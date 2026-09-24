# Module 23 — CQRS & MediatR
*Phase 5: .NET Architecture Patterns · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **CQRS is one decision — reads and writes have different requirements, so let them have different models — applied at whatever depth those requirements justify, one read path at a time; MediatR is an unrelated in-process dispatch library that most .NET teams happen to use to package their command and query handlers, and confusing the two is the single most common mistake made about this topic, in interviews and in production.**

That sentence has two halves, and the module is built around keeping them apart.

The first half is an *architectural pattern*. It starts as a principle about methods (Bertrand Meyer's Command–Query Separation, 1988: asking a question should not change the answer), becomes a principle about models (Greg Young's CQRS, around 2010: two objects where there was one), and ends as a family of designs that range from "queries use a different code path over the same tables" to "the write side is an event store and the read side is six purpose-built databases fed by projections." Every step along that range buys something specific — query speed, independent scaling, a model free of screen concerns, the ability to add a new view without touching the write side — and costs something specific: a second code path, then a second schema, then eventual consistency, then a projection pipeline that must be idempotent, ordered, monitored and rebuildable. The senior skill is not knowing CQRS; it's **choosing the lowest level that meets the requirement, per read path, and being able to say exactly what the next level would buy and cost.**

The second half is a *library and a packaging decision*. MediatR is, in its own words, a "simple, unambitious mediator implementation": an in-process request dispatcher with a pipeline around it. It is useful — pipeline behaviors are a genuinely good place for cross-cutting concerns — and it is not CQRS: you can do CQRS without it, and plenty of MediatR codebases have one model, one database and no separation at all. What changed in 2025–2026 is that MediatR became a commercial product, which turned a default into a procurement decision and forced a question that should always have been asked: **do we need a mediator at all, and if so, which one, and how do we keep our handlers independent of it?**

A mid-level candidate can define CQRS and draw two boxes labelled "write DB" and "read DB." A senior candidate can tell you why reads and writes diverge from first principles (shape, volume, consistency, scaling, security and change cadence differ), why most systems should stop at the first or second level, how a projection guarantees an exactly-once *effect* over an at-least-once *delivery*, why a projection that can't be rebuilt is a liability, how a user sees eventual consistency ("I saved it and it vanished") and the six ways to fix it, why a command handler must never decide anything based on the read model, what MediatR does under the hood on every `Send`, in which order pipeline behaviors must run and why the concurrency retry sits outside the transaction, why `INotification` is the wrong tool for integration events, what the MediatR licence actually requires and who is exempt, and how to migrate a codebase off it one vertical slice at a time.

This module closes threads that several earlier modules opened by name. **Module 20** (Concepts 34–35) introduced CQRS-lite as a layering decision — "the read side is allowed to know about the database" — and deferred the rest here, along with the MediatR question it raised in its current-platform section. **Module 21** (Concepts 38, 49) described read models and projections as the legitimate way to query across modules, listed MediatR as commercial in the in-process messaging table, and said "Module 23 takes this properly." **Module 22** (Concepts 83, 84, 88, 92) established that queries bypass the aggregate, that eventual consistency is a UX design problem, that the application service is load–decide–save, and that "whether the read side deserves a separate model, store or framework" is this module's question. **Module 11** gave you the outbox, delivery semantics and idempotent consumers — the machinery every asynchronous projection runs on. **Module 10** gave you caching as an asynchronous replica with weak consistency — which is exactly what a read model is. **Module 7** gave you the session guarantees (read-your-writes, monotonic reads) that Part E turns into design techniques. **Module 19** gave you EF Core's no-tracking queries, projections, compiled queries and interceptors — the read side's everyday tools. **Module 8** gave you replication lag and partitioning, which reappear here as read replicas and query-keyed partitions.

It shows up in five places in an interview loop: the **design round** (at step 5 of the 7-step framework — "reads outnumber writes 200 to 1 and the dashboard joins nine tables" — CQRS is usually the right move, and the interviewer is listening for the level you choose and whether you mention lag), the **deep technical round** ("how do you keep the read model consistent? what happens when the projector crashes halfway through a batch?"), the **code-review round** (a MediatR solution with a handler calling `mediator.Send` on another handler inside a transaction behavior is one of the most common things put in front of you), the **architect round** ("should we standardize on MediatR across twelve teams given the licence change?" is a build-vs-buy question with an ADR attached), and the **behavioural round** ("tell me about a time you introduced — or removed — architectural complexity").

**Current platform state (verified September 2026).** .NET 10 (November 2025) is the production-current LTS, supported to November 2028; .NET 11 RC1 shipped 8 September 2026 ahead of GA on 10 November 2026. What changed in this module's ecosystem, and why each item matters:

