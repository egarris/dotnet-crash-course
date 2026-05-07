# Module 0 — What .NET Actually Is

The word ".NET" has carried different meanings for over two decades. It started as a marketing umbrella for half a dozen Microsoft products. Today it points to something more concrete: a runtime, a standard library, a development toolchain, and a layered ecosystem of opinionated libraries built on top. Reasoning about a .NET service from the outside — as if "use .NET" were a single decision — is the kind of shorthand that works fine until something in production stops behaving the way someone expected. Then the labels stop helping and the substrate starts mattering.

This module sets up the frame the rest of the course relies on. Not as a feature tour, and not as a history lesson. The interesting question is what each part of the platform is doing for your code, and what each part commits you to once you've shipped.

That distinction matters because the substrate leaks. When a service slows down, when memory grows, when a request hangs, the explanation tends to live in the runtime's behavior at least as often as in the application code. Engineers who treat .NET as a black box can write code that works. Engineers who treat it as a system with its own commitments and failure modes can reason about why the code behaves the way it does once it leaves the laptop.

## The runtime: what manages your code while it runs

When a .NET service starts, the operating system isn't running your code directly. It's running the .NET runtime — the program historically called the CLR, the Common Language Runtime — and the runtime is what loads your compiled code and executes it. Most of what people think of as ".NET behavior" is the runtime's behavior. Five of its responsibilities tend to surface in production reasoning.

**JIT compilation.** Your C# source compiles to an intermediate representation called IL (intermediate language). The runtime translates IL to native machine code on demand, the first time a method is called. This means the first execution of a code path can be substantially slower than subsequent executions. The runtime offsets the cost with tiered compilation: it produces a fast-but-unoptimized version on first call, observes how the code is used, then replaces it with an optimized version in the background. The tradeoff is that "warmed up" performance is not the same as "first-request" performance, and the gap is usually invisible during development.

**Garbage collection.** The runtime tracks every reference-typed object you allocate and frees memory once those objects become unreachable. The garbage collector is generational — newer objects are collected more aggressively than older ones — and it runs in concurrent and background modes designed to minimize pause time. It is also non-deterministic: you do not control when collection happens. Most production memory pathologies in .NET are not leaks in the classical sense; they are unintended retention, where references stay reachable longer than the engineer expected and the collector dutifully keeps the memory alive on their behalf.

**Type checking and metadata.** The runtime enforces type safety at the level the IL describes. This is what makes generics possible without erasure (a meaningful contrast with TypeScript or Java, where generic type parameters are erased at runtime), and it's what makes reflection (runtime inspection of types and members) possible — the metadata sits inside the loaded assembly, where code can read it. This is mostly a strength. It becomes a footgun when libraries lean on reflection for routine work that the type system could express directly.

**Threading.** The runtime owns its own thread pool, a set of operating-system threads it reuses across many short tasks. When code awaits something asynchronous, the runtime parks the current logical task and frees its thread back to the pool to run other work. This is what makes async I/O cheap on .NET: a single process can have tens of thousands of pending operations without paying for tens of thousands of threads. The cost arrives when application code blocks a pool thread on synchronous work, because the pool's capacity is finite and the symptoms of starving it are usually mistaken for something else. Module 4 picks this thread up properly.

**Exception propagation.** When `throw` runs, the runtime walks the call stack looking for a `catch` block whose declared type matches the exception. It also runs `finally` blocks along the way and supports filtered catches. The mechanism is general enough that it can be used for ordinary control flow; the cost of doing so is high enough that exceptions tend to be best reserved for genuinely exceptional conditions. Module 13 takes this question seriously.

These five responsibilities are the core of what's there whenever your code runs. Everything else — the standard library, the hosting model, third-party packages — sits on top of this layer and inherits its behavior.

## From source to running code

