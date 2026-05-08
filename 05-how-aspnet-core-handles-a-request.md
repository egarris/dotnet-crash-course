# Module 5 — How ASP.NET Core Handles a Request

A request comes in; a response goes out. The middle is what this module is about — and the middle is more programmable, more ordered, and more affected by configuration than most engineers realize. Code in an ASP.NET Core service does not run in isolation. It runs after a sequence of components has had a chance to inspect, transform, or short-circuit the request, and before another sequence has a chance to do the same to the response on the way out.

Understanding the sequence is understanding what can intercept what, in what order, with what scope. A team that knows the pipeline can move authorization, logging, caching, error handling, and request-id propagation into the right place — once, declaratively, where the framework can apply them consistently. A team that doesn't ends up scattering those concerns across every endpoint, with the predictable result: rules become inconsistent, exceptions get caught in the wrong places, and changes propagate by hand instead of by configuration.

This module covers the mechanics of how an ASP.NET Core service starts up, how a request flows through it, and where the points of intervention live. The specific concerns the pipeline supports — authentication, validation, error handling, observability — get their own modules. The point of this one is the shape of what those concerns sit on top of.

## The host and the startup sequence

A .NET service is not just "the code." It is a *host process* that loads configuration, builds a dependency-injection container, configures a request pipeline, and then runs until something tells it to stop. The host owns the runtime topology of the service; the application code populates the host with what to do.

The modern entry point is `WebApplication`, the standard since .NET 6. A typical `Program.cs` looks like this:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Configuration time: register services, bind options, set up logging
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(/* ... */);
builder.Services.AddOptions<RetryOptions>()
    .Bind(builder.Configuration.GetSection("Retry"));

var app = builder.Build();

// Pipeline time: configure middleware, define endpoints
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

The split between *configuration time* (everything before `builder.Build()`) and *pipeline time* (everything after) is load-bearing. Configuration time is where DI registrations and options bindings happen — once, at startup, before any request arrives. Pipeline time is where middleware gets composed in order — also once, at startup, but the composition affects every request that follows. After `app.Run()`, the host listens for requests and dispatches them through the pipeline that was just configured.

This split shapes what's possible in code review. Code that mutates `builder.Services` after `Build()` has been called is too late; the service provider is already frozen. Code that adds middleware after `app.Run()` is unreachable; the host is already running. Mistakes in either direction usually produce silent no-ops rather than loud errors, which makes them easier to catch at the configuration site than at runtime.

The Generic Host abstraction underneath `WebApplication` is the same one used by worker services, function apps, and other long-running .NET processes. Module 6 covers the host beyond HTTP requests; the patterns there (configuration, DI, lifetime, graceful shutdown) all apply equally here.

## Configuration: layered sources and the Options pattern

Configuration in modern .NET is not a single file. It is a *layered system* that combines multiple sources into one effective configuration, with later sources overriding earlier ones for any given key. The default order:

```
Effective configuration (last source wins for any key):

  Command-line arguments                ← highest priority
  Environment variables
  User secrets (development only)
  appsettings.{Environment}.json
  appsettings.json                      ← lowest priority
```

A value in `appsettings.json` can be overridden by `appsettings.Production.json`, then by an environment variable, then by a command-line flag. The framework gives the same lookup interface — `IConfiguration`, the runtime's combined view of all sources — regardless of where any particular value originated.

The Options pattern is the right way to consume configuration in application code. Instead of reaching for `IConfiguration` everywhere, you bind sections of configuration to typed classes and inject them via DI:

```csharp
// Define the shape
public class RetryOptions
{
    public int MaxAttempts { get; set; } = 3;
    public TimeSpan Backoff { get; set; } = TimeSpan.FromSeconds(1);
}

// Register in Program.cs
builder.Services
    .AddOptions<RetryOptions>()
    .Bind(builder.Configuration.GetSection("Retry"))
    .ValidateDataAnnotations()   // checks [Required], [Range], etc. on the options class
    .ValidateOnStart();          // fail fast at startup if config is invalid

// Inject into a service
public class FlakyApiClient(IOptions<RetryOptions> options)
{
    private readonly RetryOptions _opts = options.Value;
    // use _opts.MaxAttempts, _opts.Backoff
}
```