- **MediatR is a commercial product, and the version boundary is 13.0.0.** Jimmy Bogard announced the commercialization of MediatR and AutoMapper on 2 April 2025; the commercial editions launched on 2 July 2025 with **MediatR 13.0.0**, published by his company Lucky Penny Software. Current versions are licensed under the **Reciprocal Public License 1.5** (a copyleft licence whose reciprocal obligations most proprietary software cannot accept) **or** a commercial licence. The release line since then: **14.0** (December 2025 — .NET 10 support and signed packages), **14.1** (March 2026 — perpetual licensing, de-duplication of notification handlers), **14.2** (2 July 2026 — licence keys read from `MEDIATR_LICENSE_KEY` or `LUCKYPENNY_LICENSE_KEY` environment variables, a fix for a licence-validation deadlock in synchronous contexts, validation moved off the mediator's construction path). Versions **up to and including 12.5.0 remain Apache-2.0 forever** — without support or security fixes. `MediatR.Contracts` (the `IRequest`/`INotification`/`IStreamRequest` interfaces only) remains Apache-2.0. **Why it matters here:** the most common .NET CQRS starting point is now a licensing conversation, and the answer depends on facts you should be able to state precisely (Concept 74).
- **The licence terms are specific, and more lenient in enforcement than people assume.** A free **Community licence** covers individuals and organizations with under \$5M annual gross revenue that have never taken more than \$10M in outside capital — excluding government and quasi-government bodies and universities' institutional (non-teaching) software. Paid tiers are team-sized — Standard (1–10 developers), Professional (11–50), Enterprise (unlimited) — counting only developers with "programmatic access" (those who write, debug or compile code that calls the library). List prices on mediatr.io in September 2026 are \$499 / \$1,499 / \$3,999 per year for MediatR alone, and \$799 / \$2,399 / \$6,399 bundled with AutoMapper. **No licence is needed for development, CI, staging or other non-production environments.** Enforcement is **log messages only** — no licence server, no network call, no runtime limits — and deployed applications keep running after a subscription lapses; what lapses is the right to develop new applications on, or upgrade to, newer versions. **Why it matters here:** "it'll break production when the key expires" is wrong, and "we can ignore the warning" is a compliance decision, not a technical one.
- **Two widely copied templates split on the question.** `jasontaylordev/CleanArchitecture` still uses MediatR and prints the Lucky Penny licence warning at startup — a January 2026 GitHub discussion (#1413) asked whether that was intentional, and the maintainer has defended keeping it. `ardalis/CleanArchitecture` moved to **`Mediator`** — the MIT-licensed, source-generated library by Martin Othamar. And `dotnet/eShop`, Microsoft's reference application, pins `MediatR` **13.0.0** under a comment reading "Before license change," while Lucky Penny's licence explicitly applies to "MediatR version 13.0.0 and later." **Why it matters here:** even carefully maintained reference code gets the boundary wrong; an architect checks it rather than copying a pin.
- **The free alternatives matured at the same time.** **`Mediator`** (martinothamar) reached 3.0 stable (3.0.2, March 2026; 3.1 at release candidate) — MIT, compile-time generated dispatch, `ValueTask`-based, Native AOT-compatible, singleton lifetime by default, compile-time diagnostics for missing or duplicate handlers, and in the 3.1 line built-in OpenTelemetry metrics and tracing following the messaging semantic conventions. **Wolverine 6.0** (May 2026, part of the "Critter Stack 2026" release wave with Marten 9.0 and Polecat 4.0) is MIT, targets .NET 9/10, improved cold start, can be compiled with AOT compliance (with caveats), lets its Roslyn runtime compiler become a development-time-only dependency, and makes "no service location" the default; its `DurabilityMode.MediatorOnly` turns it into a pure in-process mediator. **FastEndpoints** (MIT; 8.3 stable, 8.4 in beta in September 2026) has a command bus with middleware and streaming commands that can be used standalone via `FastEndpoints.Messaging`. **Why it matters here:** you can now name at least three production-grade MIT options and explain the trade-off each makes (Concepts 76–80).
- **ASP.NET Core 10 absorbed some of the mediator pipeline's job.** Minimal APIs now have **built-in validation** (`builder.Services.AddValidation()`, source-generated, DataAnnotations-based, now in the `Microsoft.Extensions.Validation` package so it's usable outside HTTP), endpoint filters for cross-cutting concerns, and **`TypedResults.ServerSentEvents`** for pushing updates — including "your read model has caught up" — without WebSockets. **Why it matters here:** the "we need MediatR for validation" argument is weaker than it was; the read-your-writes push technique got cheaper (Concept 52).
- **Azure's data platform now does some projection work for you.** **Azure Cosmos DB Global Secondary Indexes** became generally available (announced at Build, 2 June 2026): read-only containers with a different partition key, maintained **asynchronously and eventually consistently** from the source container's change feed, with a propagation-latency metric to alert on — a platform-managed projection for the "same document, queried by a different key" case. The **all versions and deletes change feed mode** also became GA in June 2026, giving projectors deletes, TTL expirations and intermediate versions (it requires continuous backup and only retains changes for the backup retention period). **SQL Server 2025, Azure SQL Database and Managed Instance** gained **change event streaming (CES)** — DML changes streamed from the transaction log straight to Azure Event Hubs as CloudEvents — which is **still in preview** as of August 2026, doesn't snapshot existing data, doesn't emit DDL, and, since 15 August 2026, requires the Kafka protocol (AMQP deprecated) for new stream groups on Azure SQL Database. **Why it matters here:** "feed the read side from the database's own change stream" is now a first-party option on both of Azure's main databases — with specific limits you should know before proposing it (Concepts 37, 48).
- **Microsoft's guidance is consistent with the spectrum view.** The Azure Architecture Center's *CQRS pattern* distinguishes "separate models in a single data store" from "separate models in different data stores," recommends the transactional outbox and idempotent consumers for the latter, and lists "the domain or the business rules are simple" and "a simple CRUD-style UI is sufficient" as reasons *not* to use it. The *.NET Microservices* e-book's ordering service implements queries with Dapper against the same database, bypassing the domain model entirely — level 1 in this module's terms (Concept 24).

This module has nine jobs:

1. **Derive CQRS from first principles** — from CQS, through the asymmetries between reads and writes — so that you can argue for it, and against it, from reasons rather than slogans.
2. **Lay out the spectrum** from a separate query path to separate stores fed by projections, with what each level buys and costs, so you can choose a level per read path.
3. **Design the write side** — commands, validation, outcomes, idempotency, concurrency and async commands — as the home of intent and invariants.
4. **Design the read side** at each level, from EF Core projections and Dapper to replicas, caches and database-maintained views.
5. **Build projections that are correct** under at-least-once delivery, reordering, poison events and crashes — and that can be rebuilt.
6. **Make eventual consistency invisible, or at least honest,** to users: read-your-writes, monotonic reads, consistency tokens and UX.
7. **Understand MediatR properly** — its model, its internals, what pipeline behaviors are for, in which order they run, and its traps.
8. **Handle the licensing change like an architect** — the facts, the options, the alternatives, and a migration playbook.
9. **Take the architect's view** — where CQRS belongs, what it costs to operate, what reviewers look for, and how to discuss it in design and architect rounds.

Nine framings to carry through:

1. **CQRS is a spectrum, not a switch.** "Do we do CQRS?" is the wrong question; "which read paths deserve which level?" is the right one.
2. **Separating the models is cheap; separating the stores is what costs.** The first step is almost always worth it; each step after it needs a reason you can put a number on.
3. **The write side owns truth; the read side owns convenience.** Decisions are made on the write model, always. The read model is a hint, a cache, a view — never an input to an invariant.
4. **A read model is a cache with a build process.** Everything you learned about caches in Module 10 — staleness, invalidation, warm-up, stampedes — applies, plus the build process must be idempotent, ordered and rebuildable.
5. **Eventual consistency is a product decision with a number attached.** "Eventually" means nothing until someone says "within two seconds, 99% of the time."
6. **At-least-once delivery plus idempotent application equals exactly-once effect.** That's the entire correctness story of a projector, and every projector must be designed around it.
7. **A mediator is a packaging choice, not an architecture.** It changes how handlers are invoked and wrapped; it doesn't change what they do or how the system is shaped.
8. **Pipeline behaviors are where the value is, and order is semantics.** Moving the retry inside the transaction, or validation after it, changes what the system does.
9. **Your handlers should not know which dispatcher runs them.** If they do, the next licence change — or the next framework — is a rewrite instead of an afternoon.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | One decision, many depths | Different requirements deserve different models; depth is chosen per read path |
| 2 | CQS: where it starts | Meyer: asking a question should not change the answer |
| 3 | CQS's limits | `Pop`, `TryAdd`, server-generated IDs — the pragmatic exceptions and what they teach |
| 4 | From CQS to CQRS | Young: two objects where there was one — a pattern, not an architecture |
| 5 | Why reads and writes differ | Shape, volume, consistency, scaling, latency, security and change cadence |
| 6 | The single-model tax | What a shared model costs: getters for screens, `Include` chains, tracking, locks |
| 7 | What CQRS is not | Not event sourcing, not two databases, not eventual consistency, not MediatR |
| 8 | The CQRS spectrum | Levels 0–4, what each buys and what each costs |
| 9 | Commands, precisely | Intent, imperative, one handler, may be rejected, named in the language |
| 10 | Queries, precisely | No observable side effects, DTOs out, cacheable, safe to retry |
| 11 | Commands, queries, events | Three message kinds — who owns the contract, tense, cardinality |
| 12 | Can a command return a value? | Acknowledge, identify, version, outcome — never read data |
| 13 | Task-based UIs | Why intent-revealing commands and CQRS arrive together; where CRUD is fine |
| 14 | The write side's job | Protect invariants, record intent, emit facts — nothing else |
| 15 | Designing a command | A record of domain types, identity, idempotency key, expected version |
| 16 | Validation on the write side | Shape at the edge, rules in the aggregate, authorization around the handler |
| 17 | The command handler | Load, decide, save — one aggregate, one transaction |
| 18 | Command outcomes | Result types, closed hierarchies, and the HTTP mapping |
| 19 | Idempotent commands | Keys, natural idempotency, client IDs, result stored in the same commit |
| 20 | Concurrency through the command | Expected version in, new version out; `If-Match` and 412 |
| 21 | Synchronous vs asynchronous commands | 201/204 vs 202 + a status resource; queues for load leveling |
| 22 | Write sides without a domain model | Transaction scripts are valid CQRS in supporting subdomains |
| 23 | The read side's job | Answer questions in the consumer's shape — no rules, no tracking, no aggregate |
| 24 | Level 1: a separate query path | Query handlers or a query port over the same tables |
| 25 | EF Core on the read side | Projections, no-tracking, split and compiled queries, `SqlQuery<T>` |
| 26 | Dapper and SQL on the read side | When hand-written SQL is the right tool, and how to keep it honest |
| 27 | Database-maintained read models | Views, indexed views, materialized views |
| 28 | Purpose-built vs generic queries | One query per screen vs OData/GraphQL; the BFF angle |
| 29 | Paging, filtering and counting | Keyset over offset; the cost of `COUNT(*)`; search is its own read model |
| 30 | Caching as a read model | HybridCache, output caching, ETags — a read model with a TTL |
| 31 | Read replicas | Replica lag makes level 1 eventually consistent; routing in .NET |
| 32 | Rules don't leak into queries | Compute at write time or call the domain — never a second implementation |
| 33 | Authorization on the read side | Row filters, named query filters, audience-specific read models |
| 34 | What a projection is | A deterministic fold from facts to query-shaped state |
| 35 | Level 2: synchronous projections | Read tables updated in the write transaction — strong but costly |
| 36 | Level 3: asynchronous projections | A projector consumes facts and maintains read models later |
| 37 | Feeding projections | Outbox events, CDC, change event streaming, change feed, event store |
| 38 | Events vs CDC | Intent vs row diffs; schema coupling; the outbox-via-CDC hybrid |
| 39 | Ordering | Per-aggregate order is enough; version guards for the rest |
| 40 | Idempotency in projectors | At-least-once in, exactly-once effect out |
| 41 | Checkpoints and atomicity | Commit the checkpoint with the read model |
| 42 | Poison events and stuck projections | Dead letters, skip policies and alarms |
| 43 | Rebuilding projections | Replayable sources, shadow rebuilds, blue/green swaps |
| 44 | Evolving read-model schemas | Read models are disposable — version, rebuild, swap |
| 45 | Denormalization and fan-out | Copy at write time vs join at read time; write amplification |
| 46 | Multi-source read models | Composing several contexts' events; late and missing data |
| 47 | Choosing a read store | Relational, document, search, cache, columnar — by query shape |
| 48 | Cosmos DB on both sides | Partition by aggregate for writes, by query for reads; change feed; GSIs |
| 49 | Search indexes as read models | Azure AI Search: indexers vs push; freshness trade-offs |
| 50 | Measuring consistency | Lag as a metric, an SLO and an alert |
| 51 | The stale-read bug | "I saved it and it vanished" — why it happens, why users notice |
| 52 | Read-your-writes, six ways | Return, route, wait, project synchronously, optimistic UI, push |
| 53 | The consistency token | Version out of the command, minimum version into the query |
| 54 | Monotonic reads | Replicas, caches and load balancers can make time go backwards |
| 55 | Commands never trust the read model | Decide on the write model; carry the version for concurrency |
| 56 | Designing UX for lag | Pending states, confirmations, notifications |
| 57 | Deciding per query | Which reads may be stale, and for how long — a business answer |
| 58 | The mediator pattern vs MediatR | GoF mediator coordinates colleagues; MediatR dispatches requests |
| 59 | MediatR's model | Requests, notifications, streams, behaviors, processors, publishers |
| 60 | How dispatch works inside | Cached generic wrappers, DI resolution per call, behaviors composed in reverse |
| 61 | What MediatR buys | A uniform handler shape and one place for cross-cutting concerns |
| 62 | What MediatR costs | Indirection, runtime wiring, hidden coupling, AOT friction — and a licence |
| 63 | Pipeline behaviors | Logging, validation, authorization, idempotency, retries, transactions |
| 64 | Ordering the pipeline | Outermost to innermost — and why each concern sits where it does |
| 65 | The transaction behavior done right | Commands only, one commit, outbox inside, retry outside |
| 66 | The validation behavior | FluentValidation vs .NET 10 built-in validation; results vs exceptions |
| 67 | Behaviors for queries | Caching, timeouts, read-only routing |
| 68 | Notifications: the trap | In-process, synchronous, same scope — fine for domain events, wrong for integration |
| 69 | Handlers calling handlers | Nested pipelines, double transactions, hidden graphs |
| 70 | Result types through the pipeline | Returning failures without exceptions, generically |
| 71 | Testing with a mediator | Test handlers directly; don't mock the mediator |
| 72 | MediatR and vertical slices | Feature folders, one request per slice, duplication across slices |
| 73 | What happened | April 2025 announcement → July 2025 v13 → 14.x in 2026 |
| 74 | What the licence requires | RPL-1.5 or commercial; Community tier; who counts; what expiry means |
| 75 | Your four options | Pay, pin 12.5.0, migrate, or remove the mediator |
| 76 | No library: injection + decorators | Your own handler interfaces, Scrutor decorators, zero dependencies |
| 77 | Mediator (source-generated) | MIT, MediatR-shaped, compile-time wiring, AOT, singleton by default |
| 78 | Wolverine | Mediator + messaging + outbox; conventions over interfaces; `MediatorOnly` |
| 79 | Framework-native options | Endpoint filters, FastEndpoints' command bus, Brighter/Darker |
| 80 | The comparison | Licence, dispatch, AOT, pipeline, messaging, migration cost |
| 81 | The migration playbook | Insulate, move behaviors, swap, delete — one slice at a time |
| 82 | Do you need a mediator at all? | The honest decision framework |
| 83 | Scope: per context, per read path | Never system-wide |
| 84 | CQRS in Clean Architecture and VSA | Where commands, queries and ports live |
| 85 | Read models across modules and services | Consumer-owned composite views; API composition vs projection |
| 86 | CQRS and event sourcing | Event sourcing needs CQRS; CQRS doesn't need event sourcing |
| 87 | Operating a CQRS system | Lag dashboards, dead letters, rebuild runbooks, trace links |
| 88 | Security and privacy | Separate principals, audience-specific read models, erasure propagation |
| 89 | Testing CQRS | Given/when/then for commands; given-events/then-rows for projections |
| 90 | When a separate read store pays | Read:write ratios and join cost, in arithmetic |
| 91 | Projector throughput and rebuild time | Batching, partitions, and how many hours a rebuild takes |
| 92 | Lag budgets and backlog drain | Little's Law and capacity minus arrival rate |
| 93 | Fan-out arithmetic | What a denormalized name change costs |
| 94 | CQRS as a top-level architecture | Applied everywhere, justified nowhere |
| 95 | "We use MediatR, so we do CQRS" | A dispatch library is not an architectural pattern |
| 96 | Commands that read, queries that write | The violations and the legitimate exceptions |
| 97 | Deciding from the read model | Invariants checked against stale data |
| 98 | The ceremony stack | Generic repository + MediatR + AutoMapper + nine files per field |
| 99 | Async projections without a safety net | No idempotency, no lag metric, no rebuild |
| 100 | Notifications as integration events | Lost messages and partial side effects |
| 101 | The review catalogue | What a reviewer checks, in order |
| 102 | CQRS in the design round | Where it appears in the 7 steps, without jargon |
| 103 | CQRS in the architect round | Cost, teams, operations and the ADR |
| 104 | The 60-second answers | What CQRS is; when to use it; MediatR yes or no |
| 105 | The close | One paragraph that demonstrates judgment |

---

# Part A — What CQRS actually is

Most explanations of CQRS start with a diagram: a command box, a query box, two databases and an event bus between them. That's the *end* of the spectrum, not the definition, and starting there is why so many teams build the expensive version when they needed the cheap one. Part A starts from the principle, derives the pattern, and lays out the spectrum — so that every later decision in the module is a choice of level rather than a yes/no.

---

## Concept 1 — One decision, many depths

The whole of CQRS is one observation and one decision.

**The observation:** in most business systems, the operations that *change* state and the operations that *read* state have different requirements. Changes must enforce rules, serialize conflicting updates and record intent. Reads must be fast, shaped for a particular screen or consumer, cheap to scale and tolerant of a little staleness. A single model that tries to serve both is a compromise — too rich and too locked for reads, too flat and too permissive for writes.

**The decision:** let them have different models.

That's it. Everything else — separate query handlers, separate tables, separate databases, projections, event buses, eventual consistency — is a question of **how far** you take that decision, and each step further is a separate choice with its own justification.

The practical consequence is that CQRS is decided **per read path**, not per system. In one bounded context you might have:

- Commands going through aggregates (Module 22), with a single EF Core model for writes.
- An order-details screen served by an EF Core projection over the same tables — a separate *model* (a DTO and a query), same store.
- A customer dashboard served by a denormalized table maintained in the same transaction — a separate *schema*.
- A product search served by Azure AI Search, fed asynchronously — a separate *store*.
- An admin grid that just reads the entity table, because nobody will ever care.

That's five different read-side decisions in one context, each justified by what that particular read needs. The mistake is to treat "we do CQRS" as a system-wide switch — which leads either to over-building every read path or to under-building the one that matters.

The interview framing: *"I treat CQRS as a decision about each read path: how different is its shape from the write model, how much traffic does it take, and how stale can it be? Most reads get a separate query over the same tables. A few earn their own read model. Only the ones with a real scale or shape problem get their own store."*

---

## Concept 2 — CQS: where it starts

Bertrand Meyer introduced **Command–Query Separation** in *Object-Oriented Software Construction* (1988), as a principle for designing methods:

> Every method should either be a **command** that performs an action and changes state, or a **query** that returns data to the caller — but not both. Asking a question should not change the answer.

In C# terms:

```csharp
public sealed class ShoppingCart
{
    private readonly List<CartLine> _lines = [];

    // Query: returns data, no observable side effects. Call it once or a hundred times — same world.
    public Money Total => _lines.Aggregate(Money.Zero("EUR"), (sum, l) => sum.Add(l.LineTotal));
    public int ItemCount => _lines.Sum(l => l.Quantity.Value);

    // Command: changes state, returns nothing.
    public void Add(Sku sku, Quantity quantity, Money unitPrice) { /* ... */ }
    public void Remove(Sku sku) { /* ... */ }
}
```

Why Meyer cared, and why it still matters:

- **Queries become safe to call freely.** You can call `Total` in a debugger watch window, a log statement, an assertion, a retry loop or a cache without worrying that you've changed anything. A method that both returns a value and mutates state can't be called "just to look."
- **Reasoning becomes local.** If a method returns a value, you know it didn't change anything, so you can reorder, cache or eliminate calls. Referential transparency — the property functional programmers get from pure functions — is available for the query half of an object.
- **Contracts become checkable.** Meyer's Design by Contract relies on queries in preconditions and postconditions (`require Count < Capacity`); if evaluating a precondition changed state, assertions would change program behaviour.
- **Side effects become visible in signatures.** A `void` return type announces "this does something"; a non-void return type announces "this tells you something." The signature carries intent.

CQS is a *method-level* discipline. It says nothing about models, databases or architecture — that's CQRS. But every idea in CQRS is this one scaled up: *the part of the system that changes things and the part that reports on things are different kinds of thing, and treating them differently makes both simpler.*

---

## Concept 3 — CQS's limits, and what they teach

CQS is a principle, not a law, and knowing where it bends is part of understanding it. Four classic exceptions:

**1. `Stack.Pop()`.** Returns the top element *and* removes it. Splitting it into `Peek()` + `Remove()` is CQS-compliant — and wrong under concurrency, because another thread can pop between your peek and your remove. The combined operation exists because **the query and the command must be atomic.** The same logic gives you `ConcurrentDictionary.TryAdd`, `Interlocked.Increment` (returns the new value), `Queue.TryDequeue` and SQL's `UPDATE ... OUTPUT inserted.*`. The lesson: when a read and a write must happen as one atomic step, combining them is correct.

**2. Server-generated identity.** "Create an order and tell me its ID" looks like it must violate CQS. Mark Seemann's *CQS versus server generated IDs* (2014) shows it needn't: the **client** generates the identity (a GUID) and sends it with the command; the command returns nothing; the client already knows the ID. Module 22 (Concept 42) made the same move for idempotency reasons. The lesson: many apparent CQS violations are really *identity-assignment* problems, and client-assigned identity dissolves them.

**3. Returning an outcome.** A command that can fail for business reasons ("insufficient credit") has to communicate that somehow. Throwing is one way; returning an outcome is another, and it's arguably more honest (Concept 18). Returning `PlaceOutcome.InsufficientCredit` from `Order.Place` is not "returning data" in the CQS sense — it's the command's *result*, not a view of state.

**4. Observability side effects.** A query that writes a log line, increments a metric or records "last accessed at" has side effects. The first two are acceptable because they're *not observable through the model* — no other query returns a different answer because of them. The third *is* observable, and is a genuine design question (Concept 96).

What these teach: CQS's real test is not "does the method return a value?" but **"can the caller treat this as a pure observation?"** A query must be safe to call repeatedly with no change in any answer the system gives. A command may return information *about the command itself* — did it succeed, what identity or version did it produce — but not a view of the system's state. That distinction carries straight through to "can a command return a value?" (Concept 12).

---

## Concept 4 — From CQS to CQRS

Greg Young coined **Command Query Responsibility Segregation** around 2008–2010, building on conversations with Udi Dahan and others, and set it out in his *CQRS Documents* (2010). His one-line definition:

> CQRS is simply the creation of two objects where there was previously only one. The separation occurs based upon whether the methods are a command or a query.

That's CQS applied one level up. Instead of separating *methods* within an object, you separate *objects* — and, by extension, models: a write model whose methods are all commands, and a read model whose methods are all queries.

```
Before (one model):                      After (CQRS):
┌───────────────────────────┐            ┌────────────────────────┐   ┌─────────────────────────────┐
│ CustomerService           │            │ CustomerWriteService   │   │ CustomerReadService         │
│  void ChangeAddress(...)  │     →      │  void ChangeAddress()  │   │  CustomerDetails Get(id)    │
│  void Deactivate(...)     │            │  void Deactivate()     │   │  IReadOnlyList<CustomerRow> │
│  Customer Get(id)         │            │  void Register()       │   │     Search(criteria)        │
│  List<Customer> Search()  │            └────────────────────────┘   └─────────────────────────────┘
└───────────────────────────┘
```

Three things Young and Dahan were explicit about that the industry often forgot:

**1. It's a pattern, not an architecture.** Young: "CQRS is not an architecture, it's a pattern." Dahan: CQRS belongs inside a bounded context where it's justified, not across a whole system. It's in the same category as the Repository or the Strategy pattern — a local design decision — not in the category of "microservices" or "layered architecture."

**2. It enables things; it doesn't require them.** Once the models are separate, you *can* give the read side its own store, its own schema, its own scaling. You don't *have to*. Young's documents discuss separate stores and event sourcing because those are what the separation makes possible, and because they were the systems he was building — not because the pattern requires them.

**3. It's motivated by the write side as much as the read side.** The usual pitch is "faster queries," but Young's and Dahan's primary motivation was the **write model**: once it doesn't have to serve screens, it can be designed purely around behaviour and invariants — which is exactly where DDD's aggregates (Module 22) live comfortably. A domain model that has to expose every field for display tends to collapse into getters, setters and anaemia; a write-only domain model can be *behaviour-only*.

Martin Fowler's *CQRS* bliki entry (2011) adds the warning that should accompany every mention: *"for most systems CQRS adds risky complexity,"* and it should be used "on specific portions of a system (a BoundedContext in DDD lingo) and not the system as a whole." That's the sentence to quote when someone proposes CQRS for everything.

---

## Concept 5 — Why reads and writes differ

The pattern is only worth applying where reads and writes genuinely diverge. Knowing the dimensions of divergence lets you argue for it — or against it — with specifics. There are seven:

| Dimension | Write side | Read side |
|---|---|---|
| **Shape** | Normalized; organized around invariants and aggregates; one aggregate per transaction | Denormalized; organized around screens and consumers; often joins many aggregates or contexts |
| **Volume** | Usually low — tens to hundreds per second per context | Often 10–1,000× writes: every list, detail page, dashboard refresh and API poll |
| **Consistency** | Must be linearizable *within an aggregate* (Module 7): decisions on the latest state | Usually tolerates staleness — seconds is often fine, sometimes minutes |
| **Scaling** | Scales by partitioning (aggregates are serialization points, Module 22 C65) | Scales by replication and caching — copies are cheap because reads don't conflict |
| **Latency** | Tolerates tens of milliseconds; correctness first | Often needs single-digit milliseconds at high percentiles |
| **Security** | "Who may change this?" — per command, per aggregate state | "Who may see this?" — per row, per field, per audience |
| **Change cadence** | Invariants change when the business changes its rules — slowly | Screens, reports and API shapes change weekly — fast |

Each row is a reason the models drift apart:

- **Shape** is the most common and the most persuasive. A write model shaped for invariants (an `Order` aggregate with lines) is the wrong shape for "a customer's last 20 orders with product names, shipment status and invoice amounts" — that's four aggregates across possibly three contexts.
- **Volume** determines whether shape matters. A nine-table join that takes 40 ms is irrelevant at 2 reads per second and fatal at 5,000.
- **Consistency** is what makes separate stores *possible*: if reads tolerate two seconds of staleness, you can serve them from a copy.
- **Scaling** is why separate stores are *useful*: reads scale out almost for free once they don't have to go through the serialization points that writes do.
- **Change cadence** is the underrated one. If every new screen requires a change to the write model (a new navigation property, a new getter, a new `Include`), then the part of the system that should change least — the rules — is changing most. Separating the models protects the write model from the churn of the read side.

The flip side, which a senior answer includes: **if the dimensions don't diverge, don't separate.** A settings page that reads and writes the same six fields for the same user, a few times a day, has the same shape, trivial volume, no staleness tolerance question and no security asymmetry. A single model is correct there, and CQRS would be ceremony.

---

## Concept 6 — The single-model tax

What does it actually cost to serve reads from the write model? The symptoms are recognizable in almost any mature .NET codebase that didn't separate them:

**1. The write model grows getters and navigations for screens.** An `Order` aggregate gains a `Customer` navigation property "so the order page can show the customer name," which silently lets order use cases load and modify the customer (Module 22, Concept 61). It gains a `ShipmentStatus` property copied from fulfilment "for the list page." Each addition is small; together they turn a behaviour-focused aggregate into a data bag with methods.

**2. Queries load whole aggregates to use three fields.** Rendering a list of 50 orders loads 50 `Order` aggregates, each with its lines (via `AutoInclude`, Module 22 Concept 89), each tracked by the change tracker, then maps them to DTOs and throws most of it away. At low volume nobody notices; at scale it's the dominant cost.

```csharp
// The single-model version: loads, tracks and materializes whole aggregates to show a list
var orders = await db.Orders
    .Include(o => o.Lines)                        // lines not even displayed
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Take(50)
    .ToListAsync(ct);                             // tracked: snapshot per entity, identity map
return orders.Select(o => mapper.Map<OrderSummaryDto>(o)).ToList();

// The read-side version: one SQL statement, only the columns needed, no tracking, no mapping layer
return await db.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Take(50)
    .Select(o => new OrderSummaryDto(o.Id, o.Status, o.Total.Amount, o.Total.Currency, o.Lines.Count, o.CreatedAt))
    .ToListAsync(ct);
```

**3. Mapping layers multiply.** Entity → application DTO → API response, often via AutoMapper, with configuration that nobody fully understands and that fails at runtime (Module 20, Concept 35's "your DTO inventory shrinks" is the cure).

**4. Include chains and N+1.** Screens want data from many related entities; the ORM-shaped path to them is `Include().ThenInclude()`, lazy loading, or a loop of queries (Module 19). Each is a performance problem the read side wouldn't have if it just wrote the join it needed.

**5. Locking and contention.** Reads through the transactional model can take shared locks (under `READ COMMITTED` without row versioning on SQL Server, Module 12), and long report queries against the OLTP tables compete with writes for I/O and buffer pool.

**6. Security leaks through the model.** When the same entity serves every audience, every field is exposed to every screen unless someone remembers to exclude it in a mapping. Read models per audience make exposure explicit.

**7. Change coupling.** Every new screen touches the domain model or its mapping. The part of the code that should be most stable becomes the most frequently edited.

None of this means a single model is wrong everywhere — for simple, low-volume, same-shape data it's right (Concept 5). It means that when you see these symptoms, you are paying a tax that the cheapest level of CQRS (Concept 24) removes almost for free.

---

## Concept 7 — What CQRS is not

Interviewers probe this because the misconceptions are so common, and each one leads to an expensive design. CQRS is **not**:

**1. Event sourcing.** Event sourcing (Module 24) means storing state as a sequence of events. CQRS means separating read and write models. They're independent: you can do CQRS with a plain relational write model (most CQRS systems do), and while event sourcing *practically requires* CQRS (you can't query an event stream by attributes, so you need projections), CQRS doesn't require event sourcing. Oskar Dudycz's *CQRS facts and myths explained* makes this its first point.

**2. Two databases.** Separate *models* can live in one database — even over the same tables (Concept 24). Separate stores are one option at the far end of the spectrum.

**3. Eventual consistency.** Eventual consistency only arrives when the read model is updated *asynchronously* (levels 3–4). At levels 1 and 2 reads are exactly as consistent as the database makes them.

**4. Messaging, queues or a service bus.** Commands can be method calls. Projections can be updated in-process or in the same transaction. A broker is needed only when you choose asynchronous distribution.

**5. MediatR.** A dispatch library is a packaging choice (Part F). Plenty of MediatR codebases have one model and no separation; plenty of CQRS systems have no mediator.

**6. Commands and queries as separate HTTP endpoints.** `POST` vs `GET` is CQS at the protocol level; it's necessary for CQRS but not sufficient. If both endpoints use the same entity class and the same repository, there's no segregation of models.

**7. Microservices, or a "command service" and a "query service."** Splitting reads and writes into separate *deployments* is a distribution decision (Module 21) and is rarely justified: they share a model's language and the team that owns it, and splitting them creates exactly the lockstep deployment Module 21 warned about. It happens — a read API scaled independently from the write API — but it's the far end of the far end.

**8. A top-level architecture.** It's a pattern within a bounded context (Concept 4; Concept 83).

**9. Always asynchronous commands.** Commands can be synchronous and return an outcome immediately (Concept 21). Queuing commands is a separate choice.

The interview line: *"CQRS itself is small: separate models for reads and writes. Separate databases, eventual consistency, event sourcing and buses are all options it enables — and each one is a separate decision I'd need a reason for."*

---

## Concept 8 — The CQRS spectrum

Vladimir Khorikov's *Types of CQRS* (2015) described three types beyond "no CQRS"; extending it to the event-sourced end gives five levels. This table is the backbone of the module — every later part is about operating at one of these levels.

| Level | Name | Write side | Read side | Consistency of reads | What it buys | What it costs |
|---|---|---|---|---|---|---|
| **0** | No CQRS | One model | Same model, via repositories/entities | Strong | Nothing to build | The single-model tax (Concept 6) |
| **1** | Separate query path | Domain model / aggregates | Query handlers or query port projecting DTOs from the **same tables** (EF projection, Dapper, SQL) | Strong (same DB, same tables) | Write model free of screen concerns; fast, purpose-built queries; fewer mapping layers | A second code path; SQL that knows the schema |
| **2** | Separate read schema, same store | Aggregates | **Read tables/views** in the same database, updated in the **same transaction** (or maintained by the DB: indexed/materialized views) | Strong (same transaction) | Precomputed joins and aggregates; read-optimized schema; cheap reads | Write latency and contention grow; projection code runs in the write path |
| **3** | Separate read store, async projections | Aggregates + outbox | **Separate store(s)** — another database, Cosmos container, search index, cache — maintained by **projectors** consuming events or changes | **Eventual** — lag of ms to minutes | Independent scaling and technology; read models per consumer; isolation of read load from writes | Lag and its UX; idempotent, ordered, monitored, rebuildable projectors; more infrastructure |
| **4** | Event-sourced write side | Event store is the source of truth (Module 24) | Projections from the event stream — inline or async | Strong for inline projections, eventual for async | Full history; any new read model can be built from the past; audit | Everything in level 3 plus event-schema evolution, snapshots, and a different way of thinking |

Four things to notice:

**The jump from 2 to 3 is the big one.** Levels 1 and 2 keep a single database and a single transaction, so there's no new consistency model and no new failure mode — just more code. Level 3 introduces a distributed-systems problem (Module 11's delivery semantics, Module 7's session guarantees) that must be designed for. Most of Parts D and E exists because of that jump.

**Level 1 is almost always worth it.** It's the CQRS-lite of Module 20 (Concepts 34–35) and the "reads bypass the aggregate" of Module 22 (Concept 92). It costs a second code path and removes most of the single-model tax. Khorikov's own view is that his type 1 (separate classes, same data) is sufficient for most enterprise applications.

**Level 2 is underused.** A read table maintained in the same transaction gives you precomputed, denormalized reads with *no* eventual consistency. Its cost is write-path latency and contention on shared summary rows — which is fine at modest write rates and becomes a problem exactly when level 3 starts to pay (Concept 35).

**Levels coexist.** The same context can serve one screen at level 1, a dashboard at level 2 and search at level 3 (Concept 1). The level belongs to the read path, not the system.

The senior sentence: *"I'd default to level 1 — queries projecting straight from the write tables — and move a specific read path up only when I can name what it buys: a join that's too expensive at the read rate, a store that answers the query better, or read load I need to isolate. Level 3 is where eventual consistency starts, so that's the step I'd want a number for."*

---

## Concept 9 — Commands, precisely

A **command** is a request to change the system's state, expressing an **intent**, addressed to **exactly one handler**, which may **accept or reject** it.

Each part of that definition has a consequence:

- **Intent, named in the ubiquitous language, in the imperative.** `PlaceOrder`, `CancelReservation`, `AcceptLiability`, `ChangeDeliveryAddress`. Not `UpdateOrder`, not `SaveCustomer`, not `SetStatus`. The name is the business operation, and the name is what a domain expert would recognize (Module 22, Concept 5). If you can't name a command without "update" or "set," the operation probably doesn't have a business meaning yet — see Concept 13.
- **Exactly one handler.** A command is addressed: *somebody* is responsible for deciding whether it happens. If two handlers could both act on it, it's either two commands or an event. That "one handler" rule is why mediators throw when a request has zero or multiple handlers.
- **May be rejected.** A command is a *request*. The handler checks it against the current state and the rules, and may say no — "the order has already shipped," "insufficient stock." Rejection is a normal, expected outcome, not an error in the system (Concept 18).
- **Changes state — usually of one aggregate.** A command handled by the write side changes one consistency boundary (Module 22, Concept 59). A command that must change several aggregates atomically is a design smell or a deliberate exception.
- **Carries what the decision needs.** The command contains the user's input and the identifiers of what it targets — not the state of the system (the handler loads that), and not a DTO copied from a form with every field.

In code, a command is a small immutable record, built from domain types once the edge has parsed the raw input:

```csharp
public sealed record ChangeDeliveryAddress(
    OrderId OrderId,
    Address NewAddress,           // a value object — already validated at the edge (Module 22, Concept 48)
    int ExpectedVersion,          // optimistic concurrency (Concept 20)
    IdempotencyKey Key);          // safe retries (Concept 19)
```

Two distinctions that often come up:

**Command vs request DTO.** The HTTP body (`ChangeDeliveryAddressRequest { string Street; string City; ... }`) is a transport contract; the command is an application contract made of domain types. The edge translates one into the other — that translation is where format validation lives.

**Command message vs command object.** A command can be dispatched in-process (a method call, a mediator `Send`) or sent as a message over a queue (Concept 21). The *concept* is the same; the delivery semantics differ — a queued command needs idempotency and a way to report its outcome asynchronously.

---

## Concept 10 — Queries, precisely

A **query** is a request for information that **does not change any state observable through the system**, answered with **data shaped for the caller**.

Consequences:

- **No observable side effects.** Calling a query twice returns the same answer (given no intervening commands) and changes nothing any other query could see. Logging, metrics and tracing are fine; writing "last viewed at" is not a pure query (Concept 96).
- **Safe to retry, cache, prefetch and parallelize.** Because queries don't change anything, infrastructure can retry them on timeout, cache them, run them against replicas, and fan them out in parallel. HTTP's `GET` semantics (safe and idempotent) exist for precisely this reason.
- **Returns DTOs, not domain objects.** The result is a data structure for the consumer: flat, serializable, possibly combining several aggregates, with no behaviour. Returning an aggregate from a query exposes the write model to the read side's needs and invites callers to modify it.
- **No business rules — but may apply presentation logic and authorization.** Formatting, sorting, filtering and security trimming are the read side's job. Deciding whether something is *allowed* is not (Concept 32).
- **Named for what it answers.** `GetOrderDetails`, `SearchOrders`, `GetCustomerDashboard`, `ListOverdueInvoices` — named for the question, often for the screen or consumer.

```csharp
public sealed record GetOrderDetails(OrderId OrderId, int? MinVersion = null);   // MinVersion: Concept 53

public sealed record OrderDetailsDto(
    Guid OrderId, string Status, decimal Total, string Currency,
    string CustomerName, IReadOnlyList<OrderLineDto> Lines, int Version);
```

A useful mental test: *could this query be answered from a copy of the data that's a few seconds old?* If yes, it's a normal query, and it could move to any level of the spectrum. If no — if the answer must reflect the very last write — it's either a **read-your-writes** situation (Part E) or a sign that the "query" is really part of a command's decision and belongs on the write side.

---

## Concept 11 — Commands, queries and events

Three message kinds, often confused. Module 11 covered them as messaging concepts; here's the CQRS view:

| | **Command** | **Query** | **Event** |
|---|---|---|---|
| Meaning | "Please do X" — an intent | "Tell me Y" — a question | "X happened" — a fact |
| Tense | Imperative: `PlaceOrder` | Interrogative: `GetOrderDetails` | Past: `OrderPlaced` |
| Receivers | Exactly one | Exactly one | Zero, one or many |
| Can be refused? | Yes | No (can fail, but doesn't change anything) | No — it already happened |
| Changes state? | Yes (the receiver's) | No | No — but receivers may change their own state in response |
| Contract owned by | The **receiver** (it defines what it accepts) | The **receiver** | The **publisher** (it defines what it announces) |
| Coupling | Sender knows the receiver's intent | Asker knows who answers | Publisher doesn't know who listens |
| Typical result | Outcome (accepted/rejected, ID, version) | Data (DTO) | None |

The "contract owned by" row matters architecturally. When Ordering sends `CreateInvoice` to Billing, Ordering is coupled to Billing's intent — it knows Billing exists and what it does. When Ordering publishes `OrderPlaced` and Billing subscribes, the coupling is inverted: Billing depends on Ordering's published language, Ordering knows nothing about Billing (Module 21, Concept 39; Module 22, Concept 83). In CQRS, **commands flow into the write side, events flow out of it, and projections consume the events to build the read side.** Queries never touch the write side at all.

```
      commands                                   events                               queries
UI ─────────────▶ [ write side: handlers → aggregates ] ─────────▶ [ projectors ] ─▶ [ read models ] ◀──── UI
                                                     (outbox)
```

That diagram is level 3. At level 1, the "events" arrow and the projectors disappear and the read side queries the write side's tables directly. At level 2, the projectors run inside the write transaction.

---

## Concept 12 — Can a command return a value?

The most-asked CQRS question, and the one where dogma does the most damage. The strict CQS answer is "no." The useful answer is **"yes — information about the command, not about the system's state."**

What a command may legitimately return:

| Return | Why it's fine | Example |
|---|---|---|
| **Acknowledgement / outcome** | It's the result of the command, not a view of state | `Accepted`, `Rejected(InsufficientCredit)`, `NotFound` |
| **Identity of what it created** | The caller needs it to refer to the new thing (though client-generated IDs remove the need — Concept 3) | `OrderId` |
| **The new version / position** | Enables optimistic concurrency on the next command and read-your-writes on the next query (Concepts 20, 53) | `Version = 17` |
| **Minimal data the command itself produced** | Values computed by the command that the caller can't know otherwise — a generated reference number, a reservation expiry | `ClaimNumber = CLM-2026-004211`, `ExpiresAt` |

What a command should **not** return:

- **The full updated object or a view model.** "Place order and return the order details page" couples the command to a screen, drags read concerns into the write path (loading customer names, product descriptions), and stops being true the moment you move that screen's read model to level 3.
- **Data about other things.** "Accept liability and return the claimant's other open claims" — that's a query that happens to follow the command. Make it one.

Oskar Dudycz's *Can command return a value?* (2021) lands in the same place, with a useful framing: the HTTP layer is a translation layer between protocol and application. `POST /orders` can return `201 Created` with a `Location` header and even a response body — the *endpoint* may run the command and then a query — while the command handler itself returns only the ID and version. The protocol's convenience doesn't have to dictate the application's contract.

```csharp
public sealed record PlaceOrderResult(PlaceOutcome Outcome, int? NewVersion);   // outcome + version, nothing else

app.MapPost("/orders/{id}/place", async Task<Results<Ok<PlacedResponse>, NotFound, Conflict<ProblemDetails>>>
    (OrderId id, [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
     PlaceOrderHandler handler, CancellationToken ct) =>
{
    var result = await handler.Handle(new PlaceOrder(id, new IdempotencyKey(idempotencyKey)), ct);
    return result.Outcome switch
    {
        PlaceOutcome.Placed             => TypedResults.Ok(new PlacedResponse(id, result.NewVersion!.Value)),
        PlaceOutcome.NotFound           => TypedResults.NotFound(),
        PlaceOutcome.InsufficientCredit => TypedResults.Conflict(Problem("insufficient-credit")),
        _                               => TypedResults.Conflict(Problem(result.Outcome.ToString())),
    };
});
```

(`Problem(...)` stands for a small helper that builds `ProblemDetails` with a type URI — Concept 18.)

The interview answer: *"A command can return information about itself — whether it was accepted, the ID it created, the new version — because the caller needs those to continue. It shouldn't return a view of the system; that's a query, and keeping it separate means the read side can change or move without touching the command."*

---

## Concept 13 — Task-based UIs

CQRS and **task-based user interfaces** tend to arrive together, and understanding why explains a lot about when CQRS pays.

A **CRUD UI** presents an entity as a form: every field editable, one Save button. The resulting request is "here is the new state of the customer" — and the only command you can derive from it is `UpdateCustomer`. The system can't tell *why* the address changed (moved house? typo?), can't apply different rules to different changes, and can't emit meaningful events (Module 22, Concept 79's `CustomerRelocated` vs `CustomerAddressCorrected`).

A **task-based UI** presents the operations the user can perform: *Relocate customer*, *Correct name*, *Change credit limit*, *Deactivate*. Each is a command with its own small form, its own validation, its own authorization and its own event.

```
CRUD:                                    Task-based:
┌─────────────── Edit customer ───┐      ┌──── Customer: Ana Petrović ─────────────────┐
│ Name      [Ana Petrović      ]  │      │  Address: Knez Mihailova 12, Belgrade        │
│ Street    [Knez Mihailova 12 ]  │      │   [Customer moved]  [Correct a typo]         │
│ City      [Belgrade          ]  │      │  Credit limit: €5,000   [Request change ▸]   │
│ Credit    [5000              ]  │      │  Status: Active         [Deactivate…]        │
│ Active    [x]                   │      └──────────────────────────────────────────────┘
│                    [ Save ]     │
└─────────────────────────────────┘
```

Why they go together with CQRS:

- **Commands need intent.** A write model built on behaviour (Module 22) can't do anything useful with "here's a new blob of state." Task-based UIs produce intent-revealing commands naturally.
- **The read side is free to show everything.** The screen that *displays* the customer shows whatever combination of data is useful — from several aggregates and contexts — because it's a read model. The screen doesn't have to mirror the write model's structure, and the write model doesn't have to expose everything the screen shows.
- **Commands become small and conflict less.** Two users changing different aspects of the same customer (one the address, one the credit limit) issue different commands that don't overwrite each other's fields. A CRUD Save sends the whole state and silently overwrites the other user's change unless you add concurrency checks.

The Azure Architecture Center's CQRS guidance makes the same point with a hotel example: use a command like "Book hotel room," not "Set ReservationStatus to Reserved."

Where CRUD is fine — and a senior answer says so: **supporting subdomains and reference data** (Module 22, Concept 8). An admin screen for maintaining a list of countries, product categories or notification templates has no rules worth expressing as tasks, and a form with a Save button is the right UI. Forcing a task-based design onto it is ceremony. The rule: *task-based where the business has rules about how things change; CRUD where it doesn't.*

---

# Part B — The write side

The write side is where intent enters the system and where the business's rules are enforced. Module 22 designed the aggregates; Module 20 placed the application layer; this part designs the thing that sits between an HTTP request and an aggregate method — the command and its handler — with the properties a production write path needs: layered validation, honest outcomes, idempotency, concurrency, and a choice between answering now and answering later.

---

## Concept 14 — The write side's job

The write side has exactly three responsibilities, and a useful review question is whether each piece of code on it serves one of them:

1. **Protect invariants.** Decide, against the *current* authoritative state, whether a requested change is allowed — and make it atomically if it is. This is the aggregate's job (Module 22, Part F), orchestrated by the handler.
2. **Record intent.** Capture *what the user meant* — not just the resulting state. That's why commands are named for business operations (Concept 9) and why they produce intent-revealing events (Module 22, Concept 79).
3. **Emit facts.** Publish what happened, reliably, so that everything downstream — read models, other aggregates, other contexts — can react. That's the outbox (Module 11) written in the same transaction as the state change.

What the write side is **not** for:

- **Serving screens.** No "load the order and its customer and its shipments so the UI can render them." That's a query.
- **Computing views.** No "recalculate the dashboard totals" inside the command handler unless you've *chosen* a level-2 synchronous projection (Concept 35) and accepted its cost.
- **Talking to the outside world directly.** No emails, no HTTP calls to partners, no broker publishes inside the transaction (Module 22, Concept 82). Those happen after commit, from the outbox.

The shape that falls out is a narrow, predictable pipeline:

```
HTTP / message ─▶ edge: parse & shape-validate ─▶ command ─▶ [authorize] ─▶ [idempotency] ─▶ handler
                                                                                                │
                                   load aggregate (repository) ◀────────────────────────────────┘
                                   decide (aggregate method)
                                   save (unit of work: state + domain events → outbox rows, one commit)
                                                                                                │
                                   outcome (accepted/rejected, id, version) ◀──────────────────┘
```

That's the whole write side. When one grows much beyond it, something from the read side or the integration side has usually leaked in.

---

## Concept 15 — Designing a command

A well-designed command is a small, immutable record of **domain types** that carries exactly what the decision needs. The checklist:

| Element | Include? | Why |
|---|---|---|
| **Target identity** | Always | Which aggregate the command is about — as a strongly-typed ID (`OrderId`, not `Guid`) |
| **The user's input** | Always | The parameters of the intent, parsed into value objects at the edge (`Address`, `Money`, `Quantity`) |
| **New identity** (for creation) | Usually | Client-generated ID for idempotent creation and CQS-friendly results (Concept 3; Module 22, Concept 42) |
| **Expected version** | When the user acted on something they saw | Optimistic concurrency against stale screens (Concept 20) |
| **Idempotency key** | When the caller may retry | Safe retries across timeouts (Concept 19) |
| **Actor** | When the decision depends on who's asking | Authorization and audit — taken from the authenticated principal by the edge, never from the request body |
| **Correlation / causation IDs** | In the envelope, not the payload | Tracing a command through events and projections (Concept 87) |
| **State of the system** | **Never** | The handler loads current state; a command carrying "the current balance is 500" is trusting the client |
| **Every field from the form** | **Never** | A command is an intent, not a DTO passthrough |

```csharp
// Good: intent-revealing, domain-typed, carries only what the decision needs
public sealed record RelocateCustomer(
    CustomerId CustomerId,
    Address NewAddress,
    DateOnly EffectiveFrom,
    int ExpectedVersion,
    IdempotencyKey Key);

// Smells:
public sealed record UpdateCustomer(                 // "Update" — no intent
    Guid Id,                                          // primitive identity
    string? Name, string? Street, string? City,       // optional fields = several commands in a trench coat
    decimal? CreditLimit, bool? IsActive,             // flags that change which rules apply
    decimal CurrentBalance);                          // client-supplied state
```

The second record is the output of a CRUD form (Concept 13). The handler for it has to work out which of five operations the caller meant from which fields are non-null, apply different rules to each, and trust the client's view of the balance. Every one of those is a bug waiting to happen.

Two design questions that come up in reviews:

**Where does the actor come from?** From the authenticated principal, attached at the edge or by a pipeline step, never from the request body. Whether it travels *inside* the command (`PlaceOrder(..., UserId By)`) or via an ambient `ICurrentUser` is a style choice; inside the command is more explicit and easier to test, and it means queued commands (Concept 21) carry the actor with them.

**Should commands be "PATCH-shaped" for flexibility?** No. `ChangeCustomerDetails { Name?, Phone?, Email? }` looks flexible and is the same smell as `UpdateCustomer`. If the business treats changing an email differently from changing a phone number (verification, notifications), they're different commands. If it doesn't, one command with all required fields is simpler than one with optional fields.

---

## Concept 16 — Validation layers on the write side

Module 22 (Concepts 48 and 94) distinguished input validation from invariants. On the write side of a CQRS system there are six kinds of check, each with one correct home, one failure mode and one HTTP status. Putting a check in the wrong layer is a common review finding.

| Check | Question | Home | Failure → HTTP |
|---|---|---|---|
| **Shape** | Is the request well-formed? Required fields, lengths, formats, ranges | Edge: model binding + validation (.NET 10 `AddValidation`, FluentValidation) and value-object parsing | 400 with field errors |
| **Authentication / authorization** | Who is asking, and may they issue this command at all? | Endpoint auth policies + an authorization step before the handler | 401 / 403 |
| **Existence** | Does the target exist? | Handler (repository returns null) | 404 |
| **Invariants** | Is this change allowed given the aggregate's current state? | **Aggregate** method | 409 or 422 with a domain reason |
| **Concurrency** | Is the caller acting on the current version? | Handler / unit of work (Concept 20) | 412 (human edits) or retry (machine) |
| **Set-based / cross-aggregate** | Does this violate a rule over many aggregates (uniqueness, capacity)? | Unique constraint, reservation, registry aggregate (Module 22, Concept 70) | 409 |

Three rules that follow:

**1. Invariants are never checked in a validator that runs before the handler.** A FluentValidation rule like `RuleFor(x => x.Email).MustAsync(BeUniqueEmail)` looks tidy and is wrong twice over: it races (two requests both pass the check, both insert), and it queries *some* view of the data outside the transaction — often the read model, which may be stale (Concept 97). Use it, if at all, as a *friendly early message*, and keep the database constraint as the actual guard.

**2. Authorization that depends on aggregate state belongs in the aggregate or the handler, not a generic step.** "Only the claim's assigned adjuster may accept liability" needs the loaded claim. A pipeline authorization step can check "is this user an adjuster?"; the handler or aggregate checks "is this user *this claim's* adjuster?"

**3. Shape validation lives in one place — the value object — and the edge calls it.** `EmailAddress.TryParse` is the rule; the endpoint (or a validation step) calls it and maps failure to a 400. Duplicating the regex in a FluentValidation rule and in the value object guarantees they drift.

---

## Concept 17 — The command handler

Module 22 (Concept 88) gave you the application service's shape. Here it is as a CQRS command handler, with every line accounted for:

```csharp
public sealed record RelocateCustomer(
    CustomerId CustomerId, Address NewAddress, DateOnly EffectiveFrom, int ExpectedVersion, IdempotencyKey Key);

public abstract record RelocateOutcome
{
    public sealed record Relocated(int NewVersion) : RelocateOutcome;
    public sealed record NotFound : RelocateOutcome;
    public sealed record StaleVersion(int CurrentVersion) : RelocateOutcome;
    public sealed record Rejected(string Reason) : RelocateOutcome;
}

public sealed class RelocateCustomerHandler(
    ICustomerRepository customers,
    IUnitOfWork unitOfWork,
    TimeProvider clock) : ICommandHandler<RelocateCustomer, RelocateOutcome>
{
    public async Task<RelocateOutcome> Handle(RelocateCustomer cmd, CancellationToken ct)
    {
        // 1. Load exactly one aggregate — the one the command targets.
        var customer = await customers.GetAsync(cmd.CustomerId, ct);
        if (customer is null) return new RelocateOutcome.NotFound();

        // 2. Concurrency: the user decided based on what they saw.
        if (customer.Version != cmd.ExpectedVersion)
            return new RelocateOutcome.StaleVersion(customer.Version);

        // 3. Decide — the rules live in the aggregate, not here.
        var decision = customer.Relocate(cmd.NewAddress, cmd.EffectiveFrom, clock.GetUtcNow());
        if (decision is Customer.RelocationDecision.Refused refused)
            return new RelocateOutcome.Rejected(refused.Reason);

        // 4. Save — one commit: state + domain events → handlers → outbox rows (Module 22, Concept 81).
        await unitOfWork.SaveChangesAsync(ct);
        return new RelocateOutcome.Relocated(customer.Version);
    }
}
```

The rules, each with the reason:

- **One aggregate loaded and modified.** The transaction boundary is the consistency boundary (Module 22, Concept 56). If the handler loads a second aggregate, it should be to *read* a value the decision needs (Module 22, Concept 51), not to change it — unless you've chosen and documented a Module 22 (Concept 63) exception.
- **No query-side code.** The handler doesn't load customer names for a response, doesn't build a DTO, doesn't touch read models (unless a level-2 projection is deliberately in play — and even then that happens in a domain-event handler during save, not here).
- **No business rules.** The handler orchestrates; "may this customer relocate?" is decided in `customer.Relocate`. The test: delete the handler and rebuild it from scratch — if a rule could be forgotten, it's in the wrong place.
- **One commit.** `SaveChangesAsync` is called exactly once. If you find a second call, something is trying to be two transactions.
- **Returns an outcome, not data** (Concept 12).
- **Nothing irreversible before the commit.** No emails, no HTTP calls, no broker publishes. That's what makes it safe to retry the whole handler on a concurrency conflict (Concept 20; Module 22, Concept 69).

Notice what's *not* in the handler: logging, validation of shape, authorization, idempotency and transaction management. Those are cross-cutting concerns that belong in a pipeline around the handler — which is the entire case for pipeline behaviors or decorators (Part F).

---

## Concept 18 — Command outcomes

A command can end in several ways, and how you represent them determines how honest your API is. The two schools:

**Exceptions for everything.** The handler throws `NotFoundException`, `DomainException`, `ConcurrencyException`; a global exception handler maps types to status codes. Simple to write, and it keeps signatures clean. The costs: business outcomes are invisible in the signature (callers must know which exceptions to expect), exceptions are expensive enough to matter under load (Module 17), and "the order already shipped" — an entirely expected outcome — is modelled as an exceptional condition.

**Results for business outcomes, exceptions for broken programs.** Module 22 (Concept 95) set out this rule. Expected outcomes — not found, rejected by a rule, stale version — are values in the return type; unexpected conditions — the database is down, a null that shouldn't be — are exceptions. The costs: more types, and every caller must handle every outcome. The benefit: the compiler tells you when a new outcome isn't handled.

With C# 15 (.NET 11) `closed` hierarchies, outcomes become exhaustively checked sum types (Module 16, Concepts 31–33); on .NET 10, use an `abstract record` base and a throwing discard arm:

```csharp
public abstract record PlaceOrderOutcome          // on .NET 11: public closed record PlaceOrderOutcome;
{
    public sealed record Placed(int NewVersion) : PlaceOrderOutcome;
    public sealed record NotFound : PlaceOrderOutcome;
    public sealed record NotDraft(OrderStatus Actual) : PlaceOrderOutcome;
    public sealed record Empty : PlaceOrderOutcome;
    public sealed record InsufficientCredit(Money Available, Money Required) : PlaceOrderOutcome;
    public sealed record StaleVersion(int Current) : PlaceOrderOutcome;
}

static IResult ToHttp(PlaceOrderOutcome outcome, OrderId id) => outcome switch
{
    PlaceOrderOutcome.Placed p              => TypedResults.Ok(new { orderId = id.Value, version = p.NewVersion }),
    PlaceOrderOutcome.NotFound              => TypedResults.NotFound(),
    PlaceOrderOutcome.StaleVersion s        => TypedResults.Problem(statusCode: 412, title: "The order has changed",
                                                   extensions: new Dictionary<string, object?> { ["currentVersion"] = s.Current }),
    PlaceOrderOutcome.NotDraft n            => Problem(409, "order-not-draft", $"The order is {n.Actual}."),
    PlaceOrderOutcome.Empty                 => Problem(422, "order-empty", "An order needs at least one line."),
    PlaceOrderOutcome.InsufficientCredit c  => Problem(422, "insufficient-credit", $"Available {c.Available}, required {c.Required}."),
    _ => throw new UnreachableException(),                      // .NET 11 closed hierarchies make this arm unnecessary
};

static IResult Problem(int status, string type, string detail) =>
    TypedResults.Problem(statusCode: status, type: $"https://errors.example.com/{type}", detail: detail);
```

The HTTP mapping worth knowing cold, because interviewers ask:

| Outcome | Status | Notes |
|---|---|---|
| Created, synchronously | **201 Created** + `Location` | The new resource's URI; body optional |
| Done, nothing to return | **204 No Content** | Or 200 with the outcome/version |
| Accepted for asynchronous processing | **202 Accepted** + `Location` to a status resource | Concept 21 |
| Malformed request | **400 Bad Request** | Field-level problem details |
| Not allowed to issue this command | **403 Forbidden** | 401 if not authenticated |
| Target doesn't exist | **404 Not Found** | |
| Conflicts with current state | **409 Conflict** | "already shipped," "email already in use" |
| Stale version / precondition | **412 Precondition Failed** | With `If-Match`; include the current version/ETag |
| Well-formed but semantically invalid | **422 Unprocessable Content** | Business rule rejection; 409 is also common — pick one convention and document it |
| Rate limited | **429 Too Many Requests** | With `Retry-After` |

Use RFC 9457 problem details with a stable `type` URI per domain error, so clients can switch on `type` rather than parsing messages.

---

## Concept 19 — Idempotent commands

Networks time out after the server has committed. Clients, gateways, message brokers and resilience policies retry. Without idempotency, a retried `PlaceOrder` places two orders and a retried `CapturePayment` charges twice. On the write side of a CQRS system there are four mechanisms, from cheapest to most general:

**1. Natural idempotency.** Some commands are idempotent by their semantics: `SetDeliveryAddress(X)` applied twice leaves address X. `MarkAsRead` twice leaves it read. Design commands as "set to" rather than "change by" where the business allows — "set quantity to 3" rather than "add 1."

**2. Client-generated identity for creation.** `RegisterCustomer(CustomerId NewId, ...)` — the second attempt finds the customer already exists (primary-key violation or a lookup) and returns the original outcome. Cheapest possible idempotency for creates (Module 22, Concept 42).

**3. State-based idempotency.** The aggregate itself knows the command was applied: `Place()` on an already-placed order returns `AlreadyPlaced` (and the handler maps that to the original success). Works when the command moves the aggregate to a state it can't leave by the same command.

**4. Idempotency keys with stored results.** The general mechanism, for commands that aren't naturally idempotent (`AddLine`, `CapturePayment`, `TransferFunds`). The client sends a unique key (commonly an `Idempotency-Key` HTTP header); the server records the key and the **outcome** in the **same transaction** as the state change; a retry with the same key returns the stored outcome without re-executing.

```csharp
// Stored in the same database, written in the same transaction as the aggregate
public sealed class ProcessedCommand
{
    public required string Key { get; init; }                // PK — scoped: e.g. "{userId}:{endpoint}:{clientKey}"
    public required string RequestHash { get; init; }        // detect "same key, different body" → 422
    public required string OutcomeJson { get; init; }
    public required DateTimeOffset ProcessedAt { get; init; }
}

// In the idempotency step (decorator/behavior) around the handler:
var existing = await db.ProcessedCommands.AsNoTracking().SingleOrDefaultAsync(p => p.Key == key, ct);
if (existing is not null)
    return existing.RequestHash == hash
        ? Deserialize<TOutcome>(existing.OutcomeJson)            // replay the original answer
        : throw new IdempotencyKeyReusedException(key);          // same key, different request → 422

var outcome = await next(ct);                                    // handler loads and decides; it does NOT save
db.ProcessedCommands.Add(new ProcessedCommand { Key = key, RequestHash = hash,
    OutcomeJson = Serialize(outcome), ProcessedAt = clock.GetUtcNow() });
await db.SaveChangesAsync(ct);                                   // one commit: aggregate change + outcome record
return outcome;
```

(The ordering detail matters: the processed-command row must be written *in the same commit* as the aggregate change. Here the idempotency step owns the commit — handlers don't call `SaveChanges`, the same convention as the unit-of-work behavior in Concept 65. The alternative is for the step to register the row with the unit of work *before* a handler that saves for itself. Either way, two concurrent requests with the same key race on the primary key, and the loser's whole transaction — including its aggregate change — rolls back.)

The details that separate a production design from a sketch:

- **Scope the key** — per user and per operation — so two users can't collide and a key reused on a different endpoint isn't mistaken for a replay.
- **Hash the request** and reject a reused key with a different body; otherwise a client bug silently returns the wrong result.
- **Store outcomes, including rejections.** A retried command that was *rejected* must get the same rejection, not a second chance under different state.
- **Expire keys** after a retention window longer than any client's retry horizon (24 hours to 7 days is typical), with a cleanup job.
- **Don't store "in progress" in a separate transaction** unless you're prepared to handle crashed in-progress markers; the same-transaction approach avoids that failure mode entirely.

Queued commands (Concept 21) need the same guarantee from the consumer side: the message ID is the idempotency key, recorded in an inbox table in the same transaction (Module 11).

---

## Concept 20 — Concurrency through the command

Module 22 (Concepts 68–69) made the aggregate the unit of optimistic concurrency and distinguished two strategies: **retry** for machine-issued commands, **surface the conflict** for human edits. CQRS adds a wrinkle: the user decided what to do based on the **read model**, which may be older than the write model. The expected version is how that staleness gets caught.

The flow:

```
1. Query:   GET /orders/42          → read model returns { ..., "version": 16 }  (ETag: "16")
2. User edits a form for a while.
3. Command: POST /orders/42/relocate  If-Match: "16"  → command carries ExpectedVersion = 16
4. Handler: loads aggregate at version 17 (someone else changed it) → StaleVersion(17) → 412
5. UI:      "This order was changed by someone else — review and resubmit."
```

Two ways to implement step 4:

**Check explicitly in the handler** (as in Concept 17): compare `aggregate.Version` with `cmd.ExpectedVersion` after loading. Clear and testable. It doesn't close the race *between* load and save — but the aggregate's own concurrency token does (Module 22, Concept 68), so a concurrent writer after your load still produces a `DbUpdateConcurrencyException`.

**Make EF use the expected version as the token's original value:**

```csharp
var entry = db.Entry(order);
entry.Property(o => o.Version).OriginalValue = cmd.ExpectedVersion;   // UPDATE ... WHERE Version = @expected
// decide, then SaveChanges: a mismatch now surfaces as DbUpdateConcurrencyException at commit
```

This closes the check and the save into one atomic comparison, at the cost of turning a clean outcome into an exception you must catch and translate.

Three details worth stating:

- **The version must travel through the read model.** Every read model that feeds an editing screen should carry the aggregate's version, so the UI can send it back. That's a small, easily forgotten requirement on projections (Concept 34).
- **Machine-issued commands usually don't send an expected version.** A background process reacting to an event wants the command applied to the *latest* state, so it omits the expected version and relies on the retry behavior around the handler (Concept 64).
- **Return the new version.** The command's outcome includes the version it produced, so the client can issue the next command without re-reading, and so the next *query* can demand a read model at least that fresh (Concept 53).

---

## Concept 21 — Synchronous vs asynchronous commands

Commands don't have to be processed while the caller waits. The choice:

| | **Synchronous** | **Asynchronous (queued)** |
|---|---|---|
| Flow | Request → handler → outcome in the response | Request → validate & enqueue → **202 Accepted** + status URI → handler later |
| Outcome delivery | In the HTTP response | Status resource polling, push (SignalR/SSE), or an event the client listens for |
| Rejections | Immediate, with reason | Later — the user may have moved on |
| Load profile | Handler capacity must meet peak request rate | Queue absorbs spikes; handlers process at their own pace (load leveling) |
| Failure handling | Caller sees the error and retries | Broker retries; poison commands dead-letter |
| Ordering | Whatever the callers produce | Per-aggregate ordering via partitioned queues (Service Bus sessions, Kafka keys) |
| Complexity | Low | Status tracking, idempotent consumers, outcome notification, dead-letter handling |

**Choose asynchronous when:**

- The work is **long-running** (seconds to minutes): generating a large document, calling a slow external system, a multi-step process.
- Load is **spiky** and handlers are expensive — a queue lets you size handlers for average load instead of peak (Module 6).
- The command depends on an **unreliable external system**, so retries with backoff belong in the infrastructure rather than in the request.
- Ordering per aggregate matters and you want the **single-writer** property (Module 22, Concept 66, remedy 5): a session-enabled queue keyed by aggregate ID serializes commands per aggregate, so queued commands never conflict with each other (commands arriving by other paths still can).

**Choose synchronous when:** the user needs an immediate yes or no ("was my booking accepted?"), the command is fast, and the load is manageable. That's most interactive commands.

The **asynchronous request-reply** pattern (Azure Architecture Center) is the standard shape for the async case:

```
POST /reports/generate            → 202 Accepted
                                    Location: /operations/7f3c…
GET  /operations/7f3c…            → 200 { "status": "Running" }            (Retry-After: 2)
GET  /operations/7f3c…            → 303 See Other  Location: /reports/991   (or 200 { "status": "Succeeded", "resultUri": ... })
```

Two points a senior answer includes:

**A queued command is still a command.** It has exactly one handler and can still be rejected — so the *rejection* must reach the user somehow, which is the part designs forget. A status resource that can say `Rejected: insufficient stock` is part of the design, not an afterthought.

**Validate before you enqueue.** Shape validation and authorization happen synchronously at the edge, so the 202 means "well-formed and accepted for processing," not "we'll tell you in an hour that you forgot a field." Only state-dependent decisions are deferred.

---

## Concept 22 — Write sides without a domain model

CQRS doesn't require DDD. Module 22 (Concept 8) established that **transaction scripts** are the correct choice for supporting subdomains with simple rules; a CQRS write side built from transaction-script command handlers is perfectly valid — and usually the right call for supporting areas.

```csharp
// Supporting subdomain: maintain notification templates. No aggregate — a transaction script.
public sealed record RenameTemplate(TemplateId Id, string NewName, int ExpectedVersion);

public sealed class RenameTemplateHandler(NotificationsDbContext db)
{
    public async Task<RenameOutcome> Handle(RenameTemplate cmd, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(cmd.NewName) || cmd.NewName.Length > 100)
            return RenameOutcome.Invalid;

        var rows = await db.Templates
            .Where(t => t.Id == cmd.Id && t.Version == cmd.ExpectedVersion)
            .ExecuteUpdateAsync(s => s
                .SetProperty(t => t.Name, cmd.NewName)
                .SetProperty(t => t.Version, t => t.Version + 1), ct);

        if (rows == 1) return RenameOutcome.Renamed;
        return await db.Templates.AnyAsync(t => t.Id == cmd.Id, ct) ? RenameOutcome.Stale : RenameOutcome.NotFound;
    }
}
```

That's still CQRS: the command is intent-shaped, the handler owns the write, the read side projects DTOs separately. It just doesn't need aggregates, because there are no invariants worth encapsulating. (Note that `ExecuteUpdateAsync` bypasses the change tracker and therefore interceptors and domain-event dispatch — Module 19, Concept 34 — which is fine here precisely because nothing needs to react. If something did, you'd write the outbox row explicitly.)

The broader point: **CQRS and DDD are complementary, not coupled.** DDD without CQRS works (queries go through repositories, at a cost). CQRS without DDD works (commands are transaction scripts). The combination is strongest in core domains, where the write model is a rich domain model that benefits most from being freed of read concerns. The interview line: *"I'd separate reads from writes everywhere it's cheap, and use a full domain model on the write side only where the rules justify it."*

---

# Part C — The read side, in the same store

Levels 1 and 2 of the spectrum keep one database and one transaction model, so the read side can be as consistent as the writes — and most of a system's read paths should live here. This part covers the read side's job, the two ways to structure a level-1 query path, the tools (EF Core, Dapper, database-maintained views), query shape, paging, caching, replicas, and the two things that must not leak in: business rules and unfiltered data.

---

## Concept 23 — The read side's job

The read side answers questions, fast, in the shape the asker wants. Its responsibilities:

1. **Shape** data for a specific consumer — a screen, an API client, a report, an export — combining whatever aggregates and contexts that consumer needs.
2. **Be fast and cheap** at the read rate the system actually sees.
3. **Enforce visibility** — who may see which rows and fields (Concept 33).
4. **Be honest about freshness** — when the answer may be stale, carry a version or timestamp that lets the caller tell (Concept 53).

What the read side does **not** do:

- **Enforce invariants or make decisions.** It has no aggregates, no repositories, no domain services that decide anything. If a read path needs to know whether something is *allowed*, that knowledge is computed on the write side and stored, or evaluated by a domain function, not re-implemented (Concept 32).
- **Track changes.** Nothing read on the query side is ever saved back, so change tracking, identity maps and unit-of-work machinery are pure overhead.
- **Go through the aggregate.** Module 22 (Concept 92): the domain model exists to protect invariants during changes; a read changes nothing.

The mental model that helps most: **the read side is a set of purpose-built views over the truth, each optimized for one kind of question.** At level 1 the "views" are SQL queries; at level 2 they're tables or indexed views; at level 3 they're separate stores. The *job* is the same at every level; only the machinery changes.

---

## Concept 24 — Level 1: a separate query path

Level 1 is the cheapest CQRS and the one most systems should start — and often stay — at: **queries use their own code path and their own DTOs, reading the write side's tables directly.** No new schema, no new store, no eventual consistency. Microsoft's *.NET Microservices* e-book implements its ordering service's reads this way, with Dapper queries returning view models and bypassing the DDD model entirely.

There are two common ways to structure the query path, and both are fine:

**Style 1 — a query port per area.** One interface declares the questions an area can answer; one infrastructure class answers them. Module 20 (Concept 34) introduced this shape.

```csharp
// Application layer: the questions, in application vocabulary — no query language in sight
public interface IOrderQueries
{
    Task<OrderDetailsDto?> GetDetailsAsync(OrderId id, CancellationToken ct);
    Task<IReadOnlyList<OrderSummaryDto>> RecentForCustomerAsync(CustomerId customer, int take, CancellationToken ct);
    Task<Page<OrderListItemDto>> SearchAsync(OrderSearch criteria, CancellationToken ct);
}

// Infrastructure: unapologetically database-aware
internal sealed class OrderQueries(OrderingDbContext db) : IOrderQueries { /* EF projections, or Dapper */ }
```

**Style 2 — one handler per query.** Each question is a query object with a handler — the vertical-slice style, and the shape mediators encourage.

```csharp
public sealed record GetOrderDetails(OrderId OrderId) : IQuery<OrderDetailsDto?>;

internal sealed class GetOrderDetailsHandler(OrderingDbContext db) : IQueryHandler<GetOrderDetails, OrderDetailsDto?>
{
    public Task<OrderDetailsDto?> Handle(GetOrderDetails q, CancellationToken ct) =>
        db.Orders
          .Where(o => o.Id == q.OrderId)
          .Select(o => new OrderDetailsDto(
              o.Id.Value, o.Status.ToString(), o.Total.Amount, o.Total.Currency, o.Version,
              o.Lines.OrderBy(l => l.LineNumber)
                     .Select(l => new OrderLineDto(l.LineNumber, l.Sku.Value, l.Quantity.Value, l.UnitPrice.Amount))
                     .ToList()))
          .TagWith("Ordering.GetOrderDetails")
          .SingleOrDefaultAsync(ct);
}
```

How to choose: a query port groups related questions and makes the read surface of an area easy to see and review; per-query handlers make each question independently changeable and fit vertical slices (Concept 72). Mixed codebases are common and fine. What matters is that **the query side never uses the write side's repositories and never returns aggregates.**

Three structural choices within level 1:

- **Same `DbContext`, projections only.** The simplest. The write-side `DbContext` maps the tables; the read side uses `Select` projections (no tracking needed for DTOs — Concept 25).
- **A separate read-only `DbContext`.** Maps the same tables (or views) with read-shaped types — often keyless entity types over views — and sets `QueryTrackingBehavior.NoTracking` globally. Useful when the read side wants different mappings from the write side (flattened types, views, different conversions) or connects to a replica (Concept 31).
- **No ORM — Dapper or raw ADO.NET** against an `IDbConnection` factory (Concept 26).

The interview line: *"The cheapest useful CQRS is a separate query path over the same tables. Queries project straight to DTOs with no tracking and no aggregate, and the write model stops growing getters for screens. I'd need a specific reason to go further."*

---

## Concept 25 — EF Core on the read side

Module 19 covered EF Core's query pipeline in depth; this is the read-side toolkit, with the traps that matter.

**1. Project with `Select`, always.** A projection generates SQL that fetches only the columns named, and projecting into DTOs means nothing is tracked:

```csharp
db.Orders
  .Where(o => o.CustomerId == customerId)
  .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
  .Take(20)
  .Select(o => new OrderSummaryDto(o.Id.Value, o.Status.ToString(), o.Total.Amount, o.Total.Currency,
                                   o.Lines.Count, o.CreatedAt))    // Lines.Count → correlated subquery, lines not loaded
  .ToListAsync(ct);
```

Precision on tracking: EF tracks *entity instances* that appear in a result, including entities nested inside anonymous types. A projection to DTOs containing only scalars produces no entities, so there's nothing to track and `AsNoTracking()` is redundant (harmless). If a projection returns entity types — `new { Order = o, o.Lines }` — then tracking applies and `AsNoTracking()` matters.

**2. Turn tracking off by default on read-only contexts.**

```csharp
services.AddDbContext<OrderingReadDbContext>(o => o
    .UseSqlServer(readConnectionString)
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

Use `AsNoTrackingWithIdentityResolution()` only when you load entities with shared references and need them de-duplicated in the result graph.

**3. Watch for the Cartesian explosion.** Projecting two collections (`Lines` and `Payments`) in one query joins them, multiplying rows. `AsSplitQuery()` issues one SQL statement per collection — fewer rows, more round trips, and no single-snapshot consistency across the statements unless you use a transaction or snapshot isolation (Module 19).

**4. Compile hot queries.** For a query executed thousands of times per second, EF's query cache already avoids re-translation, but compiled queries skip the cache lookup and expression-tree work:

```csharp
private static readonly Func<OrderingReadDbContext, Guid, CancellationToken, Task<OrderHeaderDto?>> GetHeader =
    EF.CompileAsyncQuery((OrderingReadDbContext db, Guid id, CancellationToken ct) =>
        db.Orders.Where(o => o.Id == new OrderId(id))
                 .Select(o => new OrderHeaderDto(o.Id.Value, o.Status.ToString(), o.Version))
                 .SingleOrDefault());
```

Worth it only where a profiler says query compilation or cache lookup is measurable (Module 17's "measure first").

**5. Drop to SQL without leaving EF when LINQ is the wrong tool.** `Database.SqlQuery<T>(...)` (EF 8+) maps a raw SQL result to an **unmapped** type, parameterized safely through interpolation:

```csharp
var rows = await db.Database.SqlQuery<MonthlyRevenueRow>($"""
    SELECT DATEFROMPARTS(YEAR(PlacedAt), MONTH(PlacedAt), 1) AS Month,
           Currency, SUM(TotalAmount) AS Revenue, COUNT(*) AS Orders
    FROM ordering.Orders
    WHERE Status IN ('Placed','Paid','Shipped') AND PlacedAt >= {from} AND PlacedAt < {to}
    GROUP BY DATEFROMPARTS(YEAR(PlacedAt), MONTH(PlacedAt), 1), Currency
    """).ToListAsync(ct);
```

**6. Map views as keyless entity types.** A database view designed for a screen becomes a first-class queryable type:

```csharp
modelBuilder.Entity<CustomerDashboardRow>(b => { b.HasNoKey(); b.ToView("CustomerDashboard", "reporting"); });
```

**7. Tag queries.** `.TagWith("Ordering.RecentForCustomer")` puts a comment into the SQL, so the query shows up by name in Query Store, `pg_stat_statements` and APM traces. On the read side, where queries multiply, this is the difference between "some query is slow" and "the dashboard query is slow."

**8. Ignore auto-includes on the read side.** If the write model uses `AutoInclude()` for aggregate children (Module 22, Concept 89), projections ignore it automatically; entity queries on the read side should use `IgnoreAutoIncludes()`.

The traps: **client evaluation** of the final projection is allowed (a method call in `Select` runs in memory after the SQL), but untranslatable expressions in `Where`/`OrderBy` throw — which is correct but surprises people; **paging without a total order** (`Take` without a unique `OrderBy`) returns nondeterministic pages; and **projecting value objects** through value converters sometimes needs the converted scalar (`o.Id.Value`) rather than the object.

---

## Concept 26 — Dapper and SQL on the read side

Hand-written SQL is not a failure of abstraction on the read side; for many queries it's the most direct, most reviewable and fastest option. Microsoft's own microservices reference uses Dapper for exactly this reason.

**When SQL is the right tool:**

- **Reporting and analytical queries** — window functions, CTEs, `GROUP BY ROLLUP`, `PIVOT`, recursive hierarchies — that LINQ expresses badly or not at all.
- **Queries tuned against a plan** — with hints, specific join orders, or index-targeted shapes — where you need to control exactly what runs.
- **The team reads SQL better than LINQ-generated SQL.** A query that a DBA can review directly has real operational value.
- **Maximum throughput** for simple, hot lookups — though EF Core's gap has narrowed a lot, and "Dapper is faster" alone is rarely a good enough reason (Module 19).

```csharp
internal sealed class CustomerReportQueries(IDbConnectionFactory connections) : ICustomerReportQueries
{
    public async Task<IReadOnlyList<TopCustomerRow>> TopCustomersAsync(DateOnly from, DateOnly to, int take, CancellationToken ct)
    {
        const string sql = """
            WITH revenue AS (
                SELECT o.CustomerId, SUM(o.TotalAmount) AS Revenue, COUNT(*) AS Orders
                FROM ordering.Orders o
                WHERE o.Status IN ('Placed','Paid','Shipped') AND o.PlacedAt >= @From AND o.PlacedAt < @To
                GROUP BY o.CustomerId
            )
            SELECT TOP (@Take) r.CustomerId, c.DisplayName, r.Revenue, r.Orders,
                   RANK() OVER (ORDER BY r.Revenue DESC) AS Rank
            FROM revenue r
            JOIN ordering.CustomerSnapshots c ON c.CustomerId = r.CustomerId   -- Ordering's own replicated copy (Module 21, C41)
            ORDER BY r.Revenue DESC;
            """;

        await using var conn = await connections.OpenAsync(ct);
        var rows = await conn.QueryAsync<TopCustomerRow>(new CommandDefinition(sql,
            new { From = from.ToDateTime(TimeOnly.MinValue), To = to.ToDateTime(TimeOnly.MinValue), Take = take },
            cancellationToken: ct));
        return rows.AsList();
    }
}
```

**How to keep hand-written SQL honest:**

- **Put it behind the query port** (Concept 24), so the SQL lives in one infrastructure class per area, not scattered through endpoints.
- **Test it against a real database.** A query that compiles can still reference a renamed column. Integration tests with Testcontainers (Module 12) run each query against the migrated schema in CI; that's the only thing that catches schema drift.
- **Always parameterize.** Dapper parameters or EF's interpolated `SqlQuery` — never string concatenation (SQL injection, and plan-cache pollution).
- **Name and tag queries** (a leading comment works in raw SQL) so they're identifiable in Query Store and traces.
- **Stay inside your context's schema.** The read side of Ordering reads Ordering's tables, including its own replicated copies of other contexts' data — never another module's tables (Module 21, Concept 38).

For Native AOT, Dapper's reflection-based materialization is a problem; `Dapper.AOT` (a source generator) addresses it, and EF Core's `SqlQuery<T>` with compiled models is the other route.

---

## Concept 27 — Database-maintained read models

Before building a projection, ask whether the database can maintain the read model for you. It often can, and the result is level 2 with zero application code.

**The cheapest read model is an index.** A covering index — key columns for the filter and sort, `INCLUDE` for the projected columns — turns a scan into a seek and makes the "read model" a structure the engine maintains synchronously on every write (Module 12). Many "we need CQRS for performance" conversations end here.

**Views** name a query. A regular view stores nothing and costs nothing to maintain; its value is giving a screen-shaped query a stable name, a single place to review it, and a mapping target for EF (Concept 25, point 6). It doesn't make anything faster by itself.

**Indexed views (SQL Server)** are *materialized*: the engine stores the view's result and maintains it **synchronously inside every transaction** that modifies the base tables. They're ideal for precomputed aggregates ("order count and revenue per customer per month") read far more often than the base tables change. The restrictions are strict: `WITH SCHEMABINDING`, deterministic expressions only, `COUNT_BIG(*)` required with `GROUP BY`, no outer joins, no subqueries, no `DISTINCT`, and a unique clustered index to materialize. Every write to a base table pays to maintain every indexed view over it — and a hot aggregate row in an indexed view becomes a contention point, exactly like a hot aggregate (Module 22, Concept 66). Outside Enterprise edition (and Azure SQL, which behaves like it), queries must reference the view `WITH (NOEXPAND)` to use it reliably.

**Materialized views (PostgreSQL)** store a query's result but are **not maintained automatically**: `REFRESH MATERIALIZED VIEW` recomputes the whole thing, and `REFRESH ... CONCURRENTLY` (which requires a unique index on the view) does so without blocking readers. That makes them a *periodically refreshed* read model — eventually consistent with a staleness equal to the refresh interval, and expensive to refresh on large data. Good for dashboards refreshed every few minutes; wrong for anything that must reflect a user's last action.

**Computed and persisted columns** move a derived value (a normalized search key, a `LineTotal`) into the row, maintained on write and indexable.

The decision rule: **if the database can maintain the read model synchronously and cheaply, let it** — you get level 2 without writing a projector. Move to application-maintained projections when the read model needs data the database can't join (other contexts' events), a different store, or logic that SQL can't express.

---

## Concept 28 — Purpose-built vs generic queries

Two philosophies for the shape of the read API:

**Purpose-built queries** — one query per screen, consumer or question: `GetOrderDetails`, `GetCustomerDashboard`, `SearchOpenInvoices`. Each returns exactly what its consumer needs.

- ✅ Each query can be optimized, indexed, cached and secured individually; there are no surprise query shapes in production.
- ✅ The API documents the product: the list of queries is the list of things users can see.
- ⚠️ More endpoints; each new screen needs a new or changed query.

**Generic queries** — a flexible query surface the client composes: OData (`$filter`, `$select`, `$expand`), GraphQL, or a homegrown "filter by any field" endpoint.

- ✅ New screens often need no backend change; clients fetch exactly the fields they want.
- ⚠️ The set of queries that can hit your database is unbounded. Someone will filter on an unindexed column over ten million rows. You need query cost analysis, depth and complexity limits, maximum page sizes and allow-listed filters — which is work.
- ⚠️ Authorization is harder: it must be enforced per field and per row, for any combination the client can express (Concept 33).
- ⚠️ With EF-backed GraphQL or OData, you're exporting `IQueryable` semantics to the client — Module 20's `IQueryable` leakage concern, now at the API level. It's more defensible on the read side (no invariants to break) but the performance and security concerns remain.

The pragmatic middle, which most systems land on: **purpose-built queries for the product's core screens, and a constrained search endpoint** (an allow-list of filters and sort keys, a maximum page size) for list views. GraphQL earns its place when many different clients — web, mobile, partners — need different slices of the same data and the team can invest in its operational safety.

The **Backend-for-Frontend (BFF)** pattern (Module 21, Concept 81) is CQRS's natural companion at the API level: each client gets a read API shaped for it, which may compose several contexts' read models. The BFF owns *presentation-shaped* reads; the contexts own their domain-shaped read models.

---

## Concept 29 — Paging, filtering and counting

List screens are where read-side performance problems concentrate. Three techniques and one warning.

**Offset paging** (`OFFSET 10000 ROWS FETCH NEXT 20`) reads and discards every row before the offset — page 500 costs 500 times page 1 — and returns duplicates or gaps when rows are inserted between requests. Fine for small, admin-style lists; wrong for deep or infinite-scroll lists.

**Keyset (seek) paging** remembers the last row's sort key and asks for rows after it. With a matching index, every page costs the same:

```csharp
// Cursor = the sort key of the last row returned. Sequence is a bigint (identity or DB sequence) used as a
// unique tiebreaker; GUID tiebreakers work too, but ordering and translation of GUID comparisons vary by provider.
public sealed record OrderPageCursor(DateTimeOffset CreatedAt, long Sequence);

public async Task<CursorPage<OrderListItemDto>> PageAsync(CustomerId customer, OrderPageCursor? after, int size, CancellationToken ct)
{
    var query = db.Orders.Where(o => o.CustomerId == customer);

    if (after is not null)
        query = query.Where(o => o.CreatedAt < after.CreatedAt
                              || (o.CreatedAt == after.CreatedAt && o.Sequence < after.Sequence));

    var items = await query
        .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Sequence)
        .Take(size + 1)                                            // fetch one extra: "has more" without COUNT(*)
        .Select(o => new OrderListItemDto(o.Id.Value, o.Sequence, o.Status.ToString(),
                                          o.Total.Amount, o.Total.Currency, o.CreatedAt))
        .ToListAsync(ct);

    var hasMore = items.Count > size;
    if (hasMore) items.RemoveAt(items.Count - 1);
    var next = hasMore ? new OrderPageCursor(items[^1].CreatedAt, items[^1].Sequence) : null;
    return new CursorPage<OrderListItemDto>(items, next);          // serialize the cursor opaquely (base64) for clients
}
// Index: (CustomerId, CreatedAt DESC, Sequence DESC) INCLUDE (Status, TotalAmount, Currency)
```

**Counting.** `COUNT(*)` over a filtered set is a scan of every matching row, on every page request. Alternatives: "fetch n+1" to show *Next* without a total (above); an approximate count from statistics; a count computed once per search session and cached; or a precomputed count in a level-2 read model.

**The warning — combinatorial filters.** A list screen with eight optional filters and five sort orders can't be indexed for every combination in a relational store. When filter combinations explode, the right read model is usually a **search index** (Concept 49), not more B-tree indexes.

---

## Concept 30 — Caching as a read model

A cache in front of the query side is a read model with a time-based consistency policy. Module 10 treated caching as an asynchronous replica with weak consistency; everything there applies. Three layers, cheapest to invalidate first:

**1. HTTP caching.** Queries are `GET`s, so HTTP's machinery works: `ETag` from the read model's version, `If-None-Match` from the client, `304 Not Modified` when nothing changed, `Cache-Control: private, max-age=…` for browser caching. The version you already carry for concurrency (Concept 20) doubles as the ETag — the cheapest consistency-correct cache there is.

**2. Output caching** (ASP.NET Core) caches whole responses on the server, with tags you can evict when something changes:

```csharp
builder.Services.AddOutputCache(o => o.AddPolicy("customer-dashboard", p => p
    .Expire(TimeSpan.FromSeconds(30))
    .SetVaryByRouteValue("customerId")
    .Tag("customers")));

app.MapGet("/customers/{customerId}/dashboard", GetDashboard)
   .CacheOutput("customer-dashboard");

// In a projector or event handler, after the read model changes:
await outputCacheStore.EvictByTagAsync("customers", ct);    // coarse; use per-customer tags where the policy allows
```

Output caching must never be applied to responses that vary by user without varying the cache key by user — a classic data leak.

**3. Data caching with `HybridCache`** (.NET 9+, the recommended default from Module 10): an in-process L1 plus a distributed L2, with stampede protection (concurrent misses for the same key trigger one factory call) and tag-based invalidation:

```csharp
public Task<CustomerDashboardDto> GetDashboardAsync(CustomerId id, CancellationToken ct) =>
    cache.GetOrCreateAsync(
        $"dashboard:{id.Value}",
        async token => await queries.LoadDashboardAsync(id, token),
        new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(5), LocalCacheExpiration = TimeSpan.FromSeconds(30) },
        tags: [$"customer:{id.Value}"],
        cancellationToken: ct);

// Invalidate on the facts that change it — in a handler for OrderPlaced, CustomerRelocated, ...
await cache.RemoveByTagAsync($"customer:{evt.CustomerId}", ct);
```

Two points that make this CQRS-specific:

- **Invalidation comes from events.** The same domain or integration events that feed projections (Part D) are the natural invalidation signal. "Evict `customer:{id}` whenever an event about that customer is processed" is simple and correct up to the event-processing lag.
- **A cache makes a level-1 read path eventually consistent.** Even if the query reads the primary database, a cached answer can be as old as its TTL or its invalidation lag. That's fine — but it means read-your-writes (Part E) applies to cached level-1 reads just as it does to level-3 read models. The usual fix is to skip or bypass the cache for the user's own freshly written data.

---

## Concept 31 — Read replicas

Routing queries to read replicas is the infrastructure version of CQRS: same schema, separate copy, read load isolated from the primary. On Azure:

- **Azure SQL Database read scale-out** — in the Premium, Business Critical and Hyperscale tiers, a built-in readable secondary is available by adding `ApplicationIntent=ReadOnly` to the connection string.
- **Hyperscale named replicas** — up to 30 independently sized read replicas, useful for isolating heavy read workloads (reporting, search indexing) from each other and from the primary.
- **Azure Database for PostgreSQL flexible server read replicas** — asynchronous physical replication, in-region or cross-region.
- **Active geo-replication** readable secondaries — for regional read locality as well as disaster recovery.

**The catch: replicas are asynchronous.** Replication lag is usually well under a second and occasionally much longer — during heavy write bursts, index rebuilds or failovers (Module 8). So **level 1 over a replica is eventually consistent.** Everything in Part E applies: a user who saves and immediately reloads may read from a replica that hasn't caught up.

Routing in .NET is simplest with two context registrations:

```csharp
builder.Services.AddDbContext<OrderingDbContext>(o =>                 // write side: primary
    o.UseSqlServer(config.GetConnectionString("Ordering")));

builder.Services.AddDbContext<OrderingReadDbContext>(o => o             // read side: replica
    .UseSqlServer(config.GetConnectionString("OrderingReadOnly"))       // "...;ApplicationIntent=ReadOnly"
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

Query handlers depend on `OrderingReadDbContext`; command handlers on `OrderingDbContext`. That makes the routing a compile-time property of each handler — visible in code review — rather than a runtime switch.

Three refinements for production:

- **Read-your-writes routing.** For a short window after a user's command (a few seconds, or until the replica's position passes the command's commit), route that user's reads to the primary. A cookie or a claim carrying "wrote at T" is enough to implement it (Concept 52).
- **Per-query choice.** Some queries must be current (a balance shown just before a payment); give those handlers the primary context explicitly and document why.
- **Monitor lag.** Alert when replica lag exceeds the staleness budget the business agreed (Concept 50). A replica that's ten minutes behind during an incident is serving wrong answers to everyone.

---

## Concept 32 — Business rules don't leak into queries

The most insidious read-side defect: a business rule re-implemented in a query, because a screen needed to show its result.

```sql
-- "Can this order still be cancelled?" — the UI wants a button enabled or disabled
SELECT o.Id,
       CASE WHEN o.Status = 'Draft'
              OR (o.Status = 'Placed' AND o.PlacedAt > DATEADD(MINUTE, -30, SYSUTCDATETIME()))
            THEN 1 ELSE 0 END AS CanCancel
FROM ordering.Orders o WHERE o.Id = @Id;
```

The domain model has the same rule in `Order.Cancel()`. The day the business changes the window to 60 minutes for premium customers, one of the two implementations won't be updated, and the button will be enabled for orders the command rejects — or disabled for orders it would accept.

Four ways to avoid the second implementation, in order of preference:

**1. Compute at write time and store the result.** The aggregate knows the rule; when the order is placed, it records `CancellableUntil`. The query just compares a stored timestamp with now — presentation, not policy. Any rule whose answer changes only when the aggregate changes (or at a known time) can be handled this way.

**2. Project it from events.** A projection that handles `OrderPlaced` and `OrderShipped` sets `CanCancel` in the read model — the rule's *outcome* is carried by the events the domain raised.

**3. Share the rule as a pure function or expression.** If the rule must be evaluated at query time, define it once in the domain as a specification that both the aggregate and the query use:

```csharp
public static class OrderPolicies
{
    public static readonly TimeSpan FreeCancellationWindow = TimeSpan.FromMinutes(30);

    // Translatable by EF when used in a query, compilable for in-memory use by the aggregate
    public static Expression<Func<Order, bool>> IsCancellableAt(DateTimeOffset now)
    {
        var cutoff = now - FreeCancellationWindow;
        return o => o.Status == OrderStatus.Draft || (o.Status == OrderStatus.Placed && o.PlacedAt > cutoff);
    }
}
```

This keeps one definition but couples the rule's *expression* to what the ORM can translate — acceptable for simple predicates, awkward for rich ones.

**4. Ask the write side.** For rare, important cases ("show the exact refund amount before the user confirms"), call a domain function or a dedicated endpoint that loads the aggregate and evaluates the rule. It's slower and it's honest.

The general rule — Module 20 (Concept 35) and Module 22 (Concept 92) both stated it: **business calculations stay in the domain even when their results are only displayed.** The read side may format, filter, sort and combine; it may not decide.

---

## Concept 33 — Authorization on the read side

Commands are authorized per operation ("may this user relocate this customer?"). Queries are authorized per **row** and per **field** ("which customers may this user see, and which of their fields?") — a different problem, and the read side's responsibility.

**Row-level filtering.** Every query must be restricted to the rows the caller may see: their tenant, their own records, their region, their assigned accounts. Doing this in each query by hand is how rows leak — the one query someone forgot. Centralize it:

- **EF Core global query filters** apply a predicate to every query for an entity type. EF Core 10 added **named query filters**, so an entity can have several (tenant, soft-delete) and a query can disable one selectively without disabling the rest:

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter("Tenant", o => o.TenantId == _tenant.Id)          // _tenant: a scoped service on the DbContext
    .HasQueryFilter("SoftDelete", o => !o.IsDeleted);

// An admin "deleted orders" screen keeps tenant isolation but shows deleted rows:
db.Orders.IgnoreQueryFilters(["SoftDelete"])...
```

- **Database row-level security** (SQL Server RLS, PostgreSQL RLS) enforces the same predicate inside the engine, as defense in depth — it also protects Dapper queries, reporting tools and ad-hoc access that bypass EF.

**Field-level exposure.** The safest approach is **different DTOs — or different read models — per audience.** A support agent's view of a customer and the customer's own view are different read models with different fields, not one DTO with fields nulled out conditionally. At level 3 this becomes natural: an audience-specific projection simply never copies the fields that audience may not see, which also limits the blast radius of a leak.

**Caches must vary by audience** (Concept 30). A cached response computed for an administrator and served to a customer is a data breach.

**Generic query APIs raise the stakes** (Concept 28): with OData or GraphQL, the client chooses fields and filters, so authorization must be enforced for every field and every relationship the schema exposes.

**Privacy lives here too.** Read models are copies of personal data. When a customer exercises the right to erasure, every read model and cache holding their data must be updated too (Concept 88) — which is much easier to do if you know which read models copy which fields.

---

# Part D — Separate read models: projections

Levels 2 and 3 give the read side its own schema, and at level 3 its own store. The component that makes that possible is the **projection** — code that turns facts about changes into query-shaped state. A projection is simple to write and surprisingly hard to make correct under real delivery semantics: duplicates, reordering, crashes mid-batch, poison messages, schema changes and the need to rebuild from scratch. This part covers all of it, then the stores projections write to, then how to know whether they're keeping up.

---

## Concept 34 — What a projection is

A **projection** is a function that consumes a sequence of facts — domain events, integration events or row changes — and maintains a query-optimized representation of them.

Formally it's a **left fold**, the same shape as the decider's `evolve` from Module 22 (Concept 73):

```
readModel = facts.Aggregate(initialState, apply)       // apply : (state, fact) → state
```

In practice, a projection is a set of handlers, one per fact type, each updating rows in a read store:

```csharp
// Read model row: one per order, shaped for the "my orders" list and the order header
public sealed class OrderSummaryRow
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = "";
    public decimal Total { get; set; }
    public string Currency { get; set; } = "";
    public int LineCount { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? PlacedAt { get; set; }
    public DateTimeOffset? ShippedAt { get; set; }
    public int Version { get; set; }                  // last applied aggregate version: ordering + idempotency (Concepts 39–40)
}

// The projection consumes the context's event *contracts* — flat, primitive, versioned records
// (Module 22, Concept 83) — plus an envelope (EventMeta) with event ID, aggregate version and position.
public sealed class OrderSummaryProjection(ReadModelsDbContext db)
{
    public Task Apply(OrderDrafted e, EventMeta m, CancellationToken ct) =>
        Upsert(e.OrderId, m.Version, row =>
        {
            row.CustomerId = e.CustomerId; row.Status = "Draft";
            row.Currency = e.Currency; row.CreatedAt = e.OccurredAt;
        }, ct);

    public Task Apply(OrderLineAdded e, EventMeta m, CancellationToken ct) =>
        Upsert(e.OrderId, m.Version, row => { row.LineCount++; row.Total = e.NewOrderTotal; }, ct);

    public Task Apply(OrderPlaced e, EventMeta m, CancellationToken ct) =>
        Upsert(e.OrderId, m.Version, row => { row.Status = "Placed"; row.PlacedAt = e.PlacedAt; row.Total = e.Total; }, ct);

    // Upsert: see Concepts 39–41 for the version guard and checkpointing that make this safe
    private async Task Upsert(Guid orderId, int version, Action<OrderSummaryRow> change, CancellationToken ct) { /* … */ }
}
```

The properties a production projection must have — each is a concept in this part:

| Property | Meaning | Concept |
|---|---|---|
| **Deterministic** | Same facts in, same state out — no `DateTime.Now`, no random, no external calls that vary | here |
| **Order-aware** | Applies facts for one aggregate in the order they happened | 39 |
| **Idempotent** | Applying a fact twice has the same effect as once | 40 |
| **Checkpointed** | Knows how far it got, durably, consistently with its writes | 41 |
| **Failure-tolerant** | One bad fact doesn't silently stop everything, or silently get skipped | 42 |
| **Rebuildable** | Can be dropped and recreated from the source of truth | 43 |
| **Observable** | Its lag and errors are measured and alerted on | 50 |

Two design rules:

**Carry what the projection needs in the fact.** `OrderLineAdded` above carries `NewOrderTotal`, so the projection doesn't have to recompute the total — which would re-implement the pricing rule on the read side (Concept 32). This is the *event-carried state transfer* style (Module 22, Concept 78) applied inside a context.

**Projections don't call back.** A projector that handles `OrderPlaced` by querying the write database for the customer's name has reintroduced coupling, a race (the write side may have moved on) and a dependency on the write side's availability. If the read model needs the customer name, the projection maintains its own copy from customer events (Concept 45).

---

## Concept 35 — Level 2: synchronous projections

A **synchronous (inline) projection** updates read-model tables **in the same transaction** as the write. In an EF Core system it's a domain-event handler dispatched during `SaveChanges` (Module 22, Concept 81) that writes to a read table instead of to the outbox:

```csharp
// Runs inside SaveChanges, before commit — same connection, same transaction as the aggregate change
internal sealed class CustomerOrderStatsProjection(OrderingDbContext db) : IDomainEventHandler<OrderPlaced>
{
    public async Task Handle(OrderPlaced e, CancellationToken ct)
    {
        var stats = await db.CustomerOrderStats.FindAsync([e.CustomerId.Value], ct);
        if (stats is null)
        {
            stats = new CustomerOrderStats { CustomerId = e.CustomerId.Value, Currency = e.Total.Currency };
            db.CustomerOrderStats.Add(stats);
        }
        stats.OrderCount++;
        stats.LifetimeValue += e.Total.Amount;
        stats.LastOrderAt = e.PlacedAt;
    }
}
```

**What it buys:** read tables that are **exactly as consistent as the write** — no lag, no read-your-writes problem, no broker, no checkpoint, no rebuild tooling beyond a SQL script. A projection bug surfaces immediately as a failed command in development rather than as silent drift in production.

**What it costs:**

- **Write latency.** Every command now also updates every inline read model affected by its events. Three inline projections might add three reads and three writes to every command.
- **Contention.** Summary rows are shared across aggregates. `CustomerOrderStats` above is updated by *every* order a customer places; a business customer placing many orders concurrently turns that row into a hot spot, and a global "orders today" counter would serialize every order in the system. This is the hot-aggregate problem (Module 22, Concept 66) reintroduced by the read side.
- **Failure coupling.** A projection that throws fails the command. The read side can now break the write side.
- **Same store only.** The projection can only write to the database the transaction spans. You can't maintain a search index or a Cosmos container synchronously.
- **Deadlock risk.** Updating shared summary rows in varying orders across transactions is a classic deadlock recipe (Module 12).

**When level 2 is right:** a small number of cheap read models in the same context and database, updated by low-to-moderate write rates, where lag would be unacceptable to users — per-aggregate summaries, per-customer counters for B2C (one writer per customer), audit-style history tables.

**When to move to level 3:** when inline projections measurably add to command latency, when a summary row becomes hot, when the read model belongs in another store, or when a projection's failure shouldn't be able to fail a business operation.

---

## Concept 36 — Level 3: asynchronous projections

At level 3, projections run **after** the write commits, in a separate process or background worker, consuming facts from a durable source. The canonical pipeline:

```
[Command handler] ──commit──▶ [write DB: state + outbox rows]
                                         │
                            outbox relay / CDC / change feed
                                         ▼
                              [durable log or queue]            ← Service Bus topic, Event Hubs/Kafka, Cosmos change feed
                                         │  at-least-once, per-key ordering
                                         ▼
                              [projector(s)]  ──▶ [read store A: SQL tables]
                                              ──▶ [read store B: search index]
                                              ──▶ [cache invalidation]
```

The projector itself is usually a `BackgroundService` (or an Azure Functions trigger, or a Container Apps job) that loops: receive a batch, apply each fact through the projection's handlers, persist the read-model changes and the checkpoint, acknowledge.

What level 3 buys, beyond level 2:

- **Any store.** Each read model can live in the technology that answers its queries best (Concept 47).
- **Isolation.** Read-model work no longer lengthens the command's transaction or contends with it; a slow projection slows only itself.
- **Independent scaling.** Projectors scale by partition; read stores scale by replicas.
- **New read models without touching the write side** — add a projector, backfill it (Concept 43), done.
- **Failure isolation.** A projection bug affects its read model's freshness, not the business operation.

What it costs — and this is the list interviewers want:

- **Eventual consistency**, and everything in Part E to make it tolerable.
- **At-least-once delivery**, so projections must be idempotent (Concept 40).
- **Ordering** only within partitions, so projections must cope with it (Concept 39).
- **A checkpoint** to maintain, atomically if possible (Concept 41).
- **Poison-message handling** (Concept 42).
- **Rebuild machinery**, which requires a replayable source (Concept 43).
- **Monitoring** of lag and errors (Concept 50).
- **More moving parts to operate**: a relay, a broker, projector hosts, read stores.

None of these are exotic — they're Module 11's messaging concerns applied to one consumer type — but each must be designed deliberately. A level-3 read model built without them works in the demo and drifts in production (Concept 99).

---

## Concept 37 — Feeding projections: the sources

Where does the projector get its facts? The options differ in what they carry, how they order, whether they include deletes and — critically — whether you can **replay** them to rebuild.

| Source | What it carries | Ordering | Deletes | Replay for rebuild | Coupling | Notes |
|---|---|---|---|---|---|---|
| **Outbox → broker** (Service Bus, Event Hubs, Kafka) | Your domain or integration events — **intent** | Per session/partition key | As explicit events (`OrderCancelled`) | Only with retention: Kafka/Event Hubs (retention window), Service Bus (none after consumption) | To your event contracts | The default for level 3 (Module 11) |
| **Outbox table polled directly** | Same events | Outbox sequence order | Explicit events | While rows are retained | To event contracts | Simplest when projector and read store share the write DB server; no broker |
| **Database CDC** (SQL Server CDC, Debezium on Postgres/SQL/MySQL) | **Row changes** — before/after images | Commit (log) order per table | Yes | Limited to CDC retention; Debezium can re-snapshot | To your **table schema** | Captures changes that bypass the domain (bulk updates, manual fixes) |
| **Change event streaming** (SQL Server 2025, Azure SQL DB/MI) — **preview** | Row changes as CloudEvents to Event Hubs | Per configured partitioning | Yes | Event Hubs retention; no initial snapshot of existing data | To table schema | Lower overhead than CDC; no DDL events; full recovery model; Kafka protocol for new stream groups on Azure SQL DB since 15 Aug 2026 |
| **Cosmos DB change feed — latest version** | Latest state of each changed item | Per logical partition key | **No** (use soft delete + TTL) | **Yes, from the beginning** — but only the *current* version of each item | To document shape | Change feed processor handles leases and scaling |
| **Cosmos DB change feed — all versions and deletes** (GA June 2026) | Every create, replace, delete, TTL expiry, with metadata | Per logical partition key | Yes | Only within the **continuous-backup retention period** | To document shape | Requires continuous backup |
| **Event store subscription** (Module 24) | Events — the source of truth | Per stream, often global position | Explicit events | **Yes, full history** | To event contracts | The best rebuild story of any source |
| **Database triggers** | Whatever the trigger writes | Transactional | Yes | No | Hidden, schema-level | Avoid: logic invisible to the application, runs inside every write |

Three observations:

**Replayability is the column that decides your rebuild strategy.** Only an event store (or a log with indefinite retention) lets you rebuild a projection purely by replaying facts. Everything else needs a **bootstrap-from-current-state** step followed by a catch-up from the live feed (Concept 43).

**The Cosmos latest-version change feed replays state, not history.** Starting a new change feed processor from the beginning gives you every *existing* item once, in its current version — which is ideal for building a *current-state* read model and useless for anything that needs intermediate history or deletes.

**CES is promising and not yet a default.** It removes the outbox relay for the "stream my table changes to Event Hubs" case, but as of September 2026 it's in preview, doesn't seed existing data, doesn't emit schema changes, and truncates column values larger than 1 MB. It's a reasonable choice for analytics feeds and for outbox tables (Concept 38); for core read models, verify its supportability status before depending on it.

---

## Concept 38 — Events vs CDC as a projection source

The biggest source decision: project from **events your code publishes** or from **row changes the database captures**?

| | **Events (outbox)** | **CDC / change streams** |
|---|---|---|
| Carries | Intent: `CustomerRelocated` vs `CustomerAddressCorrected` | Diffs: `Customers` row, `Street` changed from A to B |
| Completeness | Only what code raises — changes that bypass the domain (bulk `ExecuteUpdate`, migrations, manual fixes) are invisible unless someone raises events | Everything that hits the table, however it got there |
| Schema coupling | To an event contract you design and version | To the **physical table schema** — renaming a column is a breaking change for every consumer |
| Transaction shape | One event per business fact, often spanning several rows | One change per row; a multi-row business change arrives as several unrelated-looking records |
| Business meaning | Explicit | Must be inferred ("status went from Placed to Cancelled — was that the customer or fraud?") |
| Effort on the write side | Raise events, run an outbox | None beyond enabling capture |

The judgment:

**Within a bounded context, for its own read models, CDC is acceptable** — the team owns both the schema and the projection, so schema coupling is local coupling (Module 22, Concept 32: high strength at low distance is fine). CDC's completeness is a genuine advantage when bulk operations or data fixes are common.

**Across contexts, CDC is intrusive coupling** — another team's projection depends on your table layout, which is exactly the "reading another context's database" anti-pattern with extra steps (Module 21, Concept 35). Across contexts, publish events in a published language (Module 22, Concept 83).

**The hybrid gets both:** use CDC or change streaming **on the outbox table only**. The code writes intent-shaped events to the outbox in the same transaction (Module 11); CDC streams the outbox inserts to the broker without a polling relay. Debezium ships an "outbox event router" for exactly this, and SQL Server's change event streaming can target an outbox table the same way. You keep intent-revealing, versioned contracts and lose the polling latency and relay process.

The interview line: *"I'd project from events for anything crossing a context boundary, because CDC couples consumers to my table layout. Inside a context, CDC is fine and catches changes that bypass the domain. The best of both is running CDC on the outbox table."*

---

## Concept 39 — Ordering

Projections are order-sensitive: applying `OrderShipped` before `OrderPlaced` produces a row that says "Placed" after the order shipped. But the ordering you actually need is weaker than people assume.

**What order is needed?** Almost always **per aggregate** (per stream): events about order 42 must be applied in the order they happened. Events about order 42 and order 43 can be applied in any interleaving, because they touch different rows. Global ordering is rarely required — and it's expensive, because it rules out parallelism.

**What sources guarantee:**

- **Service Bus** — FIFO per **session**; set `SessionId = aggregateId` when publishing, and a session-aware processor handles one session at a time.
- **Event Hubs / Kafka** — ordered per **partition**; publish with `PartitionKey = aggregateId` so one aggregate's events land in one partition.
- **Cosmos DB change feed** — ordered per **logical partition key**; if the container is partitioned by aggregate ID, you get per-aggregate order.
- **Outbox polled directly** — ordered by the outbox's sequence column, if you process sequentially.

**What breaks ordering anyway:** consumers processing a partition with parallelism inside it; retries that re-deliver an earlier message after a later one was processed; multiple sources for one read model (Concept 46); and events for the same aggregate published through different topics.

**The defence is the version guard.** Every event carries its aggregate's version (Module 22, Concept 68 — the version bumps on every event), and every read-model row stores the last version applied:

```sql
-- Apply an event only if it's the next one — or, for "latest state" events, any newer one
UPDATE rm.OrderSummary
SET    Status = @Status, PlacedAt = @PlacedAt, Total = @Total, Version = @Version
WHERE  OrderId = @OrderId AND Version = @Version - 1;          -- strict: exactly the next version
-- rows affected = 0 → duplicate (Version >= @Version) or gap (Version < @Version - 1): check which
```

Two policies, depending on what the event carries:

- **Delta events** (`OrderLineAdded` with the added line) must be applied **strictly in sequence**. A gap means an event is missing — typically still in flight — so park the event and retry shortly, or re-read from the source.
- **State-carrying events** (the event includes the full new state of the fields the projection cares about) can use a **"newer wins"** rule: `WHERE Version < @Version`. Out-of-order older events are simply ignored. This is why event-carried state transfer makes projections dramatically more robust — design events to carry the resulting state when you can.

---

## Concept 40 — Idempotency in projectors

Every practical transport is **at-least-once** (Module 11): a projector will see duplicates after crashes, lease rebalancing, lock expiries and retries. The goal is an **exactly-once effect** from at-least-once delivery. Four techniques, often combined:

**1. Naturally idempotent writes.** Setting a field to a value is idempotent; incrementing a counter is not. Prefer writes of the form "set the row to the state described by this event":

```csharp
row.Status = "Placed"; row.Total = e.Total;          // idempotent: applying twice is harmless
row.OrderCount++;                                     // NOT idempotent: a duplicate double-counts
```

**2. Version guards** (Concept 39). If the row records the last applied version, a duplicate has `Version <= row.Version` and is skipped. One mechanism gives you both ordering and idempotency for per-aggregate rows.

**3. A processed-events table (inbox)**, written in the **same transaction** as the read-model change. The event ID is the primary key; a duplicate violates it or is found on lookup, and is skipped. Needed for read models that aggregate *across* aggregates — `CustomerOrderStats.OrderCount++` for every `OrderPlaced` — where a per-row version guard doesn't apply.

**4. Deterministic identities for inserted rows.** If a projection inserts a row per event (a timeline entry, a ledger line), key it by the event ID so a duplicate insert is a primary-key conflict, not a second row.

```csharp
public async Task Handle(OrderPlaced e, EventMeta meta, CancellationToken ct)
{
    // Cross-aggregate counter: needs an inbox row because a version guard can't protect it
    if (await db.ProcessedEvents.AnyAsync(p => p.Projection == Name && p.EventId == meta.EventId, ct))
        return;                                                                   // duplicate

    db.ProcessedEvents.Add(new ProcessedEvent(Name, meta.EventId, clock.GetUtcNow()));
    await db.CustomerStats                                                         // e: the flat contract (Guid, decimal)
        .Where(s => s.CustomerId == e.CustomerId)
        .ExecuteUpdateAsync(s => s.SetProperty(x => x.OrderCount, x => x.OrderCount + 1)
                                  .SetProperty(x => x.LifetimeValue, x => x.LifetimeValue + e.Total), ct);
    // The ExecuteUpdate runs immediately; the inbox row is saved by SaveChanges. Both must share one transaction —
    // the projector's batch transaction (Concept 41) — or a crash between them double-counts or drops the event.
}
```

The review heuristic: **for every projection handler, ask "what happens if this runs twice?"** If the answer is "a number is wrong," it needs a guard.

---

## Concept 41 — Checkpoints and atomicity

A projector must remember how far it got — its **checkpoint** (a stream position, an offset, a lease continuation token, the last outbox sequence). The correctness question is whether the checkpoint and the read-model changes are saved **atomically**.

**Case 1 — checkpoint and read model in the same database.** Save both in one transaction. Then after a crash, either both the changes and the checkpoint advanced or neither did: on restart you re-process from the old checkpoint, and the version guards absorb any replay. This is **exactly-once effect**, and it's the strongest guarantee available.

```csharp
public async Task ProcessBatchAsync(IReadOnlyList<StoredEvent> batch, CancellationToken ct)
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);

    foreach (var evt in batch)
        await dispatcher.ApplyAsync(evt, ct);               // projection handlers: version-guarded upserts, inbox rows

    var checkpoint = await db.Checkpoints.SingleAsync(c => c.Projection == Name, ct);
    checkpoint.Position = batch[^1].Position;              // how far we got
    checkpoint.UpdatedAt = clock.GetUtcNow();

    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);                               // read-model changes + checkpoint: all or nothing
}
```

**Case 2 — checkpoint and read model in different stores.** A Cosmos DB read model with a change feed processor (checkpoint in the lease container), an Azure AI Search index fed from Service Bus (checkpoint is message completion), a Redis read model with an Event Hubs processor (checkpoint in blob storage). No shared transaction is possible, so the order must be **write the read model, then advance the checkpoint** — and a crash between the two replays the batch. Correctness then rests entirely on **idempotent writes** (Concept 40). The change feed processor, for example, checkpoints only after your delegate completes successfully, and retries the same batch from the last checkpoint if it throws — at-least-once by design.

The two rules to state:

- **Never advance the checkpoint before the read-model write succeeds.** That's at-most-once — events silently lost on a crash.
- **Where the checkpoint can't be atomic with the write, the write must be idempotent.** There's no third option.

**Batching** interacts with both: larger batches amortize round trips and checkpoint writes (Concept 91) but increase the amount replayed after a crash and the time a poison event takes to isolate. A few hundred events per batch is a common starting point; measure.

---

## Concept 42 — Poison events and stuck projections

A projection that throws on one event has two bad options if nobody designed a third: **retry forever** (the partition stops; every later event for every aggregate in that partition waits — a *stuck projection*) or **skip and continue** (the read model silently diverges). Many default configurations choose the first: the Cosmos change feed processor retries the same batch until the delegate succeeds; a Service Bus session blocks behind a message that keeps failing until it reaches its max delivery count.

Design the third option explicitly:

**1. Classify failures.**
- **Transient** (timeouts, throttling, deadlocks): retry with backoff, in place.
- **Deterministic** (deserialization failure, a missing required field, a bug that throws for this event shape): retrying will never succeed.

**2. Park, don't drop, deterministic failures** — and do it **per aggregate** if order matters. Move the failed event to a dead-letter store with the exception, the projection name and the position. If the projection is order-sensitive, subsequent events *for the same aggregate* must also be parked (applying them would build on a missing step), while other aggregates continue. That keeps one bad order from stopping everyone else's.

**3. Alert on it.** A parked event is an incident with a known blast radius: "order 42's summary is stale." Alert on the dead-letter count per projection.

**4. Fix, then replay the parked events** through the corrected projection — which is safe only because projections are idempotent.

**5. Know your "stuck" signal.** A stuck projection shows up as **lag that grows without bound** while throughput drops to zero (Concept 50). Alert on lag *growth*, not only on lag level — a projector that has fallen 30 seconds behind and is catching up is fine; one that's 30 seconds behind and not moving is not.

A useful discipline: **every projector has a documented skip policy.** "Deserialization failures are dead-lettered with the aggregate parked; everything else retries with exponential backoff up to five minutes, then pages the on-call." Writing that sentence forces the decision before production makes it for you.

---

## Concept 43 — Rebuilding projections

You will need to rebuild read models — to fix a projection bug that corrupted rows, to add a new read model over historical data, to change a read model's schema, to move it to a different store. A read model that can't be rebuilt isn't a read model; it's a **second source of truth** with no authority, and it will eventually disagree with the first.

**What makes a rebuild possible** is the source (Concept 37):

- **Replayable history** (an event store, a Kafka topic with indefinite retention): drop the read model, reset the checkpoint to zero, replay. Simple and complete.
- **No replayable history** (outbox events already consumed, a Service Bus topic, CDC with short retention): rebuild in two phases — **bootstrap** the read model from the write side's *current state*, then **catch up** from the live feed.

The bootstrap-and-catch-up approach has one subtlety: you must record the feed's position **before** reading the snapshot of current state, then replay the feed from that position. Events that happened during the snapshot are then applied a second time — harmlessly, *if* the projection is idempotent and version-guarded (a row bootstrapped at version 17 ignores the replayed event for version 17). Get the order wrong — snapshot first, position second — and changes made between the two are lost forever.

**The operational shape: rebuild alongside, then swap.** Never rebuild in place while the read model is serving traffic.

```
1. Create OrderSummary_v2 (new schema or clean copy); register projector v2 with its own checkpoint.
2. v2 bootstraps / replays while v1 keeps serving queries and keeps up with live events.
3. v2 catches up to the head of the feed; its lag drops to ≈ v1's.
4. Compare: row counts, checksums, a sample of rows — v1 vs v2.
5. Switch queries to v2 (a view swap, a config flag, a synonym, a search-index alias).
6. Keep v1 for a rollback window; then drop it and its projector.
```

This is blue/green deployment applied to a read model, and it makes rebuilds a routine, low-risk operation rather than an outage.

Two more things a senior answer includes:

- **Rebuild time is a design parameter.** Replaying 400 million events at 20,000 events per second takes about 5.5 hours (Concept 91). If the business can't tolerate that for a bug fix, you need faster projectors, parallel replay by partition, or snapshots. Know the number before you need it.
- **Rebuilds must not trigger side effects.** A projector that also sends emails or calls external systems will re-send them during a replay. Projectors build read models and nothing else; side effects belong to other handlers (process managers, notification handlers) that are *not* replayed.

---

## Concept 44 — Evolving read-model schemas

Read models change as often as screens do (Concept 5), and — unlike the write model — they're **disposable**: any read model can be rebuilt from the source of truth. That changes how you evolve them.

- **Additive changes** (a new nullable column populated by new events): change in place, deploy the projector update, and — if historical rows need the value — backfill by rebuilding (Concept 43).
- **Breaking changes** (a column's meaning changes, the grain changes from per-order to per-order-line, a different store): create a new version alongside, rebuild into it, swap. Don't migrate a read model's data with hand-written SQL when you can recompute it from facts — recomputation is tested code; migration scripts are one-off code.
- **Query API versions are separate from read-model versions.** A v2 read model can serve both the v1 and v2 query endpoints through different projections of its columns, or a v1 endpoint can keep reading v1 until clients migrate.

Two ownership rules keep this manageable:

- **Only the projection writes to a read model.** No command handler, admin script or other service "fixes" read-model rows directly — fixes go through the source of truth and propagate.
- **Only the query side reads a read model.** No command handler uses it for decisions (Concept 55), and no other context queries it directly. If another context needs it, it gets a published contract (Module 22, Concept 28).

Those two rules are what make "drop it and rebuild it" safe.

---

## Concept 45 — Denormalization and fan-out

Read models are denormalized by design: the order list shows the customer's name, the product name and the shipment status without joins. But copying data creates a maintenance cost the moment the copied value changes.

**The fan-out problem.** An order read model copies `CustomerName` into each order row. A customer with 12,000 historical orders changes their name. The projection must now update 12,000 rows — one small event, a large write. At scale, a single `ProductRenamed` event for a popular product can touch millions of rows.

Three strategies:

| Strategy | Read cost | Write cost on change | Freshness of copies | Use when |
|---|---|---|---|---|
| **Copy into every row** | Lowest — no joins | O(rows referencing it) | Lags by the fan-out time | The copied value rarely changes, or history should keep the old value |
| **Reference table + join at read time** | One cheap join (by key) | O(1) — update one row | Immediate within the read store | The value changes often or is referenced by many rows |
| **Copy, updated lazily in batches** | Lowest | Spread over time | Eventually, with a longer tail | Very high fan-out; staleness of old rows is acceptable |

The reference-table middle ground is often the best: the read store holds its own `CustomerNames(CustomerId, Name)` table maintained from customer events, and order queries join to it. Denormalize the *expensive* joins — across contexts, across stores, across large tables — not every join.

**The business question hiding here:** should the historical order show the customer's name *now* or *at the time of the order*? For an invoice, legally, it's the name at the time — the "fan-out" is wrong, and the copy should never be updated. For a support screen, it's the current name. That's a product decision, and asking it is a senior signal: *"Before I decide how to propagate the rename, is this a snapshot of the past or a view of the present?"*

---

## Concept 46 — Multi-source read models

The most valuable read models often combine several contexts: an "order tracking" page showing Ordering's order, Billing's payment status and Shipping's delivery estimate. Module 21 (Concept 38) established that such a read model is legitimate — a read-only projection, owned by its consumer, rebuildable from the sources — and that reading the other contexts' tables directly is not.

The problems specific to multiple sources:

**1. No ordering across sources.** `ShipmentDispatched` (from Shipping) can arrive before `OrderPlaced` (from Ordering) has been projected — different topics, different partitions, different lags. The projection must handle facts for an entity it hasn't seen yet.

**2. Strategy: each source owns its columns, and rows are upserted.** Whichever event arrives first creates the row with its own columns filled and the others null. No event overwrites another source's columns.

```sql
-- Shipping's handler: creates the row if Ordering's event hasn't arrived yet; touches only Shipping's columns
MERGE rm.OrderTracking AS t
USING (SELECT @OrderId AS OrderId) AS s ON t.OrderId = s.OrderId
WHEN MATCHED AND (t.ShippingVersion IS NULL OR t.ShippingVersion < @Version) THEN
    UPDATE SET ShipmentStatus = @Status, EstimatedDelivery = @Eta, ShippingVersion = @Version
