# Module 4 — Async, Await, and the Thread Pool

Most engineers learn async/await as a way to make code faster, or as a way to use multiple threads. Neither framing is right. async/await is a way to handle many concurrent I/O operations without paying for many threads. The thread pool stays small and busy with CPU work; the I/O happens off-thread, in the kernel or in network adapters or in whatever resource the operation actually touches. The calling thread is freed during the wait, available to handle other work.

This distinction is the difference between async code that scales and async code that mysteriously hangs in production. A team that thinks async makes code faster reaches for it on CPU-bound work and finds no improvement. A team that thinks async parallelizes work assumes calling an async method launches it on another thread and writes code that deadlocks. A team that understands what async actually buys — concurrency for I/O, free of the thread cost — reaches for it correctly and avoids the failures the first two encounter.

Module 1 covered the mechanism preview: the state machine, suspension, the boxed continuation, the thread pool. This module takes async seriously as the topic it is. Most of what follows is about the production-relevant consequences of the mechanism — what async actually buys, what the thread pool can absorb, what cancellation is for, and the pitfalls that cause real outages.

## What async actually buys you

A request that waits for a database query is *I/O-bound* — the work happens at the database, and the calling thread spends most of its time waiting for a network packet to come back. A request that computes a hash of a large blob is *CPU-bound* — the calling thread does the work, and there is no network or disk to wait for.

async/await is a tool for I/O-bound work. When the calling thread reaches an `await` on an I/O operation, the runtime registers the operation with the relevant subsystem (the OS, the network adapter, the database driver) and frees the thread. The thread goes back to the pool, available to handle other work. When the I/O completes, the runtime schedules a continuation on a thread-pool thread to pick up where the awaiter left off.

For CPU-bound work, async/await offers nothing useful by itself. The calling thread has to do the work, and wrapping it in `async`/`await` does not make it run on more threads — it just adds the overhead of state-machine construction. To parallelize CPU-bound work, the right tools are `Task.Run` (to push work onto the thread pool) or `Parallel.ForEachAsync` (for many similar items). `async`/`await` and these tools compose, but they are not the same thing.

The resource trade is concrete. Without async, a thousand concurrent requests waiting on the database would require a thousand threads. Each thread costs roughly a megabyte of stack space and contributes to context-switching overhead. With async, the same thousand concurrent requests share a thread pool of perhaps a few dozen threads — because each thread is "in use" only for the small time it takes to dispatch each operation, not for the time the operation itself takes to complete.

## The thread pool: a finite shared resource

The .NET thread pool is the runtime's pool of operating-system threads available to run short tasks. The pool starts small (around the number of logical CPU cores), grows as work backs up, and tries not to grow too quickly because creating threads is expensive.

The pool sizes itself dynamically. The minimum number of threads is roughly the number of logical cores; the maximum is much higher (typically 32,767), but the pool is reluctant to grow past the minimum quickly. Growth is throttled to roughly one new thread at a time, because the pool is designed to handle short, well-behaved work items, not long-blocked ones. The runtime uses a *hill-climbing algorithm* — an iterative approach that adds or removes threads based on whether the change improves throughput, settling on a count that matches the workload — and the algorithm is conservative on purpose.

This throttling is what makes thread-pool starvation a thing. If many work items are blocked (rather than running on the CPU and finishing quickly), the pool can't replace them fast enough to keep up with new work. Requests pile up waiting for threads. CPU stays low (because the threads are blocked, not running), memory stays steady (because nothing is doing much), and the system *feels* idle while requests time out.

```
Healthy pool:                                Starved pool:

Thread pool (8 active threads)               Thread pool (8 active threads)
[T1: handling request A]                     [T1: blocked on .Result]
[T2: handling request B]                     [T2: blocked on .Result]
[T3: idle, available]                        [T3: blocked on .Result]
[T4: idle, available]                        [T4: blocked on .Result]
[T5: idle, available]                        [T5: blocked on .Result]
[T6: idle, available]                        [T6: blocked on .Result]
[T7: idle, available]                        [T7: blocked on .Result]
[T8: idle, available]                        [T8: blocked on .Result]

Queue: empty                                 Queue: 200 pending requests, growing
CPU: handling 2 active requests              CPU: low — nothing is actually running
                                             (every thread is waiting on something)
```