`ValidateOnStart()` is the mechanism that makes options validation happen at startup rather than at first use. With it, the host fails on launch if the bound options don't pass validation — for example, if a required property is missing or an annotated range is violated. Without it, validation runs lazily (the first time something resolves the options), and a misconfiguration becomes a runtime failure deep in a request rather than a startup failure visible in deployment logs. *Fail fast* is the discipline; `ValidateOnStart()` is what implements it.

Three flavors of options injection exist, each with a different lifetime:

- `IOptions<T>` is singleton. It captures the configuration as it was at startup. Changes to the underlying source are not reflected.
- `IOptionsSnapshot<T>` is scoped. It re-reads the configuration once per scope (typically once per request), so per-request changes can take effect.
- `IOptionsMonitor<T>` is singleton but reactive. It watches the underlying source and surfaces changes through a `CurrentValue` property and an `OnChange` callback. The right choice when long-lived components need to react to configuration changes without a restart.

Picking the wrong one is a real bug. A service that injects `IOptions<RetryOptions>` and expects to see configuration changes after a hot reload is not going to see them. A service that injects `IOptionsSnapshot<T>` into a singleton has captured a scoped dependency — Module 3's anchor anti-pattern in configuration form:

```csharp
// Wrong: scoped IOptionsSnapshot captured by a singleton
public class CachedClient(IOptionsSnapshot<ApiOptions> options)   // captive dependency
{ /* ... */ }
services.AddSingleton<CachedClient>();

// Right: pick the lifetime that matches the consumer
public class CachedClient(IOptionsMonitor<ApiOptions> options)
{
    private ApiOptions _opts = options.CurrentValue;
    // and subscribe via options.OnChange(updated => _opts = updated) if reacting matters
}
services.AddSingleton<CachedClient>();
```

The `ValidateScopes` setting from Module 3 catches this class of bug at startup — turning it on in non-production environments is the cheapest way to find captive options dependencies before they ship.

### Feature flags

Feature flags are a specific shape of runtime configuration: boolean (or per-tenant, per-user) decisions that can change without redeploying the service. They are usually layered on top of the regular configuration system, often through `Microsoft.FeatureManagement` (the in-framework library) or a third-party service such as LaunchDarkly or Azure App Configuration.

The mechanics matter mostly for lifetime. A feature flag evaluated once at startup and cached in a singleton field will not respond to changes; it has to be re-evaluated at the right granularity for what the flag controls. Most feature-flag libraries offer scoped or transient evaluation by default, with caching tunable per provider.

Feature flags carry a discipline of their own. They should be temporary. A flag that's been on in production for a year and is never going to be off again is dead code waiting to be removed; the longer it sits, the more it tangles with surrounding code that assumes the flag's current state. Module 17 covers flag retirement in depth; here the point is that "we have many feature flags" can be a sign of healthy experimentation or a sign of unfinished migrations no one has time to retire, and reviewing flag age periodically is what tells the difference.

## The middleware pipeline

The middleware pipeline is the sequence of components every request passes through, in order. Each middleware receives an `HttpContext` (the request, the response, and the per-request services) and a `next` delegate. It can do work before calling `next` (passing control to the next middleware), do work after `next` returns (with the response on its way back), short-circuit by not calling `next` at all, or any combination.

The pipeline is bidirectional: requests flow inward, responses flow outward, and each middleware can do work on each leg.