WHEN NOT MATCHED THEN
    INSERT (OrderId, ShipmentStatus, EstimatedDelivery, ShippingVersion)
    VALUES (@OrderId, @Status, @Eta, @Version);
```

(On SQL Server, use `MERGE ... WITH (HOLDLOCK)` or an `UPDATE`-then-`INSERT` with a unique key and retry, because `MERGE` alone can race under concurrency.)

**3. One version column per source.** A single `Version` can't order facts from independent aggregates in different contexts. Each source gets its own guard.

**4. Completeness is a state.** A row with Shipping's columns but no Ordering columns is *incomplete*. The query side must decide what to do: hide incomplete rows, show them with placeholders, or show a "processing" state (Concept 56). Make it explicit — an `IsComplete` computed from which sources have reported — rather than letting the UI render nulls.

**5. Correlation keys.** Every source's events must carry the key the read model is organized by — which is a contract requirement on the publishers. If Shipping's events carry only a `ShipmentId`, the tracking projection needs a mapping from shipment to order, itself maintained from an event (`ShipmentCreated { ShipmentId, OrderId }`).

**6. Ownership.** The consumer (the team that owns the tracking page) owns the projection, subscribes to the published contracts of each source, and is responsible for rebuilding it. The sources don't know it exists.

---

## Concept 47 — Choosing a read store

At level 3, each read model can live where its queries are best answered. The decision is by **query shape**, not by fashion:

| Query shape | Store | Why | Azure/.NET options |
|---|---|---|---|
| Lists, details, filters and joins on a few known keys; transactional updates | **Relational tables** | Mature, indexable, joins available, SQL skills common | Azure SQL, PostgreSQL flexible server; EF Core, Dapper |
| "Give me this whole document by key" — detail pages, API resources | **Document store** | One read, no joins, schema-flexible, partitioned for scale | Cosmos DB for NoSQL, PostgreSQL `jsonb` |
| Full-text, fuzzy matching, facets, relevance ranking, many optional filters | **Search index** | Inverted indexes answer combinatorial filters that B-trees can't | Azure AI Search, Elasticsearch/OpenSearch |
| Hot keys, leaderboards, counters, sessions, sub-millisecond reads | **Cache / in-memory store** | Memory speed, rich data structures | Azure Managed Redis (Module 10), HybridCache |
| Aggregations over large history, time series, ad-hoc analytics | **Columnar / analytical store** | Scans and aggregates billions of rows efficiently | Microsoft Fabric, Azure Data Explorer, Azure Databricks |
| Relationships and traversals ("friends of friends who bought…") | **Graph** | Traversal as a first-class operation | Cosmos DB for Apache Gremlin, PostgreSQL recursive CTEs for simpler cases |
| Geospatial proximity | Spatial indexes | Distance and containment queries | Azure SQL geography, PostGIS, Cosmos DB spatial |

Two principles:

- **Default to the store you already run.** A second relational schema in the database you already operate is a much smaller commitment than a new technology. Every new store adds backup, monitoring, security, cost and on-call knowledge. Polyglot read stores should be earned by a query shape the existing store genuinely can't answer.
- **One read model per question type, not per store.** It's normal for one context to have three read models in the same SQL database and one search index — the number of stores is a consequence of query shapes, not a goal.

---

## Concept 48 — Cosmos DB on both sides

Cosmos DB is a natural fit for CQRS because partitioning forces the question: **a container can be efficiently queried only by its partition key.** The write model wants one key, the read models want others — which is the CQRS argument made physical.

**The write side** partitions by aggregate ID (Module 22, Concept 75): one aggregate = one logical partition = transactional batch scope = the natural unit of change-feed ordering.

**The read side** has two options:

**Option 1 — Global Secondary Indexes (GA, June 2026).** A GSI is a **read-only container** that Cosmos DB maintains automatically from the source container's change feed, with a **different partition key**, its own throughput and its own indexing policy. It turns "find orders by customer" — a cross-partition query on a container partitioned by order ID — into a single-partition query on a GSI partitioned by customer ID. What to know before choosing it:

- It's **eventually consistent** with the source, regardless of the account's consistency level; monitor the *Global Secondary Index Propagation Latency* metric and alert on it.
- The mapping is **one-to-one**: one GSI item per source item. The definition query can project properties but can't filter (`WHERE`), join, aggregate, sort or call functions.
- The source container and definition **can't be changed** after creation — a schema change means a new GSI.
- The GSI container **must use autoscale** throughput; change-feed reads cost RUs on the source, GSI writes cost RUs on the GSI, and **replace and delete operations on the source cost an extra 50–100% RUs** (to persist previous versions).

That makes a GSI the right tool when the read model is **the same documents, queried by a different key** — and the wrong tool when it must aggregate, join, filter or reshape.

**Option 2 — your own projector on the change feed processor**, for read models that aggregate or reshape:

```csharp
var processor = cosmos.GetContainer("ordering", "orders")
    .GetChangeFeedProcessorBuilder<OrderDocument>("customer-dashboard", HandleChangesAsync)
    .WithInstanceName(Environment.MachineName)          // unique per host instance
    .WithLeaseContainer(cosmos.GetContainer("ordering", "leases"))
    .WithStartTime(DateTime.MinValue.ToUniversalTime())  // on first start only: from the beginning (bootstrap)
    .Build();

