# Module 3 — Dependency Injection and Inversion of Control

Dependency injection is widely treated as a tool for making code testable. That is a real benefit, but it sits downstream of the actual frame. Dependency injection is the practice of supplying a class's collaborators from outside the class, rather than letting the class construct or look them up itself. *Inversion of control* is the broader principle it serves — control over what the class collaborates with shifts from the class up to whichever code constructs it. The substitutability that makes testing easy is downstream of the inversion.

A *container* is one way to do dependency injection at scale: the runtime bookkeeping that resolves a graph of dependencies on demand, given a set of registrations. The *graph of dependencies* is the network of types that depend on each other — if `OrderProcessor` needs `IOrderRepository`, and `IOrderRepository` needs further collaborators, those connections form a graph:

```
OrderProcessor
    ├─> IOrderRepository
    │       └─> AppDbContext
    └─> IPaymentGateway
            └─> HttpClient
```

*Registrations* are the lookup table that tells the container which concrete type to construct when something asks for a given interface. The container is one specific tool, not the practice itself. Manually composing the dependency graph at the entry point — sometimes called *pure DI* — is also dependency injection, and is the right approach for some applications. Most production backend services use a container because the graph is too large to wire by hand; this module's vocabulary follows that majority case while naming where it doesn't apply.

> **Dependency injection in one example**
>
> Without dependency injection, a class chooses its own collaborator:
>
> ```csharp
> public class TokenIssuer
> {
>     private readonly SystemClock _clock = new SystemClock();
>
>     public Token Issue(UserId id)
>         => new Token(id, issuedAt: _clock.UtcNow);
> }
> ```
>
> With dependency injection, the class declares what it needs and someone else supplies it:
>
> ```csharp
> public class TokenIssuer(IClock clock)
> {
>     public Token Issue(UserId id)
>         => new Token(id, issuedAt: clock.UtcNow);
> }
> ```
>
> The first version chooses `SystemClock`. The second is told which clock to use — by production code (passing a `SystemClock`), by a test (passing a `FakeClock`), by a tenant-specific configuration (passing a tenant clock). The choice has moved from inside the class to whoever constructs it. (Two notes on the syntax: in a *class*, primary-constructor parameters like `clock` here are available inside the class body but are not automatically exposed as properties; in a *record*, they would become public properties. `IClock` is used throughout this module as a small teaching abstraction; production code would typically inject the framework's `TimeProvider`, introduced in Module 1.)

This module is one specific instance of the discipline Module 2 named: an interface separating contract from implementation is the abstraction; constructor injection of that interface — declaring the dependency as a constructor parameter, as in the example above — is the runtime mechanism that makes the abstraction operational. The decisions dependency injection hides are which concrete type collaborates with which, and the lifetime each instance lives at. Most production bugs attributed to "DI" come from those commitments — particularly the commitments about lifetime. Knowing what the container is doing is the difference between recognizing a registration as coherent and writing one because the surrounding code looked similar.

## What inversion of control actually inverts

Without dependency injection, a class constructs its collaborators directly. A `TokenIssuer` that needs a clock might write `new SystemClock()` somewhere inside it. The class decides which clock to use. The control over that decision sits with the class itself.

Inversion of control is the move that takes that decision out of the class. The class still depends on a clock, but it no longer chooses which clock. It declares the dependency — usually as a constructor parameter — and waits to be told what to use. The choice has moved up, to whichever code constructs the class.

`new TokenIssuer(new SystemClock())` is already inversion of control. The container is not what makes it inversion; the container is the bookkeeping that makes the inversion practical at scale. When a system has dozens of services with dozens of dependencies each, manually constructing the graph at the call sites becomes its own management problem. The container automates the construction by reading the registrations and resolving the graph at runtime.

This framing matters because it correctly locates what the container does. The interface that separates contract from implementation is the abstraction; the container is the runtime mechanism that hands callers the bound implementation when they ask for the contract. The container is bookkeeping, not architecture.

## The three forms of injection, and the one that wins

Constructor injection accepts dependencies as constructor parameters. The class's constructor signature documents what the class needs. The class is impossible to instantiate without supplying all its dependencies, which means there is no half-constructed state to defend against.

Property injection accepts dependencies through public settable properties after construction. The dependencies don't appear in the constructor, which means they aren't visible in the type's signature; you have to read the class to know what it depends on. Property injection is used when a dependency is genuinely optional, or when constructor injection would create a cycle.

Method injection passes dependencies into specific methods that need them. Method injection is used when the dependency is only needed for that one method and would otherwise pollute the class's identity.

```csharp
// Constructor injection — the default
public class TokenIssuer(IClock clock)
{
    public Token Issue(UserId id) => new Token(id, clock.UtcNow);
}

// Property injection — for genuinely optional dependencies
public class TokenIssuer
{
    public IClock? Clock { get; set; }
    public Token Issue(UserId id)
        => new Token(id, (Clock ?? new SystemClock()).UtcNow);
}

// Method injection — when the dependency varies per call
public class TokenIssuer
{
    public Token Issue(UserId id, IClock clock)
        => new Token(id, clock.UtcNow);
}
```

In modern .NET, constructor injection wins by default. It makes dependencies visible in the type's signature, prevents instances from existing in incompletely-configured states, and integrates cleanly with the framework's container, which uses constructor parameters to drive resolution. A reader knows what a class needs by glancing at its constructor; a test instantiates it by passing fakes; a registration in the composition root binds the abstract types to concrete implementations. Property and method injection are exceptions, not defaults.

## The three lifetimes

The container has to decide, for any registration, how long the instance it constructs lives. Modern .NET's built-in container offers three answers.

*Transient* registrations produce a fresh instance every time the container is asked for one. Two callers asking for the same transient service get two different instances. Transient is right when the service has no state worth sharing — typically pure-behavior helpers, formatters, validators.

*Scoped* registrations produce one instance per *scope*. A scope is a runtime-managed boundary the framework creates around a logical unit of work: in ASP.NET Core, the scope is created automatically per HTTP request; in Azure Functions, per function invocation; in worker services, by whatever code calls `IServiceScopeFactory.CreateScope()`. Two services resolved from the same scope get the same scoped instance. Two services resolved from different scopes get different ones. Scoped is right for things that should live "for the unit of work the system is currently processing" — repositories, request-bound contexts, EF Core's `DbContext`.

*Singleton* registrations produce one instance for the lifetime of the application. Every caller, every scope, every thread shares the same instance. Singleton is right for stateless services, for services holding genuinely process-wide cached data, or for services representing an external resource that's expensive to create. Singletons must be thread-safe, because everything that calls them is calling concurrently.

Each lifetime is a commitment about identity and concurrency.

```csharp
services.AddSingleton<IClock, SystemClock>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddTransient<IOrderValidator, OrderValidator>();
```

The three lines look similar. They commit the system to very different runtime topologies. To make the difference concrete, here is what each lifetime returns when the same service is resolved twice within a scope and once in a separate scope:

```csharp
// sp is the application's IServiceProvider.
// Two resolutions in scope A, one in scope B:
using var scopeA = sp.CreateScope();
var a1 = scopeA.ServiceProvider.GetRequiredService<IFoo>();
var a2 = scopeA.ServiceProvider.GetRequiredService<IFoo>();

using var scopeB = sp.CreateScope();
var b1 = scopeB.ServiceProvider.GetRequiredService<IFoo>();

// AddSingleton:  a1 == a2 == b1   (one instance for the whole process)
// AddScoped:     a1 == a2,  a1 != b1   (same within a scope, different across scopes)
// AddTransient:  a1 != a2 != b1   (a fresh instance every resolution)
```

A reviewer who reads `AddSingleton<IOrderRepository, OrderRepository>` should immediately ask what state the repository holds and what its dependencies are — because a singleton repository is a registration that almost always wants to be scoped.

## Lifetime mismatch: the anchor anti-pattern

When a longer-lived service holds a reference to a shorter-lived service, the shorter-lived service is *captured*. It lives at least as long as the service that captured it, regardless of what its registered lifetime says. The captured instance is a stale version of itself, called from contexts it wasn't meant to serve.

The canonical case: a singleton service depends on a scoped `DbContext`. The `DbContext` was registered scoped because it represents a per-request unit of work — the session of changes a single request makes against the database. Captured by a singleton, it now lives for the lifetime of the application:

```
OrderProcessor [Singleton]    ← lives for the whole process
        │
        │ holds a reference to
        ↓
DbContext [Scoped]            ← supposed to live for one request,
                                but captured — now outlives its scope
```

It gets reused across many requests. It collects tracked entities indefinitely. It is not thread-safe, and it's now being called from concurrent requests. State bleeds between requests. Performance degrades as the change tracker accumulates entities. The bug surfaces hours after deployment, in patterns that look random.

The fix many teams reach for is to inject `IServiceScopeFactory` into the singleton, create a scope per call, resolve the dependency from the scope, and dispose:

```csharp
public class OrderProcessor(IServiceScopeFactory scopeFactory)
{
    public void Process(Order order)
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // ...
    }
}
```

This compiles, runs, and "fixes" the symptom. It also makes the design worse. Each manual scope is its own `DbContext`, which means the two repository calls happen in two separate *units of work* — sequences of changes that should commit or roll back together as one logical operation. Transactional consistency across them becomes impossible without explicit coordination. The unit-of-work boundary the scoped lifetime was modeling is gone, replaced with whatever ad-hoc boundary each call to `CreateScope()` happens to define. New engineers absorb the pattern as the standard way to bridge lifetime mismatches, and the mismatch propagates.

The actual fix is to match the lifetime to the unit of work. If a service's purpose is per-request — orchestrating an HTTP request, processing a function invocation, handling a unit of work — the service should be scoped, not singleton. If a service genuinely needs to be a singleton, it should not depend on per-request state at all. It should accept the per-request data it needs as method parameters, or it should be split: a singleton for the process-wide concern, plus a scoped collaborator for the per-request work.

The diagnostic discipline is to read the lifetime up the dependency graph. If a service is registered with lifetime X, every dependency it holds must have a lifetime equal to or longer than X. Anything shorter is captured.

This is the anchor anti-pattern of the course. It will reappear: in EF Core (Module 9), where the `DbContext` is the most common captured dependency; in observability (Module 14), where logger scopes are similarly affected; in testing (Module 18), where lifetime-mismatched designs are precisely the designs that resist clean test setup.

## What the container actually does

The mechanics described here are those of `Microsoft.Extensions.DependencyInjection`, the built-in container that ships with .NET. Third-party containers (Autofac, Lamar, DryIoc, others) historically offered capabilities the built-in one lacked; as Module 0 noted, the built-in container has matured to the point where reaching for a third-party container is a special-case decision rather than a default. The concepts in this section apply to all of them.

Calling `services.AddSingleton<IFoo, Foo>()` does not create a `Foo`. It records metadata: "when someone asks for `IFoo`, give them a `Foo`; lifetime is singleton." The registrations live in an `IServiceCollection` during startup, then get baked into an `IServiceProvider` when the host is built.

When the framework needs to resolve a service — typically when constructing a controller, a function entry point, or any other root object — it asks the provider for the type, and the provider walks the constructor's parameters, resolving each recursively, until it reaches a graph it can fully construct. Resolving an `OrderController` whose constructor depends on an `IOrderService` and a logger looks roughly like this:

```
Resolve(OrderController)
    │
    ├─> Resolve(IOrderService)             ← scoped: cached for this scope
    │      └─> Resolve(IOrderRepository)
    │             └─> Resolve(AppDbContext) ← scoped: cached for this scope
    │
    └─> Resolve(ILogger<OrderController>)  ← singleton: cached process-wide
```

For singletons, the provider caches the instance after the first resolution. For scoped, it caches per scope. For transient, it doesn't cache. The cost of resolution is the cost of walking the graph plus instantiating each entry; modern .NET uses compiled expression trees and other optimizations to avoid reflection on the *hot path* (the code path executed often enough to matter for performance), but the conceptual model is "the container walks the dependency graph and constructs it on demand."

A few consequences worth noticing.

The container resolves by type, not by name. If two registrations of the same interface exist, the last one wins. .NET 8 added *keyed services* — `AddKeyedSingleton<IFoo, Foo>("primary")` — for cases where you genuinely need multiple registrations of the same interface distinguished by a key, but for most code one registration per type is the right discipline.

The container knows nothing your registrations don't tell it. If a constructor depends on `IFoo` and no registration for `IFoo` exists, resolution fails at runtime — usually deep inside a request, with a stack trace that points at the framework rather than at the missing registration. The framework can validate registrations at startup if you opt in (`builder.Host.UseDefaultServiceProvider(o => { o.ValidateOnBuild = true; o.ValidateScopes = true; })`), and turning that on in non-production environments catches both missing registrations and lifetime mismatches before deployment.

A few mechanisms round out what the container does that the registration count alone doesn't show. *Disposal*: the container tracks `IDisposable` and `IAsyncDisposable` instances it constructed, and disposes them when their scope ends — at end-of-request for scoped services, at host shutdown for singletons. Instances *not* constructed by the container (those returned from a factory delegate, for instance) are not automatically disposed. *Factory registrations* let registration logic exceed what the container can express by reflection: `services.AddSingleton<IFoo>(sp => new Foo(sp.GetRequiredService<IBar>(), staticConfig))` runs the lambda the first time `IFoo` is resolved, then caches the result according to the lifetime. *Open generics* — `services.AddSingleton(typeof(IRepository<>), typeof(Repository<>))` — register the type shape `IRepository<>` itself, rather than a specific instantiation. The container can then construct any closed instantiation (`IRepository<Order>`, `IRepository<Customer>`) on demand, without you having to enumerate every entity type the system handles. *Decorators* wrap a registered service in another service that has the same interface. The built-in container doesn't natively support this; the Scrutor library adds it with `services.Decorate<IFoo, LoggingFoo>()`. This is the standard way to compose *cross-cutting concerns* (logging, caching, retry, authorization — behavior that wraps many services in the same way) around an existing implementation.

The container is fundamentally a runtime mechanism. Its decisions happen when the application runs, not when it compiles. A class that compiles cleanly can fail to resolve. A test that passes locally can resolve a different graph than production. The *composition root* — the place where the system's dependency graph is wired up, covered in its own section below — is what makes those decisions auditable.

## When DI earns its weight, and when it doesn't

It's worth being honest about cost. DI is not free. The indirection makes debugging harder — clicking through to the implementation requires knowing which registration applies, and registrations live separately from the code that uses them. The runtime resolution adds a layer that can fail in ways the compiler cannot catch. The container itself is a dependency the team has to understand.

Some applications don't pay for what DI offers. A small console utility with one path through, no substitution needs, and no testing surface beyond integration tests gains nothing from a container. Wiring such a project through DI is overhead with no offsetting benefit; the right structure is `new ConcreteType(...)` inside `Main`.

Most production backend services are not in that category. They have substitution needs (tests want fakes, multi-tenant deployments want different configurations, A/B experiments want pluggable behavior). They have lifetime concerns (per-request `DbContext`s, per-process caches, per-tenant configuration). They have cross-cutting services that would otherwise be threaded through every method as parameters. For these systems, DI's bookkeeping pays for itself many times over, and the discipline is using it well rather than questioning whether to use it at all.

The honest position: ask the four questions from Module 2 of any proposed DI registration. What is it for? What does it promise? What does it hide? What does it cost? Most backend services answer in DI's favor. Some don't, and the design is better for not pretending otherwise.

## The composition root

The composition root, a term from Mark Seemann's *Dependency Injection Principles, Practices, and Patterns* (2019, with Steven van Deursen), is the one place in a system where abstract types are bound to concrete types. In modern .NET, it is `Program.cs` — specifically, the sequence of `services.Add...()` calls between `WebApplication.CreateBuilder(args)` and `builder.Build()`. Anywhere else in the codebase, code is consuming the bound types. Only at the composition root are the bindings decided.

This is a load-bearing concept. The composition root is where the system's runtime topology is set: which interface maps to which implementation, what lifetime each instance lives at, what cross-cutting middleware is wired up, what configuration each service sees. Reading the composition root tells you what the system is composed of, faster than reading the application code. Adding a cross-cutting concern — a new interceptor, a new piece of logging, a new authorization policy — is done at the composition root. Tests substitute the composition root with a different composition root, configured for testability.

A common pattern is the named-extension-method composition root: an extension method that aggregates a coherent group of registrations behind an intent-revealing name. Each service's `Program.cs` calls one method and inherits a known infrastructure baseline.

Without the pattern, every service's `Program.cs` carries the same registrations:

```csharp
// service-a/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddSingleton<ITelemetry, ApplicationInsightsTelemetry>();
builder.Services.AddScoped<IRequestContext, RequestContext>();
builder.Services.AddOptions<RetryOptions>()
    .Bind(builder.Configuration.GetSection("Retry"));
// ... fifteen more lines, repeated in service-b/Program.cs and service-c/Program.cs
```

With the pattern, the registrations live in one place and each service's `Program.cs` reduces to a single intent-revealing line:

```csharp
// shared library
public static IServiceCollection AddCommonServices(
    this IServiceCollection services,
    IConfiguration config)
{
    services.AddSingleton<IClock, SystemClock>();
    services.AddSingleton<ITelemetry, ApplicationInsightsTelemetry>();
    services.AddScoped<IRequestContext, RequestContext>();
    services.AddOptions<RetryOptions>().Bind(config.GetSection("Retry"));
    return services;
}

// service-a/Program.cs
builder.Services.AddCommonServices(builder.Configuration);

// service-b/Program.cs
builder.Services.AddCommonServices(builder.Configuration);
```

The pattern works because it makes the boundary visible: a single line claims a known set of registrations. Replacing or augmenting that set is one line away, in one place, instead of in twenty.

## Anti-patterns

A few patterns recur often enough to name.

*Service locator.* Injecting `IServiceProvider` (or any service-provider-like abstraction) into a class and pulling dependencies out of it imperatively, instead of declaring them in the constructor.

```csharp
// Wrong: dependencies hidden behind the provider
public class TokenIssuer(IServiceProvider services)
{
    public Token Issue(UserId id)
    {
        var clock  = services.GetRequiredService<IClock>();
        var signer = services.GetRequiredService<ITokenSigner>();
        return new Token(id, clock.UtcNow, signer.Sign(id));
    }
}

// Right: dependencies declared in the constructor
public class TokenIssuer(IClock clock, ITokenSigner signer)
{
    public Token Issue(UserId id)
        => new Token(id, clock.UtcNow, signer.Sign(id));
}
```

The class's signature stops advertising what it depends on. The dependency graph stops being statically inspectable. Tests have to mock the provider rather than supplying fakes. The container has effectively been demoted from "the bookkeeping for inversion" to "a runtime hashtable my class queries for things." It is one of the surer ways to make a codebase resistant to refactoring. There are legitimate uses — middleware components that resolve scoped services per-invocation, framework integration points that hand the developer an `IServiceProvider` directly, factories inside the composition root that resolve types dynamically. The discipline is to confine the pattern to the places where it earns its keep, not to scatter it through application code as the default way of finding collaborators.

*Captive dependency.* Covered above. Surfaces in registrations whose lifetimes don't match — singleton holding scoped, scoped holding transient under unusual conditions, and so on.

*"Just register everything as singleton."* A common attempt at performance optimization that works until the registered services start holding state or depending on per-request collaborators. Singleton is a contract about identity and statefulness; reaching for it because it's "fast" without understanding what it commits the system to produces bugs that are hard to attribute back to the registration.

```csharp
// Holds the user identity for the current request
public class CurrentUserContext
{
    public string? UserId { get; set; }
}

// Wrong: a per-request concept registered as a process-wide singleton
services.AddSingleton<CurrentUserContext>();   // every request shares one

// Right: scoped — one instance per request, isolated across requests
services.AddScoped<CurrentUserContext>();
```

*"Just register everything as transient."* The opposite over-correction: a fresh instance every time, defensively, to avoid sharing. Defeats per-request unit of work. Multiplies allocations. Breaks the assumptions of services that expected scoped collaboration.

```csharp
// Wrong: each repository gets its own DbContext, breaking the per-request unit of work
services.AddTransient<AppDbContext>();
services.AddTransient<OrderRepository>();
services.AddTransient<CustomerRepository>();

// Right: scoped — one DbContext per request, shared across the request's repositories
services.AddDbContext<AppDbContext>();          // scoped by default
services.AddScoped<OrderRepository>();
services.AddScoped<CustomerRepository>();
```

*Hidden constructor injection.* Using reflection or framework-specific tricks to construct objects outside the container's awareness. The constructor signature stops being authoritative; the team starts to distrust the type system because it isn't telling the whole story.

```csharp
// Wrong: bypassing the container with reflection
var handler = (IHandler)Activator.CreateInstance(handlerType)!;
// Whatever dependencies the handler needs go un-injected; if the constructor
// requires an IClock, this code constructs an object with no clock at all
// (or throws because there is no parameterless constructor).

// Right: register the type and let the container construct it
services.AddScoped<OrderHandler>();
// then resolve through the provider, which honors the constructor's dependencies:
var handler = serviceProvider.GetRequiredService<OrderHandler>();
```

When this pattern shows up in a codebase, treat it as a signal that the team has fought with the container and lost — and look for what they were trying to avoid.

*Hidden global dependencies.* A class whose constructor declares no dependencies but whose body calls `DateTime.UtcNow`, `Console.WriteLine`, `Random.Shared`, `Environment.GetEnvironmentVariable`, or any static accessor for global state.

```csharp
// Wrong: depends on the system clock without declaring it
public class TokenIssuer
{
    public Token Issue(UserId id)
        => new Token(id, issuedAt: DateTime.UtcNow);
    // A test cannot substitute the clock; the dependency is invisible.
}

// Right: declare every collaborator the class actually depends on
public class TokenIssuer(TimeProvider time)
{
    public Token Issue(UserId id)
        => new Token(id, issuedAt: time.GetUtcNow());
}
```

The constructor of the wrong version lies — the class depends on the global, but the type's signature doesn't show it. The class compiles cleanly and looks injectable until a test tries to substitute the dependency and discovers there is nothing to substitute. The discipline is to inject every collaborator the class actually depends on, including the clock (Module 1's `TimeProvider` discussion), the random source, the environment accessor, and any other dependency that varies between production and a test.

## A production failure that lives in the container

A scenario from a real codebase, anonymized. A service registered virtually every domain service and repository as singleton, on the theory that the resulting graph would have lower allocation overhead. The `DbContext`, registered by the EF Core extension method as scoped, was the one exception. To bridge the lifetime gap, the team added an `IServiceScopeFactory` parameter to every repository, created a scope per method call, and resolved the `DbContext` from the scope.

For a while, this looked fine. Reads worked. Writes worked. Performance tests didn't reveal anything obviously wrong. Months in, customer-facing bugs started surfacing in patterns the team couldn't reproduce locally. A user would update their profile and see stale data on the next page. Two operations in the same logical request would partially succeed — one persisted, the other lost. Background tasks would seem to run successfully but their results never appeared.

The root cause was the same in every case. Each repository call created its own scope, its own `DbContext`, its own change-tracking session, its own transaction boundary. There was no single unit of work for a request, even though every layer of the application thought there was one. Calls that should have been atomic happened in independent transactions. Reads that should have seen pending writes in the same request didn't, because the writes hadn't been committed by the other `DbContext` yet. The "fix" — `IServiceScopeFactory` — had hidden a lifetime mismatch behind a manual scope, and the manual scope had silently severed the unit-of-work boundary.

The repair was to make the service layer scoped, restore the framework's per-request `DbContext`, and remove every `IServiceScopeFactory` reference. The diagnostic took longer than the fix because the team had treated `IServiceScopeFactory` as an officially endorsed bridge between lifetimes rather than as a workaround for a lifetime mismatch. The first task in the repair was disabusing the team of the workaround's status.

The strongest counterargument is that some scenarios genuinely require a scope created on demand — a long-running worker that processes one item per scope, for instance, where there is no inherent request to align scopes against. That is true. The `IServiceScopeFactory` pattern is appropriate when the scope is the natural unit of work, not as a way to give a singleton access to scoped state. The diagnostic question is whether the manual scope is creating a unit-of-work boundary or papering over a lifetime mismatch. Workers genuinely need the former. Singletons that depend on per-request state are signaling the latter.

## A load-bearing point

The first place DI fluency is exercised under pressure is the code review of a new service registration. Someone proposes adding a service. Why is it a singleton? What does it depend on? Does anything in its dependency graph have a shorter lifetime than the registration claims? Does the service hold state? Is the state thread-safe? Does the service participate in a unit of work, and if so, is the unit of work the request scope or something else?

The questions are routine in shape. They are also load-bearing. The difference between a service that quietly captures a scoped `DbContext` and one that doesn't is whether the reviewer asked. The difference between a registration that defines a coherent runtime topology and one that signals trouble six months later is whether someone slowed down at the moment of registration to walk the dependency graph.

The same questions apply to AI-generated code. AI tools tend to mirror the lifetime patterns of nearby registrations. When the surrounding code has a captive dependency masked by `IServiceScopeFactory`, the AI's contributions extend the pattern. A reviewer with mechanism-level fluency in DI can see this — the registration looks plausible, but the lifetimes don't add up — and is doing work the producer of the diff couldn't do.

The composition root is the natural place to do this work. A composition root that reads cleanly is one where every registration is defensible by the four questions of Module 2. A composition root that doesn't read cleanly is the system telling you, in advance, where the bugs will live.

## Closing

The shortest version of this module is that dependency injection is the runtime bookkeeping for an inverted pattern of control, and the most common way to use it badly is to mismatch lifetimes. Constructor injection makes dependencies visible. The three lifetimes commit the system to different topologies. The composition root is where those commitments are defensibly made.

The question worth carrying forward into the rest of the foundation arc: when you read a service registration, what is the lifetime committing the system to about that service's identity, concurrency, and statefulness — and is every dependency in its graph living at least as long as the service is? If you can't answer in a sentence, the registration hasn't finished deciding what it is.