```
Request flow:                              Response flow (in reverse):

  Client ──>                                 Client <──
                                                          
  [UseExceptionHandler]                      [UseExceptionHandler]
       │                                          ↑
       v                                          │
  [UseHttpsRedirection]                      [UseHttpsRedirection]
       │                                          ↑
       v                                          │
  [UseRouting]                               [UseRouting]
       │                                          ↑
       v                                          │
  [UseAuthentication]                        [UseAuthentication]
       │                                          ↑
       v                                          │
  [UseAuthorization]                         [UseAuthorization]
       │                                          ↑
       v                                          │
  [Endpoint] ────── action runs here ──────> [Endpoint]
```

Middleware order is the most consequential thing to get right in `Program.cs`. Authentication middleware must run before authorization middleware (the latter checks claims set by the former). Routing middleware must run before authorization middleware uses route-bound policies. Exception-handling middleware must wrap everything else if it's going to catch what they throw. The framework's documentation publishes a recommended order; most deviations are mistakes.

```csharp
// Wrong: auth middleware after the endpoint mapping
// — authentication and authorization run nowhere useful for these endpoints
app.MapControllers();
app.UseAuthentication();
app.UseAuthorization();

// Right: auth middleware before the endpoint mapping
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

Middleware can be conditional. `app.UseWhen(predicate, branch => ...)` runs a sub-pipeline only when the predicate matches — useful for "apply this only to admin routes" or "skip CORS for internal endpoints" kinds of decisions. Conditional middleware is the right tool when an entire pipeline branch should differ based on the request, rather than a single middleware deciding internally what to do.

A middleware class implements `IMiddleware` (or just exposes an `InvokeAsync(HttpContext, RequestDelegate)` method, where `RequestDelegate` is the function the runtime hands you for "call the next middleware in the pipeline"); a middleware delegate is just a lambda registered with `app.Use(...)`. Both compose the same way in the pipeline.

```csharp
public class CorrelationIdMiddleware(ILogger<CorrelationIdMiddleware> logger) : IMiddleware
{
    public async Task InvokeAsync(HttpContext ctx, RequestDelegate next)
    {
        var id = ctx.Request.Headers["X-Correlation-Id"].FirstOrDefault()
                 ?? Guid.NewGuid().ToString();
        ctx.Response.Headers["X-Correlation-Id"] = id;
        using (logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = id }))
        {
            await next(ctx);   // every log line within this scope inherits CorrelationId
        }
    }
}

builder.Services.AddTransient<CorrelationIdMiddleware>();
app.UseMiddleware<CorrelationIdMiddleware>();
```

A middleware that captures scoped dependencies in fields will produce the captive-dependency pattern this course keeps returning to. The discipline: middleware classes registered as singletons should pull scoped services off `HttpContext.RequestServices` per invocation, not from constructor-injected fields.

## Routing and model binding

The routing middleware (`app.UseRouting()` and `app.MapControllers()` / `app.MapGet(...)` and similar) is where the framework decides which endpoint will handle a request. Routing happens before authorization, so policies can be bound to specific endpoints; it happens after exception handling, so routing failures (404s, method-not-allowed) can be caught.

Routes match by template — `/orders/{id:int}` matches `GET /orders/42` and binds `id = 42`. The `:int` is a *route constraint* that tells the framework to match this route only when the value parses as an integer; constraints exist for `int`, `guid`, `bool`, length-bounded strings, regular-expression patterns, and others. Routes can also be declared by attribute on a controller method:

```csharp
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<ActionResult<OrderResponse>> GetOrder(int id, CancellationToken ct)
    {
        // id has been bound from the route; ct is provided by the framework
    }
}
```

*Model binding* is the framework's process of taking the request — its route values, query string, headers, body, and form data — and producing typed parameters for the action method. The framework uses attributes when explicit: `[FromRoute]` for values in the URL path, `[FromQuery]` for the query string, `[FromBody]` for JSON or other content-typed payloads, `[FromHeader]` for header values, `[FromForm]` for form-encoded posts. Without explicit attributes, the framework picks a source based on the parameter type and the binding convention (simple types from route or query; complex types from the body for `POST`/`PUT`). Model state — a record of which parameters bound successfully and which failed — is available to the action via `ControllerBase.ModelState` (or, in minimal APIs, surfaced through validation hooks).

The discipline that matters: model binding is the framework's job. Code that reads `HttpContext.Request.Body` directly, deserializes it manually, and ignores binding failures is reimplementing the binder badly. The framework's binder handles content negotiation, error reporting, and validation hooks consistently; manual parsing handles none of that without bespoke effort.

```csharp
// Wrong: manual parsing bypasses the binder and its validation hooks
public async Task<IActionResult> Create()
{
    using var reader = new StreamReader(HttpContext.Request.Body);
    var body = await reader.ReadToEndAsync();
    var req = JsonConvert.DeserializeObject<CreateOrderRequest>(body);
    // No validation; no model state; no consistent error responses
}

