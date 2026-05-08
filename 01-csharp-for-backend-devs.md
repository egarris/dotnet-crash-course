# Module 1 — C# for Backend Devs

There is a layer of programming that runs on computational feedback — the compiler accepts the syntax, the linter is quiet, the test passes. There is another layer where computational feedback isn't enough. Whether the choices that look correct are the right choices for this code, in this system, on this team is a question someone has to be on the hook for. Answering it requires reading at the level of mechanism rather than syntax: knowing what each language feature does at runtime, what each abstraction commits the system to, and where each guarantee starts and stops.

Module 2 framed language features as the toolkit for expressing design boundaries. This module is the mechanism that toolkit rests on. Knowing what dispatch happens at runtime when a caller invokes an interface method is what lets you judge whether the indirection is paying for itself; knowing what equality the compiler generates for a record is what lets you trust that two records compared in production produce the answer you expected. The reader is assumed to be able to write C# or to ask an AI to write it. The module is about the layer underneath.

> **Foundational vocabulary**
>
> A few terms recur across the sections below. The *runtime* is the program that loads and executes compiled code (Module 0 covers it). It manages two memory regions referred to constantly here. The *stack* holds the parameters and local variables of the currently-executing chain of method calls; space on the stack is reclaimed automatically when each method returns. The *heap* is a longer-lived region holding objects that need to outlive the method that created them; objects on the heap are reclaimed by the *garbage collector* (GC) when nothing references them anymore. *Allocation* is the act of reserving memory for an object, usually on the heap. *Reference types* — classes, interfaces, arrays, delegates — live on the heap; a variable of a reference type holds a *reference*, a small piece of data identifying where the object lives. *Value types* — the built-in numeric types, `bool`, `enum`, `struct`, `record struct` — hold their data directly, usually on the stack or as fields of a containing object. *Dispatch* is the runtime's process of resolving a method call to the code that actually runs; some calls dispatch directly to a known implementation, while others (virtual methods, interface methods) require the runtime to look up the implementation based on the object's actual type at the call site.
>
> A diagram for the stack-and-heap split:
>
> ```
>        STACK                              HEAP
>  (per active method)             (longer-lived objects)
>
>  ┌────────────────────┐         ┌────────────────────┐
>  │ Main()             │         │ Customer           │
>  │   int count = 5    │         │   Name = "Alice"   │
>  │   Customer c ──────┼───┐ ┌──>│   Age  = 42        │
>  ├────────────────────┤   │ │   └────────────────────┘
>  │ Process()          │   │ │   ┌────────────────────┐
>  │   int total = 100  │   └─┼──>│ List<Order>        │
>  │   List<Order> os ──┼─────┘   │   [Order, Order]   │
>  └────────────────────┘         └────────────────────┘
> ```
>
> Local `int` variables (`count`, `total`) sit on the stack with their value as bytes. Reference-typed locals (`c`, `os`) sit on the stack too, but they only hold a small piece of data identifying where the object lives; the object itself is on the heap. When `Process()` returns, its stack frame is reclaimed; the heap objects stay alive until the GC notices nothing still points at them.

Each section below takes one feature or feature cluster and treats it the way a senior engineer would read it in a code review: what is the feature, what does it do at the mechanism level, what does it let the design express, and where does it commonly go wrong.

## Classes, constructors, and how objects come into existence

A class is a reference type — a region of heap memory holding fields, with methods that operate on them, and an identity distinct from its contents. The `new` keyword starts a sequence with several steps. The runtime allocates memory and zero-initializes every field (reference fields become `null`, numeric fields become `0`, `bool` fields become `false`, and so on). Then *derived-class* field initializers (the values assigned at field declaration, like `private int _count = 0`) run in declaration order. Then the *base class* constructor runs (which itself does base-class field initializers and constructor body, recursively, all the way up to `object`, the type all classes ultimately derive from). Then the derived-class constructor body runs. By the time the outermost constructor returns, the reference handed back to the caller points to an object that should be in a valid state. *Valid* means the constructor has established every invariant the class's methods will assume.

