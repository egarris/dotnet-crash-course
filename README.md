# C#/.NET Crash Course

*Evan Garris, Software Engineering Manager, May 2026*

## Why this exists

Software engineering, when it works, is a discipline of judgment. It runs on two kinds of correction. One is *computational* — a test fails, a monitor alerts, telemetry indicates drift, the system updates. The other is *normative* — an engineer commits to a description of how a system will work, and bears the discrepancy when the system behaves differently. Tools handle the first kind well, and they're getting better at it. The second kind is what holds the practice together at the points where commitments bind.

Most backend work runs on the first kind of correction, and it should. Implementation, refactoring, routine debugging, scaffolding, large parts of testing — computational feedback handles all of it well, and using good tools where they fit is sensible. The second kind of correction shows up at specific moments: architectural decisions where someone has to say *this is what we are building, and this is what I will defend if it fails*; customer-facing contracts where someone has to commit to behavior the customer can hold them to; incident ownership, where someone answers for what failed and what comes next; design reviews, where the design gets put on the table by a person who will be held to it; postmortems, where someone has to make the link between a specific judgment and a specific failure and carry the learning forward.

These are the load-bearing points. They are a small fraction of the visible throughput, and they carry most of the weight. Routing around them is fast. The cost arrives later. When conditions change, the team finds that no one was in the path of the corrections that would have built the judgment they now need.

This course is shaped around making the engineer who can be load-bearing at those points. The foundation arc builds the conceptual scaffolding those commitments need. The topical modules are the depth that lets the commitments stand up under contact with production. The workshops are where you practice taking them on. Where the curriculum addresses code production, it does so for the engineer who will be held to what gets shipped.

I looked at what's available. Microsoft Learn is an excellent reference — accurate, comprehensive, dry, shaped like an API surface. Encyclopedic books like *Pro ASP.NET Core* are 1,300-page feature tours: "here is what this method does." Online video courses are skill-acquisition-shaped: "build your first API," "master Entity Framework." Brilliant blogs — Andrew Lock, Mark Seemann, Milan Jovanović, Nick Chapsas — cover deep topics atomically, but no one has stitched them into a coherent curriculum aimed at the engineer who will be held to what they ship. That gap is what this course fills.

It's a thinking course. It teaches you to read .NET code critically, recognize good and bad shapes, design abstractions that earn their weight, and reason about how the system will behave when it runs in production. Where it shows code, the code is concrete. The lesson is how to read code of that shape, not which snippet to memorize.

## What makes this course different

- **It is about judgment that binds.** Much of the work — implementation, refactoring, routine debugging — runs on computational feedback. Tools handle it well. The judgment at the load-bearing points is different: someone has to take on a commitment, defend it, and bear the discrepancy when the system behaves differently. This course is shaped around making that engineer.
- **Plain language, technical precision.** Every term is glossed on first use. Every claim is accurate enough to bet your production behavior on.
- **A coherent curriculum, not a topic collection.** The modules build on each other. A single anchor anti-pattern — *lifetime mismatch* — threads through five separate modules and ties them together. Two integrative workshops bookend the course: one to build the mental model, one to test it.
- **Anchored in real, anonymized anti-patterns.** Every module includes at least one concrete production failure mode — the kind that shows up in pull requests, not the kind invented for tutorials.
- **No assumed background.** If you've written C# professionally, the foundation arc will move briskly. If you haven't, every term gets glossed and every concept gets grounded before it's used. You don't need to know any specific .NET library, framework, or product before you start.

## Who this is for

Engineers who want to be load-bearing on backend systems written in C# and .NET. You'll write code, review code, design systems, and answer for what ships under your name. Your job is to take on commitments — and to make sure the work that ships honors them.

**Prerequisites:** You can read code in some statically typed language. You've shipped something, somewhere, that talked to a database or another service. You don't need to know C# already; the course teaches what you need as it goes.

**The bar this course aims for:** Someone who completes the full course should be qualified to be load-bearing on a backend service — to design the API, choose the right abstractions, recognize the wrong ones, evaluate any code that lands in their review (whoever or whatever produced it), diagnose production incidents methodically, and reason about whether a proposed change will survive at scale. That is senior technical reasoning. The organizational dimensions of leading a team — earning trust, calibrating under deadline, communicating tradeoffs to non-engineers — come from real reps with shipped consequences; the conceptual scaffolding comes from here.