// Right: let the binder do its job
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateOrderRequest req,
                                        CancellationToken ct)
{
    if (!ModelState.IsValid) return ValidationProblem(ModelState);
    // req is bound; validation hooks have run; consistent error shape if they failed
}
```

If you find yourself reading `HttpContext.Request.Body` by hand, the framework is offering you a model binder you've decided to ignore. Pick a serializer (Module 0 covers `System.Text.Json` as the modern default), configure it once at startup, and let the binding pipeline do its work.

## Filters

Middleware runs around every request. *Filters* run inside the controller pipeline, around action methods. The distinction is small but real: middleware sees every request, regardless of which endpoint it routes to; filters see only requests that have already routed to a controller action and have already been bound.

The five filter types, ordered by where they sit in the nested pipeline:

- *Authorization filters* run first, after routing but before model binding. They decide whether the request can reach the action at all.
- *Resource filters* wrap everything that follows, including model binding. They are the right tool for caching (returning a cached response without binding) and for short-circuiting on coarse criteria.
- *Action filters* run before and after the action method. They can inspect or modify bound arguments and the result.
- *Result filters* run before and after result execution (the conversion of the action's return value into a response).
- *Exception filters* catch unhandled exceptions thrown from the action or from action filters.

Filters nest rather than execute in a strict line; the inner filters run inside the wrapping of the outer ones:

```
Request ──> Authorization filter
                └─> Resource filter
                      ├─> Model binding
                      └─> Action filter
                            └─> Action method
                            (returns)
                      └─> Result filter
                            └─> Result execution
                      (returns)
                (returns)
            (Exception filter catches throws from the action and action filters)
