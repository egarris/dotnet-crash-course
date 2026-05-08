# Module 6 — Backend Beyond the Request

Some backend work happens because someone asked for it. Most production backend systems also run work that no one specifically asked for — queue consumers, scheduled jobs, retry workers that pick up failed operations. That second category is the subject of this module.

The shape is different from the request/response pattern Module 5 covered. There is no incoming HTTP request to bound the work; no per-request DI scope to inherit; no obvious place for cancellation tokens to come from. The framework provides primitives — `IHostedService` and `BackgroundService` — that make this work first-class within the same Generic Host that runs HTTP services. The disciplines from Module 5 carry over (lifetimes, cancellation, graceful shutdown), but they apply differently: when there's no request to align with, you have to define the unit of work yourself.

This module covers what the host runs when no one is asking. The mechanics are not exotic; they reuse the hosting model from Module 5 and the DI patterns from Module 3. The point is that backend work outside the request lifecycle is its own discipline, and most of the lifetime, scope, and shutdown questions that produce bugs in HTTP code reappear in slightly different forms when the work is triggered by a message, a timer, or just "the host started."

## The host beyond HTTP

Module 0 named the Generic Host as the unified hosting abstraction across .NET service types. Module 5 covered its use for HTTP services. The same host runs *worker services* — long-running processes that don't accept HTTP requests at all — and HTTP services that include background work alongside their request handling. The two cases share their startup and shutdown sequence; they differ only in whether the host listens for HTTP and what kinds of services are registered with it.