The healthy pool has threads available; the starved pool has every thread blocked waiting for work that itself needs a thread to complete. The runtime tries to add more threads, but the *injection rate* — the rate at which new threads get added when work is backing up — is far slower than the request arrival rate. New requests wait.

The discipline that prevents this is straightforward: don't block thread-pool threads. Async I/O doesn't block; `await` frees the thread. Synchronous I/O (file reads on a synchronous API, blocking network calls, `.Result` on a `Task`) blocks the thread until the operation completes. A thread blocked on synchronous I/O is a thread the pool cannot use for anything else.

## The mechanism, recapped

Module 1 covered the state-machine mechanism in depth. The short version: the compiler turns each async method into a struct that holds the local variables and tracks the current `await` point. When the method completes synchronously, the struct stays on the stack. When the method actually suspends at an `await`, the struct is boxed onto the heap so it can survive across the suspension. A continuation — the chunk of work that picks up after the `await` — is registered with the awaited operation. The thread that ran the method up to the `await` is returned to the pool.

Two consequences of this mechanism are load-bearing for the rest of this module.

**The continuation does not necessarily run on the same thread that started the method.** When the awaited operation completes, the runtime picks any available thread from the pool to run the continuation. Locals are restored from the heap-resident state machine; the method resumes at the next instruction. Code that depends on thread-affinity — thread-local state, thread-bound resources, anything that assumes the same thread runs the whole method — breaks across `await` boundaries. The exception is `AsyncLocal<T>`, a context mechanism that flows with the logical async chain rather than with the underlying OS thread; it is the right tool for "I want this value to be visible across awaits but not bleed into other concurrent work."

**Anything captured by the state machine that has a shorter lifetime than the continuation will misbehave when the continuation runs.** This is the recurring lifetime-mismatch failure mode in async form. An async method captures a reference (by holding it in a local variable) to an object that gets disposed before the awaited operation completes. When the continuation runs, the object is gone or in an undefined state, and the method either throws, fails subtly, or silently produces wrong results. Module 3's discussion of captive dependencies in DI is one form of this; fire-and-forget tasks that outlive their request scope are another, covered below.

## Cancellation is a contract

A `CancellationToken` is a small struct passed through async chains as a way for a caller to signal "stop." The work doesn't have to stop the instant the signal arrives; it has to stop at the next reasonable checkpoint. Cancellation in .NET is *cooperative* — the cancelled work checks the token and exits gracefully. The runtime cannot pre-empt code; the code has to participate.

When a method declares a `CancellationToken` parameter, it is making a contract: I will check this token at points where stopping is reasonable, and I will pass it to anything I await that takes one. Forgetting to propagate the token is a defect — a quiet one, because the code still works, but the work is no longer cancellable. A request the client cancelled keeps running. A long-running operation cannot be aborted. Production load grows.

```csharp
// Wrong: ct is accepted but never propagated; the work is uncancellable
public async Task<int> CountActiveOrdersAsync(CancellationToken ct)
{
    await using var conn = await _factory.CreateConnectionAsync();    // no token
    return await conn.QueryAsync<int>("SELECT COUNT(*) FROM ...");    // no token
}

// Right: every awaitable inside this method receives the token
public async Task<int> CountActiveOrdersAsync(CancellationToken ct)
{
    await using var conn = await _factory.CreateConnectionAsync(ct);
    return await conn.QueryAsync<int>("SELECT COUNT(*) FROM ...", ct);
}
```

(`await using` is the async equivalent of `using`: it calls `DisposeAsync` instead of `Dispose` and is the right pattern for resources whose cleanup itself involves async work, such as database connections.)

Two patterns are worth knowing. `ct.ThrowIfCancellationRequested()` is the lightweight check at the start of a long-running synchronous loop or before an expensive operation; it throws `OperationCanceledException` if cancellation has been requested, and most of the .NET ecosystem is set up to handle that exception cleanly. `CancellationTokenSource.CreateLinkedTokenSource(token1, token2, ...)` produces a token that fires when *any* of its sources fire — useful for combining a per-operation timeout with a request-scoped token, so the operation cancels on either trigger.

```csharp
public async Task<Order?> GetOrderAsync(OrderId id, CancellationToken ct)
{
    using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(2));
    using var linked  = CancellationTokenSource.CreateLinkedTokenSource(ct, timeout.Token);
    return await _repo.GetAsync(id, linked.Token);
    // Cancels on either: client cancels the request, OR the 2-second timeout fires.
}
```