That ordering produces one of C#'s subtler hazards: calls to *virtual methods* from inside base-class constructors. A virtual method is one whose actual implementation is selected based on the runtime type of the receiver — a method declared on a base class can be replaced in a derived class, and a call to that method on `this` resolves to the derived class's version even when the call is written in base-class code. If a base-class constructor makes such a call and a derived class has overridden the method, the override runs at a moment when the derived-class field initializers have run but the derived-class constructor body has not. The object the override dispatches to is partially constructed in a way the override almost certainly isn't expecting:

```csharp
class Base
{
    public Base()
    {
        DoSetup();   // resolves to Derived.DoSetup at runtime
    }

    protected virtual void DoSetup() { }
}

class Derived : Base
{
    private string _label;          // initialized to null by zero-init

    public Derived(string label)
    {
        _label = label;             // runs *after* Base() returns
    }

    protected override void DoSetup()
    {
        Console.WriteLine(_label.ToUpper());  // NullReferenceException
    }
}
```

`_label` is null when `DoSetup` runs because the derived constructor body — the line that assigns it — hasn't executed yet. The fix is usually to avoid virtual calls in constructors, or to mark methods sealed when they aren't meant to be overridden.

Modern C# adds two pieces of vocabulary worth knowing. *Required members* (the `required` modifier on a property) tell the compiler that callers must initialize the property at construction; the type cannot be instantiated without it. *Primary constructors* (added in C# 12) let constructor parameters appear in the class header, in scope across the type body. Both are language-level ways to make construction-time invariants more visible — turning conventions into things the compiler enforces.

The construction sequence matters in production because the invariants established by the constructor are what the rest of the codebase depends on. Code that lets a half-constructed object escape the constructor — by handing `this` to another method, by storing it in a static collection during construction, or by registering callbacks that fire before the constructor finishes — produces a class of bugs that are hard to reproduce and harder to diagnose. The discipline is to treat the constructor as the place where the object becomes itself, and to ensure nothing outside the constructor sees the object until that transition is complete.

## Methods and how data flows

C# passes parameters by value. For a value type, that means the bytes are copied; the method has its own instance, and changes inside the method do not affect the caller's copy. For a reference type, the reference is copied; the method gets a new variable holding the same address, and operations on the referenced object are visible to the caller:

```csharp
class Box   { public int Value; }
struct Point { public int X; }

void Mutate(Box b, Point p)
{
    b.Value = 99;   // visible to caller — b holds the same reference
    p.X = 99;       // local copy only — invisible to caller
}

var box = new Box   { Value = 0 };
var pt  = new Point { X = 0 };
Mutate(box, pt);
// box.Value is now 99
// pt.X is still 0
```

What's happening at the parameter boundary, in the stack-and-heap picture from earlier:

```
After Mutate(box, pt) is called:

       caller's stack frame                         Mutate's stack frame
  ┌────────────────────────┐                  ┌────────────────────────┐
  │ box ───────────────┐   │                  │ b ───────────────┐     │
  │ pt = { X = 0 }     │   │                  │ p = { X = 0 }    │     │
  └────────────────────┼───┘                  └──────────────────┼─────┘
                       │                                         │
                       │                                         │
                       └─────────────┬───────────────────────────┘
                                     ↓
                                 ┌─────────────┐  on the heap
                                 │ Box         │
                                 │  Value = 0  │
                                 └─────────────┘
```

`box` and `b` are two stack variables holding the same reference, both pointing at the one `Box` on the heap; mutations through `b` are mutations to that single object. `pt` and `p` are two independent stack-resident `Point` structs; `p.X = 99` modifies `Mutate`'s copy and leaves `pt` alone. The distinction explains the perennial confusion that "C# passes objects by reference." What is passed is the reference, by value.

The `ref`, `out`, and `in` modifiers change this. `ref` passes a reference to the caller's variable itself, allowing the method to reassign what the caller's variable points to. `out` is `ref` with a contract: the method must assign the variable before returning. `in` is `ref` with a different contract: the method may not write through the reference, but avoids the copy cost for large value types. None of these are common in everyday backend code, but recognizing them is part of fluent reading.