For pure background work, the entry point is `Host.CreateApplicationBuilder` (or `WebApplication.CreateBuilder` when mixing with HTTP):

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<OrderProcessor>();
builder.Services.AddSingleton<IOrderQueue, AzureServiceBusOrderQueue>();
var host = builder.Build();
await host.RunAsync();
```

The `host.RunAsync()` call starts every registered hosted service and blocks until shutdown is requested — via a signal (`Ctrl+C`, `SIGTERM`), a fatal exception, or a programmatic call to `IHostApplicationLifetime.StopApplication()`. When shutdown begins, the host signals each hosted service's stop method with a cancellation token; each service has the host's configured shutdown timeout (default 30 seconds) to finish in-flight work cleanly.

```
Host lifecycle:

  Startup ──> IHostedService.StartAsync called for each registered service
              (BackgroundService starts ExecuteAsync on a thread-pool thread,
               which continues for the host's lifetime)
     │
     ↓
  Running ──> All hosted services running in parallel
              HTTP endpoints accept requests (if web host)
     │
     ↓
  Shutdown signal received  (Ctrl+C, SIGTERM, programmatic, or fatal exception)
     │
     ↓
  ApplicationStopping token fires
     │
     ↓
  IHostedService.StopAsync called  (BackgroundService's stoppingToken cancels)
     │   Host waits up to ShutdownTimeout for services to finish
     ↓
  Service container disposed  (scoped/singleton IDisposables disposed)
     │
     ↓
  Process exits
```

Two implications follow from this lifecycle. First, hosted services run as long as the host runs; their lifetime is the process lifetime, not a request lifetime. Anything they hold by reference lives that long too. Second, shutdown is a finite window; work that doesn't honor the shutdown signal will be interrupted mid-operation when the timeout expires.

## `IHostedService` and `BackgroundService`

`IHostedService` is the framework interface for "something the host should run." It has two methods: `StartAsync(CancellationToken)`, called once during host startup, and `StopAsync(CancellationToken)`, called once during host shutdown. The contract is shape-dependent. For *long-running* services (queue consumers, schedulers), `StartAsync` should kick off the work and return quickly so the host can keep starting other services; the long-running work runs separately. For *one-shot startup* work that the rest of the host depends on (database migrations, cache warming, configuration verification), `StartAsync` should do the work in line and return only when it's complete — the host blocks here until the method returns, which is exactly what's wanted when nothing else should start until the work has succeeded. `StopAsync` in either case should wait for in-flight work to finish, respecting the cancellation token to bound how long to wait.

The framework also runs hosted services in registration order. If a `StartupMigrator` must complete before an `OrderProcessor` begins consuming messages, `AddHostedService<StartupMigrator>()` must come before `AddHostedService<OrderProcessor>()` in `Program.cs`. Order is part of the contract.

For one-shot startup work, implementing `IHostedService` directly is the right shape:

```csharp
public class StartupMigrator(MigrationRunner runner, ILogger<StartupMigrator> logger)
    : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        logger.LogInformation("Running migrations…");
        await runner.RunAsync(ct);
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

For long-running work, the abstract `BackgroundService` class is the right base. It implements the `IHostedService` contract correctly and exposes a single `ExecuteAsync(CancellationToken stoppingToken)` method that is expected to run for the lifetime of the host:

```csharp
public class OrderProcessor(
    IOrderQueue queue,
    IServiceScopeFactory scopes,
    ILogger<OrderProcessor> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in queue.ReadAllAsync(stoppingToken))
        {
            await ProcessOneAsync(message, stoppingToken);
        }
    }

    private async Task ProcessOneAsync(QueueMessage msg, CancellationToken ct)
    {
        // see "Defining the unit of work" below — one scope per message
    }
}
```

The `stoppingToken` is the host's signal for "begin graceful shutdown." A correctly written `ExecuteAsync` propagates it through every awaitable and exits cleanly when it fires. A `BackgroundService` that ignores the stopping token is the canonical shutdown failure: the host signals, waits the shutdown timeout, then forcibly terminates the service mid-work.

The `IOrderQueue` in the example above is deliberately abstract. The work source could be an external queue (Azure Service Bus, AWS SQS, RabbitMQ), or it could be an in-process `Channel<T>` (Module 4 covered briefly) for work that doesn't need to cross process boundaries — a request handler that needs to enqueue background work for the same process can write to a `Channel<T>` that the `BackgroundService` reads from. The `BackgroundService` shape is the same regardless of where the work comes from.

A `BackgroundService` is registered as a singleton in DI by default — once via `AddHostedService<T>()`, the host owns the single instance for the process lifetime. This is the same lifetime characterization Module 3 covered, with the same constraint: every dependency the service holds by reference must have a lifetime equal to or longer than the host's lifetime, or the lifetime-mismatch anchor reappears.

## Defining the unit of work

Module 5's HTTP services have a unit of work given to them by the framework: each request gets a scope, scoped services live for that request, the scope is disposed when the response is sent. Background work has no equivalent given to it. The author has to decide what counts as a unit of work and create the scope themselves.

For a queue consumer, the natural unit of work is one message. For a scheduled job, it's one tick. For a long-running sync, it might be one batch. Whatever the choice, the discipline is the same: create a scope at the start of the unit, resolve scoped services from that scope, do the work, dispose the scope at the end.

```csharp
private async Task ProcessOneAsync(QueueMessage msg, CancellationToken ct)
{
    using var scope = scopes.CreateScope();
    var processor = scope.ServiceProvider.GetRequiredService<IMessageProcessor>();
    var db        = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    try
    {
        await processor.HandleAsync(msg, ct);
        await db.SaveChangesAsync(ct);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to process message {MessageId}", msg.Id);
        // Decide: retry, dead-letter (move to a separate queue for manual
        // inspection or later retry), or escalate. Module 16 covers retry policies.
    }
}
```

This is the legitimate use of `IServiceScopeFactory` that Module 3 named. The scope per message is a unit-of-work boundary, not a workaround for a lifetime mismatch. Each message gets its own scoped `DbContext`, its own change-tracking session, its own transaction. Failures in one message don't poison the next.

The most common mistake is to skip the scope and inject scoped services directly into the `BackgroundService` constructor:

```csharp
// Wrong: BackgroundService is a singleton; AppDbContext is scoped — captured for the
// lifetime of the host. Change tracker grows forever; transactions span messages
// they shouldn't; eventual collapse.
public class OrderProcessor(AppDbContext db, IOrderQueue queue) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in queue.ReadAllAsync(stoppingToken))
        {
            // db is reused across every message
        }
    }
}

// Right: capture only IServiceScopeFactory; create a scope per message
public class OrderProcessor(IServiceScopeFactory scopes, IOrderQueue queue)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in queue.ReadAllAsync(stoppingToken))
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            // db is fresh for each message
        }
    }
}
```

This is Module 3's lifetime-mismatch anchor in background-work form. The `ValidateScopes` setting from Module 3 catches this at startup if enabled — the captive scoped dependency fails to resolve out of the root scope and the host fails fast with a clear error. Turning it on in non-production environments is the cheapest way to find this class of bug before it ships.

## Cancellation and graceful shutdown

The host's shutdown sequence is what makes background services bounded. After a shutdown signal, the host runs through a sequence: the application-stopping cancellation token fires; `StopAsync` is called on each hosted service (or the `stoppingToken` cancels for `BackgroundService`); the host waits up to the shutdown timeout for services to finish; the service container is disposed; the process exits. The shutdown timeout is configurable on the host (`HostOptions.ShutdownTimeout`, default 30 seconds). After it elapses, services that haven't stopped get cancelled; the host then proceeds to disposal regardless. Work in flight that hasn't checked the cancellation token in time is interrupted mid-operation.

The discipline that makes shutdown clean: every `await` inside `ExecuteAsync` propagates the `stoppingToken`; long synchronous loops check `stoppingToken.IsCancellationRequested` periodically; in-flight work has a chance to commit or roll back rather than being killed mid-state.

```csharp
// Wrong: ignores the stopping token; the host has to force-terminate
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (true)
    {
        var message = await queue.ReceiveAsync();   // no token; can't be cancelled
        await ProcessAsync(message);                // no token; can't be cancelled
    }
}

// Right: propagates stoppingToken; exits cleanly on shutdown
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await foreach (var message in queue.ReadAllAsync(stoppingToken))
    {
        await ProcessAsync(message, stoppingToken);
    }
}
```

`IHostApplicationLifetime` is the abstraction for interacting with the host's lifecycle from inside the application. Its `ApplicationStarted`, `ApplicationStopping`, and `ApplicationStopped` cancellation tokens fire at the corresponding lifecycle points. Code that needs to react to shutdown — flush a buffer, drain a connection pool, send a final telemetry batch — registers with `ApplicationStopping`. Code that needs to wait for startup to complete before doing something registers with `ApplicationStarted`.

One subtlety to be aware of: an unhandled exception escaping `ExecuteAsync` does not, by default, stop the host. The exception is logged by the framework, the `BackgroundService` exits, and the rest of the host keeps running — silently, with one fewer service active. The default behavior is governed by `HostOptions.BackgroundServiceExceptionBehavior`; setting it to `StopHost` makes the host shut down on an unhandled background exception, which the orchestrator can then restart cleanly. The framework defaults to `Ignore` to preserve pre-.NET-6 behavior, not because `Ignore` is the better choice; many production teams set it to `StopHost`. Either choice should be deliberate.

## Scheduled work

Periodic work — running every minute, every hour, on a cron schedule — fits naturally into a `BackgroundService` that loops with a delay. The right primitive is `PeriodicTimer`, which gives a cancellable async wait that's better-suited to hosted services than `Timer.Change` or `Task.Delay` in a loop. `PeriodicTimer` accepts a `TimeProvider` (Module 1's testable clock abstraction); injecting one makes the schedule unit-testable without waiting wall-clock time:

```csharp
public class HourlyReporter(
    IServiceScopeFactory scopes,
    TimeProvider time) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1), time);
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            using var scope = scopes.CreateScope();
            var reporter = scope.ServiceProvider.GetRequiredService<IReportGenerator>();
            await reporter.RunAsync(stoppingToken);
        }
    }
}
```

`WaitForNextTickAsync` returns `false` when the timer is disposed or `stoppingToken` cancels, which is what makes the loop exit cleanly on shutdown. Two specific concerns recur with scheduled work.

**Overlapping invocations.** If the work takes longer than the interval, what happens? `PeriodicTimer` does not start a new tick while the previous one is running; it skips ticks the consumer didn't `await`. This is usually the right behavior. If overlapping is genuinely needed, fan-out with `Task.Run` per tick — but be careful about resource exhaustion and ordering.

**Multiple instances.** If the service runs in multiple replicas (which most cloud-deployed services do), the scheduled work will run in every replica unless something coordinates. The standard approaches are *leader election* (a coordination protocol where the replicas agree on which one runs the work), *distributed locking* (a shared resource — a database row, a Redis key — that only one replica can hold at a time), or moving the schedule out of the application entirely (Azure Functions timer triggers, Kubernetes CronJobs, or dedicated scheduling services like Hangfire or Quartz).

For anything more than per-instance scheduled work, treat the schedule as infrastructure rather than as application code — the application's job is to do the work; the platform's job is to decide who runs.

## Health checks

Health checks are the framework's mechanism for telling the platform — Kubernetes, Azure App Service, a load balancer — whether the service is ready to handle traffic. They sit alongside hosted services in the host and expose HTTP endpoints (typically `/health/live` and `/health/ready`) that the platform polls.

Two distinctions matter:

*Liveness* checks answer "is the process still alive and responsive?" A failing liveness check tells the platform to restart the process. Liveness should be cheap and local — pinging the process itself, not its dependencies.

*Readiness* checks answer "is the process ready to serve traffic?" A failing readiness check tells the platform to remove the instance from the load-balancing rotation but leave it running, so it might recover. Readiness usually verifies dependencies — the database is reachable, the message queue is connected, configuration is loaded.

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags: ["ready"])
    .AddCheck<QueueHealthCheck>("queue", tags: ["ready"]);