```

Filters can be registered globally (in `services.AddControllers(o => o.Filters.Add<X>())`), per-controller (attribute on the class), or per-action (attribute on the method). The same filter type can appear at multiple scopes; global filters run outermost, action-scoped filters innermost.

The most common production use is authorization. Declaring `[Authorize(Policy = "RequireAdmin")]` on a controller or action attaches an authorization filter that runs before the action. An *authorization policy* is a named set of requirements (claims, roles, or custom rules) registered at startup; the attribute references the policy by name. The policy is evaluated against the current `ClaimsPrincipal` — the framework's representation of the authenticated user, with claims (typed key-value assertions about the user) populated by the authentication middleware that ran earlier in the pipeline.

```csharp
[Authorize(Policy = "RequireAdmin")]
[HttpDelete("{id:int}")]
public async Task<IActionResult> DeleteOrder(int id, CancellationToken ct)
{
    // The filter has already verified the policy.
    // If the policy fails, this method never runs and a 403 is returned.
}
```

The default authorization policy denies unauthenticated requests, so an endpoint with `[Authorize]` and no specific policy will reject anonymous traffic out of the box. Custom policies can be more restrictive (require specific claims or roles) or, occasionally, more permissive (allow anonymous if certain conditions hold) — the latter is unusual but possible, and worth keeping in mind when reasoning about why an endpoint accepted a request it shouldn't have.

Imperative authorization — calling `_authService.AuthorizeAsync(...)` from inside the action method — is a recurring anti-pattern. It works, but the security check is no longer auditable from the action's signature; reviewing the API surface for what's authorized requires reading every method body, and missing the call once is enough to ship an unsecured endpoint with no compile-time signal. Declarative attributes (or `RequireAuthorization` on minimal-API endpoints) are the discipline that keeps the auth model reviewable. Module 12 covers authorization in depth; here the point is just that the framework offers a place for the check to live.

## Minimal APIs and the modern shape

Minimal APIs, introduced in .NET 6 and continuing to evolve, are a lighter-weight alternative to controllers. Endpoints are declared as method calls in `Program.cs` rather than as classes:

```csharp
app.MapGet("/orders/{id:int}",
    async (int id, IOrderService orders, CancellationToken ct) =>
{
    var order = await orders.GetAsync(id, ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
})
.WithName("GetOrder")
.RequireAuthorization("RequireAuthenticated");
```

Minimal APIs use the same routing, model binding, and filter infrastructure as controllers. Parameters bind from the same sources; cancellation tokens are injected; DI works the same way. (`Results` is the minimal-API helper that produces strongly-shaped HTTP responses — `Results.Ok(value)` returns 200 with the value as JSON; `Results.NotFound()` returns 404. `TypedResults` is the strongly-typed sibling that returns concrete types like `Ok<Order>`, which integrate better with OpenAPI generation.) The difference from controllers is structural: minimal APIs put the endpoint definition next to its handler, often in fewer lines of code.

Minimal APIs have their own filter mechanism, separate from MVC's: `MapGet(...).AddEndpointFilter(...)` attaches a filter that runs around the endpoint's invocation, with access to the bound arguments and the result. Endpoint filters are how minimal APIs share cross-cutting behavior (logging, validation, custom authorization checks) without controller-style attributes. Related endpoints can be grouped with `MapGroup`, which lets a route prefix, authorization requirement, and shared filters apply to a whole group at once:

```csharp
var orders = app.MapGroup("/orders").RequireAuthorization("RequireAuthenticated");

orders.MapGet("/{id:int}", async (int id, IOrderService svc, CancellationToken ct) =>
    await svc.GetAsync(id, ct) is { } o ? TypedResults.Ok(o) : (IResult)TypedResults.NotFound());

orders.MapPost("/", async (CreateOrderRequest req, IOrderService svc, CancellationToken ct) =>
    TypedResults.Created($"/orders/{(await svc.CreateAsync(req, ct)).Id}"));
```

For services with a small number of endpoints, minimal APIs are usually the right default. For larger services with many related endpoints sharing authorization, validation, and serialization behavior, controllers remain the more navigable choice. Mixed codebases — controllers for the main API surface, minimal APIs for health checks and similar light endpoints — are common and supported.

The strongest counter-position is that a small service can become a large service, and the cost of migrating from minimal APIs to controllers later is non-trivial. That's true in principle. In practice, the migration is mechanical when it does become necessary, and starting with the lighter shape is rarely the regret. The harder regret is over-engineering controllers for a service that stays small.

## A production failure that lives in the pipeline

What follows is a real failure mode, with the team's details removed. A team had a custom *internal-call* authentication middleware — a small piece of code that read a signed header for service-to-service traffic and populated a `ClaimsPrincipal` with the caller's service identity. The middleware was added to `Program.cs` to authenticate internal requests; controllers used `[Authorize(Policy = "RequireInternalCaller")]` to require it.

Several days after a refactor, an alert fired: an external request had reached an internal-only endpoint and succeeded. The request had no signed header, no service identity, and yet had been accepted. The authorization filter, which was supposed to deny anonymous traffic, had not done so.

The investigation found that the custom middleware had been moved during the refactor. The `Program.cs` file now had `app.MapControllers()` *before* `app.UseInternalAuthentication()`. With the controllers mapped first, requests reached the endpoint without the custom middleware ever running. The authorization filter ran and evaluated the `RequireInternalCaller` policy — which checked for the *presence* of a specific service-identity claim. With no authentication middleware in the path, no claims were present, but the policy's evaluation logic had a bug: it fell through to allow when the claim collection was empty rather than denying explicitly. The combination of misordered middleware and a permissive policy default produced silent success.

The fix was a one-line reorder in `Program.cs` and a one-line policy correction. The diagnostic took two days because the integration tests bypassed authentication entirely (a common testing pattern that swaps the auth scheme for a test-only one), so the production-only ordering bug never showed up in tests.

The strongest counterargument is that the test suite should have caught this. That's fair as far as it goes. Tests for "authentication middleware is in the right place" are unusual in practice; most teams test endpoint behavior with auth bypassed because the test scaffolding is simpler that way. The `Program.cs` ordering is a cross-cutting concern that lives outside the action under test, and the conventional unit and integration test layers don't naturally exercise it. Module 18's testing discussion picks this up: a small set of pipeline-shape tests — assertions about the order of middleware in the configured pipeline — would have caught this without requiring the rest of the suite to run with full authentication.

## A load-bearing point

The first place pipeline fluency is exercised under pressure is the code review of `Program.cs` and any change that touches it. The questions are routine; getting them wrong is how unsecured endpoints, missing telemetry, and refresh-blind options ship to production.

Is the middleware in the right order? Authentication before authorization. Routing before authorization (so policies can be route-bound). Exception handling outside everything else. Each line in the pipeline section is a commitment; the order is the system's behavior.

Is authorization declared declaratively (attributes, `RequireAuthorization`) or imperatively (calls in the action body)? Imperative authorization is route-around-the-framework; the auth model becomes invisible to anyone reading the API surface.

Are options registered with `ValidateOnStart()`? A service that fails at startup with a clear error is better than a service that fails three minutes later with a stack trace deep inside an action.

Do options consumers use the right lifetime? `IOptions<T>` for static config, `IOptionsSnapshot<T>` for per-request config consumed by scoped services, `IOptionsMonitor<T>` for long-lived components that need to react to changes. A singleton holding `IOptionsSnapshot<T>` is the lifetime-mismatch anchor in configuration form.

Does code reach for `IConfiguration` directly? Direct use is sometimes appropriate (especially in `Program.cs`); pervasive use throughout application code is usually a sign that the Options pattern hasn't been adopted consistently. The audit's "fifteen `Environment.GetEnvironmentVariable` calls scattered through the codebase" is the recurring shape.

Does any middleware capture scoped state in singleton fields? Middleware classes registered as singletons should pull scoped services from `HttpContext.RequestServices` per invocation, not hold them as fields.

Does manual code read `HttpContext.Request.Body`? If so, what model binding behavior is it bypassing — content negotiation, validation, error responses, consistent serialization?

These are the things a fluent reviewer can ask in a few minutes when a PR touches the pipeline. Code that ships under review — including the increasing volume produced by AI tooling — usually orders middleware correctly. Where it deviates — ordering off by one, options without `ValidateOnStart`, manual body reads — the reviewer is the only line of defense before the deviation becomes the next production failure.

## Closing

The shortest version of this module is that a request becomes a response by traversing an ordered, programmable pipeline, and the order is set in `Program.cs`. Configuration is layered. Options have lifetimes. Middleware composes in sequence. Filters run inside the controller pipeline. Routing happens before authorization. Model binding is the framework's job, not yours.

The question worth carrying forward into the rest of the foundation arc: when you read a `Program.cs`, can you trace what each middleware will do to a request, in order, and identify the points where the request can be short-circuited? When you read an action method, can you state what's already been done by the time it runs?