The .NET toolchain has a clean separation between what builds your code and what runs it. The SDK contains the compiler, the build system (MSBuild — the engine that turns project files into compiled output), the package manager integration (NuGet — .NET's public package registry), and the project tooling. The runtime contains the CLR, the standard library, and the components needed to execute compiled output. Both ship in versioned releases. A production server typically has only the runtime; a developer machine has both.

C# is compiled to IL, packaged into assemblies (DLLs and EXEs that contain IL plus metadata). The runtime loads those assemblies and JIT-compiles their IL to native code. This decoupling has consequences worth noticing. The same compiled assembly can target multiple platforms because the runtime handles platform-specific code generation. The runtime can apply optimizations that depend on actual execution patterns, because it sees the code while it runs. The runtime can also choose to skip JIT entirely.

That last option is the AOT path. Native AOT — Ahead-of-Time compilation, mature in .NET 10 — translates IL to native machine code at build time. The output is a single binary that does not need a runtime to JIT anything; it includes only the runtime services it actually uses. The benefits show up at startup, where there is no JIT to warm, and in deployment size. The costs show up in flexibility: features that depend on runtime code generation (some reflection patterns, dynamic assembly loading, certain serialization libraries) require workarounds or are unavailable. Whether AOT is the right choice depends on what the service does and how often it starts. For long-running services that handle steady traffic, JIT is usually fine and the optimization payoff over time is real. For serverless functions that scale to zero and cold-start frequently, AOT eliminates a class of latency problems that JIT cannot.

## What comes in the box

What people often praise about .NET for backend work is not the language itself; it's the breadth and quality of what comes with the platform. The standard library — traditionally called the Base Class Library, or BCL — includes implementations of nearly everything a backend service routinely needs: collections, JSON serialization, HTTP client and server, file I/O, cryptography, regular expressions, dates and times. None of this is third-party. None of it is at risk of being abandoned by an unpaid maintainer. The tradeoff is that some of the BCL has accumulated decades of API surface that doesn't always reflect current best practice; reasoning about which BCL types are the modern recommendation is part of being fluent in the platform.

On top of the BCL sits a smaller set of Microsoft-maintained libraries that have effectively become standard for backend services: `Microsoft.Extensions.*` for hosting, configuration, dependency injection, logging, and resilience; `Microsoft.AspNetCore.*` for the web framework; `Microsoft.EntityFrameworkCore` for ORM-shaped data access (an ORM — Object-Relational Mapper — is a library that translates between application objects and database tables). These are not formally part of .NET, but they ship on the same cadence and are designed to work together. Most production .NET services lean heavily on them.

The hosting model is the part that surprises people coming from other ecosystems. A .NET service is not just "the code that handles requests." It is a host process — typically built using the Generic Host abstraction — that owns configuration loading, dependency injection wiring, logging, lifetime management, and graceful shutdown. The framework provides the host; your code populates it. This is why the entry point of most .NET services looks like a sequence of `builder.Services.Add...()` calls followed by `app.Use...()` calls and a final `app.Run()`. The program loop is already written; the application code configures it.

This is one of the platform's stronger ideas, because it means concerns that other ecosystems treat as application code (config layering, DI wiring, structured logging, health checks, graceful shutdown coordination) come pre-built and consistent across services. It also means a meaningful portion of any service's behavior lives in the host configuration rather than in the visible application logic — a fact that becomes load-bearing when something misbehaves and the engineer in front of the screen doesn't yet know to look at the host.

## Choosing .NET 10: what you commit to and what's moving

.NET releases on an annual cadence. Even-numbered versions are designated LTS (Long-Term Support), supported for three years from release. Odd-numbered versions are STS (Standard-Term Support), supported for eighteen months. .NET 10, released in November 2025, is the current LTS. .NET 11, due late 2026, will be STS. .NET 12, due late 2027, will be the next LTS.

This cadence sets the rhythm of platform commitments. When a service standardizes on .NET 10, it's signing up for at least one major-version upgrade within three years if it wants to stay on a supported runtime. Migrations between adjacent .NET versions are usually uneventful — the platform takes backward compatibility seriously — but they aren't free. Behavior changes happen, especially in the libraries above the runtime. Default settings shift. Deprecated APIs eventually disappear. A team that treats version upgrades as ambient maintenance handles them well; a team that defers them until forced has the harder version of the problem.

Several choices in .NET 10 are worth designing around now because they are stable and modern.

The hosting and DI infrastructure has converged. The Generic Host pattern is the default for web, worker, and function services; the same configuration patterns and lifetime semantics apply across all of them. The platform's built-in DI container has matured to the point where third-party containers are a special-case decision rather than a default.

OpenAPI generation moved into the platform itself. `Microsoft.AspNetCore.OpenApi` shipped in .NET 9 and is refined further in .NET 10. (OpenAPI is the standard format for describing HTTP API contracts as a machine-readable schema; Module 10 covers it properly.) The previous community standard, Swashbuckle, is on a long sunset. Designing new services with the built-in package is the durable choice.

`System.Text.Json` is the recommended JSON serializer. Newtonsoft.Json (Json.NET) remains widely used and excellent for many cases, but the platform's investment is in `System.Text.Json` — better performance, better source-generation support, better AOT compatibility. New code can usually start with `System.Text.Json` and reach for Newtonsoft only when a specific behavior requires it.

Native AOT is genuinely production-ready for many service shapes. Cold-start-sensitive workloads (serverless, scale-to-zero containers) benefit measurably. Long-running services usually don't, but the option exists and the constraints are well-documented.

The resilience story has consolidated. Polly v8 (a popular library for retry, timeout, and circuit-breaker patterns) was integrated into `Microsoft.Extensions.Resilience`, which means those patterns are first-class with the host. Hand-rolled retry loops are now a clearer regression than they were three years ago.

A few things are also worth approaching with care. Some packages built on assumptions older than the current platform — heavy reflection-based serializers, dynamic assembly loaders, AutoMapper-style scanning configurations — may continue to work but increasingly fight against AOT and against the platform's direction. Their use isn't wrong, but choosing them today imposes a constraint that didn't apply when they were written.

## Where .NET is the right tool, and where it isn't

It is worth being honest about platform fit. .NET is one of the strongest backend platforms available today for a particular shape of work: typed services with non-trivial business logic, multiple integration points, structured data, long-running processes that handle steady traffic. Within that shape, the combination of language, runtime, BCL, hosting model, and library ecosystem is hard to match. The strongest counter-positions deserve their due.

Go's deployment story — a single static binary, no runtime to install, fast startup, small memory footprint — is genuinely simpler when operational simplicity matters more than ecosystem breadth. Teams running many small services on lean infrastructure often have less to apologize for in choosing Go than .NET advocates sometimes acknowledge.

Node.js with TypeScript is more idiomatic when a backend is mostly glue between I/O endpoints and the team's investment is already in JavaScript. The ramp-up cost of learning .NET's hosting model and its standard library is real, and a small team that already thinks in JavaScript can usually ship faster on Node for typical CRUD-shaped work.

Python is irreplaceable for data and machine-learning work. The ecosystem there is so dominant that suggesting .NET is asking the team to give up tools they need.

Rust earns its place when memory predictability and performance ceiling matter more than developer velocity. For most backend services, the velocity tradeoff outweighs the performance ceiling. For some — high-throughput proxies, infrastructure components — it doesn't.

The stronger version of "should this be in .NET" is rarely "is .NET capable of this," because the answer is usually yes. The better question is what choosing .NET commits this team to that another platform wouldn't. That question lands differently depending on the team's existing fluency, the operational environment, and the lifecycle of the service. A team comfortable with the platform, on Azure or a Linux container host, building a service that will run for years, has unusually little to gain from leaving .NET. A team building a stateless edge function that needs to start in fifty milliseconds has reasons to look elsewhere.

The platform decision isn't always hard. It is, however, a commitment, and commitments are worth making for reasons that hold up later when someone asks why.

## A production failure that lives in the runtime

Here is a real-shaped scenario, anonymized. A team rolled out a new HTTP service onto a serverless host. Locally, requests took thirty milliseconds end-to-end. Deployed, the first request after any quiet period took 2.3 seconds; subsequent requests in the same warm container dropped back to the expected range. The team checked their database connection pool, their network configuration, the documented cold-start behavior of the platform. They added timing around their controllers, their service layer, their data access. Nothing obviously slow. Hours into the investigation, someone noticed that the slow latency was nearly identical across cold starts, regardless of what the request did or what data it touched.

The runtime was JIT-compiling frequently-used code paths on first hit. Tiered compilation produced an unoptimized version, executed it, then started recompiling in the background. By the third or fourth request, the warm container's hot paths were optimized and fast. The first request paid for compiling them.

The fix had two parts. The team enabled Native AOT for the service, which translated IL to native code at build time and removed first-execution JIT cost from the path. They also added a small warm-up routine on container start, which exercised a representative request through the in-process pipeline before the platform reported the container as ready. Both changes were small. The diagnostic took longer than the fix because no one on the team had a working mental model of what the runtime was doing during the first request. They had treated the runtime as a black box, and the black box leaked.

This is the failure mode Module 0 trains against. The team didn't need to know the answer in advance. They needed to know where to look. Knowing where the runtime sits in the stack of things that can go slow is what makes the difference between a four-hour outage and a forty-minute one.

The strongest counterargument here deserves an answer. A skeptic could note that JIT-warmup latency is a well-known issue with workable mitigations, and that platform engineers shouldn't need to understand the runtime to apply the standard fix. That's true as far as it goes. The trouble is that the standard fix only works once the symptom has been correctly identified, and the symptom in this case was indistinguishable at first glance from a dozen other things that produce slow first requests. The mental model is what shortens the diagnostic, not the fix.

## A load-bearing point

The first place an engineer is held to commitments around .NET is the architecture review where someone asks why the team picked the platform. The answers that don't hold up — "because we know it," "because Microsoft supports it," "because the previous service is on it" — aren't wrong, exactly, but they aren't defenses. They tell the room what was true at the moment of the decision; they don't tell the room what makes the decision still right two years from now when conditions have changed.

The defensible answers tend to be more specific. "Because the service has substantial business logic that benefits from a strong type system, the team is fluent in the runtime model, and the LTS cadence aligns with our maintenance budget." "Because we're on Azure and the platform integration reduces our operational surface meaningfully." "Because we have one engineer who knows three runtimes well, and choosing this one means everyone else can ramp up quickly." Specifics survive the second meeting.

Platform choice is rarely the most important architectural decision a team makes. The point is not that it is. The point is that even small commitments need to be defensible if the engineer is going to be the one defending them. Module 0 is the first place this course asks the reader to look at a familiar thing as a commitment rather than as a default.

## Closing

The most useful frame for the rest of the course is this. .NET is not one thing. It is a runtime, a toolchain, a library, an ecosystem, and a release cadence, each with their own commitments and failure modes. Treating them as one thing is fine for most days. The days when the system surprises you are usually days when one of those layers — almost always the one no one was paying attention to — was doing more than the engineer realized.

The question worth carrying into the rest of the foundation arc: which of the runtime's responsibilities do you implicitly trust, and which would you know to inspect first when the system surprises you?