await processor.StartAsync();

async Task HandleChangesAsync(ChangeFeedProcessorContext ctx, IReadOnlyCollection<OrderDocument> changes, CancellationToken ct)
{
    var dashboards = cosmos.GetContainer("ordering", "customer-dashboards");   // partitioned by /customerId
    foreach (var order in changes)                                              // ordered per partition key (order ID)
    {
        // Idempotent, version-guarded upsert of this order's contribution into the customer's dashboard document.
        // Throwing here makes the processor retry the whole batch from the last checkpoint — at-least-once.
        await ApplyToDashboardAsync(dashboards, order, ct);
    }
}
```

The change feed processor distributes partition-key ranges across instances via the lease container, checkpoints after your delegate completes, and gives you a **change feed estimator** for lag. In latest-version mode it doesn't deliver deletes — use soft deletes with a TTL, or the all-versions-and-deletes mode (which needs continuous backup and only reaches back as far as its retention).

A detail that bites: **the change feed gives you document states, not events.** If two updates to an order happen between reads in latest-version mode, you see only the second. That's fine for "current state" read models and wrong for anything that must count transitions — which is a reason to write events (as documents in an events container, or via an outbox) when the read side needs history.

---

## Concept 49 — Search indexes as read models

When a screen has many optional filters, free-text search, facets ("Status: Placed (1,204) · Shipped (3,891)") or relevance ranking, the right read model is a **search index**. Relational indexes can't serve arbitrary filter combinations efficiently (Concept 29); inverted indexes can.

**Azure AI Search** offers two ways to keep an index current:

**Indexers (pull).** The service periodically reads a supported source — Azure SQL, Cosmos DB, Blob Storage and others — using **change detection** (a high-water-mark column such as a `rowversion` or `_ts`, or SQL Server's integrated change tracking) and **deletion detection** (typically a soft-delete column). No projector code, but freshness is bounded by the schedule (minutes, not seconds), the indexer reads your source's schema directly (coupling), and complex reshaping is limited.

**Push API.** Your projector writes documents to the index as events arrive — the same pattern as any level-3 projection, with freshness in seconds and full control over the document shape. You own batching, retries, idempotency (documents are upserted by key, so re-pushing the same document is naturally idempotent) and rebuilds.

The patterns worth knowing:

- **The index document is a denormalized read model** — one document per searchable thing, with every field a filter, facet or result card needs, including data from other aggregates maintained through the projection.
- **Search results carry keys, not truth.** A common, robust design: the index returns matching IDs and display fields; the detail page loads from the authoritative read model. The index can be seconds stale without a user acting on stale detail data.
- **Schema changes usually mean rebuilding the index** — many field changes (type, analyzers, filterable/facetable attributes) require a new index. Build the new index alongside, repopulate it, and switch queries to it (Concept 43's swap).

---

## Concept 50 — Measuring consistency

"Eventually consistent" is only acceptable if you know how eventual. **Lag is the read side's primary health metric**, and a level-3 system without it is operating blind.

**Two ways to measure lag:**

- **Position lag** — how far the projector's checkpoint is behind the head of its source (messages or events outstanding). Tells you backlog size; doesn't tell you user-visible staleness directly.
- **Time lag** — `now − occurredAt` of the event most recently applied, or — better — per applied event, `appliedAt − occurredAt`, recorded as a histogram. This *is* the staleness users experience, and it's what the SLO should be written in.

```csharp
private static readonly Histogram<double> ProjectionLag =
    Meter.CreateHistogram<double>("projection.lag", unit: "s", description: "Time from event occurrence to read-model apply");

// After a batch commits:
foreach (var evt in batch)
    ProjectionLag.Record((clock.GetUtcNow() - evt.OccurredAt).TotalSeconds,
        new KeyValuePair<string, object?>("projection", Name));
```

**Platform-provided lag signals:** the Cosmos DB change feed estimator (per lease), the Global Secondary Index propagation latency metric, Service Bus active message count and oldest-message age, Event Hubs consumer-group lag, SQL replica lag (`redo_queue_size` and related DMVs, or the Azure monitoring metrics).

**End-to-end freshness canaries.** Per-projector metrics miss problems between components (a relay that stopped publishing, a subscription that was deleted). A canary closes the gap: every few seconds, a heartbeat command writes a tiny change through the real write path; each read model records when it saw the heartbeat; an alert fires when any read model's canary age exceeds its budget. It measures exactly what users experience — "how old is the newest thing I can see?"

**Write the SLO in business terms and alert on two things:**

- **Level:** "99% of order-status changes visible on the tracking page within 2 seconds; alert if p99 lag > 10 s for 5 minutes."
- **Growth:** "alert if lag has increased monotonically for 3 minutes" — the signature of a stuck projection (Concept 42), which may still be under the level threshold when it starts.

The SLO number comes from the business (Concept 57), and the monitoring is how you prove you're meeting it — Module 28 takes SLOs and error budgets in depth.

---

# Part E — Eventual consistency, as the user sees it

The moment a read path is served from anything that updates asynchronously — a level-3 read model, a replica, a cache — users can see the past. Most of the time nobody notices. The time they always notice is **right after they changed something**. Module 7 gave you the session guarantees (read-your-writes, monotonic reads) as theory; Module 22 (Concept 84) said eventual consistency is a UX problem. This part turns both into concrete techniques, including the one that answers most interview follow-ups: the consistency token.

---

## Concept 51 — The stale-read bug

The canonical bug report, which every level-3 system receives in its first month:

> "I added an item to my order and when the page reloaded it wasn't there. I clicked Add again and now I have two."

What happened:

```
t=0 ms     POST /orders/42/lines        → command commits, version 17; outbox row written
t=40 ms    201 Created                  → client navigates / refetches
t=55 ms    GET /orders/42               → read model still at version 16 (projector hasn't run yet)
t=60 ms    page renders without the new line
t=180 ms   projector applies version 17 → read model now correct — but the user is already confused
t=2 s      user clicks Add again        → a second, genuinely new line (idempotency keys can't help:
                                          it's a new user action, not a retry)
```

Why it's so common: the client's refetch happens tens of milliseconds after the command returns, and projection lag — outbox relay polling interval, broker delivery, batch accumulation, processing — is typically hundreds of milliseconds, sometimes seconds. **The user's own next read is the read most likely to be stale**, because it's the one that happens soonest after a write.

The same bug appears at level 1 whenever reads go through a replica (Concept 31) or a cache (Concept 30), which is why "we're not doing async projections, so we don't have this problem" is often untrue.

Two consequences worth naming:

- **It generates duplicate actions and support tickets**, not just confusion — users "fix" what looks like a failed save.
- **It erodes trust in the whole system.** Once users have seen a save "disappear," they start refreshing, double-checking and phoning support. The cost of eventual consistency is paid in trust, not milliseconds.

---

## Concept 52 — Read-your-writes, six ways

The session guarantee you need is **read-your-writes**: after a user's write, that user's subsequent reads reflect it. Other users may see it later; *this* user must see it now. Six techniques, which combine well:

| # | Technique | How | Cost | Best for |
|---|---|---|---|---|
| 1 | **Use the command's result** | Client renders the change it just made from the command response (and merges it into what it has) | Client logic; the command returns enough to render (ID, version, maybe the created item's display data) | Most interactive edits; the cheapest fix |
| 2 | **Route the user's reads to the source of truth** | For a short window after a write (or until the read side catches up), that user's queries go to the primary / write-side tables | A second query path; primary load for recent writers | Replicas and caches at level 1 |
| 3 | **Consistency token** | Command returns a version/position; the next query demands a read model at least that fresh and waits briefly if needed | Query-side waiting logic; the read model must record versions/positions | Level 3 detail views; the general-purpose fix (Concept 53) |
| 4 | **Project the user's own view synchronously** | The "my stuff" read model is level 2 (same transaction); shared views stay level 3 | Write latency for that projection | "My orders," "my cart," "my requests" |
| 5 | **Optimistic UI** | Client shows the change immediately, marked pending, and reconciles when the read model catches up | UI state management; handling rejections gracefully | Chat, collaboration, list additions |
| 6 | **Push when caught up** | Server notifies the client (SignalR, SSE) when the projection has applied the user's change; client refetches then | A push channel; subscription management | Longer lags; async commands (Concept 21) |

**Technique 1 deserves emphasis because it's so often overlooked.** If `POST /orders/42/lines` returns `{ "lineNumber": 3, "sku": "...", "quantity": 2, "version": 17 }`, the client can add the line to the page it's already showing without refetching anything. No server-side consistency machinery is needed. The command still returns information *about itself* (Concept 12) — the line it created — not a view of the system.

**Technique 6 got cheaper in .NET 10**, which added `TypedResults.ServerSentEvents` for streaming events over a plain HTTP response — a lightweight alternative to SignalR for one-way "your data is ready" notifications:

```csharp
app.MapGet("/me/read-model-updates", (IReadModelNotifications notifications, ClaimsPrincipal user, CancellationToken ct) =>
    TypedResults.ServerSentEvents(
        notifications.StreamForUserAsync(user.GetUserId(), ct),   // IAsyncEnumerable<ReadModelUpdated>, fed by the projector
        eventType: "read-model-updated"));
```

The projector publishes "applied version 17 of order 42" to a per-user channel (Redis pub/sub, Azure Web PubSub, or an in-process `Channel<T>` on a single instance); the client refetches when its version arrives.

In practice most systems use **1 for the common case, 3 for detail views reached right after a command, and 6 for asynchronous commands** — and choose technique 2 when the lag comes from replicas rather than projections.

---

## Concept 53 — The consistency token

The general-purpose read-your-writes mechanism, and the one to describe in a design round. The idea: **the command tells the client how fresh a read must be; the query refuses to answer from anything older.**

**Step 1 — commands return a freshness marker.**
- For per-aggregate reads: the aggregate **version** the command produced (`17`).
- For cross-aggregate reads (lists, dashboards): a **position** in the source feed — the outbox sequence number of the command's last event, or the event store's global position.

**Step 2 — read models record what they've applied.** Per-aggregate rows carry `Version` (they already do, for idempotency — Concept 39). Each projection records its **checkpoint position** (it already does — Concept 41).

**Step 3 — queries accept a minimum and wait, briefly, if behind.**

```csharp
public sealed record GetOrderDetails(OrderId OrderId, int? MinVersion = null);

internal sealed class GetOrderDetailsHandler(ReadModelsDbContext db, TimeProvider clock)
    : IQueryHandler<GetOrderDetails, ReadResult<OrderDetailsDto>>
{
    private static readonly TimeSpan MaxWait = TimeSpan.FromSeconds(2);
    private static readonly TimeSpan PollInterval = TimeSpan.FromMilliseconds(50);

    public async Task<ReadResult<OrderDetailsDto>> Handle(GetOrderDetails q, CancellationToken ct)
    {
        var deadline = clock.GetUtcNow() + MaxWait;
        while (true)
        {
            var row = await db.OrderDetails.AsNoTracking()
                              .SingleOrDefaultAsync(r => r.OrderId == q.OrderId.Value, ct);

            if (q.MinVersion is null)                                           // no token: answer now, never wait
                return row is null
                    ? ReadResult<OrderDetailsDto>.NotFound()
                    : ReadResult<OrderDetailsDto>.Fresh(row.ToDto());

            if (row is not null && row.Version >= q.MinVersion)
                return ReadResult<OrderDetailsDto>.Fresh(row.ToDto());

            if (clock.GetUtcNow() >= deadline)
                return row is null
                    ? ReadResult<OrderDetailsDto>.NotYetAvailable()                 // token says it exists: "not yet", not 404
                    : ReadResult<OrderDetailsDto>.Stale(row.ToDto(), row.Version);  // caller decides: show + banner, or 503

            await Task.Delay(PollInterval, clock, ct);                          // TimeProvider overload: testable with FakeTimeProvider
        }
    }
}
```

On the HTTP side, the client passes the token as a query parameter or header (`GET /orders/42?minVersion=17`), and the endpoint maps the outcome: fresh → 200; not found without a token → 404; stale after waiting → 200 with a staleness indicator (or `503` with `Retry-After`, if showing stale data would mislead); not yet available → a retry hint (`202`, or `404` with `Retry-After`) rather than a flat 404, because "doesn't exist" and "doesn't exist *yet*" are different answers — and only a caller holding a token can know the second case applies.

Note the first branch: **a query without a token must never enter the wait loop.** Otherwise every request for a genuinely missing order would hang for two seconds before answering — an easy bug to write and a nasty one under a crawler or a broken client link.

Design details that matter:

- **Bound the wait** — a second or two at most. The token turns a stale read into a slightly slower fresh read in the common case; it must not turn a projection outage into hung requests. When the wait expires, return something honest.
- **Poll cheaply** — a primary-key lookup on the read model every 50 ms for up to two seconds is trivial load. For many waiters, a notification from the projector (Concept 52, technique 6) avoids polling entirely.
- **Only the writer sends a token.** Other users' queries have no minimum and never wait.
- **Positions for lists need the checkpoint.** For "my orders" after placing an order, the command returns the outbox position `P`; the query checks `checkpoint(OrdersByCustomer) >= P` before answering. That requires the projection to process the feed in position order — true for a single ordered feed, not for a partitioned one without per-partition positions.

This is the same idea as **Cosmos DB's session consistency**: the SDK carries a session token from each write and ensures reads in the same session see a replica at least that fresh (Module 7). The consistency token is session consistency implemented at the application level, across whatever stores your read side uses.

---

## Concept 54 — Monotonic reads

Read-your-writes is about *your* writes. **Monotonic reads** is about not going backwards: once a user has seen version 17, they shouldn't later see version 16.

How time goes backwards in a CQRS system:

- **Replicas with different lag.** Request 1 hits replica A (caught up), request 2 hits replica B (behind). The user sees an order as "Shipped," refreshes, and sees "Placed."
- **Multiple read-model instances.** Two projector deployments — or blue and green read models mid-rebuild (Concept 43) — at different positions behind a load balancer.
- **Caches with different ages.** An L1 cache on instance 1 holds an older value than instance 2's.
- **Search vs detail.** The search results (updated in 5 s) show a status the detail page (updated in 200 ms) has already moved past — not strictly monotonicity, but experienced the same way.

Fixes:

- **Carry the last-seen version on every read**, not only after writes. If the client always sends the highest version it has seen for an entity, the consistency token mechanism (Concept 53) enforces monotonicity for free.
- **Session affinity** to a replica or instance, where the infrastructure supports it — simple, but it breaks on failover and scales poorly.
- **One freshness source per screen.** Don't let a single page mix data from read models with very different lags without making the difference visible.
- **Rebuild swaps happen only when the new read model has caught up** — never switch reads to a read model that's behind the old one.

Monotonic reads matter less than read-your-writes for most business screens, and much more for anything showing progress or status (tracking pages, job status, workflow state), where going backwards looks like a bug in the business process, not a UI glitch.

---

## Concept 55 — Commands never trust the read model

The most important safety rule in CQRS, and a common source of subtle production bugs:

> **Decisions are made against the write model's current state, inside the command's transaction. The read model is never an input to an invariant.**

The failure mode: a command handler or validator checks something against the read model because it's convenient — it's already shaped for the question.

```csharp
// WRONG: deciding from a read model
public async Task<ReserveOutcome> Handle(ReserveStock cmd, CancellationToken ct)
{
    var available = await readDb.ProductAvailability                 // projected, possibly seconds old
                                .Where(p => p.Sku == cmd.Sku.Value)
                                .Select(p => p.Available).SingleAsync(ct);
    if (available < cmd.Quantity.Value) return ReserveOutcome.InsufficientStock;
    ...                                                               // two concurrent reservations both pass: oversold
}
```

The read model is stale by design (level 3) or at least not locked for this transaction (level 1–2), so the check can pass for two concurrent commands, or pass on data that was true a second ago. The invariant — "never reserve more than is available" — must be checked on the aggregate that owns it (Module 22, Concepts 56–58), or enforced by a conditional write (Module 22, Concept 66).

What the read model *may* legitimately do for commands:

- **Pre-validate for UX.** Disable the *Reserve* button when the product page says "out of stock." It's a hint to avoid obviously doomed commands; the command still checks for real.
- **Supply the expected version** the user saw, so the command can detect that the user acted on stale information (Concept 20).
- **Supply display data** that doesn't affect the decision.

**The quote pattern** handles the case where the business *does* want to honour what the user saw — "the price you were shown is the price you pay." The read side shows a price; the *write side* issues a **quote** (an aggregate or a signed token: price, SKU, expiry) when the user reaches checkout; the command references the quote ID; the write side validates the quote (exists, unexpired, not used) — not the read model. The user's view becomes an explicit, time-limited commitment enforced by the write model, rather than an accidental dependency on a stale projection.

---

## Concept 56 — Designing UX for lag

Technical techniques reduce staleness; UX design makes the remainder honest. Jakob Nielsen's classic response-time limits give a useful calibration: about **0.1 s** feels instantaneous, about **1 s** keeps the user's flow of thought, and about **10 s** is the limit of keeping attention on the task. Map your lag SLO onto them:

| Lag you actually have | What users experience | Design response |
|---|---|---|
| < 100 ms typical | Invisible, except on immediate refetch | Technique 1 (use the command's result) is enough |
| 100 ms – 1 s | Occasional "where did it go?" on refetch | Consistency token or optimistic UI for the writer's own views |
| 1 – 10 s | Noticeable; users re-click and re-submit | Explicit pending state; disable resubmission; push when done |
| > 10 s | Users leave the page | Treat the command as asynchronous: "we've received your request," status page, notification (Concept 21) |

Patterns that work:

- **Say what's true.** "Order received — confirming availability…" rather than "Order confirmed!" before the confirmation exists. The pending state is a real domain state (Module 22, Concept 84), and the UI should name it.
- **Prevent the duplicate action** the stale read provokes: disable the button after submission, show the pending item, and — for creates — use client-generated IDs so an accidental second submission is idempotent (Concept 19).
- **Show incompleteness explicitly.** A multi-source read model's row that has Ordering's data but not yet Shipping's (Concept 46) shows "Delivery estimate: calculating…", not an empty cell or a zero.
- **Timestamp the data** where staleness matters for interpretation: "Updated 12 seconds ago" on a dashboard sets expectations and makes a stuck projection visible to users before monitoring notices.
- **Design the rejection path.** With asynchronous commands, rejections arrive later. Where does the user find out that their order couldn't be fulfilled? An in-app notification, an email, a status on the order list — decided up front, not when the first complaint arrives.

---

## Concept 57 — Deciding per query

The staleness a read path can tolerate is a **business decision**, and it's the input that decides the level for that path (Concept 8). The question to ask stakeholders — never "is eventual consistency OK?", which invites a reflexive "no" — but:

> *"If this screen showed data that was five seconds old, what's the worst thing that would happen? What about a minute? An hour?"*

A worked classification for an e-commerce context:

| Read path | Tolerable staleness | Why | Level |
|---|---|---|---|
| Order confirmation page right after checkout | 0 for the buyer's own order | User just acted; must see their order | Level 1 read of the write tables, or level 3 + consistency token |
| "My orders" list | ~1 s for the owner; seconds for others | Owner expects to see new orders; nobody else is looking | Level 2 (synchronous, per customer) or level 3 + token |
| Product page stock indicator ("Only 3 left") | 5–30 s | A hint; the reservation checks for real (Concept 55) | Level 3 or cache |
| Product search and facets | 30 s – a few minutes | Catalogue changes slowly; search is exploratory | Search index, push or indexer |
| Account balance shown before a payment | 0 | The user decides how much to pay based on it | Level 1 against the primary |
| Customer-service dashboard | 5–10 s | Agents refresh often; they act through commands | Level 3 |
| Monthly revenue report | Hours | Analytical; reconciled later anyway | Analytical store, batch |

The pattern that emerges is typical: **the writer's own immediate views need freshness; shared, exploratory and analytical views don't; anything a user makes a financial or legal decision from needs to come from the source of truth.** Writing the table down — in an ADR (Module 31), with the business's sign-off on the numbers — is the artifact that turns "eventual consistency" from a risk into a specification, and it's what the lag SLOs of Concept 50 are measured against.

---

# Part F — MediatR and in-process dispatch

Parts A–E were about the pattern. Part F is about the tool most .NET teams reach for when they implement it. MediatR deserves a proper look — its model, what it does on every call, where its real value lies and where it quietly causes harm — because interviewers ask about it constantly, because most codebases you'll review use it, and because you can't make a sensible decision about the licensing change (Part G) without understanding what you'd be replacing.

---

## Concept 58 — The mediator pattern vs MediatR

The name causes confusion, so start with the definitions.

**The Mediator pattern** (Gang of Four, 1994) is a *behavioural* pattern for reducing coupling among a set of **colleague objects** that interact in complex ways. Instead of each colleague referencing the others, they all talk to a mediator, which encapsulates how they interact. The canonical example is a dialog box: the list box, the text field and the OK button don't know about each other; the dialog (the mediator) knows that selecting an item fills the text field and enables the button. The mediator contains **coordination logic** specific to that set of colleagues.

**MediatR** is something different: an **in-process request dispatcher** with a **pipeline**. You send it a request object; it finds the single handler registered for that request type, wraps it in any pipeline behaviors, and invokes it. It contains no coordination logic of its own — it doesn't know what your handlers do or how they relate. In pattern terms it's closer to a **command dispatcher** (or "command bus") combined with **chain of responsibility** / decorators for the pipeline. Jimmy Bogard describes it as a "simple, unambitious mediator implementation."

Why the distinction matters in an interview:

- **"We use the mediator pattern" usually means "we use a request dispatcher."** Saying so precisely signals that you know what the tool actually does.
- **The decoupling MediatR provides is narrow.** It decouples the *caller* (a controller, a job, a message consumer) from the *concrete handler type* — the caller depends on a request type and `ISender`. It doesn't decouple handlers from each other, from the database, or from the domain; that's your design's job.
- **A request dispatcher is not an architecture.** It changes how handlers are *invoked* and *wrapped*. Whether the system has separated read and write models, bounded contexts or aggregates is entirely independent of it (Concept 95).

---

## Concept 59 — MediatR's model

MediatR's surface is small. As of the 13/14 lines (September 2026):

| Type | Role | Handler contract |
|---|---|---|
| `IRequest<TResponse>` | A request with a response — command or query | `IRequestHandler<TRequest, TResponse>`: `Task<TResponse> Handle(TRequest, CancellationToken)` |
| `IRequest` | A request with no response | `IRequestHandler<TRequest>`: `Task Handle(TRequest, CancellationToken)` |
| `INotification` | A message for **zero or more** handlers | `INotificationHandler<TNotification>`: `Task Handle(TNotification, CancellationToken)` |
| `IStreamRequest<TResponse>` | A request answered with a stream | `IStreamRequestHandler<TRequest, TResponse>`: `IAsyncEnumerable<TResponse> Handle(...)` |
| `ISender` / `IPublisher` / `IMediator` | Entry points: `Send`, `CreateStream` / `Publish` / both | — |
| `IPipelineBehavior<TRequest, TResponse>` | Wraps request handling — the middleware | `Task<TResponse> Handle(TRequest, RequestHandlerDelegate<TResponse> next, CancellationToken)` |
| `IStreamPipelineBehavior<,>` | The same, for streams | — |
| `IRequestPreProcessor<TRequest>` / `IRequestPostProcessor<TRequest, TResponse>` | Simplified before/after hooks (implemented as behaviors) | `Task Process(...)` |
| `IRequestExceptionHandler<TRequest, TResponse, TException>` | May handle an exception and supply a response | `SetHandled(response)` on a state object |
| `IRequestExceptionAction<TRequest, TException>` | Runs on an exception without handling it (logging) | — |
| `INotificationPublisher` | Strategy for invoking notification handlers | Built-in: `ForeachAwaitPublisher` (default, sequential), `TaskWhenAllPublisher` (concurrent) |

Registration (MediatR 12+, where the DI integration moved into the main package):

```csharp
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<PlaceOrder>();   // scans for handlers, processors, exception handlers
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));              // order of registration = order of execution
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    cfg.AddOpenBehavior(typeof(UnitOfWorkBehavior<,>));
    // Licence key: cfg.LicenseKey = ..., or the MEDIATR_LICENSE_KEY / LUCKYPENNY_LICENSE_KEY env vars (14.2+)
});
```

By default `IMediator`, `ISender`, `IPublisher`, handlers and scanned types are registered **transient**. A request and handler look like this:

```csharp
public sealed record PlaceOrder(OrderId OrderId, IdempotencyKey Key) : IRequest<PlaceOrderOutcome>;