Returns are by value too. A method that returns a `List<T>` hands back a reference, copied to the caller; the caller and the method then share the same list. A method that returns a `decimal` returns the bytes of the decimal. Tuple returns (`(int Count, string Name) GetThing()`) and deconstruction (`var (count, name) = GetThing()`) are syntactic sugar over a small struct created at the call site and decomposed inline.

Modern C# code uses *extension methods* extensively. An extension method is a static method declared in a static class with `this` prefixing its first parameter, which the compiler treats at the call site as if it were an instance method on the parameter's type. Every LINQ operator is an extension method, as are most fluent configuration APIs in `Microsoft.Extensions.*`. Dispatch is by the static type of the receiver at compile time, not by runtime type; an extension method called on an `IEnumerable<T>` reference dispatches to the extension method even if the underlying object happens to define a method with the same name. The implication for design is that extension methods don't participate in polymorphism (the property of a single call site dispatching to different implementations depending on the receiver's actual runtime type) — they're a way to add operations to a type without modifying it, but they aren't the same as adding methods to the type itself, and a derived class cannot override them.

## Types and equality

Reference types live on the heap; value types live where they're declared, which is usually the stack or inside a containing object. The "stack vs heap" framing is a useful first approximation, but the runtime can elide allocations in some specific cases (synchronous async completion, certain small-struct optimizations, recent work on object stack allocation), and value types end up on the heap whenever they're boxed (treated as `object` or as a non-generic interface) or stored as fields of reference types. The distinction that matters for design is identity: a reference type's identity is its address, and a value type's identity is its contents.

Equality follows from identity. Two reference-type variables are equal by default if they refer to the same object. Two value-type variables are equal if their fields are equal. The default `Equals` and `GetHashCode` implementations encode this, but the defaults are notoriously easy to get wrong — particularly for structs with reference-type fields, where the default hash code computation can be slow and ill-distributed. Records, introduced in C# 9, generate value equality automatically over their declared members, and they are the right starting point for any type whose identity is its data rather than its address.

*Boxing* is what happens when a value-type instance is assigned to an `object`-typed slot, an interface variable, or a collection that takes `object`: the runtime allocates a heap object containing the value's bytes and hands back a reference to it. Boxing is invisible in source — there is no keyword for it — and it costs an allocation and a level of indirection:

```csharp
int x = 42;
object boxed = x;        // allocation: x's bytes copied to a heap object
int y = (int)boxed;      // unboxing: copies them back out

var list = new List<object>();
list.Add(42);            // 42 is silently boxed before being added
```

In hot paths, boxing in unintended places is a common cause of allocation pressure that profiles identify and source readers miss.

Three modern struct modifiers are now central to performance-conscious C# code. `readonly struct` declares that no member of the struct mutates its fields; the compiler enforces this and avoids defensive copies the runtime would otherwise make when calling members through `in` parameters or readonly references. `record struct` is a struct with auto-generated value equality across its declared members — the value-type counterpart to a record class. A declaration like `public record struct Point(int X, int Y);` produces a struct with `X` and `Y` properties, an equality implementation that compares both, and a constructor that takes both. `ref struct` is a struct that may not live on the heap at all; it cannot be a field of a reference type, captured by a lambda (an inline function expression like `x => x + 1`), or boxed. `Span<T>` is the canonical `ref struct` and shows up in any modern code that handles parsing, byte manipulation, or memory-conscious work — a typical use looks like `ReadOnlySpan<char> slice = source.AsSpan(1, 3);`, which gives the method a window into the source without copying it. The `ref struct` category exists because some types only make sense when their lifetime is bounded to a single stack frame; the language enforces that bound at compile time.

Mutable value types are an established C# anti-pattern. A struct with a mutable field or property, passed by value, gets copied at the call site, and any "mutation" through the copy leaves the original untouched:

```csharp
struct Counter { public int Value; }

void Increment(Counter c) => c.Value++;   // mutates the local copy

var counter = new Counter { Value = 0 };
Increment(counter);
Console.WriteLine(counter.Value);  // 0, not 1
```