## What you'll get out of this

After working through this material, you should be able to:

- **Read .NET code critically.** Not as syntax, but as a system. When you look at a class, you should know what it is for, what it promises, what it hides, and where it is lying about any of those.
- **Recognize good and bad abstraction shapes.** When an abstraction earns its weight and when it is ceremony. When a boundary is in the right place and when it points the wrong way.
- **Design APIs and services that survive production.** Not just "it works in the happy path" but "it works when the database is slow, when an upstream is flapping, when the client retries, when the load spikes."
- **Evaluate code you'll be answerable for.** Tell the difference between code that looks right and code that *is* right, regardless of who or what produced it. Recognize the failure modes that recur in production — mocked dependencies that are wrong, swallowed exceptions, missing cancellation tokens, captive lifetimes, log messages that aren't queryable — so the work that ships under your name can be defended when conditions change.
- **Diagnose production issues methodically.** Not by guessing. By following a process: form a hypothesis, run the cheapest test that distinguishes hypotheses, narrow, repeat. Know what telemetry to ask for and what it tells you.
- **Make architectural decisions you can defend.** When to introduce a service and when not to. When to make something asynchronous and when to keep it synchronous. When *YAGNI* ("you aren't gonna need it" — the discipline of not building what you don't yet need) is right and when it is an excuse. When a cache will help and when it will hide a different problem.

None of this requires memorization. The cheat sheets and appendices exist so you can look things up. What matters is the mental model — accurate enough that you can reason about situations you haven't seen before.

There is a team dimension to this too. The deeper goal is **shared vocabulary**. When someone says "this is a captive dependency" in a PR review, or "this looks like thread-pool starvation, not CPU pressure" during an incident, or "the Include chain here is leaking persistence shape to the API," the rest of the team should know what that means and engage with the reasoning. That is what turns a group of individuals into a team that can own a production system together. If you are a manager introducing this to a team, that shared vocabulary is probably the highest-leverage outcome.

## The course

Nineteen modules and two workshops, structured in three acts.

**Act 1 — Foundation arc.** What .NET actually is, how to read C# critically, what abstraction is for, and how the platform assembles a working service. Sequential; each module builds on the previous.

**Act 2 — Topical depth.** The day-to-day concerns of production backend code: data access, API design, validation, security, errors, observability, diagnosis, resilience, architecture, testing. Read in any order once you have the foundations.

**Act 3 — Judgment.** Two workshops. The first integrates the foundations by tracing a real request through every layer. The second applies the topical depth by reviewing pull requests — AI-generated and otherwise — for production fitness.

### Foundation arc (sequential)

| # | Module | What you'll learn |
|---|---|---|
| 0 | [What .NET Actually Is](00-what-dotnet-actually-is.md) | The runtime, the BCL, the SDK, what's new in .NET 10, where .NET fits |
| 1 | [C# for Backend Devs](01-csharp-for-backend-devs.md) | Language features as windows into the runtime — types, async, generics, LINQ, exceptions |
| 2 | [Boundaries, Contracts, and Concerns](02-boundaries-contracts-and-concerns.md) | The course's spine. What abstraction is for, how to recognize it, how to design it |
| 3 | [Dependency Injection and Inversion of Control](03-dependency-injection-and-ioc.md) | Lifetimes, captive dependencies, what the container hides, when DI earns its weight |
| 4 | [Async, Await, and the Thread Pool](04-async-await-and-the-thread-pool.md) | The state machine, cancellation, what "async" actually buys you |
| 5 | [How ASP.NET Core Handles a Request](05-how-aspnet-core-handles-a-request.md) | The host, configuration, middleware, routing, model binding, filters |
| 6 | [Backend Beyond the Request](06-backend-beyond-the-request.md) | Hosted services, queue consumers, scheduled work, graceful shutdown |

### Workshop A

| # | Workshop | |
|---|---|---|
| 7 | [Trace a Request End-to-End](07-workshop-trace-a-request.md) | Constructive — build the integrative mental model |

### Topical (any order)

| # | Module | |
|---|---|---|
| 8 | [How Backend Code is Organized](08-how-backend-code-is-organized.md) | Layered architecture, repositories, mediators, cross-cutting concerns |
| 9 | [Entity Framework Core Without the Magic](09-entity-framework-core-without-the-magic.md) | What EF generates, change tracking, projections, the leaky abstraction |
| 10 | [API Design: Designing and Consuming Contracts](10-api-design.md) | REST, versioning, ProblemDetails, OpenAPI, idempotency, calling other services |
| 11 | [Validation and Trust Boundaries](11-validation-and-trust-boundaries.md) | Where to validate, what never to trust, layered defense |
| 12 | [Security Patterns](12-security-patterns.md) | Authentication vs authorization, claims, policies, OWASP mapped to .NET, secrets |
| 13 | [Error Handling](13-error-handling.md) | Exception philosophy, global handlers, ProblemDetails, what to catch and what not to |
| 14 | [Logging, Telemetry, and Observability](14-logging-telemetry-and-observability.md) | Structured logging, Application Insights, OpenTelemetry, the production gotchas |
| 15 | [Diagnosing Production Issues](15-diagnosing-production-issues.md) | The diagnostic loop, methodology under fire, symptom-to-cause-to-fix |
| 16 | [Resilience, Caching, and Distributed Systems Realities](16-resilience-caching-distributed-systems.md) | Retries, circuit breakers, idempotency, eventual consistency, the outbox pattern |
| 17 | [Architecture Decisions at System Scale](17-architecture-decisions-at-system-scale.md) | When to split, when to merge, sync vs async, when abstractions earn their weight |
| 18 | [Testing the Backend](18-testing-the-backend.md) | Unit vs integration, WebApplicationFactory, testability as a boundary check |

### Workshop B

| # | Workshop | |
|---|---|---|
| 19 | [AI-PR Review Workshop](19-workshop-ai-pr-review.md) | Evaluative — use the mental model to judge code |

### Appendices

| | Appendix | |
|---|---|---|
| A | [Glossary](appendix-a-glossary.md) | Every term defined, in the order you encounter it |
| B | [Cheat Sheets](appendix-b-cheat-sheets.md) | DI lifetimes, middleware order, EF pitfalls, error handling decision tree, logging dos and don'ts, useful telemetry queries |
| C | [Resources](appendix-c-resources.md) | Curated books, blogs, talks, and tools by module |
| D | [Onboarding to a Production .NET Service](appendix-d-onboarding.md) | Questions to ask, code to read, telemetry to look at when joining a project |

## Time commitment

Roughly **35 to 45 hours of reading** for the full course, plus the two workshops at 2 to 4 hours each. The foundation arc (Modules 0 to 6) is about 12 to 15 hours; the topical modules range from 1 to 3 hours each.

This is not meant to be done in one sitting. **One module per week, with a short team discussion after each, is the recommended cadence.** That puts the full course at about five months for a team — long, yes, but proportional to the bar of senior technical reasoning. If five months feels too long, pick one of the focused reading plans below instead.

## How to use this course

Several reading plans, depending on what you need.

### "I just joined a .NET project and need to be useful fast"

You don't need the whole course before you can contribute. You need enough to read existing code with comprehension and ask the right questions in your first PR review.

1. [Module 0 — What .NET Actually Is](00-what-dotnet-actually-is.md)
2. [Module 1 — C# for Backend Devs](01-csharp-for-backend-devs.md) (skim if already comfortable)
3. [Module 5 — How ASP.NET Core Handles a Request](05-how-aspnet-core-handles-a-request.md)
4. [Module 7 — Trace a Request End-to-End (Workshop A)](07-workshop-trace-a-request.md)
5. [Appendix D — Onboarding to a Production .NET Service](appendix-d-onboarding.md)

About **10 to 12 hours**. Enough to be useful in your first two weeks. Come back to the rest later.

### "I want to build the foundations properly"

The foundation arc is the conceptual scaffolding the rest of the course assumes. If you'll spend any time on this material, this is the part that compounds.

Read [Modules 0 through 7](00-what-dotnet-actually-is.md) in order, including Workshop A. About **15 to 20 hours**. After this you can read the topical modules in any order; let your current work set the priority.

### "I want to deepen my understanding beyond the foundations"

You've done the foundation arc. You are looking to go deeper on the abstractions and design decisions that distinguish workmanlike code from senior code.

1. [Module 8 — How Backend Code is Organized](08-how-backend-code-is-organized.md)
2. [Module 9 — Entity Framework Core Without the Magic](09-entity-framework-core-without-the-magic.md)
3. [Module 10 — API Design: Designing and Consuming Contracts](10-api-design.md)
4. [Module 17 — Architecture Decisions at System Scale](17-architecture-decisions-at-system-scale.md)
5. [Module 18 — Testing the Backend](18-testing-the-backend.md)

About **10 to 12 hours** after the foundations. The modules picked are the ones where senior judgment most distinguishes itself from junior implementation.

### "I want to reason deeply about production systems"

You've done the foundation arc. You want the operational and architectural muscle.

1. [Module 14 — Logging, Telemetry, and Observability](14-logging-telemetry-and-observability.md)
2. [Module 15 — Diagnosing Production Issues](15-diagnosing-production-issues.md)
3. [Module 16 — Resilience, Caching, and Distributed Systems Realities](16-resilience-caching-distributed-systems.md)
4. [Module 17 — Architecture Decisions at System Scale](17-architecture-decisions-at-system-scale.md)

About **8 to 10 hours** after the foundations. This is the path for the engineer who will be on call, in incident review, and in architecture decision conversations.

### "I want to get sharper at reviewing code I'll answer for"

You've done the foundation arc. You're looking to sharpen the reviewer's eye — particularly in the modules where AI-generated code (and, often, hurried human code) most reliably produces plausible-but-wrong shapes.

1. [Module 9 — Entity Framework Core Without the Magic](09-entity-framework-core-without-the-magic.md)
2. [Module 11 — Validation and Trust Boundaries](11-validation-and-trust-boundaries.md)
3. [Module 13 — Error Handling](13-error-handling.md)
4. [Module 14 — Logging, Telemetry, and Observability](14-logging-telemetry-and-observability.md)
5. [Module 19 — AI-PR Review Workshop](19-workshop-ai-pr-review.md)
6. [Appendix B — Cheat Sheets](appendix-b-cheat-sheets.md)

About **10 to 12 hours** after the foundations.

### "I'm leading a greenfield service design"

You've done the foundation arc. You are designing a new service from scratch and want the system-shaping decisions to be defensible.

1. [Module 2 — Boundaries, Contracts, and Concerns](02-boundaries-contracts-and-concerns.md) (re-read; this is the spine)
2. [Module 8 — How Backend Code is Organized](08-how-backend-code-is-organized.md)
3. [Module 10 — API Design: Designing and Consuming Contracts](10-api-design.md)
4. [Module 17 — Architecture Decisions at System Scale](17-architecture-decisions-at-system-scale.md)
5. [Module 16 — Resilience, Caching, and Distributed Systems Realities](16-resilience-caching-distributed-systems.md)

About **10 to 12 hours** after the foundations.

### Reference

Each module stands alone. Jump to the module that matches your current problem — once you have the foundations, you don't need to read in order.

### Cheat sheets

Print [Appendix B](appendix-b-cheat-sheets.md). Pin it next to your monitor.

### For managers introducing this to a team

A suggested approach: **one module per week, with a 30-minute team discussion after each**. The discussion is where the value compounds — people connect the material to real things they've seen in your codebase. Workshops A and B work well as group exercises; running Workshop A on a service the team owns is especially high-leverage.

Plan for the conversations, not just the reading. Shared vocabulary — the kind that makes PR reviews and incident calls move faster — only develops through talking. Reading alone is not enough.

## What no course can manufacture

Three honest limits worth naming so your expectations are calibrated:

1. **Pattern recognition at speed.** Seniors recognize anti-patterns in seconds; the course-trained reader will recognize them in minutes. That gap closes only with reps.
2. **Calibration under ambiguity.** Knowing which 60% of a problem to ignore so you can ship the 40% that matters. This requires having shipped under deadline several times, with consequences, and lived with the results.
3. **Communicating tradeoffs to non-engineers.** A different skill, adjacent to the course but not in it.

A graduate of this course is **senior-ready**: capable of senior-level reasoning when given time to think, with the conceptual scaffolding to make real reps compound. Cultivated taste — the perception that names what's wrong about a system before the articulation arrives — comes from years of being wrong about specific systems and being corrected by them. The course can't manufacture those corrections. It can prepare you to recognize them when they happen, externalize your reasoning in a form the practice can hold you to, and make the next year of real work compound rather than blur.

That is the honest contract. Don't expect to finish this and feel senior overnight. Expect to finish it and find that the next year of real work makes you visibly faster, sharper, and more useful — because you went into it with the conceptual scaffolding that lets the corrections take.