internal sealed class PlaceOrderHandler(IOrderRepository orders, ICreditService credit, TimeProvider clock)
    : IRequestHandler<PlaceOrder, PlaceOrderOutcome>
{
    public async Task<PlaceOrderOutcome> Handle(PlaceOrder cmd, CancellationToken ct)
    {
        var order = await orders.GetAsync(cmd.OrderId, ct);
        if (order is null) return new PlaceOrderOutcome.NotFound();
        var available = await credit.GetAvailableCreditAsync(order.CustomerId, order.Total.Currency, ct);
        return order.Place(available, clock.GetUtcNow()) switch
        {
            PlaceOutcome.Placed             => new PlaceOrderOutcome.Placed(order.Version),
            PlaceOutcome.NotDraft           => new PlaceOrderOutcome.NotDraft(order.Status),
            PlaceOutcome.Empty              => new PlaceOrderOutcome.Empty(),
            PlaceOutcome.InsufficientCredit => new PlaceOrderOutcome.InsufficientCredit(available, order.Total),
            _ => throw new UnreachableException(),
        };
        // No SaveChanges here: in this codebase the UnitOfWorkBehavior commits (Concept 65)
    }
}

// Endpoint: depends on ISender and the request type — not on the handler
app.MapPost("/orders/{id}/place", async (OrderId id, [FromHeader(Name = "Idempotency-Key")] string key,
                                        ISender sender, CancellationToken ct) =>
    ToHttp(await sender.Send(new PlaceOrder(id, new IdempotencyKey(key)), ct), id));
```

---

## Concept 60 — How dispatch works inside

Knowing what happens on each `Send` lets you reason about MediatR's costs, its failure modes and why the alternatives differ.

On `sender.Send(new PlaceOrder(...), ct)`:

1. **Wrapper lookup.** `Mediator` keeps a static `ConcurrentDictionary<Type, …>` keyed by request type. On the first `Send` of a given type it builds a closed generic wrapper — `RequestHandlerWrapperImpl<PlaceOrder, PlaceOrderOutcome>` — via `MakeGenericType` and `Activator.CreateInstance`, and caches it. Later calls hit the cache.
2. **Handler resolution.** The wrapper asks the `IServiceProvider` for `IRequestHandler<PlaceOrder, PlaceOrderOutcome>` — on **every call**. With transient registration, that constructs a new handler and its transient dependencies.
3. **Behavior resolution.** It asks for all `IPipelineBehavior<PlaceOrder, PlaceOrderOutcome>` — again on every call — including closed instances of each open behavior whose generic constraints the request satisfies.
4. **Pipeline composition.** It reverses the behavior list and folds it into nested delegates, each capturing the next — so the **first-registered behavior is the outermost**:

```csharp
// Paraphrasing the wrapper's core:
Task<TResponse> Handler(CancellationToken t = default) =>
    serviceProvider.GetRequiredService<IRequestHandler<TRequest, TResponse>>()
                   .Handle((TRequest)request, t == default ? cancellationToken : t);

return serviceProvider.GetServices<IPipelineBehavior<TRequest, TResponse>>()
    .Reverse()
    .Aggregate((RequestHandlerDelegate<TResponse>)Handler,
               (next, behavior) => t => behavior.Handle((TRequest)request, next, t == default ? cancellationToken : t))();
```

5. **Invocation.** The outermost delegate runs; each behavior decides whether and when to call `next`.

(Recent versions let `next` take a `CancellationToken`, so a behavior can substitute a linked token — useful for timeouts, Concept 67.)

The consequences:

- **Per-call overhead is small in absolute terms and large relative to a direct call.** Dictionary lookup, DI resolution of the handler and every behavior, a LINQ reversal, a closure and delegate per behavior, and the async state machines — typically well under a few microseconds and a handful of allocations. That's irrelevant next to a database round trip of a millisecond, and it's why "MediatR is slow" is usually the wrong objection. It matters in tight in-memory loops, very high-throughput services, and for allocation-sensitive code (Module 17).
- **Missing handlers are runtime errors.** Nothing checks at compile time that `PlaceOrder` has a handler; you find out when the first request arrives (Concept 71's registration test catches it in CI).
- **Reflection-based wiring is Native AOT-hostile.** Assembly scanning and `MakeGenericType` over arbitrary types don't survive trimming reliably. Source-generated alternatives exist precisely for this (Concept 77).
- **Behaviors are resolved per request type.** Constrained open behaviors (`where TRequest : ICommand`) are applied only to matching requests — a powerful way to target concerns (Concept 65), and dependent on the container honouring generic constraints (Microsoft's DI does for enumerable resolution).

---

## Concept 61 — What MediatR buys

Stated honestly, the benefits are real and specific:

**1. A uniform handler shape.** Every use case is a request type and a handler class. New team members learn one pattern; code review has one shape to check; the list of request types is the list of things the application does. This is most of the value of vertical slices (Concept 72).

**2. One place for cross-cutting concerns.** Logging, tracing, validation, authorization, idempotency, concurrency retries and transaction management can be written once as pipeline behaviors and applied to every handler — or to every *command* handler — without touching the handlers. This is the strongest argument for any mediator, and the one to lead with.

**3. Thin entry points.** Controllers, minimal-API endpoints, background jobs and message consumers become one-liners that translate transport to request and back. The same handler serves HTTP, a scheduled job and a queue consumer (Derek Comartin's "application requests" argument).

**4. Handlers decoupled from transport.** Handlers don't know about HTTP — no `HttpContext`, no `IActionResult`. That's good layering (Module 20), though achievable without a mediator.

**5. Convention and ecosystem.** Templates, samples, blog posts and team members' experience all assume it. That's a genuine productivity benefit — and, since 2025, also the reason the licence change affects so many codebases.

What's notably **not** on the list: "enables CQRS" (any code path can separate reads and writes), "decouples the application" (only callers from handler types), and "makes handlers testable" (handlers are plain classes regardless).

---

## Concept 62 — What MediatR costs

**1. Indirection.** In the IDE, *Go to Definition* on `sender.Send(new PlaceOrder(...))` lands on `ISender.Send`, not on the handler. Finding the handler means searching for `IRequestHandler<PlaceOrder`. In a large codebase this is a constant small tax on understanding, and it's the most common complaint from engineers joining a MediatR codebase. (Conventions help: request and handler in the same file.)

**2. Runtime wiring.** Missing handlers, duplicate handlers, and behaviors that don't apply because a generic constraint doesn't match are all discovered at runtime (Concept 60).

**3. Hidden coupling through `IMediator`.** When handlers inject `IMediator` and call other handlers (Concept 69) or publish notifications that trigger other handlers (Concept 68), the application's real call graph becomes invisible: a handler's constructor no longer tells you what it depends on. This is the **service locator** anti-pattern with a nicer API.

**4. Everything becomes a request.** Teams that adopt MediatR tend to route *everything* through it — including trivial reads that would be a single method call — adding ceremony without benefit.

**5. Notifications invite misuse** as an event bus — synchronous, non-durable, same-transaction — which fails in exactly the ways a real event bus is designed not to (Concepts 68, 100).

**6. Performance and AOT** (Concept 60): small per-call overhead, reflection-based wiring, no Native AOT story.

**7. Version churn.** Major versions have changed registration and signatures (v12 merged the DI package into the main one and changed how void requests are handled; later versions extended the pipeline delegate and added licence keys), and a breaking upgrade touches every registration and often every behavior.

**8. Licensing** (Part G): from v13, a commercial product with RPL-1.5 as the alternative — a procurement and compliance cost, not a technical one, and now the most frequent trigger for revisiting the decision.

The balanced view to give: *"MediatR's real value is the pipeline. Its real costs are indirection and the temptation to use it as an internal event bus. Whether it's worth it depends on how many cross-cutting concerns you're applying uniformly — and since 2025, on the licence."*

---

## Concept 63 — Pipeline behaviors: the real value

A pipeline behavior wraps every matching handler, like ASP.NET Core middleware wraps every request. Everything that should happen for *every* command (or query) belongs here rather than being copy-pasted into handlers. The catalogue:

| Concern | Applies to | What it does |
|---|---|---|
| **Logging / tracing / metrics** | All | Start an `Activity`, log request type and outcome, record duration and failures |
| **Authorization** | All (with per-request policies) | Reject before any work if the principal may not issue this request type |
| **Validation** | Mostly commands | Run shape validators; reject with errors before the handler |
| **Idempotency** | Commands with keys | Replay stored outcomes for duplicate keys (Concept 19) |
| **Concurrency retry** | Machine-issued commands | Re-run on `DbUpdateConcurrencyException` with a fresh unit of work (Module 22, Concept 69) |
| **Unit of work / transaction** | Commands | Begin, invoke handler, `SaveChanges` once, commit (Concept 65) |
| **Caching** | Queries | Return cached responses; populate on miss (Concept 67) |
| **Timeouts** | Queries (sometimes commands) | Link a cancellation token with a deadline |
| **Performance warning** | All | Log requests slower than a threshold, with the request type |
| **Tenancy / context** | All | Establish tenant, culture or correlation context for the handler |

A logging-and-tracing behavior shows the shape:

```csharp
internal sealed class TelemetryBehavior<TRequest, TResponse>(ILogger<TelemetryBehavior<TRequest, TResponse>> log)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    private static readonly ActivitySource Source = new("Ordering.Application");
    private static readonly string RequestName = typeof(TRequest).Name;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        using var activity = Source.StartActivity(RequestName, ActivityKind.Internal);
        var start = Stopwatch.GetTimestamp();
        try
        {
            var response = await next(ct);
            activity?.SetTag("outcome", OutcomeName(response));           // e.g. the outcome record's type name
            return response;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            log.LogError(ex, "{Request} failed", RequestName);
            throw;
        }
        finally
        {
            log.LogInformation("{Request} handled in {ElapsedMs:0.0} ms", RequestName,
                Stopwatch.GetElapsedTime(start).TotalMilliseconds);
        }
    }

    private static string OutcomeName(TResponse? r) => r?.GetType().Name ?? "null";
}
```

Two design rules:

- **Target behaviors with marker interfaces.** Declare `ICommand<TResponse> : IRequest<TResponse>` and `IQuery<TResponse> : IRequest<TResponse>`, and constrain behaviors: `where TRequest : ICommand<TResponse>` for transactions and retries, `where TRequest : IQuery<TResponse>` for caching. The request's type then declares which concerns apply — visible in the code, checked by the container.
- **Behaviors hold no business logic.** A behavior that knows about orders or customers is a handler in disguise. Behaviors are infrastructure for *all* requests of a kind.

---

## Concept 64 — Ordering the pipeline

Behaviors run in registration order, outermost first — and **the order is semantics**, not style. Here's the order for commands, with the reason each concern sits where it does:

```
 ┌── 1. Telemetry (tracing, logging, metrics)     outermost: measures everything, including rejections below
 │ ┌── 2. Authorization                           cheapest rejection; unauthorized callers learn nothing else
 │ │ ┌── 3. Validation (shape)                    before any state is read; invalid requests consume nothing
 │ │ │ ┌── 4. Idempotency (replay lookup)         a duplicate returns the stored outcome without retrying or re-running
 │ │ │ │ ┌── 5. Concurrency retry                 wraps the whole attempt: fresh unit of work per attempt
 │ │ │ │ │ ┌── 6. Unit of work / transaction      begin → handler → SaveChanges (state + outbox + idempotency record) → commit
 │ │ │ │ │ │     └── Handler                      load → decide
```

Why each boundary matters — the questions interviewers use to test understanding:

- **Why is retry outside the transaction?** Because a `DbUpdateConcurrencyException` means this transaction's view is stale. Retrying *inside* it would re-use the same `DbContext` with the same stale tracked entities and the same failed transaction. The retry must start over: new transaction, cleared change tracker (or a fresh scope), reload, re-decide (Module 22, Concept 69).
- **Why is idempotency outside retry?** So that a request whose key was already processed is answered from the store once, not re-checked on every retry attempt — and so the *recording* of the outcome (inside the unit of work) is part of whichever attempt finally commits.
- **Why is validation outside idempotency?** So invalid requests don't reserve or consume idempotency keys, and so a client that fixes a validation error can retry with the same key.
- **Why is authorization before validation?** Validation messages can leak information ("customer 42 does not exist") to callers who aren't allowed to ask. Reject unauthorized requests first.
- **Why is telemetry outermost?** So the trace and duration include time spent in every other behavior, and so rejected requests are still observed.
- **Where do transient-fault retries go?** Not in this pipeline for EF Core: the provider's execution strategy (`EnableRetryOnFailure`) handles transient database errors, and when you open your own transaction it must be wrapped in `strategy.ExecuteAsync(...)` (Concept 65). Composing that with the concurrency retry so the two don't multiply is Module 25's subject; the rule for now is **one retry loop per failure kind, each at the level that can safely redo the work.**

For queries, the pipeline is shorter: telemetry → authorization → caching → timeout → handler. **Caching must come after authorization** (and be keyed by audience), or cached responses are served to callers who shouldn't see them.

---

## Concept 65 — The transaction behavior done right

The unit-of-work behavior is the most valuable behavior and the one most often written wrong. A correct version for EF Core:

```csharp
// Marker interfaces declare which concerns apply
public interface ICommand<TResponse> : IRequest<TResponse> { }
public interface IQuery<TResponse>   : IRequest<TResponse> { }

internal sealed class UnitOfWorkBehavior<TRequest, TResponse>(OrderingDbContext db)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>                                   // commands only — queries never open transactions
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (db.Database.CurrentTransaction is not null)
            throw new InvalidOperationException(                          // nested command: see Concept 69
                $"{typeof(TRequest).Name} was sent from inside another command's transaction.");

        var strategy = db.Database.CreateExecutionStrategy();              // transient-fault retries (EnableRetryOnFailure)
        return await strategy.ExecuteAsync(async token =>
        {
            db.ChangeTracker.Clear();                                      // each strategy attempt starts clean
            await using var tx = await db.Database.BeginTransactionAsync(token);

            var response = await next(token);                              // handler: load → decide (no SaveChanges)
            if (request is IIdempotentCommand idem)                        // record the outcome in the same commit (Concept 19)
                db.ProcessedCommands.Add(ProcessedCommand.For(idem, response));
            await db.SaveChangesAsync(token);                              // state + domain-event handlers + outbox rows
            await tx.CommitAsync(token);
            return response;
        }, ct);
    }
}
```

The decisions embedded in it:

- **Commands only.** The constraint `where TRequest : ICommand<TResponse>` means queries never get a transaction. A query wrapped in a transaction holds locks (or a snapshot) for no reason and can't be routed to a replica cleanly.
- **One commit, owned by the behavior.** Handlers don't call `SaveChanges`. This makes "one transaction per command" structural rather than a convention. (The alternative — handlers call `SaveChanges` themselves, as in Concept 17 — is more explicit and equally valid. **Pick one convention per codebase**; mixing them produces double commits and missing commits.)
- **Domain events and the outbox commit with the state.** `SaveChanges` runs the domain-event interceptor (Module 22, Concept 81), which writes outbox rows in the same transaction. Nothing leaves the process until after commit.
- **Concept 19's idempotency step, split in two.** In a pipeline, the replay *lookup* stays in its own behavior outside the retry (Concept 64), and the *recording* moves in here, because only the unit of work knows which attempt actually commits.
- **Rejections commit nothing — or commit their idempotency record.** If the handler returns a rejection outcome without changing state, `SaveChanges` writes only what was added (for example the idempotency record, if you store rejections — Concept 19).
- **Nested commands are refused, loudly.** A command sent from inside another command would otherwise either join the outer transaction silently (making two "commands" one transaction, and running validation and retries twice — Concept 69) or open a second transaction on the same connection. Failing fast makes the design problem visible.
- **The execution strategy wraps the whole attempt.** EF Core's retrying execution strategy can't retry an operation inside a user-initiated transaction unless the whole transactional unit is passed to `ExecuteAsync`. The concurrency retry behavior sits outside this one and handles `DbUpdateConcurrencyException`, which the execution strategy deliberately doesn't retry.
- **Commit-time ambiguity remains.** If the connection drops *during* `CommitAsync`, the strategy can't tell whether the commit landed, and it re-runs the whole attempt. EF Core's connection-resiliency docs call this the commit-failure idempotency issue; their remedies are a verification check (`ExecuteInTransaction` with a `verifySucceeded` delegate) or making the attempt itself idempotent. For keyed commands the idempotency record already gives you the second: at worst the re-run fails loudly on the key's unique constraint instead of applying the command twice. Turning that into the *right response* rather than an error is part of Module 25's retry composition.

---

## Concept 66 — The validation behavior

Validation behaviors are the most common behavior in MediatR codebases, usually built on FluentValidation (Apache-2.0, 12.x — unaffected by the licensing changes):

```csharp
internal sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
    where TResponse : IFailureFactory<TResponse>                          // Concept 70: lets us return a failure generically
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next(ct);

        var context = new ValidationContext<TRequest>(request);
        var results = await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, ct)));
        var errors = results.SelectMany(r => r.Errors)
                            .Select(f => new Error(f.PropertyName, f.ErrorMessage))
                            .ToList();

        return errors.Count == 0 ? await next(ct) : TResponse.ValidationFailed(errors);
    }
}
```

Three judgments to make:

**1. Where should shape validation live at all?** Since .NET 10, minimal APIs validate request types at the edge with `builder.Services.AddValidation()` — a source-generated, DataAnnotations-based filter that returns a `400` problem-details response before your endpoint runs. If your requests are validated at the edge and parsed into value objects (Module 22, Concept 48), a pipeline validation behavior may be redundant for HTTP — but it still protects the *other* entry points (jobs, message consumers) that send the same commands. Choose deliberately: edge validation for HTTP concerns, pipeline validation when the same requests arrive from several transports.

**2. Results or exceptions?** Throwing a `ValidationException` from the behavior and mapping it to 400 in exception-handling middleware is simple and common. Returning a failure result (above) avoids exceptions for an expected outcome and keeps failures visible in types — at the cost of the generic plumbing in Concept 70.

**3. Keep validators to shape.** Validators that query the database — "customer must exist," "email must be unique" — are running invariant checks outside the transaction, often against stale data (Concepts 16, 55). They're acceptable as friendly early messages only if the write side still enforces the rule for real.

---

## Concept 67 — Behaviors for queries

Queries get a different, shorter pipeline. Three behaviors earn their place:

**Caching**, keyed by the query's parameters **and the caller's audience** (Concept 33):

```csharp
public interface ICacheableQuery
{
    string CacheKey { get; }                     // must include tenant/user where results vary by caller
    TimeSpan Duration { get; }
    IReadOnlyList<string> Tags { get; }          // invalidated by events (Concept 30)
}

internal sealed class QueryCachingBehavior<TRequest, TResponse>(HybridCache cache)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IQuery<TResponse>, ICacheableQuery
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct) =>
        await cache.GetOrCreateAsync(
            request.CacheKey,
            async token => await next(token),
            new HybridCacheEntryOptions { Expiration = request.Duration },
            request.Tags,
            ct);
}
```

**Timeouts**, so one pathological query can't hold a connection and a thread for minutes:

```csharp
internal sealed class QueryTimeoutBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IQuery<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(5));
        return await next(cts.Token);                  // the handler and EF see the linked token
    }
}
```

**Slow-query warnings** — log any query over a threshold with its type and parameters (carefully — no personal data in logs), feeding the read-side tuning loop.

What doesn't belong in a query pipeline: transactions (Concept 65), concurrency retries (queries don't conflict), and idempotency (queries are idempotent by definition). And one thing a behavior *can't* do well: **route a query to a replica.** Routing is better expressed at compile time by which `DbContext` the query handler depends on (Concept 31).

---

## Concept 68 — Notifications: the trap

`INotification` looks like an event bus: publish once, any number of handlers react. It isn't one, and treating it as one is among the most damaging mistakes in MediatR codebases (Concept 100).

What `Publish` actually does:

- **In-process and synchronous with the caller.** `await publisher.Publish(evt)` runs the handlers *now*, in the same process, and doesn't return until they finish.
- **In the same DI scope.** Handlers get the same scoped `DbContext` as the publisher. If the publisher is inside a transaction, so are they.
- **Sequentially by default, and the first exception stops the rest.** The default `ForeachAwaitPublisher` awaits handlers one by one; if the second throws, the third never runs, and the exception propagates to the publisher. The alternative `TaskWhenAllPublisher` runs them concurrently — which, with a shared scoped `DbContext`, produces "a second operation was started on this context instance" errors unless handlers use their own scopes.
- **Not durable.** If the process crashes after the transaction commits but before a notification handler finishes, that handler's work is lost. There's no retry, no dead-letter, no record that it was supposed to happen.
- **No ordering guarantee you should rely on** between handlers.

So when is `INotification` appropriate?

- ✅ **In-transaction domain-event dispatch** (Module 22, Concept 81): handlers that must succeed or fail *with* the command, write only to the same database, and include writing integration events to the outbox. Microsoft's eShop uses MediatR notifications this way. The synchronous, same-transaction, all-or-nothing behaviour is exactly what you want *here*.
- ✅ **In-process decoupling of trivial reactions** where losing the reaction on a crash is acceptable — cache invalidation on a single instance, say.
- ❌ **Integration events** — anything another context or service must reliably receive. Those go through the outbox and a broker (Module 11).
- ❌ **Side effects outside the database** — emails, HTTP calls, pushes. If published inside the transaction, they happen even if the transaction rolls back; if published after commit, they're lost on a crash. Either way, wrong. Use the outbox.

(MediatR 14.1 began de-duplicating notification handlers before dispatch — a fix for the common bug where assembly scanning registered the same handler twice and every notification was handled twice.)

---

## Concept 69 — Handlers calling handlers

The pattern: a handler injects `ISender` and calls another request from inside its own `Handle`.

```csharp
internal sealed class CheckoutHandler(ISender sender, ...) : ICommandHandler<Checkout, CheckoutOutcome>
{
    public async Task<CheckoutOutcome> Handle(Checkout cmd, CancellationToken ct)
    {
        await sender.Send(new ReserveStock(cmd.CartId), ct);         // another command, through the full pipeline
        await sender.Send(new PlaceOrder(cmd.OrderId, cmd.Key), ct);  // and another
        ...
    }
}
```

What goes wrong:

- **The pipeline runs again for each nested request.** Telemetry and validation run twice. The idempotency behavior sees a nested request — with which key? The concurrency retry nests: three outer attempts times three inner attempts is nine executions of the inner handler under contention. The transaction behavior either joins the outer transaction silently or opens a second one.
- **Two "commands" become one transaction** — or, worse, don't. The design says one command changes one aggregate (Concept 17); the nested calls change two or three, with whatever atomicity the transaction behavior happens to provide.
- **The call graph disappears.** `CheckoutHandler`'s constructor says it depends on `ISender`. Its real dependencies — stock reservation, order placement and everything *they* depend on — are invisible without reading the body.
- **Slices couple to each other.** In a vertical-slice codebase, one slice now depends on the internals of two others (Concept 72).

What to do instead:

- **Extract the shared logic** into a domain service, an application service or the aggregates themselves, and call it directly. If two handlers need the same logic, that logic isn't a handler.
- **If it's really a workflow across aggregates**, make it one: raise an event from the first step and let a process manager or event handler issue the next command in its own transaction (Module 22, Concept 85). That makes the eventual consistency explicit.
- **If an endpoint genuinely needs two commands in sequence**, have the *endpoint* (the composition layer) send them, and design for the second failing after the first succeeded.

The rule for reviews: **handlers don't inject `ISender` or `IMediator`.** An architecture test can enforce it.

---

## Concept 70 — Result types through the pipeline

If commands return outcome types rather than throwing (Concept 18), generic behaviors face a problem: a validation behavior that finds errors must *return a failure* of type `TResponse` — but it doesn't know what `TResponse` is. Three solutions:

**1. Throw anyway, in the behavior.** Behaviors throw `ValidationException`; handlers return results for business outcomes. Pragmatic, common, and inconsistent.

**2. A common result base class**, with the behavior constrained to it and creating failures by reflection or `Activator`. Works, but brittle.

**3. Static abstract interface members** (C# 11+) — the clean solution. The response type declares how to construct a failure; the behavior calls it through the generic constraint, with no reflection:

```csharp
public sealed record Error(string Code, string Message);

public interface IFailureFactory<TSelf> where TSelf : IFailureFactory<TSelf>
{
    static abstract TSelf ValidationFailed(IReadOnlyList<Error> errors);
    static abstract TSelf Forbidden();
}

public abstract record PlaceOrderOutcome : IFailureFactory<PlaceOrderOutcome>
{
    public sealed record Placed(int NewVersion) : PlaceOrderOutcome;
    public sealed record Invalid(IReadOnlyList<Error> Errors) : PlaceOrderOutcome;
    public sealed record Denied : PlaceOrderOutcome;
    // ... business outcomes as in Concept 18

    public static PlaceOrderOutcome ValidationFailed(IReadOnlyList<Error> errors) => new Invalid(errors);
    public static PlaceOrderOutcome Forbidden() => new Denied();
}

// The validation behavior (Concept 66) constrains TResponse : IFailureFactory<TResponse>
// and returns TResponse.ValidationFailed(errors) — resolved at compile time per closed type.
```

A behavior constrained this way applies only to requests whose responses opt in — and because Microsoft's DI skips open-generic implementations whose constraints aren't satisfied when resolving enumerables, requests that return plain types simply don't get it. That's a feature (explicit opt-in) and a trap (a request that forgot to implement the interface silently skips validation) — which is what a registration test is for (Concept 71).

The ecosystem has noticed the demand: Wolverine 6 added formal support for the result pattern in its mediator usage, and libraries such as FluentResults and ErrorOr provide ready-made result types.

---

## Concept 71 — Testing with a mediator

Three layers, each testing one thing:

**1. Test handlers directly.** A handler is a plain class. Construct it with real domain objects and fakes (or a real database via Testcontainers for handlers that are mostly data access), call `Handle`, assert on the outcome and the resulting state. No mediator involved:

```csharp
[Fact]
public async Task Placing_an_order_over_the_credit_limit_is_rejected()
{
    var order = OrderMother.DraftWithTotal(Money.Of(1_200m, "EUR"));
    var handler = new PlaceOrderHandler(new InMemoryOrders(order), new FixedCredit(Money.Of(1_000m, "EUR")), TimeProvider.System);

    var outcome = await handler.Handle(new PlaceOrder(order.Id, IdempotencyKey.New()), CancellationToken.None);

    outcome.ShouldBeOfType<PlaceOrderOutcome.InsufficientCredit>();
    order.Status.ShouldBe(OrderStatus.Draft);
}
```

**2. Test behaviors in isolation** with a fake `next` delegate: does the validation behavior short-circuit on errors and call `next` when valid? Does the retry behavior retry on `DbUpdateConcurrencyException` and not on other exceptions? Does the caching behavior key by audience?

**3. Test the wiring end to end** through HTTP with `WebApplicationFactory` (or Alba): does `POST /orders/42/place` produce a 422 with the right problem type when credit is insufficient? These tests exercise registration, behaviors, handlers and persistence together.

Two anti-patterns:

- **Mocking `ISender` in endpoint tests** — "verify that the endpoint called `Send` with a `PlaceOrder`" tests the wiring you wrote one line ago and nothing about behaviour. Test the endpoint through HTTP, or not at all.
- **Relying on production traffic to find missing handlers.** Because MediatR resolves handlers at runtime, add a **registration test**: reflect over every type implementing `IRequest<>`/`IRequest` in the application assembly and assert the container resolves exactly one handler for each (and, if you use result types, that every command's response implements the failure factory). Source-generated mediators make this a compile-time diagnostic instead (Concept 77).

---

## Concept 72 — MediatR and vertical slices

MediatR's popularity is bound up with **Vertical Slice Architecture** (VSA), which Jimmy Bogard has advocated since around 2018 and Module 20 (Concept 51) introduced: organize code by **feature** rather than by technical layer, so everything for one use case — request, handler, validator, endpoint, DTOs — lives together.

```
Features/
  Orders/
    PlaceOrder/
      PlaceOrder.cs             // command record + handler + validator (one file is fine)
      PlaceOrderEndpoint.cs     // HTTP mapping
    GetOrderDetails/
      GetOrderDetails.cs        // query + handler + DTO
      GetOrderDetailsEndpoint.cs
    SearchOrders/
      ...