The practice pays off twice. Cancellation tokens make work bounded — bad inputs and slow downstream services don't pin threads forever. They also make the system observable: when a request cancels and the corresponding work stops promptly, telemetry stays clean; when work keeps running after cancellation, log noise and resource use grow without anyone seeing why.

## Sync-over-async and the deadlock story

`.Result`, `.Wait()`, and `.GetAwaiter().GetResult()` all do the same thing: block the calling thread until the `Task` completes. They are how synchronous code calls into async code. They are also how code deadlocks under certain conditions, and how thread pools get starved under load.

The deadlock part is now mostly historical, but it shows up often enough in legacy code to be recognizable. In legacy ASP.NET (the .NET Framework era), Windows Forms, and WPF, async continuations by default are scheduled back to a *synchronization context* — an abstraction representing where to resume async work. In a UI app, the context is the UI thread; continuations resume on the UI thread so they can update UI elements safely. In legacy ASP.NET, the context was the request thread; continuations resumed on the same thread.

The deadlock shape:

```
Thread T1 holds the synchronization context
            │
            │ calls .Result on a Task
            ↓
T1 blocks, waiting for Task to complete
            │
            ↓
Task's continuation is scheduled to run on the sync context (i.e., on T1)
            │
            ↓
T1 is blocked, so the continuation cannot run
            │
            ↓
Task cannot complete
            │
            ↓
.Result never returns ─── deadlock
```

`ConfigureAwait(false)` was the workaround. It tells the awaiter "don't bother resuming on the original context — any thread-pool thread will do." Library code that didn't need to resume on a particular context — which is most library code — used `ConfigureAwait(false)` defensively to avoid the deadlock.

The modern reality is simpler. ASP.NET Core has no synchronization context. Continuations run on whatever thread the runtime picks. The deadlock pattern from legacy ASP.NET cannot occur for the same reason: there is no context to be blocked. `ConfigureAwait(false)` in ASP.NET Core application code is a no-op. The rule that replaces all of this is *async all the way up*: from the entry point of a request down to the lowest awaitable, every method in the chain should be async, and every awaitable should be `await`-ed. No `.Result`, no `.Wait()`, no `.GetAwaiter().GetResult()` anywhere along the path.

```csharp
// Wrong: blocking on Task.Result. On a legacy synchronization context this
// can deadlock; on the modern thread pool it ties up a thread for the whole
// I/O wait, contributing to starvation under load.
public IActionResult GetUser(int id)
{
    var user = _service.GetUserAsync(id).Result;
    return Ok(user);
}

// Right: async all the way up
public async Task<IActionResult> GetUser(int id, CancellationToken ct)
{
    var user = await _service.GetUserAsync(id, ct);
    return Ok(user);
}
```

`ConfigureAwait(false)` still has a place in libraries that ship across application models — a NuGet package can't assume the absence of a synchronization context, so library authors continue to use it as a defensive habit. Application code in ASP.NET Core, minimal APIs, or worker services doesn't need it. If a codebase is sprinkled with `ConfigureAwait(false)` in application logic, that's usually muscle memory from an older era, not a meaningful improvement.

## Fire-and-forget and lifetime mismatch

A `Task` that is created but not awaited is *fire-and-forget*. The operation runs to completion (or failure) on its own; nothing waits for the result. Fire-and-forget is sometimes the intent — a metric emit that doesn't need to block the request, an audit log that's eventual-consistency. It is also the most common source of async lifetime bugs.

Two specific failures recur.

**Exceptions thrown by an unawaited `Task` are unobserved.** The `Task` captures the exception, but no one ever calls `.Wait()` or `await` to surface it. The application keeps running. The runtime emits a `TaskScheduler.UnobservedTaskException` event eventually, but only if the GC happens to finalize the `Task` and only if a handler is registered. Production debugging surfaces the symptom (something went wrong, data is missing) without the cause (the exception that explains why).

**The `Task` may outlive the scope that created it.** A controller method that fires off `_logger.LogAuditAsync(...)` without awaiting has launched work that may run after the request scope ends. The DI scope is disposed; the `DbContext` the work depends on is disposed; the work tries to use the disposed dependency and fails. This is Module 3's lifetime-mismatch anchor in async form: a captured reference outliving the lifetime its registration claimed.