app.MapHealthChecks("/health/live");                  // no checks; just process responsiveness
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

Health checks are easy to misuse. A liveness check that pings the database means "if the database is down, restart the process" — which is almost never the right behavior, because restarting the process won't fix the database, and now the platform is churning processes uselessly. A readiness check that lies about being ready (returning healthy when dependencies are not actually responding) means the platform sends traffic to instances that will fail. The discipline is to match the check to the question it's answering and to be honest about both.

## A production failure that lives in background work

A pattern that recurs across teams, anonymized. A team built a queue consumer as a `BackgroundService`. The consumer was registered as a singleton (the framework's default for `BackgroundService`) and constructor-injected an `AppDbContext` and a domain service. Local testing worked; tests with a few hundred messages worked; deployed, the service ran for days at moderate load and looked stable on every metric.

Three weeks after deployment, the service fell over with an OOM (out-of-memory) error. Memory had been climbing slowly the whole time; it finally exhausted the container and the orchestrator (the platform that runs containerized services — Kubernetes, ECS, App Service, or similar) killed it. The replacement instance started clean and began the same drift.

The investigation found that the constructor-injected `DbContext` had been alive for the entire process lifetime, accumulating *tracked entities* — EF Core's in-memory record of every entity it had read or modified, used to translate `SaveChanges()` into the right SQL — for every message it processed. The change tracker had hundreds of thousands of references it would never release. Each batch of messages added more; nothing dropped them. The `DbContext`'s memory was the leak.

The fix was the pattern from earlier in this module: inject `IServiceScopeFactory` instead of the scoped services directly, create a scope per message, dispose the scope after. The diagnostic took longer than the fix because the bug shape — slow, gradual growth that doesn't correlate with any particular message or operation — looks exactly like ordinary cache growth, slow allocation pressure, or even GC inefficiency. Module 14's observability discussion covers the metrics that distinguish them.

The strongest counterargument is that EF Core could be configured to behave differently. Calling `ChangeTracker.Clear()` after each batch would empty the tracker; using `AsNoTracking()` on read queries would prevent EF from tracking returned entities at all. Either would mitigate the symptom. Neither addresses the underlying lifetime mismatch. Adding `Clear()` calls is the kind of defensive coding that hides design problems rather than solving them. The right fix is to scope the `DbContext` to the unit of work and let the scope's disposal do the cleanup. (If `ValidateScopes` had been enabled in non-production environments, the captive dependency would have been caught at host startup with a clear error message; the team had it off because, as is often the case, it had been off when they joined and no one remembered why.)

## A load-bearing point

The first place hosted-service fluency is exercised under pressure is the code review of a new `BackgroundService` or `IHostedService`. The questions are routine and load-bearing.

Does `ExecuteAsync` propagate the `stoppingToken` to every awaitable? An ignored stopping token is the canonical shutdown failure.

Does the service capture scoped state in fields, or does it create a scope per unit of work? Constructor-injected `DbContext`s, repositories, or other scoped services are the lifetime-mismatch anchor in background form.

Is the unit of work explicitly defined? "One message," "one batch," "one tick" — the author should be able to name it without hesitation.

What happens when the loop exits? `BackgroundService.ExecuteAsync` returning unexpectedly is rarely the desired outcome; if the work is supposed to keep going, the loop should be inside a wrapper that logs and recovers from non-fatal exceptions.

Are exceptions caught at the right level? An exception that escapes `ExecuteAsync` propagates to the host, which by default does not stop the host — the exception is logged but the service is dead, silently, until the process is restarted. Setting `BackgroundServiceExceptionBehavior.StopHost` in the host options changes this; the choice between letting the host crash and letting the service silently die deserves explicit reasoning.

What does graceful shutdown mean for in-flight work? Will it commit, roll back, or be left in an undefined state if the host shuts down while the work is mid-operation?

If the work is scheduled, what coordinates across replicas? If the answer is "nothing," is that intentional or an oversight?

These questions are answerable for someone who understands the host's lifecycle and the unit-of-work discipline. `BackgroundService` skeletons that ship under review — whether AI-produced or hurried human work — often look correct at the surface and fail one or more of these checks at the mechanism level, particularly the lifetime and shutdown ones.

## Closing

The shortest version of this module is that backend work outside the request lifecycle uses the same hosting model, the same DI container, and the same async machinery as HTTP work — but with the unit of work defined by the author rather than by the framework. `BackgroundService` is the long-running primitive; `IHostedService` is the one-shot primitive; the unit of work and the cancellation token are the discipline that makes both reliable.

The question worth carrying forward into the topical modules: when you read a `BackgroundService`, can you name the unit of work, identify where the scope is created and disposed, and trace the `stoppingToken` from `ExecuteAsync` to every operation that could be cancelled?