Domain/                         // aggregates, value objects — shared by the slices that change state
Infrastructure/                 // DbContext, outbox, read-model persistence
```

Why the two fit: a request-and-handler pair *is* a slice's natural unit, and the pipeline supplies the cross-cutting concerns that a layered architecture would otherwise put in base classes or service layers.

The rules that keep slices healthy:

- **Slices share the domain model and infrastructure, not each other.** A command slice uses aggregates; a query slice uses the read side. No slice calls another slice's handler (Concept 69).
- **Duplication between slices is acceptable** — two query slices with similar projections are cheaper than a shared abstraction that serves neither well (Module 20, Concept 72, "the wrong abstraction").
- **Each slice chooses its own depth.** A CRUD slice for reference data can be a transaction script; a core command slice goes through an aggregate; a query slice can use EF projections or Dapper. VSA is CQRS's "decide per path" applied to code organization.

And the point to make explicitly: **VSA doesn't require MediatR.** A slice can be a minimal-API endpoint that calls a handler class directly, a FastEndpoints endpoint, a Wolverine HTTP endpoint or handler, or a `Mediator` request. ardalis's `min-clean` template, Wolverine's vertical-slice tutorial and Bogard's own later writing all treat the dispatcher as a detail. Separating those two ideas — *organize by feature* and *dispatch through a mediator* — is what makes the licensing decision in Part G a packaging change rather than an architectural one.

---

# Part G — The licensing change and the alternatives

In 2025 the most common .NET implementation of the command/query handler pattern became a commercial product. For a senior engineer that's a migration question; for an architect it's a build-vs-buy decision with procurement, compliance and risk dimensions (Module 33 takes the executive framing further). This part gives you the facts precisely, the options with their costs, the credible alternatives with code, and a migration playbook — and ends with the question that should come first: do you need a mediator at all?

---

## Concept 73 — What happened

The timeline, as verified in September 2026:

| Date | Event |
|---|---|
| **1 April 2025** | MediatR **12.5.0** — the last Apache-2.0 release |
| **2 April 2025** | Jimmy Bogard announces that MediatR and AutoMapper will be commercialized "to ensure the long-term sustainability" of the projects, noting that his open-source contributions had collapsed after he moved to solo consulting in 2020 |
| **16 April 2025** | A licensing-update post sets out the intended model: dual licensing, a free tier for smaller organizations, team-based pricing |
| **2 July 2025** | Commercial editions launch under **Lucky Penny Software**; **MediatR 13.0.0** ships with licence-key support; the repository moves to `LuckyPennySoftware/MediatR` |
| **October 2025** | 13.1.0 — .NET Framework 4.x support, static licence keys, fixes |
| **December 2025** | **14.0.0** — .NET 10 support and signed packages; major versions now aligned with .NET's release cadence |
| **March 2026** | 14.1.0 — perpetual-licence support, de-duplication of notification handlers; quarterly releases become the norm |
| **2 July 2026** | **14.2.0** — licence keys from `MEDIATR_LICENSE_KEY` / `LUCKYPENNY_LICENSE_KEY` environment variables, a fix for a licence-validation deadlock in synchronous contexts, validation moved off the mediator's construction path |

The same period saw a broader pattern in the .NET ecosystem: FluentAssertions 8 moved to a commercial licence (January 2025), MassTransit announced that v9 would be commercial while v8 stays Apache-2.0 (Module 21), and — earlier — IdentityServer became Duende's commercial product. The architectural lesson isn't about any one library: **widely used open-source dependencies can change their terms, and the cost of that change is proportional to how deeply your code depends on them.** That's the argument for Concept 81's "insulate your handlers" rule, regardless of which library you pick.

---

## Concept 74 — What the licence actually requires

Precision matters here, because both over- and under-reaction are common. As stated by Lucky Penny's licence, pricing page and FAQ in September 2026:

**Which versions.** The licence agreement applies to **MediatR 13.0.0 and later** (and AutoMapper 15.0.0 and later). Earlier versions remain under their original licences (Apache-2.0 for MediatR ≤ 12.5.0) **indefinitely**, but without support, security updates, warranty or indemnity. **`MediatR.Contracts`** — the package containing only `IRequest`, `INotification` and `IStreamRequest` — is published under **Apache-2.0**.

**Two licences for current versions.**
- **Reciprocal Public License 1.5** — a strong copyleft licence. Unlike the GPL, it treats most use beyond personal or research use — including internal deployment and providing services over a network — as triggering its obligation to publish source. Most organizations' legal teams treat it as incompatible with proprietary software. That's intentional: it's the free path for open-source projects; everyone else is expected to buy. (Have your own legal team read it; don't rely on anyone's summary, including this one.)
- **The commercial licence** — a subscription.

**Who needs to pay.**
- The **Community licence** is free, self-service, and covers individuals and organizations with **annual gross revenue (or non-profit budget) under \$5M** that have **never received more than \$10M in outside capital**, and that are **not government or quasi-government entities** — a category defined broadly (public health systems, public utilities, government-controlled enterprises, municipalities…). Universities qualify for teaching and academic research but **not** for institutional/operational software.
- Everyone else buys a **team-sized** tier: **Standard** (1–10 developers), **Professional** (11–50), **Enterprise** (unlimited). List prices on mediatr.io in September 2026: **\$499 / \$1,499 / \$3,999 per year** for MediatR alone; **\$799 / \$2,399 / \$6,399** bundled with AutoMapper; monthly billing available with a 12-month minimum. Tiers don't stack (14 developers means Professional, not two Standards).
- **Only developers with "programmatic access" count** — those who regularly write, modify, debug or compile code that calls the library. Front-end developers, QA, designers and product managers don't.
- **Agencies**: if the client has its own developers, the *client* holds the licence and the agency's developers count toward the client's tier; Community eligibility follows the client, not the agency.
- A licence covers one legal entity; affiliates need an addendum.

**What's enforced, and how.**
- **Non-production environments need no licence** — development, CI/CD, staging, QA, integration testing.
- Enforcement is **self-contained and log-based**: no licence server, no network call, no runtime limits or degradation. A missing, invalid or expired key produces **warning or error log messages** (the category is `LuckyPennySoftware.MediatR.License`).
- On **expiry**, applications built during the subscription **keep running unchanged**. What you lose is the right to develop new applications with it, upgrade to newer versions, and receive support.
- Client-side apps (Blazor WebAssembly, MAUI, WPF) should omit the key rather than ship it; the licence still applies to the developers.

Two misconceptions to correct in an interview:

- *"Production will break when the key expires."* No — enforcement is logging only.
- *"So we can just ignore the warning."* That's a **compliance** decision, not a technical one. Using a commercially licensed library in production without a licence is a legal exposure for the company, and an architect who recommends it is recommending that exposure.

---

## Concept 75 — Your four options

For an existing MediatR codebase, there are four options. None is universally right.

| Option | Engineering cost | Ongoing cost | Risk | Right when |
|---|---|---|---|---|
| **1. License it** (Community if eligible, otherwise paid) | ~Zero — add a key to configuration | Subscription; tracking eligibility and renewals | Vendor dependency; future terms may change again | Most codebases with heavy MediatR use: the price is small relative to engineering time |
| **2. Pin 12.5.0** (Apache-2.0) | Low — pin and stop upgrading | None in fees | No fixes, no new .NET support; divergence from ecosystem; a forced migration later | A short-term bridge while migrating; systems nearing end of life |
| **3. Migrate to an alternative** | Proportional to handler count and to use of notifications, nested sends, processors and exception handlers | The alternative's own (usually none, MIT) | Migration bugs; learning curve | Large or growing codebases, AOT needs, or a desire to consolidate on Wolverine for messaging |
| **4. Remove the mediator** | Similar to 3 for the mechanics; often simplifies code | None | Losing the pipeline if not replaced with decorators/filters | Codebases that used MediatR mainly as a dispatcher with one or two behaviors |

A quick cost sanity check that belongs in the conversation: a Professional licence for up to 50 developers at \$1,499 a year is roughly the cost of one or two engineer-days. A migration of a codebase with several hundred handlers is engineer-*weeks*. On pure cost, licensing usually wins for an existing codebase — **if** the organization's procurement, legal and dependency policies are comfortable with a single-maintainer commercial dependency. The strongest arguments for migrating are therefore usually *not* the fee: they're AOT or cold-start requirements, a policy against commercial dependencies in the application core, consolidation onto a tool that also does messaging, or the realization that the codebase doesn't need a mediator at all.

---

## Concept 76 — No library: handler interfaces and decorators

The simplest replacement is no library. Define your own handler interfaces, inject handlers directly, and implement cross-cutting concerns as **decorators**. Milan Jovanović's *CQRS Pattern the Way It Should've Been From the Start* (May 2025) walks through the same approach.

```csharp
// Application layer: your own contracts — nothing to license, nothing to upgrade
public interface ICommand<TResult> { }
public interface IQuery<TResult> { }

public interface ICommandHandler<in TCommand, TResult> where TCommand : ICommand<TResult>
{
    Task<TResult> Handle(TCommand command, CancellationToken ct);
}

public interface IQueryHandler<in TQuery, TResult> where TQuery : IQuery<TResult>
{
    Task<TResult> Handle(TQuery query, CancellationToken ct);
}

// A decorator is a handler that wraps a handler
internal sealed class LoggingCommandHandler<TCommand, TResult>(
    ICommandHandler<TCommand, TResult> inner,
    ILogger<LoggingCommandHandler<TCommand, TResult>> log) : ICommandHandler<TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    public async Task<TResult> Handle(TCommand command, CancellationToken ct)
    {
        var start = Stopwatch.GetTimestamp();
        try { return await inner.Handle(command, ct); }
        finally { log.LogInformation("{Command} handled in {Ms:0.0} ms", typeof(TCommand).Name,
                                     Stopwatch.GetElapsedTime(start).TotalMilliseconds); }
    }
}
```

Registration with **Scrutor** (MIT) for scanning and decoration:

```csharp
builder.Services.Scan(scan => scan
    .FromAssemblyOf<PlaceOrder>()
    .AddClasses(c => c.AssignableTo(typeof(ICommandHandler<,>)), publicOnly: false)
        .AsImplementedInterfaces().WithScopedLifetime()
    .AddClasses(c => c.AssignableTo(typeof(IQueryHandler<,>)), publicOnly: false)
        .AsImplementedInterfaces().WithScopedLifetime());

// Each Decorate wraps what's registered so far: the LAST call becomes the OUTERMOST decorator.
builder.Services.Decorate(typeof(ICommandHandler<,>), typeof(UnitOfWorkCommandHandler<,>));   // innermost
builder.Services.Decorate(typeof(ICommandHandler<,>), typeof(RetryCommandHandler<,>));
builder.Services.Decorate(typeof(ICommandHandler<,>), typeof(ValidationCommandHandler<,>));
builder.Services.Decorate(typeof(ICommandHandler<,>), typeof(LoggingCommandHandler<,>));     // outermost

builder.Services.Decorate(typeof(IQueryHandler<,>), typeof(LoggingQueryHandler<,>));         // queries: a shorter chain
```

Endpoints inject the handler they need:

```csharp
app.MapPost("/orders/{id}/place", async (OrderId id, [FromHeader(Name = "Idempotency-Key")] string key,
        ICommandHandler<PlaceOrder, PlaceOrderOutcome> handler, CancellationToken ct) =>
    ToHttp(await handler.Handle(new PlaceOrder(id, new IdempotencyKey(key)), ct), id));
```

What this buys: zero dependency, compile-time types at every call site (the endpoint's parameter names the exact handler contract), no runtime dispatch lookup, and decorators that are ordinary classes. Separate `ICommandHandler`/`IQueryHandler` interfaces give you the command-vs-query targeting that MediatR achieves with generic constraints — without relying on constraint handling in the container.

What it costs: registration is still reflection-based (Scrutor scans), so it's not Native AOT-friendly without hand-written registrations; there's no built-in notification publishing (you already have a domain-event dispatcher — Module 22, Concept 81); and **the decorator order is the reverse of MediatR's registration order** — the most common bug when migrating (Concept 81).

A variation: a 40-line `IDispatcher` that resolves `ICommandHandler<TCommand, TResult>` from the service provider, for code that wants `dispatcher.Send(command)` call sites. It's easy to write — the Module 22 domain-event dispatcher is the same pattern — but direct injection is simpler and keeps the dependency visible, so prefer it unless many call sites need dispatch by runtime type.

---

## Concept 77 — `Mediator`: the source-generated option

**`Mediator`** by Martin Othamar (MIT; 3.0 stable, 3.1 at release candidate in September 2026) keeps a MediatR-like API but generates the dispatch code at compile time with a Roslyn source generator. `ardalis/CleanArchitecture` uses it.

```xml
<!-- Only in the outermost project (the web host / worker) -->
<PackageReference Include="Mediator.SourceGenerator" Version="3.0.*">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
<!-- In projects that define messages and handlers -->
<PackageReference Include="Mediator.Abstractions" Version="3.0.*" />
```

```csharp
builder.Services.AddMediator((MediatorOptions o) =>
{
    o.ServiceLifetime = ServiceLifetime.Scoped;           // default is Singleton — see the trap below
    o.Assemblies = [typeof(PlaceOrder)];
    o.PipelineBehaviors =                                 // explicit and ordered (needed for AOT); first = outermost
    [
        typeof(TelemetryBehavior<,>),
        typeof(ValidationBehavior<,>),
        typeof(UnitOfWorkBehavior<,>),
    ];
});

public sealed record PlaceOrder(OrderId OrderId, IdempotencyKey Key) : ICommand<PlaceOrderOutcome>;

public sealed class PlaceOrderHandler(IOrderRepository orders, ICreditService credit, TimeProvider clock)
    : ICommandHandler<PlaceOrder, PlaceOrderOutcome>
{
    public async ValueTask<PlaceOrderOutcome> Handle(PlaceOrder cmd, CancellationToken ct) { /* as before */ }
}

public sealed class UnitOfWorkBehavior<TMessage, TResponse>(OrderingDbContext db) : IPipelineBehavior<TMessage, TResponse>
    where TMessage : IBaseCommand                          // Mediator distinguishes commands, queries and requests
{
    public async ValueTask<TResponse> Handle(TMessage message, MessageHandlerDelegate<TMessage, TResponse> next, CancellationToken ct)
    {
        // Shortened to show the API shape. In production keep Concept 65's body: execution strategy,
        // ChangeTracker.Clear, nested-transaction guard, idempotency record.
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        var response = await next(message, ct);            // next takes the message and token: no closure allocation
        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
        return response;
    }
}
```

What it buys:

- **Compile-time wiring and diagnostics.** The generator emits DI registrations and a monomorphized `Send` per message type; a missing or duplicate handler is a build warning, not a production exception.
- **Native AOT and trimming compatibility** — no reflection or runtime code generation. Relevant for serverless cold starts and for containers where startup time matters (Module 17).
- **Lower overhead** — `ValueTask` returns, singleton lifetimes by default, delegates that pass the message and token instead of capturing closures. In absolute terms this saves fractions of a microsecond per call; it matters for allocation-sensitive services, not for a typical CRUD API.
- **Distinct `ICommand`, `IQuery` and `IRequest`** marker types, which make command-only and query-only behaviors natural.
- **Built-in OpenTelemetry metrics and tracing** in the 3.1 line, following the messaging semantic conventions.

What to watch:

- **The singleton default is a trap for typical handlers.** A singleton handler that depends on a scoped `DbContext` is a captive dependency (Module 18) — with scope validation enabled it fails at startup; without it, it shares one `DbContext` across requests. Most line-of-business apps should set `ServiceLifetime.Scoped`, accepting the (small) allocation cost.
- **Behaviors must be listed explicitly**, in order, for AOT. Forgetting to add a new behavior to the list means it silently doesn't run.
- **Install the source generator in one project only** — the outermost executable. Referencing it from several projects in the same deployed application causes generation conflicts.
- **API differences from MediatR**: `ValueTask` instead of `Task`, a different pipeline delegate signature, no generic requests, different namespaces. The migration is mechanical but touches every handler and behavior.

---

## Concept 78 — Wolverine

**Wolverine** (MIT; part of the JasperFx "Critter Stack" with Marten) is both an in-process mediator and a full messaging framework with a durable inbox/outbox, sagas, scheduled messages and broker transports. Version 6.0 shipped in May 2026 as part of the "Critter Stack 2026" wave: .NET 9/10 targets, faster cold start, the option to compile with AOT compliance (with caveats), the Roslyn runtime compiler movable to a development-time-only dependency, and service location disallowed by default.

As a pure mediator:

```csharp
builder.Host.UseWolverine(opts =>
{
    opts.Durability.Mode = DurabilityMode.MediatorOnly;   // in-process only: no local queues, no durable background agents
});

// Handlers are discovered by convention: a public type with a Handle/HandleAsync method whose first parameter is the message.
// Other parameters are resolved as services. No interfaces required; static handlers are idiomatic.
public static class PlaceOrderHandler
{
    public static async Task<PlaceOrderOutcome> HandleAsync(
        PlaceOrder cmd, IOrderRepository orders, ICreditService credit, IUnitOfWork uow, TimeProvider clock, CancellationToken ct)
    {
        var order = await orders.GetAsync(cmd.OrderId, ct);
        if (order is null) return new PlaceOrderOutcome.NotFound();
        // ... decide, save, return the outcome
    }
}

app.MapPost("/orders/{id}/place", async (OrderId id, [FromHeader(Name = "Idempotency-Key")] string key,
                                        IMessageBus bus, CancellationToken ct) =>
    ToHttp(await bus.InvokeAsync<PlaceOrderOutcome>(new PlaceOrder(id, new IdempotencyKey(key)), ct), id));
```

`InvokeAsync<T>` executes the handler inline and returns its response — which, per Wolverine's documentation, works when the handler returns (or cascades) a message of exactly type `T`. Anything else the handler returns is treated as a **cascading message** — sent on to its own handlers after the current one succeeds — which is how Wolverine expresses "this command produced these events."

Where Wolverine is distinctive:

- **One tool from mediator to messaging.** Outside `MediatorOnly` mode, Wolverine's EF Core integration applies transactional middleware and a durable outbox around handlers, so cascaded messages are persisted with the state change and delivered after commit — the Module 11 outbox without writing one. The same handler can later be invoked from a queue instead of HTTP without changing its code. That's the "in-process now, broker later" story Module 21 (Concept 49) described.
- **Low ceremony.** No handler interfaces, method injection, conventions for middleware (`Before`/`After` methods), and first-class support for pure-function handlers that return side effects instead of performing them — which makes handlers trivially unit-testable.
- **Wolverine.Http** can replace the mediator hop entirely: HTTP endpoints *are* handlers, and the documentation itself suggests this is lower ceremony than using Wolverine as a mediator underneath minimal APIs.

What it costs:

- **Conventions and generated code are a different mental model.** Discovery by method name and runtime code generation are powerful and surprising to engineers expecting interfaces; diagnostics (Wolverine's CLI can describe the generated handler code) become part of the workflow.
- **Cold start.** Runtime Roslyn code generation adds startup time; 6.0 reduced it and supports pre-generating code, but it's a consideration for serverless workloads.
- **It's a framework, not a library.** Adopting Wolverine for its mediator is reasonable; adopting it for its messaging is a larger architectural commitment (transports, durability, operational tooling), with commercial support and tooling available from JasperFx.

---

## Concept 79 — Framework-native options

Three more routes, each strong in its niche.

**1. Minimal APIs, endpoint filters and injected handlers.** ASP.NET Core already has a pipeline: middleware for the whole app and **endpoint filters** per endpoint or route group. With .NET 10's built-in validation, a lot of what teams used MediatR behaviors for — validation, logging, authorization — can live on the HTTP side:

```csharp
var commands = app.MapGroup("/orders")
    .AddEndpointFilter<IdempotencyFilter>()                    // runs for every endpoint in the group
    .RequireAuthorization("orders.write");

commands.MapPost("/{id}/place", PlaceOrderEndpoint.Handle);    // validation via builder.Services.AddValidation()
```

The limit: endpoint filters exist only for HTTP. If the same commands arrive from jobs and message consumers, those entry points don't get the pipeline — which is exactly the gap a dispatcher or decorators fill. Combining the two is common: HTTP concerns (validation of request shape, problem details, output caching) in filters; application concerns (unit of work, retries, idempotency) in handler decorators.

**2. FastEndpoints** (MIT; 8.3 stable, 8.4 in beta as of September 2026) structures APIs around the **REPR** pattern (Request–Endpoint–Response: one class per endpoint) and includes an **in-process command bus**: commands implement `ICommand` or `ICommand<TResult>`, handlers implement `ICommandHandler<TCommand, TResult>`, commands are executed with `command.ExecuteAsync()`, and **command middleware** (`ICommandMiddleware<TCommand, TResult>`) provides a chain-of-responsibility pipeline. Recent versions added streaming commands, and the command bus can be used without the rest of FastEndpoints via the `FastEndpoints.Messaging` package. A good fit when the team is adopting FastEndpoints for the API anyway.

**3. Brighter and Darker** (MIT). Brighter is a mature **command processor** — commands and events dispatched to handlers, with policies (retries, circuit breakers via Polly), an outbox and external-bus support declared through handler attributes. **Darker** is its query-processor counterpart, with query handlers and decorators (for example caching and retries). A good fit for teams that want an explicit command-processor pattern with messaging built in and are comfortable with a smaller ecosystem than MediatR's.

---

## Concept 80 — The comparison

As of September 2026:

| | **MediatR 14** | **MediatR 12.5 (pinned)** | **Own interfaces + decorators** | **Mediator 3.x** | **Wolverine 6** | **FastEndpoints command bus** | **Brighter / Darker** |
|---|---|---|---|---|---|---|---|
| **Licence** | RPL-1.5 or commercial (Community tier free) | Apache-2.0, unmaintained | None (Scrutor: MIT) | MIT | MIT | MIT | MIT |
| **Dispatch** | Reflection + DI resolution per call | Same | Direct injection, decorators | Source-generated, compile-time | Runtime-generated code (Roslyn) | Library dispatch | Library dispatch |
| **Handler contract** | `IRequestHandler<,>` | Same | Your interfaces | `ICommandHandler`/`IQueryHandler`/`IRequestHandler` (`ValueTask`) | Convention (`Handle` method), no interface | `ICommandHandler<,>` | `RequestHandler<T>` / `QueryHandler<,>` |
| **Pipeline** | Behaviors, pre/post processors, exception handlers | Same | Decorators | Behaviors, processors, exception handlers | Middleware (conventional or policies) | Command middleware | Attribute-declared handler pipeline / decorators |
| **Native AOT** | No | No | Only with manual registration | Yes | Possible in 6.0, with caveats | Check current docs per feature | Check current docs |
| **Notifications** | `INotification` (in-process) | Same | Your own dispatcher | `INotification` (in-process) | Cascading messages, durable if configured | Event bus (in-process) | Events, with external bus |
| **Durable messaging / outbox** | No | No | No (write your own, Module 11) | No | **Yes** | Remote messaging add-ons | **Yes** |
| **Missing-handler detection** | Runtime | Runtime | Types checked at compile time; a missing registration fails on first resolve (or a startup check) | **Compile time** (generator diagnostics) | Runtime (its CLI can list handlers and generated code) | Runtime | Runtime |
| **Migration effort from MediatR** | — | Trivial | Moderate (mechanical) | Low–moderate (mechanical, API-similar) | Moderate–high (different model) | Moderate | Moderate–high |
| **Best when** | You rely on it heavily and the licence is acceptable | A bridge while migrating | You want no dependency and a small, visible pipeline | You want MediatR's shape, free, fast, AOT-friendly | You also want durable messaging, sagas and an outbox | You're adopting FastEndpoints for the API | You want an explicit command processor with messaging |

The single most useful sentence to extract from the table: **the options differ mainly in how dispatch is wired and what else comes in the box — not in what your handlers do.** That's why a well-insulated codebase can move between them cheaply.

---

## Concept 81 — The migration playbook

A migration off MediatR — to any target — follows the same steps. The order matters: insulate first, change the dispatcher last.

**Step 0 — Inventory.** Count request types and handlers. List every pipeline behavior, pre/post processor, exception handler and stream request. Find every handler that injects `IMediator`/`ISender`/`IPublisher` (nested sends, Concept 69) and every `INotification` with its handlers. Most of the work — and nearly all of the risk — is in the last two lists; the rest is mechanical.

**Step 1 — Characterization tests.** Before changing anything, add HTTP-level tests (Concept 71) covering each endpoint's main outcomes, including rejections and validation failures. These are the safety net that makes the rest mechanical.

**Step 2 — Insulate.** Introduce your own `ICommand<T>`/`IQuery<T>` and handler interfaces (or the target library's), and adapt: during the transition, MediatR can dispatch to handlers that implement your interfaces via a thin generic adapter, or you can migrate slice by slice with both dispatchers registered. Request types can keep referencing **`MediatR.Contracts`** (Apache-2.0) in the meantime — only the dispatch and handler plumbing needs to move.

**Step 3 — Untangle the hard parts.**
- **Nested sends:** extract shared logic into services; turn workflows into event-driven steps (Concept 69).
- **Notifications:** classify each. In-transaction domain-event handlers move to your domain-event dispatcher (Module 22, Concept 81). Anything that is really an integration concern moves to the outbox (Concept 100). Anything that was fire-and-forget and "fine to lose" gets a conscious decision.

**Step 4 — Port the behaviors.** Rewrite each behavior as a decorator, a target-library behavior, a Wolverine middleware or an endpoint filter. **Check the order**: MediatR runs behaviors in registration order (first = outermost); Scrutor decorators wrap in the opposite sense (last `Decorate` = outermost); `Mediator` uses the order of its `PipelineBehaviors` array (first = outermost). A transaction behavior that ends up *outside* the retry behavior changes correctness (Concept 64). Write one test per behavior that asserts its position (for example, that a concurrency conflict is retried with a fresh unit of work).

**Step 5 — Switch dispatch, one slice at a time.** Vertical slices (Concept 72) are the natural unit: move a feature folder, run its tests, merge, repeat.

**Step 6 — Remove and prevent regression.** Delete the MediatR package, and add an architecture test (NetArchTest.eNhancedEdition or ArchUnitNET — Module 20) that fails the build if any type references the `MediatR` namespace — and, while you're at it, that no handler injects the dispatcher.

**Effort, as a rule of thumb:** the mechanical part scales with handler count and is fast — hundreds of handlers can move in days, especially with automated refactoring tools and AI-assisted edits. The non-mechanical part scales with nested sends and notifications and is where the weeks go. The inventory in step 0 tells you which kind of migration you have before you commit to a date.

---

## Concept 82 — Do you need a mediator at all?

Before choosing *which* mediator, ask whether you need one. The honest decision framework:

| Question | If the answer is… | Then |
|---|---|---|
| How many cross-cutting concerns must wrap **every** handler? | None or one (logging is already done by ASP.NET Core and OpenTelemetry) | No mediator; inject handlers directly |
| | Several (unit of work, retries, idempotency, validation, authorization) | A pipeline is valuable — decorators, a mediator or a framework |
| Do requests arrive from **several entry points** (HTTP, jobs, queue consumers, gRPC)? | Only HTTP | Endpoint filters may be enough |
| | Several | Put the pipeline around the handlers, not the transport |
| Do you need **durable messaging, an outbox or sagas** soon? | Yes | Consider a framework that does both (Wolverine; Brighter) instead of a mediator plus a separate bus |
| Are **cold start or Native AOT** requirements real? | Yes | Source-generated (`Mediator`) or no library |
| Is the team **already fluent** in one option, and is the codebase built around it? | Yes | The switching cost is real; weigh it honestly against the licence |
| What's the organization's **policy on commercial dependencies** in the application core? | Restrictive | Prefer MIT options or none |

The answer template for an interview:

> *"For a new service, I'd start without a mediator library: handlers as plain classes injected into endpoints, with cross-cutting concerns as decorators or endpoint filters — it's the least machinery and everything is visible. If the same commands arrive from HTTP, jobs and consumers and we want one pipeline around them, I'd use a source-generated mediator for compile-time checking and no licence, or Wolverine if we're also going to need durable messaging and an outbox. For an existing MediatR codebase, licensing is usually cheaper than migrating — I'd check eligibility and cost first — but either way I'd insulate the handlers from the dispatcher, because the lesson of 2025 is that this dependency can change under you."*

---

# Part H — CQRS in the architecture

Parts A–G were about the pattern and its tools. Part H places it in the larger architecture: where it belongs and where it doesn't, where the pieces live in a Clean Architecture or vertical-slice codebase, how read models cross module and service boundaries, how it relates to event sourcing, and what it takes to operate, secure and test.

---

## Concept 83 — Scope: per context, per read path

Udi Dahan's and Greg Young's most repeated warning, and Fowler's: **CQRS belongs inside specific bounded contexts, not across a whole system.** The more precise version, which this module has been building toward: it belongs to specific **read paths** inside specific contexts.

Where it pays most — Dahan's observation in *Clarified CQRS* and later talks:

- **Collaborative domains**, where many users act on the same data concurrently (booking, trading, dispatch, inventory). Commands that capture intent conflict less than CRUD saves; read models can show a shared picture that's allowed to lag slightly.
- **Core domains with rich rules** (Module 22, Concept 11), where freeing the write model from display concerns lets it stay behaviour-focused.
- **High read:write asymmetry with divergent shapes**, where precomputed read models save real money (Concept 90).

Where it rarely pays:

- **Supporting subdomains with CRUD semantics** — reference data, settings, admin screens. Level 0 or level 1 at most.
- **Low-traffic internal tools** where the single-model tax is paid by nobody.
- **Systems whose reads and writes have the same shape** (Concept 5).

The practical form of the rule is a **per-context decision record**: for each bounded context, which read paths exist, what staleness each tolerates (Concept 57), and which level each uses. That table is one of the most useful artifacts an architect can produce for a CQRS system, because it makes the complexity budget explicit — and reviewable.

---

## Concept 84 — CQRS in Clean Architecture and vertical slices

Module 20 established the dependency rule and the layers; Concept 72 established slices. Here's where each CQRS piece lives.

**In a layered (Clean Architecture) solution:**

```
Ordering.Domain            aggregates, value objects, domain events          — knows nothing about reads
Ordering.Application       commands + command handlers; query contracts       — depends on Domain
                           (query records, DTOs, IOrderQueries ports);
                           command/query handler interfaces, decorators
Ordering.Infrastructure    EF write model and mappings, repositories,         — implements Application's ports
                           outbox, query implementations (EF projections,
                           Dapper), read-model DbContext
Ordering.ReadModels        projection handlers, read-model schema and          — depends on event contracts, not Domain
                           migrations, checkpoint store
Ordering.Api               endpoints, composition root                        — references everything
Ordering.Projector         worker host that runs async projections            — optional: can live in Api for small systems
```

Four placement rules:

- **The domain knows nothing about queries.** No `GetOrdersForDashboard` on an aggregate, no read DTOs in the domain project.
- **Query contracts belong to the application layer; query implementations may know SQL.** Module 20 (Concept 34): the port is declared in Application, and the adapter in Infrastructure is unapologetically database-aware. That keeps the dependency rule intact while letting the read side use the database properly.
- **Projections depend on event contracts, not on the domain model.** A projector that references aggregate classes couples the read side's deployment to the write model's internals. Projections consume the context's published event types (Module 22, Concept 83) — which also means they can move to another process or service unchanged.
- **Read-model schemas are owned by the projection.** Their migrations live with the projection code, separate from the write model's migrations, so they can be dropped and rebuilt independently (Concept 44).

**In a vertical-slice solution**, each query slice owns its query implementation — often a handler containing an EF projection or SQL directly — and each projection lives in a `ReadModels/` or `Projections/` area organized by read model. The domain and infrastructure are shared. Slices trade some layering purity for locality: everything about "search orders" is in one folder, including its SQL. Both structures are defensible; the CQRS placement rules above apply either way.

**In a modular monolith** (Module 21), each module has its own write model, read models and projections, in its own schema. A read model that combines several modules' data belongs to the **consuming** module, built from the others' published events (Concept 85).

---

## Concept 85 — Read models across modules and services

A screen that needs data from several contexts — an order-tracking page combining Ordering, Billing and Shipping — has three implementation options. Module 21 (Concept 38) introduced them; here's the full comparison:

| | **API composition** (BFF or gateway calls each context's query API and merges) | **Consumer-owned projection** (subscribe to each context's events; maintain a combined read model) | **Client-side composition** (the UI calls each API; micro-frontends) |
|---|---|---|---|
| Freshness | Current (each call hits live data) | Eventually consistent (lag per source) | Current |
| Latency | Sum or max of the calls; fan-out per request | One local read | Several browser calls |
| Availability | Product of the dependencies' availability (Module 21, Concept 21) | Independent of the sources at read time | Degrades per widget |
| Query flexibility | Limited: can't filter or sort across sources efficiently ("orders where payment failed and shipment is late") | Full: the combined model is queryable | Per widget only |
| Build cost | Low | Moderate: projections, idempotency, rebuilds (Part D) | Low server-side; UI complexity |
| Best for | Detail pages, low traffic, freshness-critical views | Lists, search, dashboards, cross-source filters, high traffic | Loosely related panels on one page |

The decision rule: **if the screen needs to filter, sort or search across the sources, or needs to be fast and available regardless of the sources, it needs a projection.** If it shows one entity's details from a few sources at modest traffic, composition is simpler and fresher.

Two ownership rules keep this healthy:

- **The team that owns the screen owns the composite read model.** It subscribes to the sources' published contracts; the sources don't know it exists and don't change for it (beyond publishing the fields their consumers need, negotiated as customer–supplier — Module 22, Concept 25).
- **Operational read models and analytics are different things.** A projection that powers a product screen has latency SLOs and a user waiting. Company-wide reporting over years of data belongs in an analytical platform (Fowler's *Reporting Database*, today typically a lakehouse such as Microsoft Fabric) fed from the same events or from CDC — with different freshness, different schemas and different owners. Don't let one grow into the other.

---

## Concept 86 — CQRS and event sourcing

Module 24 covers event sourcing; the relationship needs stating here because it's the most common conflation (Concept 7).

**Event sourcing needs CQRS.** An event store answers "give me the events of stream `order-42`" efficiently and almost nothing else. It can't answer "orders over €500 placed this week by customers in Serbia" without replaying everything. So any event-sourced system needs read models built by projections — which is CQRS at level 4.

**CQRS doesn't need event sourcing.** Levels 1–3 work with a conventional state-stored write model, which is what most CQRS systems in .NET have: EF Core aggregates, an outbox, projections from integration events or CDC.

**What event sourcing adds to CQRS** is precisely what level 3 lacks: a **replayable, complete history** as the source of truth. Concept 43's rebuild problem disappears — any read model, including one invented years later, can be built by replaying from the beginning. Temporal queries ("what did this look like on 1 March?") become possible. And the events are, by construction, intent-revealing facts rather than row diffs.

**How projections map onto the levels in an event-sourced system** — using Marten's terminology as the .NET example:

| Marten projection lifecycle | When it runs | Consistency | Equivalent level |
|---|---|---|---|
| **Inline** | In the same transaction as the event append | Strong | Level 2 |
| **Async** (the async daemon) | After commit, in a background process with checkpoints | Eventual | Level 3 |
| **Live** | On demand, by aggregating the stream at query time | Strong, computed per request | Like a level-1 query over the events |

The senior sentence: *"I'd decide on event sourcing on its own merits — audit, history, temporal questions, a replayable source for read models — not because we're doing CQRS. CQRS works fine with a state-stored write model; event sourcing makes the read side's rebuild story much better, at the cost of everything Module 24 is about."*

---

## Concept 87 — Operating a CQRS system

A level-3 system has more moving parts than a CRUD application, and operating it well is what separates a CQRS design that ages gracefully from one that becomes the team's least favourite system. What must exist before go-live:

**Dashboards**, per projection:
- Lag — time-based percentiles (p50, p99) and position-based backlog (Concept 50).
- Throughput — events applied per second, batch sizes, batch durations.
- Errors — retries, parked events, dead-letter counts, per error type.
- Checkpoint positions and the time each last advanced.

Plus, for the whole context: command outcomes by type (accepted, rejected, conflict), read latency by query, replica lag, and — on Cosmos DB — change-feed estimator values, GSI propagation latency and RU consumption.

**Alerts** — on SLO burn for lag, on lag *growth* (stuck projections), on any dead-lettered event, on a projector with no heartbeat, and on replica lag beyond the staleness budget.

**Tracing across the asynchronous gap.** A command's trace should connect to the projection work it caused. Propagate W3C trace context through the outbox and the broker (store `traceparent` with each outbox row; set it as a message property). Because a projector processes batches containing events from many traces, the projector's span should **link** to the originating spans rather than pretend to be a child of any one of them:

```csharp
var links = batch.Where(e => e.TraceParent is not null)
                 .Select(e => new ActivityLink(ActivityContext.Parse(e.TraceParent!, null)));

using var activity = ProjectorSource.StartActivity(
    $"project {Name}", ActivityKind.Consumer, parentContext: default, links: links);
```

OpenTelemetry's messaging semantic conventions describe exactly this batch-consumer shape, and Module 28 covers distributed tracing in depth.

**Runbooks**, written and rehearsed:
- Rebuild a read model (Concept 43), including how long it takes (Concept 91).
- Replay parked events after a fix (Concept 42).
- Pause and resume a projector safely (for a read-store migration or incident).
- Backfill a new read model into production without affecting live projectors.
- Deploy a projection change that requires a rebuild (blue/green, not in place).

**Capacity planning**: projector throughput must exceed peak event rate with headroom, and backlog drain time after an outage must be acceptable (Concept 92).

The review question that exposes an unoperated CQRS system: *"If the projector for the order list stopped at 2 a.m., how would you find out, and how long would it take to be correct again?"*

---

## Concept 88 — Security and privacy

CQRS changes the security surface in three ways: more copies of data, more processes with database access, and read models that can be shaped per audience.

**Least privilege, by side.** Separate database principals make the CQRS boundaries enforceable rather than conventional:

| Principal | Permissions |
|---|---|
| Command side (API/handlers) | Read/write on the write schema and its outbox; **no** access to read-model schemas |
| Projector | Read on its source (outbox, change feed); read/write on **its own** read-model schema only |
| Query side (API/handlers) | **Read-only** on read-model schemas (and on write tables for level-1 queries); ideally via a read-only replica connection |

With managed identities on Azure, each process gets its own identity and grants; on Cosmos DB, data-plane RBAC roles scope identities to containers. A compromised query endpoint then can't write anything, and a buggy projector can't corrupt the write model.

**Minimize what read models copy.** Every read model is another place personal data lives. Copy only the fields its queries need; prefer identifiers and display names over full profiles; pseudonymize where the read model doesn't need the real value (analytics rarely needs names).

**Audience-specific read models** (Concept 33) make field-level exposure explicit and limit the blast radius of a leak: the partner API's read model simply doesn't contain the fields partners may not see.

**Erasure must propagate.** Under GDPR's right to erasure (Article 17) and similar laws, deleting a customer means deleting — or anonymizing — them from every read model, cache, search index, replica and backup-restoration path. That requires:
- An explicit event (`CustomerErased`) that **every** projection handles, not just the primary one.
- Cache and search-index eviction on the same event.
- An inventory of which read models hold which personal fields — maintained as projections are added.
- For event-sourced systems and replayable logs, a strategy for personal data *inside events* — commonly **crypto-shredding** (encrypt personal fields per data subject; delete the key to erase) — which Module 24 takes up.

**Data residency.** Read replicas and geo-distributed read stores can move personal data across borders. The read side's topology is part of the data-protection design, not only the performance design.

---

## Concept 89 — Testing CQRS

Each part of a CQRS system has a natural test shape.

**Commands:** given/when/then on aggregates (Module 22, Concept 93) and outcome-focused handler tests (Concept 71). Nothing new.

**Projections:** *given these events, then these rows.* Run the projection handlers against a real read store (Testcontainers) or an in-memory equivalent that honours the same semantics, and assert on the resulting read model:

```csharp
[Fact]
public async Task Order_summary_reflects_placement_and_ignores_duplicates()
{
    var orderId = Guid.NewGuid();
    var events = new StoredEvent[]
    {
        Evt(orderId, version: 1, new OrderDraftedV1(orderId, CustomerId, "EUR", T0)),
        Evt(orderId, version: 2, new OrderLineAddedV1(orderId, "SKU-1", 2, 10m, NewOrderTotal: 20m)),
        Evt(orderId, version: 3, new OrderPlacedV1(orderId, CustomerId, 20m, "EUR", LineCount: 1, T1)),
    };

    await projector.ProcessBatchAsync(events, ct);
    await projector.ProcessBatchAsync(events, ct);                 // at-least-once: replay the whole batch

    var row = await readDb.OrderSummaries.SingleAsync(r => r.OrderId == orderId, ct);
    row.Status.ShouldBe("Placed");
    row.Total.ShouldBe(20m);
    row.LineCount.ShouldBe(1);                                      // not 2: the duplicate was ignored
    row.Version.ShouldBe(3);
}
```

Four properties every projection should have a test for:

- **Idempotency** — apply the stream twice; the read model is unchanged by the second pass (above).
- **Ordering tolerance** — for state-carrying events, shuffle the events of *different* aggregates (and, where the "newer wins" rule applies, deliver an older event after a newer one); the result is the same.
- **Rebuild equivalence** — build the read model incrementally in several batches and by a single replay from zero; the results are identical. This catches projections that depend on processing history rather than on the facts.
- **Erasure** — after `CustomerErased`, no row in the read model contains the customer's personal fields.

**Contracts between publisher and projection.** When the publisher is another team's context, consumer-driven contract tests (Module 21) verify that the events the projection depends on still carry the fields it reads.

**End-to-end tests with eventual consistency.** A test that sends a command and then immediately queries a level-3 read model will be flaky. Never `Thread.Sleep` — poll with a timeout:

```csharp
public static async Task<T> Eventually<T>(Func<Task<T?>> probe, Func<T, bool> until,
                                          TimeSpan? timeout = null, CancellationToken ct = default) where T : class
{
    var deadline = DateTime.UtcNow + (timeout ?? TimeSpan.FromSeconds(10));
    while (true)
    {
        var value = await probe();
        if (value is not null && until(value)) return value;
        if (DateTime.UtcNow > deadline) throw new TimeoutException("Read model did not converge in time.");
        await Task.Delay(50, ct);
    }
}