```csharp
// Wrong: fire-and-forget, no exception handling, captures scoped state
public async Task<IActionResult> SubmitOrder(OrderRequest req, CancellationToken ct)
{
    var order = await _orders.CreateAsync(req, ct);
    _ = _audit.LogAuditAsync(order);   // discard pattern signals "I know this is unawaited"
    return Ok(order);                  // but the scoped DbContext inside _audit may be
                                       // disposed before LogAuditAsync's continuation runs
}

// Right: enqueue the work, let a hosted service own the lifetime
public async Task<IActionResult> SubmitOrder(OrderRequest req, CancellationToken ct)
{
    var order = await _orders.CreateAsync(req, ct);
    await _auditQueue.EnqueueAsync(new AuditMessage(order.Id), ct);
    return Ok(order);
}

// AuditWriter is a BackgroundService that reads the queue in its own scope.
// Module 6 covers IHostedService and BackgroundService in depth.
public class AuditWriter(IServiceScopeFactory scopes, IAuditQueue queue)
    : BackgroundService { /* reads queue, processes in its own scope */ }
```

The `_ = ...` discard syntax is sometimes used to mark "I deliberately don't await this." It silences compiler warnings about unawaited Tasks. It does not address the underlying problems — exceptions are still unobserved, the scope still ends. For work that matters, the right pattern is a hosted service that owns its own lifetime. Module 6 covers the hosting model.

`async void` is a particularly dangerous form of fire-and-forget. Methods declared `async void` don't return a `Task`, so callers cannot await them, cannot observe their completion, and cannot catch their exceptions through the normal mechanism. An exception thrown from an `async void` method is rethrown on the captured `SynchronizationContext` — or, if there is none (as in ASP.NET Core), on a thread-pool thread, where an unhandled exception terminates the process by default. The only routinely legitimate use is event handlers, which the framework expects to be `void`-returning. Anything else should be `async Task` so callers can do their job.

## Beyond await: streaming and parallelism

Three patterns extend `async`/`await` for shapes the basic keyword doesn't directly support.

**`IAsyncEnumerable<T>`** is the async equivalent of `IEnumerable<T>` for sequences that arrive over time — paginated database results, server-sent events, message-stream consumption. Each item is awaited individually; the consumer can break out of the loop early without buffering everything in memory. The `[EnumeratorCancellation]` attribute on a `CancellationToken` parameter is what bridges the consumer's cancellation token to the iterator's awaits; without it, the token the consumer passes via `await foreach` won't reach the awaits inside the iterator method.

```csharp
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct)
{
    string? continuation = null;
    do
    {
        var page = await _repo.GetPageAsync(continuation, ct);
        foreach (var order in page.Items)
            yield return order;
        continuation = page.NextToken;
    } while (continuation is not null);
}

// Caller:
await foreach (var order in service.GetOrdersAsync(ct))
{
    if (order.Status == "Cancelled") break;   // early exit; no further pages fetched
    // process order
}
```

**`Task.Run`** pushes a delegate onto the thread pool and returns a `Task` that completes when the delegate finishes. Its purpose is to move CPU-bound work off the calling thread. The classic case: an HTTP handler that needs to compute a hash of a large blob. Doing the work synchronously blocks the request thread; doing it inside `Task.Run` shifts it to a pool thread, freeing the request thread for other work.

`Task.Run` is for CPU-bound work, not for "wrapping" async I/O.

```csharp
// Wrong: Task.Run wrapping an async method adds overhead with no benefit.
// The async method already frees the thread on its own awaits.
public async Task<int> GetCountAsync(CancellationToken ct)
{
    return await Task.Run(() => _service.GetCountAsync(ct));
}

// Right: just await the async method directly
public async Task<int> GetCountAsync(CancellationToken ct)
{
    return await _service.GetCountAsync(ct);
}

// Right: Task.Run for CPU-bound work that would otherwise block the caller
public async Task<string> ComputeHashAsync(byte[] payload, CancellationToken ct)
{
    return await Task.Run(() => SHA256.HashData(payload).ToHexString(), ct);
}
```

**`Channel<T>`** is .NET's bounded or unbounded producer/consumer queue, designed for async access. A producer writes items; one or more consumers read them asynchronously. Channels are the right tool when work needs to be queued between async stages — for example, streaming a large response while a background producer fetches the next pages — without spinning up explicit thread synchronization.