The bug is invisible at the source level — the syntax looks like it should mutate — and produces a particularly confusing class of diagnostics. Modern style strongly favors immutable value types — `readonly struct`, init-only properties, `with` expressions for derived copies — for exactly this reason.

## Interfaces and inheritance

An interface declares method signatures and properties. Any type implementing the interface promises to provide those members; callers depending on the interface depend on the contract, not on any particular implementation:

```csharp
public interface IClock
{
    DateTime UtcNow { get; }
}

public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

public class FakeClock : IClock
{
    public DateTime UtcNow { get; set; }
}

// TokenIssuer depends on IClock, not on which clock is plugged in.
public class TokenIssuer(IClock clock)
{
    public Token Issue() => new Token(issuedAt: clock.UtcNow);
}
```

At runtime, calling an interface method goes through interface dispatch — a small indirection that selects the implementation based on the runtime type of the receiver. The cost is small in most code; it becomes measurable in tight loops or hot paths.

Inheritance is the other axis. A class can derive from one base class (single inheritance) and any number of interfaces. Methods can be marked `virtual` to allow overriding, `override` to override a virtual method, `sealed` to close further extension, or `new` to introduce a member that hides one from the base class. The runtime resolves virtual calls based on the actual type of the object — a base-class reference holding a derived-class instance dispatches to the derived class's override. The implication for design was named in Module 2: inheritance bundles shared behavior with no opt-out, which is why composition is often the better choice when the sharing is incidental rather than essential.

*Default interface methods*, added in C# 8, let an interface supply a default implementation for a member — callable through an interface reference even if the implementing class doesn't provide one. The feature blurs the line between interfaces and abstract classes, and is occasionally useful for evolving an interface without breaking existing implementations. It should be reached for sparingly; an interface with significant default behavior is closer to an abstract class than to a contract.