var summary = await Eventually(async () =>
{
    using var response = await client.GetAsync($"/orders/{id}", ct);
    return response.IsSuccessStatusCode                              // 404 / 202 = "not yet": keep polling
        ? await response.Content.ReadFromJsonAsync<OrderSummaryDto>(ct)
        : null;
}, s => s.Status == "Placed", ct: ct);
```

Note the probe. `GetFromJsonAsync` would be shorter, but it throws on any non-success status. A read model that doesn't have the row *yet* answers 404 (or 202 with a consistency token, Concept 53), so the test would fail on the first poll instead of waiting.

Better still, when the messaging framework supports it, **wait for processing to complete deterministically**: Wolverine's tracked sessions and MassTransit's test harness let a test wait until every message caused by an action has been handled, which removes timing from the test entirely. With a consistency token (Concept 53), the test can also simply pass `minVersion` and let the query side do the waiting — which tests that mechanism at the same time.

---

# Part I — The numbers

Most CQRS discussions are qualitative: "reads scale better," "projections add complexity." Interviewers at senior level want to see the arithmetic that turns those phrases into a decision. Four calculations cover almost every situation: whether a separate read model pays, how long a rebuild takes, how lag and recovery behave, and what denormalization costs when a copied value changes.

---

## Concept 90 — When a separate read store pays

The decision to move a read path up the spectrum is an economic one: **the cost saved per read, times the read rate, against the cost of maintaining the read model, times the write rate, plus a fixed operational cost.**

A worked example — an order-tracking dashboard for customer-service agents:

| Parameter | Value |
|---|---|
| Peak read rate | 3,000 queries/s |
| Level-1 query | a 7-table join with filters; ~15 ms of database CPU per execution at realistic cache hit rates |
| Level-3 read model | a single denormalized row per order, fetched by key; ~0.5 ms of CPU per read |
| Write rate | 60 commands/s, ~2 events each → 120 events/s |
| Projection cost | ~2 ms of CPU per event (read-modify-write with a version guard) |

The arithmetic:

- **Level 1:** 3,000 × 15 ms = **45 CPU-seconds per second** — about 45 database cores just for this screen, plus the lock and buffer-pool pressure on the write tables.
- **Level 3:** reads 3,000 × 0.5 ms = 1.5 cores; projection 120 × 2 ms = 0.24 cores. **About 2 cores in total** — plus a projector process, monitoring and the eventual-consistency design work.

At this read rate, the read model is overwhelmingly worth it. Now change one number: the same screen used by 20 back-office staff at **2 queries per second**. Level 1 costs 2 × 15 ms = 0.03 cores. The read model saves nothing worth its operational cost; level 1 is correct.

The general shape:

```
saving  ≈ R × (c_query_level1 − c_query_readmodel)
cost    ≈ E × c_projection + fixed operational cost (projector, monitoring, rebuild tooling, design effort)
move up the spectrum when saving ≫ cost — and when latency or isolation requirements demand it regardless
```

Three refinements an interviewer will appreciate:

- **The ratio that matters isn't reads:writes alone** — it's reads × per-read saving versus writes × per-write projection cost. A 1,000:1 ratio with a cheap query doesn't justify a read model; a 5:1 ratio with a 200 ms query might.
- **Try the cheap levels first.** A covering index (Concept 27) might cut the level-1 query from 15 ms to 2 ms; an 80% cache hit rate (Concept 30) cuts the effective read rate five-fold. Either can make the read model unnecessary. Quote the calculation after each step.
- **Latency and isolation are separate reasons.** Even when the CPU arithmetic is marginal, a p99 target of 20 ms that the join can't meet under load, or a need to stop dashboard traffic from affecting order placement, can justify level 3 on their own.

---

## Concept 91 — Projector throughput and rebuild time

Rebuild time (Concept 43) is determined by projector throughput, and throughput is determined almost entirely by **round trips**.

**One event at a time.** A projector that, per event, reads the row and writes it back — two round trips of ~1 ms each — processes about **500 events per second per partition**, no matter how fast the CPU is.

**Batched.** Accumulate a few hundred events, compute the read-model changes in memory, and write them in one round trip (EF Core batches the updates in one `SaveChanges`; a table-valued parameter with one `MERGE`, or `SqlBulkCopy` into a staging table, does it more efficiently). A batch of 500 events in ~50 ms is **~10,000 events per second per partition**.

**Parallel partitions.** Events for different aggregates are independent (Concept 39), so partitions can be processed in parallel: 8 partitions × 10,000 = **~80,000 events per second** — until the read store's write capacity becomes the limit.

What that means for rebuilding 300 million events:

| Projector design | Throughput | Rebuild time |
|---|---|---|
| One event at a time, one partition | 500/s | 600,000 s ≈ **6.9 days** |
| Batched, one partition | 10,000/s | 30,000 s ≈ **8.3 hours** |
| Batched, 8 partitions | 80,000/s | 3,750 s ≈ **1 hour** |

The same code, three orders of magnitude apart. That's why "can we rebuild it?" must be followed by "in how long?" — and why production projectors are almost always batched.

Two faster routes for rebuilds specifically:

- **Bootstrap from current state** when history isn't needed (Concept 43): 20 million current rows bulk-copied at 50,000 rows per second is under 7 minutes, followed by a short catch-up from the live feed.
- **Build in memory, write once.** For read models that fit in memory, replay events into dictionaries keyed by read-model row, then bulk-insert the result. This eliminates per-event I/O entirely and is often an order of magnitude faster again.

---

## Concept 92 — Lag budgets and backlog drain

**Where lag comes from.** End-to-end lag is the sum of every stage between commit and read:

| Stage | Typical contribution |
|---|---|
| Outbox relay polling interval (if polling) | Up to the interval — average half of it (1 s polling → ~0.5 s average) |
| Broker publish and delivery | ~10–50 ms |
| Batching window (waiting to fill a batch) | Up to the configured maximum wait (e.g., 100 ms) |
| Processing the batch | Batch size × per-event cost, amortized |
| Read-side caching | Up to the cache TTL, if the query is cached |

To hit a p99 lag of one second, a 1-second outbox polling interval is already the entire budget. That's the practical case for push-based relays (CDC on the outbox, Concept 38) or short polling intervals — and for measuring each stage.

**Little's Law** (Module 6) relates the pieces: with events arriving at λ = 200 per second and an average lag W = 0.5 s, on average L = λW = **100 events are in flight** between commit and read model. If a dashboard shows "events waiting," that's the number it should hover around.

**Backlog drain after an outage** is the calculation that decides projector capacity. If a projector is down for T, the backlog is B = λT; once it restarts with capacity μ, the backlog drains at the **surplus** rate μ − λ, because new events keep arriving:

```
drain time = B / (μ − λ) = λT / (μ − λ)
```

With λ = 200 events/s and a 30-minute outage, B = 360,000 events:

| Projector capacity μ | Surplus μ − λ | Drain time |
|---|---|---|
| 250/s (1.25× peak) | 50/s | 7,200 s = **2 hours** |
| 1,000/s (5× peak) | 800/s | 450 s = **7.5 minutes** |
| 10,000/s (50× peak, batched) | 9,800/s | ~37 s |

Divide through by λ and the formula gets simpler still. With **k = μ / λ** (the projector's headroom multiple):

```
drain time = T / (k − 1)
```

The event rate cancels out: drain time depends only on the outage length and the headroom. At 2× headroom, recovery takes as long as the outage; at 5×, a quarter of it; at 1.25×, four times it. The curve is a hyperbola with its knee between about 2× and 5× — below 2× every extra bit of headroom buys a lot, above 5× it buys little. That makes a clean interview sentence: *"I size projectors at around five times peak, so recovery takes about a quarter of the outage."*

A projector sized just above peak load works fine in steady state and turns every 30-minute incident into two further hours of stale reads. **Size projectors for recovery, not for steady state** — several times peak event rate is a reasonable target, which batching (Concept 91) usually makes cheap.

---

## Concept 93 — Fan-out arithmetic

Denormalization (Concept 45) makes each copied value's *change* cost proportional to how many rows copy it. Estimate the maximum, not the average.

A worked example — customer names copied into order read-model rows:

| Quantity | Value |
|---|---|
| Average orders per customer | 40 |
| Largest B2B customer's orders | 250,000 |
| Customer renames per day | 500 |
| Batched update throughput | 5,000 rows/s |

- **Average case:** 500 renames × 40 rows = 20,000 row updates a day. Trivial.
- **Worst case:** one rename of the largest customer = 250,000 row updates ≈ **50 seconds** of projector time. If that projection runs in the same partition as normal order events, every event queued behind it waits — a 50-second lag spike on the order list for everyone in that partition, caused by one customer changing their name.

The same arithmetic for a product name copied into order-line rows: a popular product on 2 million lines is **400 seconds** of updates.

Design responses, from the arithmetic:

- **Reference instead of copy** where fan-out is high (Concept 45): a `CustomerNames` table in the read store, joined by key, makes a rename one row.
- **Isolate fan-out work** in a separate, lower-priority projection or partition, so a large fan-out delays only the copies of that value, not unrelated events.
- **Question the requirement**: if historical rows should show the value *as it was* (invoices, legal records), the fan-out is not just expensive but wrong (Concept 45).

The general rule: **for every value copied into a read model, know its change rate and its maximum fan-out.** If the product of the two can exceed your lag budget, don't copy it.

---

# Part J — Anti-patterns and the architect's view

The failure modes of CQRS are well documented, remarkably consistent across organizations, and — because they're usually visible in code — exactly what code-review and architect rounds put in front of you. This part catalogues them, gives the review order, and then turns to how to talk about CQRS in design and architect rounds.

---

## Concept 94 — CQRS as a top-level architecture

**The symptom:** "Our architecture is CQRS." Every context, every entity, every screen has commands, queries, handlers and — often — separate read databases fed by events, including the admin screen for maintaining country codes.

**Why it happens:** CQRS is learned from conference talks about its far end (level 3–4), and applied uniformly because uniformity feels like good architecture.

**What it costs:** eventual consistency where nobody needed it; projection pipelines for data that changes twice a year; a support burden of stale-read bugs (Concept 51); onboarding time; and — the worst cost — the complexity budget spent on supporting subdomains leaves none for the core.

**The fix:** the per-context, per-read-path decision table (Concepts 57 and 83). Most contexts collapse to level 1; a few read paths earn more. Removing unjustified level-3 read models is a legitimate, senior simplification — it has the same shape as merging two always-co-deployed microservices (Module 21).

The review sentence: *"Which of these read models would the business notice if it were a query over the write tables instead?"*

---

## Concept 95 — "We use MediatR, so we do CQRS"

**The symptom:** a codebase where every operation is an `IRequest`, the solution has `Commands/` and `Queries/` folders, and the query handlers load aggregates through the same repositories and map them with AutoMapper. There's one model and one database; the "queries" are the write model read through a different class name.

**Why it matters:** it's not CQRS in any useful sense (Concept 7). The single-model tax (Concept 6) is still being paid in full, now with an extra layer of indirection on top. And because the team *believes* it's doing CQRS, the actual benefits — purpose-built projections, a write model free of screen concerns — are never pursued.

**The fix:** make the query handlers actually query. Replace repository-plus-mapper with direct projections to DTOs (Concept 25), remove display-only getters and navigations from the aggregates, and measure the difference. The folder structure stays; the separation becomes real.

The interview sentence: *"MediatR is a way to dispatch handlers. Whether you're doing CQRS depends on whether your query handlers use a different model from your command handlers — and in a lot of MediatR codebases they don't."*

---

## Concept 96 — Commands that read, queries that write

Two mirror-image violations of CQS, each with legitimate exceptions worth knowing.

**Commands that return read models.** `PlaceOrder` returns the full order-details DTO, complete with customer name, product descriptions and shipment estimates. The command handler now loads data from other aggregates (or other contexts), the write path depends on read concerns, and the day that screen moves to a level-3 read model, the command's response and the page disagree. **Fix:** return the outcome, identity and version (Concept 12); let the client use technique 1 or 3 for read-your-writes (Concepts 52–53). **Legitimate exception:** returning data *the command itself produced* that the client can't know otherwise — a generated reference number, a computed expiry.

**Queries that write.** Three common forms:

| Form | Example | Verdict |
|---|---|---|
| **Observable state change** | `GetNotifications` also marks them as read | ❌ A retry, prefetch or cache changes state. Make "mark as read" a separate command the client sends deliberately |
| **Lazy creation** | `GetCart` creates an empty cart if none exists | ❌ Two concurrent gets create two carts. Make creation a command (or make the query return "no cart" and let the client create one) |
| **Access auditing** | Recording who viewed a patient record | ✅ Legitimate — often legally required — as long as it's not observable through the domain's queries and doesn't make the query fail or slow down: write the audit record asynchronously (a channel, the outbox, a log pipeline) |

The test (Concept 3): *can this query be retried, cached, prefetched or run against a replica without changing any answer the system gives?* If not, part of it is a command.

---

## Concept 97 — Deciding from the read model

**The symptom:** a command handler, validator or domain service reads a read model — or a replica, or a cache — to make a decision that enforces a rule. "Check the available-stock read model before reserving." "Validate that the email is unique against the customer search index." "Look up the customer's credit status in the dashboard table."

**Why it's dangerous:** the read model is stale by design at level 3 and unlocked with respect to the command's transaction at every level. Two concurrent commands both pass the check; a command passes on data that was true a second ago. The bug is intermittent, load-dependent and invisible in tests (Concept 55).

**The fix:** enforce invariants on the write side — in the aggregate, with a database constraint, or with a conditional write (Module 22, Concepts 66 and 70). Use the read model only for UX pre-checks and for supplying the version the user saw. When the business wants to honour what the user saw, use an explicit **quote** issued and validated by the write side (Concept 55).

The review question: *"For each check this handler performs, what data is it reading, and could that data be older than the transaction?"*

---

## Concept 98 — The ceremony stack

**The symptom:** a solution where changing one field on one screen touches nine files — the request DTO, the command, the validator, the handler, the generic repository, the entity, the AutoMapper profile, the response DTO, the query handler — and the team can't explain what several of those layers protect against. Often the stack is Clean Architecture + MediatR + AutoMapper + a generic `IRepository<T>` + a DTO per layer.

**Why it happens:** templates. Each piece is defensible in isolation; together they encode "enterprise" rather than any specific requirement, and CQRS gets blamed for the ceremony it didn't require.

**The fix — CQRS actually *removes* ceremony when it's applied properly:**

- **Query side:** project straight to the response DTO (Concept 25). No repository, no entity materialization, no mapping profile. The DTO inventory shrinks (Module 20, Concept 35).
- **Command side:** a command record built from domain types at the edge, a handler, an aggregate. No generic repository (Module 22, Concept 87); no AutoMapper — mapping from a request to a command is a line of code you can read, and where mapping volume genuinely justifies a tool, a source-generated mapper (such as Mapperly) avoids both the runtime surprises and the licensing question (AutoMapper 15+ is commercial on the same terms as MediatR).
- **Validation:** shape at the edge (.NET 10's built-in validation or a validator), rules in the aggregate — not both duplicated.

Module 20 (Concept 71) gave the metric: count files touched per one-field change across the last twenty feature PRs. If the number is high and nobody can name what each layer buys, the stack is ceremony.

---

## Concept 99 — Asynchronous projections without a safety net

**The symptom:** a level-3 read model built as "a background service that listens to events and updates a table," with none of Part D's properties. Typical signs:

- **No idempotency** — counters that drift upward after every redeployment or lease rebalance.
- **No ordering guard** — rows that occasionally show an older status than the one before.
- **Checkpoint advanced before the write** — events silently lost on crashes.
- **No poison handling** — one malformed event stops a partition for hours, or is swallowed by a `catch` that logs and moves on.
- **No lag metric** — the team learns about a stuck projector from a customer.
- **No rebuild path** — the source was a Service Bus topic with no retention; the read model can never be regenerated, so a projection bug is permanent.

**Why it matters:** each of these is a data-correctness problem that surfaces weeks later as "the numbers don't match," usually without a way to reconstruct what happened.

**The fix:** Part D's checklist, applied before the first production deployment — version guards or an inbox, atomic or idempotent checkpoints, a documented skip policy, lag SLOs with alerts, and a rehearsed rebuild. If the team can't afford those, the read path doesn't need level 3 — which is itself a useful conclusion (Concept 94).

---

## Concept 100 — Notifications as integration events

**The symptom:** after saving an order, a handler calls `publisher.Publish(new OrderPlacedNotification(...))`, and notification handlers send the confirmation email, call the warehouse's API and update another module's tables.

**What goes wrong** (Concept 68):

- **Published inside the transaction** — the email is sent and the warehouse is called even if the transaction then rolls back. Customers receive confirmations for orders that don't exist.
- **Published after the commit** — if the process crashes, is recycled or scales in between commit and the handlers finishing, the email and the warehouse call never happen. There's no retry and no record that they should have.
- **Sequential by default** — if the warehouse call throws, the email handler registered after it never runs, and the exception surfaces to the user as a failed order that actually succeeded.
- **Hidden coupling** — the order module now synchronously depends on the warehouse's availability and on another module's schema.

**The fix:** the outbox (Module 11). In-transaction domain-event handlers write integration events to the outbox as part of the same commit (Module 22, Concept 83); a relay publishes them after commit; consumers — the email sender, the warehouse integration, other modules — process them idempotently, with retries and dead-lettering. `INotification` keeps its legitimate role: in-transaction domain-event dispatch within one context.

---

## Concept 101 — The review catalogue

When a CQRS codebase is put in front of you — in a review round or on your first week in a team — check these in order. The order goes from "is the pattern real?" to "is it safe?" to "is it operable?"

**1. Is there actually a separation?** Do query handlers use a different model (projections, DTOs, read models) from command handlers, or do they go through the same repositories and aggregates? (Concept 95)

**2. Is each read path at a justified level?** Can someone name, for each level-2 or level-3 read model, what it buys and what staleness it tolerates? Is there a decision table? (Concepts 57, 83, 94)

**3. Are commands intent-shaped?** `RelocateCustomer`, not `UpdateCustomer`; domain types, not primitives; no client-supplied state. (Concepts 9, 15)

**4. Does each command handler change one aggregate in one transaction and return an outcome, not a view?** (Concepts 12, 17, 96)

**5. Do any commands, validators or domain services decide from the read model, a replica or a cache?** (Concepts 55, 97)

**6. Are business rules duplicated in queries?** Look for status logic, eligibility calculations and price arithmetic in SQL or LINQ projections. (Concept 32)

**7. Is read-side authorization centralized?** Tenant and ownership filters applied globally; caches keyed by audience; audience-specific DTOs. (Concept 33)

**8. For every asynchronous projection: idempotent? ordered? checkpointed atomically or idempotently? poison-safe? rebuildable — and in how long?** (Concepts 39–43, 91)

**9. Is lag measured, with an SLO and alerts, and do users have read-your-writes on their own changes?** (Concepts 50, 52–53)

**10. In the dispatch layer: do handlers inject the mediator? Are notifications used for integration? Is the behavior order correct — retry outside the transaction, caching after authorization, transactions only for commands?** (Concepts 64, 65, 68, 69)

**11. Is the mediator dependency's licence known and compliant, and are handlers insulated from it?** (Concepts 74, 81)

**12. Can the team explain what each layer in the solution protects against?** (Concept 98)

The first two questions find the expensive design problems; questions 5 and 8 find the data-correctness bugs; questions 9–11 find the operational and compliance risks.

---

## Concept 102 — CQRS in the design round

CQRS appears at almost every step of the 7-step framework (Module 3), and the strongest candidates introduce it without the acronym until the interviewer uses it.

| Step | Where CQRS shows up | What to say |
|---|---|---|
| **1. Requirements** | Read:write ratio; which reads must be fresh | *"Which views does a user look at right after they change something, and which can be a few seconds behind?"* |
| **2. Estimation** | Read QPS vs write QPS; per-read cost | *"About 3,000 reads a second against 60 writes — the order list is the hot path."* |
| **3. API** | Commands as intent-named POSTs; queries as GETs; 202 for async commands | *"`POST /orders/{id}/cancel` rather than a PATCH on status."* |
| **4. Data model** | The write model (aggregates) and, separately, read models per hot query | *"Orders are the consistency boundary for writes; the list view reads a denormalized table keyed by customer."* |
| **5. High-level design** | The projection pipeline if level 3: outbox → broker → projector → read store | Draw it only for the read paths that need it |
| **6. Deep dive** | Projection correctness; read-your-writes; rebuilds | *"The projector is idempotent via a version guard, and after placing an order the confirmation page passes the new version so it never shows stale data."* |
| **7. Wrap-up** | Lag monitoring, backlog recovery, rebuild time | *"I'd alert on projection lag above two seconds, and size the projector for five times peak so a 30-minute outage clears in minutes."* |

Three habits that score well:

- **Say the level and the reason.** *"This view can be a query over the write tables; this one needs its own read model because of the join cost at this rate; search goes to an index."*
- **Mention lag before the interviewer does** — and the user-visible fix. Nothing signals experience faster than naming the stale-read bug unprompted.
- **Keep MediatR out of the design round** unless asked. It's an implementation detail; bringing it up in a system design discussion suggests you think it's architecture.

---

## Concept 103 — CQRS in the architect round

The architect round asks different questions: not "how would you build it?" but "should we, what will it cost, and how do we keep it healthy?"

**Cost, stated in the executive's terms (Module 33):**

- **Infrastructure:** extra stores, projector compute, broker throughput, duplicated storage — usually small relative to what a read model saves on the primary at high read rates (Concept 90).
- **People:** eventual consistency is a skill. Teams new to it produce the Concept 99 failures. Budget for training, templates and review.
- **Operations:** dashboards, alerts, runbooks and rehearsed rebuilds (Concept 87) are a permanent cost.
- **Complexity budget:** every level-3 read model is a distributed-systems component that someone must understand at 2 a.m. Spend the budget in the core domain.

**Typical architect-round questions and the shape of good answers:**

- *"Twelve teams each built CQRS differently. Should we standardize?"* — Standardize the **properties** (idempotent projections, lag SLOs, rebuild runbooks, least-privilege principals, event contracts) and provide a paved-road library or template; don't standardize the **level** — that's decided per read path. Write it as a guideline plus a small set of ADRs.
- *"Should we standardize on MediatR given the licence change?"* — Frame it as build-vs-buy: licence cost vs migration cost vs dependency risk (Concept 75). Recommend insulating handlers regardless, and choose per organization policy — often "keep and license where heavily used; no mediator or an MIT option for new services." Record it in an ADR with a revisit date.
- *"Our read models are always wrong."* — Diagnose with the review catalogue (Concept 101): usually non-idempotent projections, no ordering guards and no lag monitoring. Fix the properties before adding features, and demote read paths that don't need level 3.

**The ADR** for a read path moving to level 3 should record: the read path, the measured read rate and query cost (Concept 90), the staleness tolerance signed off by the business (Concept 57), the source and its replayability (Concept 37), the read-your-writes technique (Concept 52), the lag SLO and alerts (Concept 50), the rebuild procedure and its measured duration (Concepts 43, 91), and the conditions under which the decision should be revisited.

---

## Concept 104 — The 60-second answers

**"What is CQRS?"**
> "Separating the model that changes state from the model that answers questions, because they have different requirements — the write side protects invariants and records intent, the read side is shaped for screens and scaled for traffic. It's a spectrum: the cheapest form is a separate query path that projects DTOs straight from the write tables, with no eventual consistency at all; the far end is separate read stores fed asynchronously by projections. It's not event sourcing, not two databases by definition, and not MediatR. I choose the level per read path, based on how different its shape is, how much traffic it takes and how stale it can be."

**"When would you use it, and when not?"**
> "The first level — separate query code over the same tables — almost always, because it removes the cost of loading aggregates for screens. Separate read models when a read path has a real shape or scale problem: an expensive join at thousands of reads a second, combinatorial search, or read load I need to isolate from writes. Not for CRUD areas, supporting subdomains or low-traffic tools — and not system-wide. Each step up needs a number: the query cost it saves and the staleness the business accepts."

**"How do you keep the read model consistent with the write model?"**
> "I publish events from the write side through an outbox in the same transaction, so nothing is lost. Projectors consume them at-least-once, so every projection is idempotent — version guards per row, or an inbox table for cross-aggregate counters — and checkpoints commit with the read-model changes where possible. Ordering is per aggregate via the partition key. I measure lag as a histogram and alert on it and on its growth, and I make sure every read model can be rebuilt, with the rebuild time known. For the user who just made a change, I give read-your-writes — usually by returning the new version from the command and having the query wait briefly for it."

**"Should we use MediatR?"**
> "It depends on what you'd use it for. Its real value is the pipeline — one place for validation, transactions, retries and logging around every handler. Its costs are indirection, runtime wiring and the temptation to use notifications as an event bus — and since version 13 it's commercial, with a free tier for small organizations. For a new service I'd start with plain handlers and decorators, or a source-generated MIT mediator if we want the same shape. For an existing MediatR codebase, a licence is often cheaper than migrating. Either way I'd keep handlers independent of the dispatcher, so the choice stays cheap to change."

---

## Concept 105 — The close

The paragraph that demonstrates judgment, for the end of a design discussion or an architect round:

> "The way I think about CQRS is as a sequence of decisions, each with a price. Separating the query code from the write model is nearly free and removes most of the pain of serving screens from aggregates, so I do that by default. Giving a read path its own schema or store buys speed, independence and isolation, and costs eventual consistency, a projection pipeline that must be idempotent, ordered, monitored and rebuildable, and a user experience that has to handle lag honestly — so I only pay it where the numbers say so, and I write down the staleness the business agreed to. Whatever dispatch mechanism packages the handlers — MediatR, a source-generated mediator, Wolverine or none — is a separate, smaller decision, and I keep the handlers independent of it. The measure of a good CQRS design isn't how many read models it has; it's that every one of them has a reason, a freshness target, and a way to be rebuilt."

---

# Putting it together

Six worked examples in the shape these questions actually arrive — two design rounds, a brownfield performance problem, a production incident, an architect-round decision and a code review.

---

## Worked example 1 — "Design order history and tracking for an online retailer"

**Get the deciding inputs first** (thirty seconds):

> *"Three things change the design: the read and write rates, which views a user looks at right after they act, and whether tracking needs data from other systems — payments, shipping."*

Say the interviewer answers: 2 million orders a day (~25 placements/s averaged over the day; ~120 events/s including status changes); 4,000 reads/s at peak across list, detail and tracking views; tracking shows payment and shipment status from separate Billing and Shipping services; customer-service agents search orders by many criteria.

**Classify the read paths** (Concept 57):

| Read path | Traffic | Freshness need | Level |
|---|---|---|---|
| Order confirmation (right after checkout) | ~25/s | The buyer's own order, immediately | Level 1 read of the write tables by ID — or level 3 with a consistency token |
| "My orders" list | ~1,500/s | Owner expects new orders at once; ~1 s otherwise | Level 3 read model keyed by customer, with the consistency token after placing |
| Order tracking (order + payment + shipment) | ~2,000/s | Seconds | Level 3 multi-source projection owned by the tracking team |
| Agent search (many filters, free text) | ~500/s | 10–30 s | Search index, push-fed by the projector |

**The write side:** `Order` aggregates (Module 22) in Azure SQL; commands `PlaceOrder`, `CancelOrder`, `ChangeDeliveryAddress`, each returning an outcome and the new version; domain events written to an outbox in the same transaction; a relay publishing to a Service Bus topic with `SessionId = orderId` for per-order ordering.

**The read side:**

- **OrderSummaries** (SQL, same server, `rm` schema): one row per order, version-guarded upserts, keyset-paged by `(CustomerId, CreatedAt, Sequence)`.
- **OrderTracking** (SQL `rm` schema): one row per order with columns owned by each source — Ordering, Billing, Shipping — each with its own version column, created by whichever event arrives first (Concept 46).
- **Agent search** (Azure AI Search): documents pushed by the projector; results return IDs, and detail pages load from `OrderTracking`.

**Read-your-writes:** `PlaceOrder` returns `{ orderId, version, position }`. The confirmation page reads by ID with `minVersion`; the "my orders" query passes `minPosition` and waits up to 2 s for the projection's checkpoint to pass it (Concept 53).

**The numbers that justify it** (Concept 90): the level-1 tracking query joins across three services' data — impossible without composition calls at 2,000/s, which would multiply availability (Module 21). The projection costs ~120 events/s × ~2 ms ≈ 0.25 cores.

**Correctness and operations:** projections batched (500 events), checkpoints committed with read-model changes in the same SQL transaction; poison events parked per order; lag histogram per projection with an SLO of "p99 < 2 s," alerts on level and growth; projector sized at 5× peak so a 30-minute outage drains in minutes (Concept 92); rebuild by bootstrap from the write tables plus catch-up, measured in rehearsal.

**What to say at the end:** *"Only two of the four read paths needed a separate model, and the only store beyond SQL is the search index, because the agents' filters are combinatorial. Everything eventual has a freshness number the business agreed and a metric that proves it."*

---

## Worked example 2 — "Our customer dashboard is slow. The team wants to introduce CQRS with Cosmos DB."

**Don't accept the solution; diagnose the problem** (Concept 90's "cheap levels first"):

> *"Before a new store, I'd like the query's plan, its rate and its latency distribution — the fix might be much smaller."*

Findings: 600 reads/s at peak; p50 180 ms, p99 1.9 s; the query joins `Orders`, `OrderLines`, `Payments`, `Shipments` and `Customers`, loads full aggregates via the repository with `Include` chains, then maps with AutoMapper; the database is at 70% CPU during peaks.

**Step 1 — level 1, properly (a day's work).** Replace the repository-and-mapper path with a single `Select` projection to the dashboard DTO (Concept 25). The SQL now fetches 11 columns instead of every column of five tables, with no tracking and no mapping. Measured: p50 35 ms, p99 220 ms; database CPU at peak drops to 45%.

**Step 2 — an index (an hour).** The plan shows a key lookup per order line for two projected columns. A covering index on `OrderLines(OrderId) INCLUDE (Quantity, UnitPrice)` removes it. p50 12 ms, p99 60 ms.

**Step 3 — caching (a day).** The dashboard is per customer and tolerates 30 s of staleness (the business confirmed). `HybridCache` keyed by customer, with tag invalidation from order events. Hit rate 75%: effective database load falls four-fold.

**Decision:** stop. The dashboard now meets its target at a small fraction of its original database load, with no new store, no projector and no eventual consistency beyond a 30-second cache the business signed off. The Cosmos DB proposal is recorded in the ADR as "not needed at current scale; revisit if reads exceed ~5,000/s or if the dashboard must combine data from outside this context."

**What this demonstrates:** CQRS is a spectrum, and the cheapest levels often solve the problem. The senior move is to measure after each step and stop when the requirement is met — and to write down the trigger for the next level.

---

## Worked example 3 — "After last night's deployment, customer order counts are wrong"

**Symptoms:** the "orders placed" counter on customer profiles is higher than the actual number of orders for about 3% of customers — all of them customers who ordered during the deployment window.

**Diagnosis:**

1. The profile counter is a level-3 read model: a projector consumes `OrderPlaced` from Service Bus and runs `UPDATE CustomerStats SET OrderCount = OrderCount + 1`.
2. During the rolling deployment, projector instances were stopped mid-batch. Service Bus message locks expired and messages were redelivered to the new instances — at-least-once delivery doing exactly what it promises.
3. The increment is not idempotent (Concept 40). Every redelivered `OrderPlaced` counted twice.

**Immediate fix:**

- Add an inbox table `rm.ProcessedEvents(Projection, DedupKey)` written in the **same transaction** as the counter update; skip events already recorded (Concept 40). Make the key deterministic: for `OrderPlaced` it can simply be the order ID, because an order is placed exactly once. That choice matters for the repair below.
- Complete (acknowledge) the message only after that transaction commits (Concept 41).

**Repair the data:**

- The counter can't be "un-incremented" selectively without knowing which events were duplicated, so rebuild it (Concept 43). The source is a Service Bus subscription: no retention, no replayable position. So bootstrap from the write side, and make the bootstrap and the live feed agree on identity:
  1. **Create v2's own subscription first.** From now on, every `OrderPlaced` reaches it.
  2. **Then take one consistent snapshot of the write side** (snapshot isolation): the counts (`SELECT CustomerId, COUNT(*) FROM Orders WHERE Status <> 'Draft' GROUP BY CustomerId`) *and* the list of placed order IDs.
  3. **Write both into v2:** the counts into `CustomerStats_v2`, and one inbox row per order ID.
  4. **Start the fixed projector on the v2 subscription.** An `OrderPlaced` already counted by the snapshot finds its order ID in the inbox and is skipped; one committed after the snapshot is new and counts once.

  The order of steps 1 and 2 is the whole trick. Snapshot first, subscribe second, and any order committed in between is in neither — lost for good. Subscribe first and the overlap is harmless, because the seeded inbox absorbs it. (With a *replayable* source such as the outbox table, you'd record its position instead of creating a subscription — Concept 43.)
- Compare v2 against v1 for customers outside the incident window (they should match), then swap reads.

**Prevent recurrence:**

- A projection test that applies every event twice and asserts an unchanged result (Concept 89), required for every projection.
- A reconciliation job that samples 1,000 customers nightly and compares the read model with a level-1 query of the write side — a cheap detector for drift of any cause.
- Graceful shutdown for projectors: stop receiving, finish or abandon the in-flight batch, then exit — which reduces redeliveries but doesn't replace idempotency.

**What to say in a behavioural round:** the root cause wasn't the deployment; it was a projection written for exactly-once delivery in a system that provides at-least-once. The durable fix was a property (idempotency) plus a test that enforces it, not a change to the deployment process.

---

## Worked example 4 — "Design room booking for a hotel chain"

A collaborative domain (Concept 83): many users compete for the same inventory.

**Commands and their consistency boundaries:**

- `HoldRoom(hotelId, roomTypeId, stayDates, holdId)` → a **hold** with a 15-minute expiry. The invariant is "held + booked ≤ capacity for each night," so the aggregate is `RoomTypeNight` inventory per hotel, room type and date — small, and contended only on the same nights (Module 22, Concept 66: split along the invariant).
- `ConfirmBooking(holdId, guest, paymentReference)` → converts the hold; fails if the hold expired.
- `CancelBooking(bookingId)` → releases inventory; applies the cancellation policy.

**The availability read model** — "which hotels have a deluxe double free for these three nights, under €180?" — is a search-shaped query over thousands of hotels, updated by `InventoryHeld`, `HoldExpired`, `BookingConfirmed` and `BookingCancelled`. It's level 3, a few seconds stale, and **it never decides anything** (Concept 55): a guest may see a room as available that was held a second ago; `HoldRoom` then returns `SoldOut`, and the UI offers alternatives.

**The price the user saw:** rates change during the day. The business wants "the price shown at checkout is the price charged." That's the **quote pattern** (Concept 55): `HoldRoom` issues a quote (price, currency, expiry) on the write side and returns it; `ConfirmBooking` references the quote. The availability read model's prices are only for browsing.

**Overbooking policy:** hotels deliberately overbook some room types by a percentage. That's an explicit policy object in the `RoomTypeNight` aggregate (Module 22, Concept 53), not a read-side calculation.

**Read-your-writes:** after `ConfirmBooking`, the confirmation page renders from the command's result (technique 1); "My trips" is a level-2 synchronous projection per guest, because guests always check it immediately and write rates per guest are tiny.

**Numbers:** 300 holds/s at peak during a flash sale; per-night inventory aggregates for a popular hotel's 1,000 room-nights see contention only on the same nights; availability search at 8,000 queries/s is served from the search index, which is exactly the read-shape asymmetry CQRS exists for.

---

## Worked example 5 — "We have 340 MediatR handlers across 6 services. Should we migrate off MediatR?"

**Inventory first** (Concept 81, step 0):

| Item | Count |
|---|---|
| Request handlers | 340 (210 commands, 130 queries) |
| Pipeline behaviors | 6 (telemetry, authorization, validation, idempotency, retry, unit of work) |
| `INotification` types / handlers | 22 / 41 |
| Handlers injecting `ISender`/`IMediator` | 9 |
| Stream requests, pre/post processors, exception handlers | 0 / 2 / 1 |
| Developers with programmatic access | 34 |
| Organization | ~\$40M revenue — not Community-eligible |

**Cost of licensing:** 34 developers → Professional tier, \$1,499 per year at list price.

**Cost of migrating** (estimate): the 340 handlers and 6 behaviors are mechanical — a few days per service with good tests. The 41 notification handlers need classification: say 25 are in-transaction domain-event handlers (move to the existing domain-event dispatcher), 12 perform external side effects (**these are bugs today** — Concept 100 — and move to the outbox), 4 are fire-and-forget cache invalidations. The 9 nested sends need redesign. Realistically 4–6 engineer-weeks including testing.

**Recommendation, framed for an ADR:**

> *"Buy the Professional licence now — it's cheaper than one week of migration and removes the compliance exposure today. Independently, fix the 12 notification handlers that perform external side effects by moving them to the outbox, and remove the 9 nested sends — those are correctness and design issues regardless of which dispatcher we use. Introduce our own `ICommandHandler`/`IQueryHandler` interfaces for new code, so handlers stop depending on MediatR types. For new services, default to no mediator library, or `Mediator` (MIT, source-generated) where a shared pipeline is needed. Revisit the licence decision at renewal: if by then fewer than half the handlers depend on MediatR types, complete the migration; otherwise renew."*

**What this demonstrates:** separating the licence decision (cheap, reversible) from the design debt it exposed (important, independent), and turning a vendor change into a plan to reduce dependency risk over time rather than a big-bang rewrite.

---

## Worked example 6 — Code review: a MediatR command handler

The code put in front of you:

```csharp
public class UpdateOrderCommand : IRequest<OrderDto>
{
    public Guid Id { get; set; }
    public string? Status { get; set; }
    public string? ShippingAddress { get; set; }
    public decimal? Discount { get; set; }
}