**`Parallel.ForEachAsync`** (added in .NET 6) runs an async operation over many items with controlled parallelism. A pattern like "fetch 100 records concurrently, with at most 10 in flight at once" becomes one method call.

**`Task.WhenAll` and `Task.WhenAny`** are the two most common parallel-async patterns. `Task.WhenAll` returns a `Task` that completes when all of its input tasks complete; if any of them throw, the resulting `Task` aggregates the exceptions. `Task.WhenAny` returns the first task to complete, leaving the rest still running — useful when the first answer is the one you want and the others can be cancelled or ignored. Both compose with `await`:

```csharp
// Fan out three independent calls, await all of them
var (user, prefs, recent) = await (
    _users.GetAsync(id, ct),
    _prefs.GetAsync(id, ct),
    _orders.GetRecentAsync(id, ct)
).WhenAll();   // or: await Task.WhenAll(t1, t2, t3) and unpack the results

// Race two sources, take whichever responds first
var first = await Task.WhenAny(
    _primary.QueryAsync(ct),
    _replica.QueryAsync(ct));
```

## A production failure that lives in the thread pool

The pattern below has shipped to production in more than one codebase. A service handled requests at 50 RPS (requests per second) in load testing without issue. Deployed to production at 200 RPS, it began timing out — not all requests, but a growing percentage as load increased. The metrics dashboard showed CPU at 30% and memory steady. Database performance was unchanged. Network latency was normal. The bottleneck was somewhere else.

Traces showed many requests waiting for what looked like database queries, with the queries themselves running normally. The team eventually noticed a pattern: a helper method deep in the request path called a synchronous-looking API that, internally, used `.Result` to bridge to an async library. Each call blocked a thread-pool thread until the library's task completed.

At 50 RPS, the thread pool had enough headroom — even with each request blocking a thread, the pool grew fast enough to keep up. At 200 RPS, the pool was perpetually full of blocked threads, growing as fast as the runtime would allow but never catching up. New requests waited for threads that wouldn't come free. The metrics looked idle (CPU low, memory steady) because nothing was running; everything was waiting.

The fix was small: replace `.Result` with `await`, propagate the cancellation token through the call stack, and let the request path be async all the way up. The diagnostic took a week because the symptom didn't match what the team was monitoring — CPU was low, no exceptions thrown, no obvious queue — and load tests didn't reach the threshold that triggered starvation.

The strongest counterargument is that load testing should have caught this before deployment. That's fair as far as it goes. Load tests are usually shaped by what the team knows to test for, and thread-pool starvation is not visible in CPU, memory, or queue-depth metrics — which is what load tests typically watch. The behavior is invisible until requests start timing out, which makes it especially hard to detect proactively. Module 14's observability discussion picks up the metric — *thread-pool injection rate* combined with *thread-pool work-item count* — that surfaces this class of failure when CPU does not.

## A load-bearing point

The first place async fluency is exercised under pressure is the code review of an async method or an async-bound code path. The questions are routine, and they're load-bearing.

Is every awaitable awaited? An unawaited `Task` is either fire-and-forget — and should be explicitly so, with exception handling and a defensible lifetime story — or a bug.

Is the cancellation token propagated? A method that accepts a `CancellationToken` should pass it to every awaiter that takes one. Anything else is a silent contract violation.

Are there any `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` calls in async paths? Each one is a thread blocked when it could be freed.

Is `async void` used, and is the method actually an event handler? If not, change it to `async Task`.

Where do tasks from fire-and-forget operations live? If they capture scoped state, they're a bug waiting to surface in production.

Is `Task.Run` used to wrap async calls? That's overhead with no benefit, and almost always wrong.

Each question is answerable for someone who understands what async actually does at the thread-pool level. None is answerable at the surface. A method that compiles and "looks async" can fail any of these checks while looking correct to a less-fluent reviewer.

## Closing

The shortest version of this module is that async/await is a tool for I/O concurrency, not parallelism. The thread pool is finite. Cancellation tokens are a contract. Sync-over-async deadlocks legacy contexts and starves the modern thread pool. Fire-and-forget tasks must own their lifetime explicitly or they become the next lifetime-mismatch failure.

The question worth carrying forward into the rest of the foundation arc: when you read an async method, can you trace the cancellation token through every `await`, name where the calling thread is freed and where the continuation resumes, and identify which captures might outlive the scope they were created in?