Two more interface features matter for modern code. *Explicit interface implementation* lets a class provide a member only when accessed through the interface — useful when a class implements two interfaces with conflicting members, or when the class wants the interface member out of its public class-level surface. The implementation is invoked only when the caller has an interface-typed reference, not a class-typed one. *Static abstract members* (C# 11) let interfaces declare that implementing types must provide static members with particular signatures. This is what makes generic math possible — `IAdditionOperators<T, T, T>` requires the implementing type to provide a static `+` operator — and it's the mechanism behind the `INumber<T>` family of interfaces that lets generic code call type-level operations without runtime dispatch.

## Generics

Generics let a type or method be parameterized by another type. A `List<T>` doesn't know what `T` is at the source level, but at runtime each closed generic type — `List<int>`, `List<string>`, `List<Customer>` — is a distinct, fully realized type with its own metadata. This is *reification*: the runtime knows the concrete type parameters of every instantiation, a meaningful difference from Java and TypeScript, both of which erase type parameters at runtime. Reification is what makes `typeof(T)` work inside a generic method and what lets the runtime treat each instantiation as its own type.

The JIT-compiled code is treated differently for reference and value types. Reference-type instantiations share machine code: `List<string>`, `List<Customer>`, and any other reference-type `List<T>` use the same JIT-compiled methods, with type information passed at runtime. Value-type instantiations get specialized: `List<int>` uses code that operates directly on `int`s without boxing, distinct from the code shared by reference-type lists. This split is why generic abstractions over reference types are essentially free in code-size terms, while heavy generic specialization over value types can produce real code-size pressure in tight cases — every distinct value-type closed generic compiles to its own specialized methods.

Generic constraints (`where T : class`, `where T : struct`, `where T : new()`, `where T : SomeInterface`, and so on) tighten the contract a generic exposes. Without constraints, the only operations available on a `T` are the ones defined on `object`. With constraints, the compiler lets the body call methods declared on the constraint, instantiate `T` with a parameterless constructor, or branch on whether `T` is a value or reference type:

```csharp
public T OpenAndReturn<T>() where T : IDisposable, new()
{
    var t = new T();   // allowed because of new()
    // ... use t.Dispose() if needed; allowed because T : IDisposable
    return t;
}
```

Constraints are the language's way of making a generic abstraction precise about what it actually requires from the types it accepts.

Variance, named in Module 2, is the modifier on generic interface and delegate type parameters that lets the compiler treat related closed generic types as substitutable in one direction or the other. `out T` (covariant) means the type only appears in output positions; `in T` (contravariant) means the type only appears in input positions. The reason variance is restricted to interfaces and delegates is mechanism: classes can have fields holding `T` in either direction, and unrestricted variance over fields would break type safety.

## Nullability and pattern matching

Nullable reference types (NRT), enabled by default in modern .NET projects, are a compile-time feature. The compiler tracks whether each reference can be null, warns when potentially-null values are dereferenced without a check, and lets you annotate types with `?` to declare them nullable. The runtime does not enforce any of this — `null` is still the same `null` it always was. The gap between compile-time tracking and runtime enforcement is where NRT bugs live: code that compiles cleanly under NRT can still throw `NullReferenceException` if a null arrives through a path the compiler couldn't analyze (a generic, an unmarked third-party API, a deserializer that bypasses the constructor).

Nullable value types are different. `int?` is `Nullable<int>`, a struct with two fields — a `bool` indicating whether the value is present and the underlying value. The runtime treats this as an ordinary struct; there is no compile-time-only indirection. The `?.` and `??` operators work on both, but the underlying mechanism diverges: for reference types they are runtime null checks; for value types they are operations on `Nullable<T>`.

Pattern matching is the language feature most useful for handling these distinctions cleanly. Switch expressions, type patterns, property patterns, and list patterns let the compiler verify exhaustiveness across closed sets and surface gaps the developer didn't think about:

```csharp
public abstract record Shape;
public sealed record Circle(double Radius) : Shape;
public sealed record Square(double Side)   : Shape;

string Describe(Shape? shape) => shape switch
{
    Circle { Radius: > 10 } c => $"big circle ({c.Radius})",
    Circle c                  => $"small circle ({c.Radius})",
    Square { Side: var s }    => $"square ({s})",
    null                      => "nothing",
    _                         => "unknown shape"
};
```

In practice, pattern matching is one of the more useful tools the compiler provides for catching the case the engineer forgot — particularly when combined with sealed hierarchies the compiler can prove enumerable.

## Async and the state machine

`async` and `await` look like syntactic sugar over callbacks. They are something more interesting: the compiler rewrites an async method into a *state machine* — conceptually, a small object that remembers where the method paused, what its local variables were at that moment, and how to resume when the awaited operation finishes. Concretely, it is a compiler-generated struct (typically named something like `<MethodName>d__0`, with the angle brackets ensuring it can't collide with names a developer would write) that holds the locals as fields and tracks the method's current pause point as it moves through its `await` points. A method like:

```csharp
public async Task<int> GetCountAsync(CancellationToken ct)
{
    var data = await _api.FetchAsync(ct);   // suspension point
    return data.Count;
}
```

is rewritten by the compiler into a struct with one field per local (`data`, the awaiter for `_api.FetchAsync`, the eventual result), a `MoveNext` method that advances through labeled states, and a `Task<int>` returned to the caller as the handle to the eventual result. The method itself returns a `Task` (or `Task<T>`, or `ValueTask<T>`) — a handle to the eventual completion that callers can await in turn. If the method completes synchronously without ever suspending, the state machine stays on the stack — no heap allocation. If the method actually suspends at an `await`, the struct is boxed onto the heap at the suspension point so it can survive while the awaited operation runs.

When `await` is reached and the awaited operation hasn't completed, the state machine registers a *continuation* — the chunk of work that needs to run after the awaited operation completes — and returns. The thread that was running the method is freed. When the operation completes, the runtime schedules the continuation to run on a thread from the *thread pool*, a managed pool of operating-system threads the runtime reuses across many short tasks rather than starting a fresh thread per task (by default; the synchronization context can change this in some application models). This is what makes async I/O cheap on .NET: a process can have thousands of in-flight async operations without paying for thousands of threads.

Visually, the transition at `await` looks like this:

```
Before await:                       At await (operation in flight):

      STACK                                    STACK             HEAP
  ┌────────────────────┐                ┌──────────────────┐  ┌──────────────────┐
  │ GetCountAsync()    │                │ caller           │  │ state machine    │
  │   state machine    │                │   has the Task,  │  │   data: ?        │
  │     data           │       ───>     │   continues with │  │   awaiter        │
  │     awaiter        │                │   other work or  │  │   state = 1      │
  │     state = 0      │                │   awaits it      │  └──────────────────┘
  └────────────────────┘                └──────────────────┘         ↑
                                                                     │
                                            continuation registered with the awaiter;
                                            runs on a thread-pool thread when the operation completes
```

Before the await, the state machine struct lives on the stack inside `GetCountAsync()`'s frame. At the await, the struct gets boxed onto the heap; the calling thread is freed; the awaited operation runs on its own; when it finishes, the runtime picks a thread-pool thread, restores the locals from the heap-resident state machine, and resumes the method from where it paused.

The cost shows up in two places. Allocations: each async method that actually suspends incurs a heap allocation for the boxed state machine, plus the `Task` object for callers to await. Synchronous completion avoids the boxing, and `Task` returns can be optimized via cached completed tasks; `ValueTask` returns allow further elision when the awaited operation usually completes synchronously. Lifetime: an async operation that outlives its calling scope — fire-and-forget work started without an `await`, or work that captures references whose backing objects get disposed before the continuation runs — produces the recurring lifetime-mismatch failure mode this course names. Module 4 covers async in depth; this section is the mechanism preview.

> **Two more terms before Module 4**
>
> `async void` should usually be avoided. Its return type doesn't carry a `Task` for callers to await, which means exceptions thrown from it can crash the process and there is no way to wait for it to finish. The only routinely legitimate use is event handlers.
>
> `ConfigureAwait(false)` controls whether the continuation resumes on the original synchronization context — relevant in some application models, irrelevant in ASP.NET Core where there is no synchronization context. Library code that doesn't need to return to a particular context typically uses it.
>
> The deeper async pitfalls — sync-over-async deadlocks, missing cancellation propagation, the difference between `Task.Run` and a bare async call — live in Module 4.

## LINQ and deferred execution

LINQ (Language-Integrated Query) is two related things wearing the same syntax. The first is `IEnumerable<T>`-based LINQ-to-objects, where queries operate on in-memory sequences and the operators are extension methods on `IEnumerable<T>`. The second is `IQueryable<T>`-based LINQ-to-providers, where queries are constructed as *expression trees* — runtime data structures that represent the code of the query itself, which a library can walk and translate into another form. A provider, most commonly Entity Framework Core, translates these trees into another query language at execution time, typically SQL.

The behavior that matters most is *deferred execution*. A LINQ query is not run when the query is written. It is run when the result is enumerated — by `foreach`, by `.ToList()`, by `.First()`, or any other materialization operator. Until then, the query is a description of operations to perform:

```csharp
var query = items.Where(x => ExpensiveCheck(x));

foreach (var item in query) { /* ... */ }   // runs ExpensiveCheck on every item
foreach (var item in query) { /* ... */ }   // runs it again, on every item
```

Re-enumerating a query runs it from the source again; iterating an `IQueryable` query twice issues two SQL queries, not one. Capturing a variable in a query and changing the variable before enumeration uses the changed value. Adding `.ToList()` materializes the results once and turns subsequent operations into in-memory work, which is sometimes the right call and sometimes premature.

The boundary between `IEnumerable<T>` and `IQueryable<T>` matters most in EF Core; Module 9 covers it in depth. The short version is that operators called on an `IQueryable` are translated to SQL; once a sequence becomes an `IEnumerable` (often by an explicit `.AsEnumerable()` or by an operator the provider can't translate), the rest of the query runs in process:

```csharp
// IQueryable: composes into a single SQL WHERE clause
var users = db.Users
    .Where(u => u.Active)
    .Where(u => u.CreatedAt > cutoff)
    .ToList();

// IQueryable → IEnumerable: provider can't translate the predicate,
// so execution moves in-process at .AsEnumerable()
var users = db.Users
    .AsEnumerable()
    .Where(u => MyHelper(u.LocalTime))   // runs in C#, after pulling all rows
    .ToList();
```

The performance cliff between the two is steep, and the source code at the call site rarely makes the transition visible.

## Exceptions and disposal

Throwing an exception walks the call stack looking for a `catch` block whose declared type matches, running each `finally` along the way. Exception filters (`catch (...) when (predicate)`) let the catch be conditional without unwinding the stack — useful when the matching predicate depends on inspecting the exception's properties. The mechanism is general; the cost is high enough that exceptions are best reserved for genuinely exceptional conditions rather than used as ordinary control flow. Module 13 covers errors in depth.

Two C# patterns matter for the stack trace specifically. `throw;` rethrows the current exception preserving its stack. `throw ex;` resets the stack trace to the rethrow point, losing the original origin. The difference is invisible at the call site and shows up only when something fails in production and the stack trace points at the wrong place. Code that wants to rethrow should usually use the parameterless form.

`IDisposable` is the language's deterministic-cleanup story. An object implementing `IDisposable` has a `Dispose()` method that releases resources. The `using` statement ensures `Dispose()` is called when the variable goes out of scope, even if an exception is thrown:

```csharp
using var stream = File.OpenRead(path);
// ... use stream ...
// stream.Dispose() runs automatically when the enclosing scope exits,
// whether the scope exits normally or via an exception
```

`IAsyncDisposable` is the async equivalent, used with `await using`. The recurring confusion for engineers from JavaScript — where the garbage collector handles everything — is that .NET resources requiring deterministic cleanup do exist (file handles, network connections, database connections, lock handles) and that letting the GC clean them up "eventually" is usually wrong. The `using` discipline is what keeps that from being a production problem.

## Production primitives

Four categories of language-level decisions produce a disproportionate share of production bugs in backend systems. Each is the kind of choice that looks trivial at the call site and surfaces under load or at scale.

### Time

`DateTime` is type-uniform without timezone awareness; its `Kind` property indicates whether it represents UTC, local time, or unspecified, but the type system does not prevent mixing the three. `DateTimeOffset` includes a UTC offset and is the right default for most timestamps that cross process boundaries — it removes the ambiguity that produces the most common class of timezone bugs. It captures the offset at the moment of recording, not the timezone identifier, which means systems that need to round-trip wall-clock representations through DST transitions need to store the timezone alongside the offset (or use a library such as NodaTime's `ZonedDateTime`, which models full timezone awareness directly). `DateOnly` and `TimeOnly` (added in .NET 6) are for cases where only the date or only the time matters and storing a `DateTime` would be misleading. Clock injection — accepting an abstraction such as `TimeProvider` (added in .NET 8) rather than calling `DateTime.UtcNow` directly — is what makes time-dependent code testable without wall-clock dependencies.

### Money

`decimal` is the right type for monetary amounts. `double` and `float` are floating-point types that cannot represent most decimal fractions exactly, and using them for money produces bugs that pass cursory testing and surface as off-by-cent discrepancies at scale. The cost of `decimal` is performance — slower arithmetic, larger memory footprint — and that cost is almost always worth paying. Currency itself is a separate concern. Representing ten dollars as `10m` loses the *dollar* information; a money type that pairs amount with currency is the safer abstraction in any multi-currency system.

### Identifiers

`Guid` is .NET's 128-bit identifier type. As primary keys, randomly-generated GUIDs have a notorious cost: their non-monotonic ordering causes heavy fragmentation in clustered indexes, because newly-inserted rows scatter across the index rather than appending. Sequential GUIDs (`Guid.CreateVersion7()`, added in .NET 9) and ULIDs solve this by ordering identifiers monotonically over time. The choice between random and sequential is a database performance question. The choice of identifier scheme also has security implications: sequential or guessable IDs leak information (creation order, total volume) to anyone who sees them, which makes random or hashed identifiers the safer choice for IDs that appear in URLs.

### Strings

The `string` type is immutable — every "modification" creates a new string — and the runtime interns string literals so that two literals with identical content share storage. The pitfall most likely to ship to production is comparison. `string.Equals` and the `==` operator default to *culture-sensitive* comparison, which can produce different answers in different cultures. The canonical example is the Turkish locale, where `"i".ToUpper()` returns `"İ"` rather than `"I"`, which means a case-insensitive comparison that works on a developer's laptop can silently disagree with itself in production under a different locale. The right default for backend code that compares identifiers, configuration keys, file paths, or anything not meant for end-user display is `StringComparison.Ordinal` or `StringComparison.OrdinalIgnoreCase` — explicit, culture-independent, and faster than the cultural alternatives. Code that calls the parameterless overloads inherits whatever culture the process is running under, which is rarely what the developer meant.

## A production failure that lives in the language

A scenario from a real codebase, anonymized. A team adopts nullable reference types across their codebase and turns NRT warnings into errors. The build is clean. Several months in, a production endpoint begins throwing `NullReferenceException` from a field the type system says cannot be null. The field is annotated as a non-nullable `string`, populated from a deserialized JSON payload. The deserializer is configured for permissive parsing and writes `null` into the field whenever the corresponding JSON property is absent. The compiler doesn't see this — the deserializer's API is annotated as if it produces non-null values, and the runtime doesn't enforce the compile-time contract.

The fix is small once located: add a validation step after deserialization that converts missing required fields into a typed parse error rather than letting the half-constructed object escape into the rest of the system. The diagnostic took longer than the fix because the team had developed a working assumption that NRT-clean code would not produce null reference exceptions. That assumption is correct for code that stays within the language's enforcement. It is not correct at boundaries where serializers, ORMs, reflection, or third-party libraries cross into the type system without honoring its annotations.

The strongest counterargument is that this is a serializer configuration problem rather than a language problem, and the fix belongs in the deserializer's settings rather than in a separate validation step. That is partly fair. Stricter deserializer configuration would have caught this case. The deeper observation is that NRT is a *compile-time* contract; the runtime has no obligation to honor it, and any code path that produces a value the compiler didn't see is a place where the compile-time guarantees stop. The common offenders are deserialization, reflection (the runtime ability to inspect types and members by name and call them dynamically), generic methods that lose specific type information about `T`, and third-party APIs without nullability annotations. The discipline is to validate at trust boundaries (Module 11 covers this in depth) rather than to trust that NRT-clean code is null-free.

## A load-bearing point

The first place language fluency is exercised under pressure is the code review of code you'll answer for — your own, your team's, and the increasing volume that comes from AI tooling. Code that ships under review tends to look idiomatic — the keywords are right, the patterns are recognizable, the conventions are followed. Whether it does what it appears to do is a separate question, and answering it requires reading at the level of mechanism rather than at the level of syntax.

The questions a fluent reviewer asks are routine in shape and load-bearing in practice. A method that takes an `IEnumerable<T>` and calls `.Count()` twice on it — does the source enumerate twice, and if so, what does that cost when the source is a database query? A switch expression over a sealed hierarchy that the AI generated without an exhaustiveness arm — what happens to existing call sites when a new case is added later? A class that holds disposable fields without an explicit dispose pattern — is the cleanup story complete, or does the class quietly leak resources? An async method whose return is not awaited — what scope is the work running in, and what happens if the scope ends before the work does? A struct used as a dictionary key whose `GetHashCode` falls back to the default — how badly distributed are the resulting buckets?

Each question is answerable for someone who reads at the level of mechanism. None is answerable at the level of syntax. The reviewer who can ask them is doing work that the producer of the diff couldn't do, regardless of whether the producer was an AI or another engineer who hadn't yet developed mechanism-level fluency.

## Closing

The shortest version of this module is that C# is a language whose features carry meaning beyond their syntax — meaning the compiler enforces in some places, the runtime enforces in others, and engineering discipline enforces where neither does. Reading code at the level of mechanism means knowing which is doing the work in any given line, and being able to predict what the system will do at runtime as a result.

The question worth carrying forward into the rest of the foundation arc: when you read C# at the surface, the code uses certain features; when you read at the mechanism level, what is actually happening, and where does what is happening stop matching what the surface suggests?