public class UpdateOrderHandler(AppDbContext db, IMediator mediator, IMapper mapper, IReportingDb reporting)
    : IRequestHandler<UpdateOrderCommand, OrderDto>
{
    public async Task<OrderDto> Handle(UpdateOrderCommand request, CancellationToken ct)
    {
        var stats = await reporting.GetCustomerStatsAsync(request.Id, ct);
        if (request.Discount > 0 && stats.LifetimeValue < 1000)
            throw new ValidationException("Discount not allowed");

        var order = await db.Orders.Include(o => o.Customer).Include(o => o.Lines)
                                   .SingleAsync(o => o.Id == request.Id, ct);
        if (request.Status != null) order.Status = request.Status;
        if (request.ShippingAddress != null) order.ShippingAddress = request.ShippingAddress;
        if (request.Discount != null) order.Discount = request.Discount.Value;
        order.Customer.LastActivity = DateTime.Now;

        await db.SaveChangesAsync(ct);
        await mediator.Publish(new OrderUpdatedNotification(order.Id), ct);   // handlers email the customer and call the ERP
        await mediator.Send(new RecalculateLoyaltyPoints(order.CustomerId), ct);

        return mapper.Map<OrderDto>(await db.Orders.Include(o => o.Customer).Include(o => o.Lines)
                                                   .Include(o => o.Payments).SingleAsync(o => o.Id == order.Id, ct));
    }
}
```

**The review, in the order of Concept 101:**

1. **No intent.** `UpdateOrderCommand` with optional fields is three commands in a trench coat (Concept 15): changing status, changing the address and applying a discount have different rules, permissions and events. Split into `ChangeShippingAddress`, `ApplyDiscount` and — for status — the specific transitions (`CancelOrder`, `MarkShipped`), each intent-named.
2. **Primitive and mutable.** `Guid`, `string`, settable properties. Commands should be immutable records of domain types (`OrderId`, `Address`, `Percentage`).
3. **Deciding from a read model.** The discount rule checks `LifetimeValue` in the reporting database — stale and outside the transaction (Concept 55). The rule belongs to the write side: either the value the rule needs is maintained on the write side, or the rule is re-expressed on data the transaction can see.
4. **Invariants bypassed.** `order.Status = request.Status` sets any string, skipping every transition rule; `order.Discount = ...` bypasses pricing. These need aggregate methods (Module 22, Concept 40).
5. **Two aggregates modified.** `order.Customer.LastActivity` changes the customer aggregate via a navigation property (Module 22, Concept 61) — in the same transaction, by accident. If "last activity" matters, it's a projection from order events.
6. **`DateTime.Now`.** Local time and untestable; inject `TimeProvider` and use UTC.
7. **Notification as integration event.** Emails and ERP calls from `Publish` after `SaveChanges` are lost on a crash and block the response on the ERP's availability (Concept 100). Raise domain events; write integration events to the outbox.
8. **Handler calling handler.** `mediator.Send(new RecalculateLoyaltyPoints(...))` runs the whole pipeline again, possibly in the same transaction, and hides the dependency (Concept 69). Loyalty should react to an `OrderDiscountApplied` (or similar) event in its own transaction.
9. **Returning a read model.** Reloading with four `Include`s and mapping to `OrderDto` puts read concerns in the command path and doubles its database work (Concepts 12, 96). Return the outcome and new version; let the client re-query (with a consistency token if needed).
10. **Exceptions for business outcomes.** `ValidationException` for a business rule rejection; return an outcome type (Concept 18).
11. **Dependencies.** `IMediator`, `IMapper` and a reporting database in a command handler are three smells in one constructor — and two of the three libraries are commercial from their current major versions.

**The rewrite's shape:** `ApplyDiscount(OrderId, Percentage, int ExpectedVersion)` → handler loads the `Order`, calls `order.ApplyDiscount(percentage, discountPolicy)`, saves (state + `DiscountApplied` event → outbox) and returns `Applied(newVersion)` or a rejection. Twelve lines, one aggregate, one transaction, no reads for display.

---

# Interview questions and model answers

**"Is CQRS the same as having separate read and write databases?"** No. CQRS is separate *models*; separate databases are one option at the far end of a spectrum. The cheapest level uses the same tables with a separate query path and has no eventual consistency at all (Concepts 7–8).

**"Does CQRS require event sourcing?"** No. Event sourcing requires CQRS, because an event store can't answer attribute queries; CQRS works fine with a state-stored write model. Event sourcing adds a replayable source, which solves the read side's rebuild problem (Concept 86).

**"Can a command return a value?"** Yes — information about the command: its outcome, the identity it created, the new version. Not a view of the system's state; that's a query, and keeping it separate means the read side can move without changing the command (Concept 12).

**"How do you handle a user not seeing their own change?"** Read-your-writes: render from the command's result where possible; otherwise return a version or position from the command and have the query wait briefly until the read model reaches it; or project the user's own view synchronously. For replicas, route the user's reads to the primary for a short window (Concepts 52–53).

**"How do you make a projection idempotent?"** Version guards on per-aggregate rows (apply only newer versions), an inbox table in the same transaction for cross-aggregate aggregations, set-style writes instead of increments, and deterministic IDs for inserted rows. Then test it by applying every event twice (Concepts 40, 89).

**"What if events arrive out of order?"** Per-aggregate ordering via the partition or session key; a version guard for the rest. With delta events, a gap means waiting or parking; with state-carrying events, "newer wins" makes order irrelevant (Concept 39).

**"How would you rebuild a read model?"** Build a new version alongside the live one — by replaying a replayable source, or by bootstrapping from current state after recording the feed position and then catching up — compare, then swap. Know how long it takes: batching and partitioned parallelism are the difference between days and an hour (Concepts 43, 91).

**"Should a command handler use the read model to validate?"** Only for UX pre-checks. Invariants are enforced on the write side inside the transaction; the read model is stale by design and not locked. When the business wants to honour what the user saw, use a write-side quote (Concept 55).

**"What's the difference between the mediator pattern and MediatR?"** The GoF mediator encapsulates coordination logic among colleague objects; MediatR is an in-process request dispatcher with a pipeline — closer to a command dispatcher plus chain of responsibility (Concept 58).

**"What's the value of MediatR?"** The pipeline: one place for logging, validation, authorization, idempotency, retries and transactions around every handler, and a uniform handler shape. Its costs are indirection, runtime wiring, hidden coupling through `IMediator` and notifications, AOT unfriendliness, and — since v13 — a commercial licence (Concepts 61–62).

**"In what order should pipeline behaviors run?"** Telemetry, authorization, validation, idempotency, concurrency retry, unit of work, handler. Retry must be outside the transaction so each attempt gets a fresh unit of work; caching (for queries) must be after authorization (Concept 64).

**"Why shouldn't handlers call `mediator.Send`?"** It re-runs the pipeline — validation, retries, transactions — nests transactions or retries, and hides dependencies. Extract shared logic into services, or make it an event-driven workflow (Concept 69).

**"Is `INotification` a good way to publish domain events?"** For in-transaction domain-event handlers that write to the same database, yes. For integration events or external side effects, no: it's synchronous, same-scope, sequential by default and not durable. Use the outbox (Concepts 68, 100).

**"What changed with MediatR's licence?"** From 13.0.0 (July 2025) it's RPL-1.5 or commercial; ≤ 12.5.0 stays Apache-2.0 without fixes. Community licence free under \$5M revenue and \$10M outside capital (not government); team-sized tiers otherwise; non-production use needs no licence; enforcement is log warnings only (Concepts 73–74).

**"What would you use instead of MediatR?"** Depends on need: plain handlers with decorators (no dependency); `Mediator` (MIT, source-generated, AOT); Wolverine (MIT, adds durable messaging and an outbox); FastEndpoints' command bus; Brighter/Darker. Or no mediator at all, with endpoint filters for HTTP concerns (Concepts 76–80, 82).

**"How do you know if your CQRS system is healthy?"** Projection lag as a histogram with an SLO and alerts on level and growth, dead-letter counts, checkpoint heartbeats, end-to-end freshness canaries, rehearsed rebuilds with known durations, and periodic reconciliation between read models and the write side (Concepts 50, 87).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Defines CQRS as "two databases and an event bus" | Defines it as separate models, then lays out the spectrum and chooses a level per read path |
| Applies CQRS to the whole system | Applies it per context and per read path, with a decision table |
| Conflates CQRS with event sourcing | States the dependency precisely: ES needs CQRS; CQRS doesn't need ES |
| Conflates CQRS with MediatR | Treats MediatR as a dispatch/packaging choice, separate from the architectural pattern |
| Jumps straight to a separate read store | Tries a projection, an index and a cache first, measuring after each step |
| No numbers | Quotes read rate × per-read cost vs projection cost; rebuild time; backlog drain time |
| "It's eventually consistent" and stops | Gives the lag SLO, how it's measured, and how users get read-your-writes |
| Ignores the stale-read-after-write bug | Names it unprompted and describes the consistency token |
| Validates commands against the read model | Enforces invariants on the write side; read model for UX hints only; quote pattern when needed |
| Projections that increment counters blindly | Version guards, inbox tables, set-style writes, and a twice-applied test |
| Checkpoint advanced before the write | Atomic checkpoint with the read model, or idempotent writes when atomicity is impossible |
| No rebuild plan | Replayable source or bootstrap-plus-catch-up, blue/green swap, measured duration |
| Copies every value into every read model | Estimates fan-out and change rate; references high-fan-out values; asks snapshot vs current |
| Commands named `UpdateX` with optional fields | Intent-named commands with domain types, expected version and idempotency key |
| Commands return full DTOs | Return outcome, identity and version; the client queries separately |
| Uses `INotification` for emails and integrations | Uses the outbox; keeps notifications for in-transaction domain events |
| Handlers inject `IMediator` | Bans it by architecture test; extracts services or uses events |
| Retry behavior inside the transaction | Retry outside, fresh unit of work per attempt; transactions only for commands |
| "MediatR will break when the licence expires" | Knows enforcement is logging only — and that compliance is still required |
| "We can't afford MediatR" (without numbers) | Compares licence price to migration cost; separates licence from design debt |
| Picks an alternative by popularity | Chooses by need: no library, source-generated, messaging framework, or framework-native |
| Business rules duplicated in SQL for display | Computes at write time, projects from events, or shares one expression |
| Treats read models as permanent data | Treats them as disposable and rebuildable; only projections write them |

---

## Practice exercises

**Exercise 1 — Climb the spectrum, measuring each step (half a day).** Take an EF Core application (or eShop's ordering service) and a list screen that loads aggregates via a repository. Implement, in order: (a) a `Select` projection with no repository; (b) a covering index; (c) `HybridCache` with tag invalidation; (d) a level-2 read table maintained in the same transaction; (e) a level-3 projection via an outbox and a `BackgroundService`. Load-test each with a fixed read rate (k6, NBomber or Bombardier) and record p50/p99 latency and database CPU. Write down where you would have stopped — this is the Concept 90 argument with your own numbers.

**Exercise 2 — Break a projection, then make it correct (2 hrs).** Build a projector for `OrderPlaced → CustomerStats.OrderCount++`. Write a test harness that (1) delivers every event twice, (2) shuffles events across aggregates, (3) kills the projector between the read-model write and the checkpoint. Watch the counter drift. Then add an inbox table, atomic checkpoints and version guards until all three tests pass (Concepts 39–41).

**Exercise 3 — Read-your-writes three ways (2 hrs).** With a level-3 order list, reproduce the stale-read bug from Concept 51 (add an artificial 1-second projection delay). Fix it three ways — client-side merge of the command's result, a consistency token with bounded waiting, and an SSE notification using `TypedResults.ServerSentEvents` — and measure the latency each adds to the post-write read.

**Exercise 4 — Rebuild under traffic (2 hrs).** With the Exercise 2 projector, implement a blue/green rebuild: create `CustomerStats_v2`, bootstrap from the write tables after recording the feed position, catch up, compare with v1, switch reads, drop v1 — all while a load generator keeps placing orders. Time each phase and extrapolate to 300 million events (Concept 91).

**Exercise 5 — Order the pipeline (1 hr).** In a MediatR (or `Mediator`) project, register the six command behaviors from Concept 64 in the wrong order — retry inside the unit of work, validation after idempotency — and write tests that expose each mistake (a concurrency conflict that retries on a stale `DbContext`; a validation failure that consumes an idempotency key). Then fix the order and keep the tests.

**Exercise 6 — Migrate a slice off MediatR three ways (half a day).** Take one feature (command + query + two behaviors) and implement it with (a) your own handler interfaces and Scrutor decorators, (b) `Mediator` with source generation, (c) Wolverine in `MediatorOnly` mode. Compare: lines of code, how you navigate from endpoint to handler, how behaviors are ordered, startup time, and — for (b) — publish with Native AOT. Note which differences would matter for your team (Concepts 76–80).

**Exercise 7 — Cosmos DB read side (2 hrs).** Create an `orders` container partitioned by `/orderId`. (a) Add a Global Secondary Index partitioned by `/customerId` and query it; watch the propagation-latency metric while writing. (b) Build a change-feed-processor projection that maintains a per-customer dashboard document with totals — something a GSI can't do — with version-guarded, idempotent updates. Compare RU costs and lag of the two approaches (Concept 48).

**Exercise 8 — Write the decision table and the ADR (1 hr).** For a system you know, list every significant read path with its rate, current cost, tolerable staleness and chosen level (Concept 57). Pick the one that most needs to change and write the Concept 103 ADR for it, including the lag SLO, the rebuild procedure and the revisit trigger. Do the same for the mediator decision (Concept 82).

---

# Free resources

Every resource below is free to read or watch. Links were checked in September 2026; where a talk is best found by title, it's listed without a link.

### Primary sources — where the ideas come from

| Resource | What it covers |
|---|---|
| [CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) — Greg Young, 2010 | The original long-form statement: task-based UIs, commands, separating the models, and why the write side benefits most (**Concepts 4, 13**) |
| [Clarified CQRS](https://udidahan.com/2009/12/09/clarified-cqrs/) — Udi Dahan, 2009 | Collaborative domains, stale data as a feature, commands that rarely fail (**Concepts 4, 83**) |
| [When to avoid CQRS](https://udidahan.com/2011/04/22/when-to-avoid-cqrs/) — Udi Dahan, 2011 | The warning against applying it system-wide (**Concepts 83, 94**) |
| [Race Conditions Don't Exist](https://udidahan.com/2010/08/31/race-conditions-dont-exist/) — Udi Dahan, 2010 | Business answers to "what if two commands collide?" (**Concept 55**) |
| [CQRS](https://martinfowler.com/bliki/CQRS.html) — Martin Fowler | The concise definition and the "risky complexity" warning (**Concept 4**) |
| [CommandQuerySeparation](https://martinfowler.com/bliki/CommandQuerySeparation.html) — Martin Fowler | Meyer's principle and its pragmatic exceptions (**Concepts 2–3**) |
| [Command–query separation](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation) — Wikipedia | CQS's origin in Meyer's work and its relationship to CQRS (**Concept 2**) |
| [What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html) — Martin Fowler, 2017 | Notification, event-carried state transfer, event sourcing, CQRS — four different things (**Concepts 34, 39**) |
| [Reporting Database](https://martinfowler.com/bliki/ReportingDatabase.html) — Martin Fowler | The analytical cousin of the read model (**Concept 85**) |
| [Eager Read Derivation](https://martinfowler.com/bliki/EagerReadDerivation.html) — Martin Fowler | Computing read-optimized data at write time — the essence of a projection (**Concepts 32, 34**) |
| [Immutability Changes Everything](https://www.cidrdb.org/cidr2015/Papers/CIDR15_Paper16.pdf) — Pat Helland, CIDR 2015 | Append-only logs and derived views as a general data architecture (**Concepts 34, 86**) |
| [Turning the database inside-out](https://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html) — Martin Kleppmann | The log as source of truth; materialized views as derived, rebuildable state (**Concepts 34, 43**) |

### Practitioner articles — the CQRS debates

| Resource | What it covers |
|---|---|
| [Types of CQRS](https://enterprisecraftsmanship.com/posts/types-of-cqrs/) — Vladimir Khorikov | The spectrum this module's levels extend (**Concept 8**) |
| [CQRS facts and myths explained](https://event-driven.io/en/cqrs_facts_and_myths_explained/) — Oskar Dudycz | No, you don't need two databases, a bus or event sourcing (**Concept 7**) |
| [Can command return a value?](https://event-driven.io/en/can_command_return_a_value/) — Oskar Dudycz | The pragmatic answer, and the HTTP layer as a translator (**Concept 12**) |
| [CQS versus server generated IDs](https://blog.ploeh.dk/2014/08/11/cqs-versus-server-generated-ids/) — Mark Seemann | Client-generated identity dissolves the apparent CQS violation (**Concept 3**) |
| [CQRS Myths: 3 Most Common Misconceptions](https://codeopinion.com/cqrs-myths-3-most-common-misconceptions/) — Derek Comartin | Separate databases, event sourcing and async messaging are all optional (**Concept 7**) |
| [Why use MediatR? 3 reasons why and 1 reason not](https://codeopinion.com/why-use-mediatr-3-reasons-why-and-1-reason-not/) — Derek Comartin | Decoupling, application requests, pipelines — and the in-process notification problem (**Concepts 61, 68**) |
| [Stop Conflating CQRS and MediatR](https://milanjovanovic.tech/blog/stop-conflating-cqrs-and-mediatr) — Milan Jovanović | The distinction at the heart of this module (**Concept 95**) |
| [CQRS Pattern the Way It Should've Been From the Start](https://milanjovanovic.tech/blog/cqrs-pattern-the-way-it-should-have-been-from-the-start) — Milan Jovanović, May 2025 | Handler interfaces, decorators and Scrutor — CQRS without MediatR (**Concept 76**) |
| [CQRS Pattern With MediatR](https://milanjovanovic.tech/blog/cqrs-pattern-with-mediatr) — Milan Jovanović | The conventional MediatR approach, for comparison (**Concept 59**) |
| [Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/) — Jimmy Bogard | Organizing by feature; where MediatR's popularity came from (**Concept 72**) |
| [MediatR Pipeline Examples](https://lostechies.com/jimmybogard/2016/10/13/mediatr-pipeline-examples/) — Jimmy Bogard, 2016 | Logging, validation, authorization and pre/post processing as behaviors (**Concept 63**) |

### The MediatR licensing change — primary sources

| Resource | What it covers |
|---|---|
| [AutoMapper and MediatR Going Commercial](https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/) — Jimmy Bogard, 2 April 2025 | The announcement and the sustainability rationale (**Concept 73**) |
| [AutoMapper and MediatR Licensing Update](https://www.jimmybogard.com/automapper-and-mediatr-licensing-update/) — Jimmy Bogard, April 2025 | The intended model, before launch (**Concept 73**) |
| [AutoMapper and MediatR Commercial Editions Launch Today](https://www.jimmybogard.com/automapper-and-mediatr-commercial-editions-launch-today/) — Jimmy Bogard, 2 July 2025 | RPL-1.5 or commercial, tiers, Community edition, v13 (**Concepts 73–74**) |
| [MediatR pricing and FAQ](https://mediatr.io/) | Current tiers, prices and eligibility (**Concept 74**) |
| [Lucky Penny Licensing FAQ](https://luckypennysoftware.com/faq) | Who counts, non-production use, enforcement, expiry, agencies (**Concept 74**) |
| [Lucky Penny Software License Agreement](https://luckypennysoftware.com/license) | The EULA itself — what your legal team should read (**Concept 74**) |
| [MediatR on GitHub](https://github.com/LuckyPennySoftware/MediatR) · [releases](https://github.com/LuckyPennySoftware/MediatR/releases) · [wiki](https://github.com/LuckyPennySoftware/MediatR/wiki) | Source, release notes (13.x–14.2), licence-key configuration (**Concepts 59–60, 73**) |
| [MediatR on NuGet](https://www.nuget.org/packages/MediatR) | Version history and dates — including 12.5.0, the last Apache-2.0 release (**Concept 73**) |
| [jasontaylordev/CleanArchitecture discussion #1413](https://github.com/jasontaylordev/CleanArchitecture/discussions/1413) | A template user meets the licence warning; the maintainer's position (**Orientation**) |
| [dotnet/eShop `Directory.Packages.props`](https://github.com/dotnet/eShop/blob/main/Directory.Packages.props) | Microsoft's reference app pinning MediatR 13.0.0 "before license change" — check boundaries yourself (**Orientation**) |

### The alternatives

| Resource | What it covers |
|---|---|
| [Mediator (martinothamar)](https://github.com/martinothamar/Mediator) · [NuGet](https://www.nuget.org/packages/Mediator.SourceGenerator) | Source-generated, MIT, AOT; differences from MediatR; benchmarks (**Concept 77**) |
| [Using MediatR in .NET? Maybe replace it with this](https://www.youtube.com/watch?v=aaFLtcf8cO4) — Nick Chapsas | Video comparison of MediatR and Mediator (**Concept 77**) |
| [Wolverine as Mediator](https://wolverinefx.net/tutorials/mediator.html) | `InvokeAsync`, cascading messages, `MediatorOnly` mode (**Concept 78**) |
| [Wolverine for MediatR users](https://wolverinefx.net/introduction/from-mediatr) | Mapping MediatR concepts onto Wolverine (**Concepts 78, 81**) |
| [Wolverine — Vertical Slice Architecture tutorial](https://wolverinefx.net/tutorials/vertical-slice-architecture) | Slices without a mediator hop (**Concept 72**) |
| [Marten 9.0, Polecat 4.0 and Wolverine 6.0 are live](https://jeremydmiller.com/2026/05/24/marten-9-0-polecat-4-0-and-wolverine-9-0-are-live/) — Jeremy Miller, May 2026 | The Critter Stack 2026 release: cold start, AOT, new defaults (**Orientation, Concept 78**) |
| [FastEndpoints — Command Bus](https://fast-endpoints.com/docs/command-bus) | Commands, handlers, middleware, streaming commands, standalone use (**Concept 79**) |
| [Brighter](https://github.com/BrighterCommand/Brighter) · [Darker](https://github.com/BrighterCommand/Darker) | Command processor and query processor, MIT (**Concept 79**) |
| [Scrutor](https://github.com/khellang/Scrutor) | Assembly scanning and decorators for Microsoft DI (**Concept 76**) |
| [ardalis/CleanArchitecture](https://github.com/ardalis/CleanArchitecture) | A mainstream template that moved to the source-generated Mediator (**Orientation, Concept 77**) |
| [jasontaylordev/CleanArchitecture](https://github.com/jasontaylordev/CleanArchitecture) | A mainstream template that kept MediatR — compare the two (**Orientation**) |
| [FluentValidation](https://docs.fluentvalidation.net/) | Validators and DI integration, Apache-2.0 (**Concept 66**) |
| [Mapperly](https://github.com/riok/mapperly) | Source-generated object mapping — an alternative to AutoMapper where mapping volume justifies a tool (**Concept 98**) |

### Microsoft Learn — patterns and guidance

| Resource | What it covers |
|---|---|
| [CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) — Azure Architecture Center | Single-store vs separate-store CQRS, outbox and idempotent consumers, when not to use it (**Concepts 7–8, 13**) |
| [Materialized View pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view) | Precomputed, query-shaped views (**Concepts 27, 34**) |
| [Event Sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) | Preview of Module 24; projections from events (**Concept 86**) |
| [Asynchronous Request-Reply pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/async-request-reply) | 202 Accepted, status resources and polling (**Concept 21**) |
| [Transactional Outbox with Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-out-box-cosmos) | Atomic state change and event publication (**Concepts 36–37**) |
| [Idempotent Consumer pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/idempotent-consumer) | Handling duplicate delivery (**Concept 40**) |
| [Event-driven architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven) | The context projections live in (**Concept 36**) |
| [Implement reads/queries in a CQRS microservice](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/cqrs-microservice-reads) — *.NET Microservices* e-book | Dapper queries returning view models, bypassing the domain model — level 1 (**Concepts 24, 26**) |
| [Apply simplified CQRS and DDD patterns in a microservice](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/apply-simplified-microservice-cqrs-ddd-patterns) | The e-book's overall CQRS approach (**Concepts 8, 24**) |
| [Implement the microservice application layer using the Web API](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api) | Command handlers, MediatR behaviors and idempotent commands in the e-book (**Concepts 17, 63**) |

### EF Core and ASP.NET Core — the read side's tools

| Resource | What it covers |
|---|---|
| [Tracking vs. no-tracking queries](https://learn.microsoft.com/en-us/ef/core/querying/tracking) | When tracking applies, including inside projections (**Concept 25**) |
| [Efficient querying](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying) | Projections, indexes, loading only what you need (**Concepts 6, 25**) |
| [Advanced performance topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics) | Compiled queries, context pooling (**Concept 25**) |
| [Single vs. split queries](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries) | The Cartesian explosion and its trade-offs (**Concept 25**) |
| [SQL queries](https://learn.microsoft.com/en-us/ef/core/querying/sql-queries) | `SqlQuery<T>` for unmapped types (**Concept 25**) |
| [Keyless entity types](https://learn.microsoft.com/en-us/ef/core/modeling/keyless-entity-types) | Mapping views as read models (**Concepts 25, 27**) |
| [Pagination](https://learn.microsoft.com/en-us/ef/core/querying/pagination) | Offset vs keyset, and the unique-ordering warning (**Concept 29**) |
| [Global query filters](https://learn.microsoft.com/en-us/ef/core/querying/filters) | Tenant and soft-delete filters, including EF 10's named filters (**Concept 33**) |
| [Connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) | Execution strategies and user-initiated transactions (**Concepts 64–65**) |
| [What's new in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew) | Named query filters, complex types, `ExecuteUpdate` improvements (**Concepts 25, 33**) |
| [Output caching middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output) | Server-side response caching with tag eviction (**Concept 30**) |
| [HybridCache](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) | L1/L2 caching, stampede protection, tags (**Concepts 30, 67**) |
| [Validation in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/validation?view=aspnetcore-10.0) | .NET 10's built-in minimal-API validation (**Concept 66**) |
| [What's new in ASP.NET Core in .NET 10](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0?view=aspnetcore-10.0) | Validation, Server-Sent Events and more (**Orientation**) |
| [Create responses in Minimal API applications](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/responses?view=aspnetcore-10.0) | `TypedResults`, problem details, `ServerSentEvents` (**Concepts 18, 52**) |
| [Endpoint filters](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters) | ASP.NET Core's own pipeline for cross-cutting concerns (**Concept 79**) |
| [Dapper](https://github.com/DapperLib/Dapper) | The micro-ORM for hand-written read-side SQL (**Concept 26**) |

### Azure data platform — read replicas, change streams and read stores

| Resource | What it covers |
|---|---|
| [Read scale-out](https://learn.microsoft.com/en-us/azure/azure-sql/database/read-scale-out) — Azure SQL Database | `ApplicationIntent=ReadOnly` and the readable secondary (**Concept 31**) |
| [Hyperscale secondary replicas](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-replicas) | HA, named and geo replicas (**Concept 31**) |
| [Create indexed views](https://learn.microsoft.com/en-us/sql/relational-databases/views/create-indexed-views) | Requirements and costs of SQL Server's materialized views (**Concept 27**) |
| [REFRESH MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html) — PostgreSQL | Full and concurrent refresh (**Concept 27**) |
| [What is change event streaming?](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview?view=sql-server-ver17) · [FAQ](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/frequently-asked-questions-faq?view=sql-server-ver17) | SQL Server 2025 / Azure SQL streaming to Event Hubs — preview status, limits, the AMQP deprecation (**Concept 37**) |
| [SQL Server 2025 is now generally available](https://techcommunity.microsoft.com/blog/SQLServer/sql-server-2025-is-now-generally-available/4470570) | The release announcement, including change event streaming (**Orientation**) |
| [Change feed in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed) · [modes](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-modes) | Latest-version vs all-versions-and-deletes (**Concepts 37, 48**) |
| [Change feed processor](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-processor) · [estimator](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-use-change-feed-estimator) | Leases, at-least-once, per-partition ordering, lag (**Concepts 41, 48, 50**) |
| [Change feed design patterns](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-design-patterns) | Data movement to caches and search indexes, event sourcing (**Concept 48**) |
| [All versions and deletes change feed mode is GA](https://devblogs.microsoft.com/cosmosdb/azure-cosmos-db-all-versions-and-deletes-change-feed-mode-is-now-generally-available/) — June 2026 | Deletes and intermediate versions for projectors (**Concept 37**) |
| [Global Secondary Indexes](https://learn.microsoft.com/en-us/azure/cosmos-db/global-secondary-indexes) · [GA announcement](https://devblogs.microsoft.com/cosmosdb/announcing-general-availability-of-azure-cosmos-db-global-secondary-indexes/) | Platform-maintained, eventually consistent read containers — and their limits (**Concept 48**) |
| [Indexer overview](https://learn.microsoft.com/en-us/azure/search/search-indexer-overview) · [Cosmos DB for NoSQL indexer](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-cosmosdb-sql) — Azure AI Search | Pull-based index maintenance with change and deletion detection (**Concept 49**) |
| [Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html) — Debezium | CDC on the outbox table: the events-plus-CDC hybrid (**Concept 38**) |

### Code to read

| Repository | Why |
|---|---|
| [kgrzybek/sample-dotnet-core-cqrs-api](https://github.com/kgrzybek/sample-dotnet-core-cqrs-api) | CQRS with raw SQL reads and a DDD write side, compact enough to read in an evening |
| [kgrzybek/modular-monolith-with-ddd](https://github.com/kgrzybek/modular-monolith-with-ddd) | CQRS per module, outbox/inbox, cross-module integration (**Concepts 84–85**) |
| [oskardudycz/EventSourcing.NetCore](https://github.com/oskardudycz/EventSourcing.NetCore) | Projections, idempotency and read models, with and without event sourcing (**Part D, Concept 86**) |
| [Marten — projections](https://martendb.io/events/projections/) | Inline, async and live projection lifecycles (**Concept 86**) |
| [dotnet/eShop](https://github.com/dotnet/eShop) | Microsoft's reference app: MediatR behaviors, domain events via notifications, integration events via an outbox |
| [jbogard/ContosoUniversityDotNetCore-Pages](https://github.com/jbogard/ContosoUniversityDotNetCore-Pages) | Vertical slices with MediatR by the library's author (**Concept 72**) |

### Talks worth an hour (search by title unless linked)

| Talk | Why |
|---|---|
| **"CQRS and Event Sourcing"** — Greg Young, Code on the Beach 2014 | The originator on what CQRS is and isn't, with the history |
| **"Turning the database inside out with Apache Samza"** — Martin Kleppmann, Strange Loop 2014 | Derived views and logs as a general architecture — the conceptual basis for projections |
| **"SOLID Architecture in Slices not Layers"** — Jimmy Bogard | Vertical slices and where MediatR fits (**Concept 72**) |
| [**"Discovering The Truth About CQRS — No MediatR Required"**](https://www.youtube.com/watch?v=F3xNCfP3Xew) — Milan Jovanović | Separating the pattern from the library, with code (**Concepts 76, 95**) |
| **"CQRS Myths"** — Derek Comartin (CodeOpinion) | The video version of the three misconceptions (**Concept 7**) |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What CQRS is | Separate models for changing and reading state, chosen per read path; a spectrum, not a switch |
| Its origin | Meyer's CQS (methods) → Young's CQRS (objects/models), ~2010; a pattern, not an architecture |
| Why reads and writes differ | Shape, volume, consistency, scaling, latency, security, change cadence |
| The levels | 0 none · 1 separate query path, same tables · 2 read tables in the same transaction · 3 separate store, async projections · 4 event-sourced |
| The big jump | Level 2 → 3: eventual consistency and a projection pipeline appear |
| Whether it needs two databases / ES / a bus | None of them; each is an option the separation enables |
| Can a command return a value | Yes — outcome, identity, version; not a view of state |
| Command design | Intent-named record of domain types, expected version, idempotency key; no client-supplied state |
| Validation layers | Shape at the edge (400), invariants in the aggregate (409/422), concurrency (412), set rules via constraints |
| Idempotent commands | Natural idempotency, client IDs, aggregate state, or keys + stored outcomes in the same commit |
| Async commands | 202 + status resource; validate before enqueueing; design the rejection path |
| Level-1 read side | EF `Select` projections or Dapper; no repositories, no tracking, no aggregates |
| Cheap wins before a read store | Projection → covering index → cache; measure after each |
| Replicas | Level 1 on a replica is eventually consistent; route recent writers to the primary |
| Business rules in queries | Never re-implemented: compute at write time, project from events, or share one expression |
| What a projection is | A deterministic fold from facts to query-shaped state |
| Projection correctness | Ordered per aggregate, idempotent, atomic or idempotent checkpoint, poison-safe, rebuildable, observed |
| Idempotency in projectors | Version guards, inbox table in the same transaction, set-style writes, deterministic IDs |
| Ordering | Per-aggregate via partition/session key; version guard; "newer wins" for state-carrying events |
| Events vs CDC | Events carry intent and a contract; CDC is complete but couples to the schema; CDC on the outbox gets both |
| Rebuilds | Replayable source, or bootstrap after recording the position then catch up; with a non-replayable feed, subscribe *first*, then snapshot and seed a deterministic inbox; blue/green swap; know the duration |
| Cosmos DB read side | GSIs for "same documents, different key" (eventual, read-only, no filters/joins); change feed processor for reshaping |
| Lag | Time-based histogram, SLO in business terms, alerts on level and growth, freshness canaries |
| Read-your-writes | Use the command result; consistency token (version/position + bounded wait); route to primary; sync projection; push |
| Deciding from the read model | Never for invariants; UX hints and versions only; quote pattern to honour what the user saw |
| Staleness per query | A business answer: "what's the worst that happens if it's five seconds old?" |
| Mediator pattern vs MediatR | Coordination among colleagues vs an in-process request dispatcher with a pipeline |
| MediatR's real value | The pipeline: one place for cross-cutting concerns; a uniform handler shape |
| MediatR's costs | Indirection, runtime wiring, hidden coupling via `IMediator`/notifications, AOT, licence |
| Behavior order | Telemetry → authorization → validation → idempotency → retry → unit of work → handler |
| Transaction behavior | Commands only; one commit; outbox inside; execution strategy wraps the attempt; nested commands refused |
| Notifications | In-process, same scope, sequential, not durable — domain events yes, integration events no |
| Handlers calling handlers | Nested pipelines and transactions; extract services or use events; ban by architecture test |
| The licence | ≥ 13.0.0: RPL-1.5 or commercial; ≤ 12.5.0 Apache-2.0 unmaintained; Community < \$5M revenue & ≤ \$10M capital; team tiers; non-prod free; log-only enforcement |
| Options for existing code | License, pin 12.5.0, migrate, or remove the mediator — licence is often cheapest; insulate either way |
| Alternatives | Own interfaces + decorators · Mediator (MIT, source-gen, AOT) · Wolverine (MIT, + messaging/outbox) · FastEndpoints · Brighter/Darker |
| Migration | Inventory → characterization tests → insulate → untangle notifications and nested sends → port behaviors (check order!) → switch per slice → ban by test |
| When a read store pays | R × per-read saving ≫ E × projection cost + operational cost |
| Rebuild time | Events ÷ throughput; batching and partitions turn days into an hour |
| Backlog drain | λT ÷ (μ − λ) = T ÷ (k − 1) with k = μ/λ; size projectors for recovery — at 5× peak, recovery takes a quarter of the outage |
| Fan-out | Max fan-out × change rate vs lag budget; reference instead of copy when high |

---

## What this module closed, and what's next

Threads from earlier modules, now resolved:

- **Module 20's CQRS-lite** (Concepts 34–35) is level 1 of a five-level spectrum, with the arithmetic for when to go further (Concepts 8, 24, 90) — and its deferred MediatR question has a full answer (Parts F–G).
- **Module 21's read models and in-process messaging table** (Concepts 38, 49) are now a projection discipline (Part D) and a licensing and alternatives analysis (Part G).
- **Module 22's "reads bypass the aggregate"** (Concept 92), eventual-consistency-as-UX (Concept 84) and application-service shape (Concept 88) have become the read side's design (Part C), the read-your-writes techniques (Part E) and the command handler (Part B).
- **Module 11's outbox and idempotent consumers** are the transport and the correctness model for every asynchronous projection (Concepts 36–41).
- **Module 10's caching** reappears as a read model with a TTL (Concept 30), and **Module 7's session guarantees** as concrete techniques (Concepts 52–54).
- **Module 19's EF Core query tools** are assembled into the read side's toolkit (Concept 25), and **Module 8's replication lag** into the replica routing rules (Concept 31).

Threads left open on purpose:

- **Event sourcing** — where the write side *is* the event log, projections become the only way to query, rebuilds become replays, and events carry personal data that must be erasable — is **Module 24**.
- **Resilience with Polly** — composing the concurrency retry of Concept 64 with EF's execution strategy and HTTP retries without multiplying attempts — is **Module 25**.
- **Cosmos DB partitioning and RU economics** for write containers, read containers and GSIs — **Module 27**.
- **SLOs, error budgets and distributed tracing across asynchronous boundaries** — the lag SLOs of Concept 50 and the trace links of Concept 87 — are **Module 28**.
- **Security architecture** — managed identities and least-privilege principals per side (Concept 88) — is **Module 29**.
- **ADRs** for read-path levels and the mediator decision (Concept 103) — **Module 31**.
- **Build-vs-buy** — the executive framing of the licensing decision (Concept 75) — **Module 33**.

Next in the curriculum: **Module 24 — Event Sourcing**: what changes when the events of Module 22 and the projections of this module become the system of record — stream design, snapshots, event versioning and upcasting, projections as the only read path, the decider as the aggregate, Dynamic Consistency Boundaries versus stream-per-aggregate, erasing personal data from immutable logs, and when the whole approach is worth what it costs.
