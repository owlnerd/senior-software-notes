# Module 20 — Clean Architecture & Layering
*Phase 5: .NET Architecture Patterns · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **Clean Architecture is not a folder layout, a set of project names, or a template — it is one rule (source-code dependencies point inward, toward policy), implemented by one mechanism (dependency inversion at the boundaries you actually want to protect), paid for in one currency (indirection), and it is worth paying only where the boundary buys something specific: a decision you expect to change, a test you want to run without infrastructure, or a seam an organization needs to own.**

That reframing matters because the naive picture of Clean Architecture is a picture — four concentric circles, four projects, arrows pointing inward — and a mid-level candidate will draw it, name the layers, and say "the domain doesn't depend on anything." A senior candidate can tell you *why* a team with four projects and 180 interfaces still can't change their database (every repository returns `IQueryable`, so the domain depends on LINQ-to-SQL semantics it never named), why "the domain must not know about EF" survives first contact with a real schema only if you decide in advance which three compromises you'll accept (private setters, backing fields, and a shadow concurrency token — or you get a separate persistence model and pay for the mapping forever), why a one-line change to a validation message touched six files across four projects and nobody can defend it, why the composition root is the one place in the system that is *allowed* to know everything and why putting a `DbContext` registration in the Application layer quietly inverts the whole dependency graph, why "we can swap SQL Server for Cosmos" is a claim almost no team has ever cashed and yet the architecture that promised it still paid off for a different reason, why `IDateTime` interfaces are now dead code in .NET 8+ but `IClock`-shaped thinking is not, and why two of the libraries that define the canonical .NET Clean Architecture template became commercial licensing decisions in 2025.

This module opens **Phase 5** and settles the thread that Modules 18 and 19 both left open by name. **Module 18 (Concept 72)** built the composition root and the DI lifetime rules; this module decides *where that composition root lives* and what it is allowed to reference. **Module 19 (Concepts 65–68, 72)** gave you the honest analysis of repositories over EF, `IQueryable` leakage, EF-and-DDD mapping, and the ORM's stopping point; this module puts those in their architectural context and answers the question Module 19 deferred: **does "the domain must not know about EF" survive contact with a real schema?** (Short answer: partially, deliberately, and at a price you should be able to quote.) **Module 12** gave you storage trade-offs; **Module 13** gave you reliability patterns that mostly live in adapters; **Module 11** gave you the outbox, which is the canonical example of an application-layer port with an infrastructure implementation.

It shows up in five places in an interview loop: the **design round** ("how would you structure this service?"), the **code-review round** ("here's our solution — what would you change?"), the **deep technical round** ("what does the dependency rule actually mean? give me a case where you'd break it"), the **architect round** (defending a structure to engineers who have to live in it, and to a manager who wants to know what it costs), and the **brownfield round** ("this 400k-line app has business logic in controllers and stored procedures — what's your twelve-month plan?"). It is also the single most common place where candidates recite a diagram and get marked down for having no opinion about when *not* to use it.

**Current platform state (verified September 2026).** .NET 10 (November 2025) is the production-current LTS, supported to November 2028; .NET 11 RC1 shipped 8 September 2026 with a go-live licence ahead of GA on 10 November 2026. What changed in this module's ecosystem, and why each item matters architecturally:

- **MediatR and AutoMapper are now commercial products.** Both moved to Lucky Penny Software under a dual-licence model: the source is available under the Reciprocal Public License 1.5, and a commercial licence is the alternative for teams unwilling to take on RPL-1.5's reciprocal obligations. The split point is **MediatR v13.0.0 and later** and **AutoMapper v15.0.0 and later**; earlier versions (MediatR ≤ 12.5.0, Apache-2.0) remain under their original licences indefinitely, without support or security fixes. There is a free **Community tier** for organizations under roughly \$5M revenue that have not taken more than \$10M in outside capital, and commercial tiers are priced by team size (Standard 1–10, Professional 11–50, Enterprise unlimited) counting only developers with programmatic access. `MediatR.Contracts` is still published under Apache-2.0. MediatR's current line is v14.x. **Why it matters here:** the canonical .NET "Clean Architecture" solution is usually Clean Architecture *plus MediatR plus AutoMapper*, and two-thirds of that stack is now a procurement conversation. An architect who proposes the template without naming that is proposing an unpriced dependency. (Module 23 takes the MediatR question properly.)
- **The two reference templates both moved to .NET 10, and one of them now ships a vertical-slice alternative.** `ardalis/CleanArchitecture` is on .NET 10 and publishes **two** templates: `clean-arch` (the full Core / UseCases / Infrastructure / Web solution) and `min-clean` (a simplified single-project vertical-slice template). `jasontaylordev/CleanArchitecture` is on .NET 10 with template package 10.8.0 (March 2026), a dedicated docs site at cleanarchitecture.jasontaylor.dev as the single source of truth, Aspire-based functional tests, NSwag's runtime introspection and Swagger UI replaced by ASP.NET Core's built-in OpenAPI plus Scalar, and — deliberately — the MediatR licence warning in the build output, which the maintainer has publicly defended as an acceptable trade for maturity and community support.
- **Microsoft's own reference apps moved.** `dotnet-architecture/eShopOnWeb` — the sample behind the *Architecting Modern Web Applications with ASP.NET Core and Azure* e-book, and the app most Clean Architecture blog posts are downstream of — was **archived by Microsoft on 13 January 2025** and is now community-maintained at `NimblePros/eShopOnWeb`. The e-book itself is still live on Microsoft Learn and still names Clean Architecture as the target architecture, still pointing readers at the ardalis template. `dotnet/eShop` is the current first-party reference application: Aspire-hosted, service-per-bounded-context, and much less useful as a *layering* example than as a distributed-systems one.
- **Architecture-test tooling is split.** `NetArchTest.Rules` 1.3.2 (May 2021) is still the newest release on NuGet — the library works fine against modern .NET but is effectively frozen. The maintained path is the MIT-licensed fork **`NetArchTest.eNhancedEdition`** (1.4.5, June 2025) or **`ArchUnitNET`**, the .NET port of Java's ArchUnit, which is actively developed. If you say "we enforce the dependency rule in CI," expect to be asked which one and why.
- **`TimeProvider` killed a whole genre of hand-rolled ports.** Since .NET 8, `System.TimeProvider` is in the BCL with `Microsoft.Extensions.TimeProvider.Testing`'s `FakeTimeProvider` for tests. Every `IDateTimeProvider` / `IClock` / `IDateTime` interface in every Clean Architecture template written before 2024 is now a hand-rolled duplicate of a framework abstraction. Knowing that is a small, cheap, very specific currency signal.
- **Aspire moved the infrastructure half of the composition root out of process.** Aspire (the ".NET" was dropped from the name; the 13.x line shipped through 2026) puts resource wiring — databases, caches, queues, service discovery, connection strings — in an AppHost project that is *not* your application's composition root but sits beside it. This is genuinely new surface for the "where does configuration live?" question, and both templates now assume it.
- Minor but useful: **FluentValidation remains Apache-2.0** (12.x line) — the third leg of the usual template stack did *not* change licence. The **`.slnx`** XML solution format is supported by current SDKs, which matters more than it sounds: when the solution file is the architecture diagram, being able to review it in a pull request is a control.

This module has nine jobs:

1. **Reduce the diagram to a rule, and the rule to a mechanism.** What "dependencies point inward" actually constrains (source-code references), what it doesn't (control flow, data flow, runtime call direction), and the one polymorphism trick that makes it possible.
2. **Give the rule a lineage**, so you can say the true and useful thing: Boundary-Control-Entity, Ports & Adapters, Onion, and Clean are four names for one idea with different emphases, and knowing which emphasis you need is the actual skill.
3. **Make the .NET solution shape concrete** — projects, references, what belongs in each, and the composition root as a physical location with compile-time consequences.
4. **Settle the persistence seam** — repositories, unit of work, `IQueryable`, specifications, the `IApplicationDbContext` shortcut, and exactly how far "the domain doesn't know about EF" survives.
5. **Decide every cross-boundary question once**: mapping, validation, errors, time, IDs, configuration, external services, domain vs integration events, background work.
6. **Name the seams that are negative value** — the interfaces you should not write — because that list is what separates judgment from ritual.
7. **Present the alternatives honestly.** Vertical slices, transaction scripts, modular monoliths, functional core/imperative shell: what each optimizes, and the hybrid that most mature teams actually run.
8. **Make the architecture enforceable and measurable** — project references, banned-API analyzers, architecture tests, fitness functions, and the testing pyramid the structure is supposed to buy you.
9. **Raise it to the architect's job**: introducing it into a brownfield system, defending it in cost language, documenting it, and knowing when to collapse it.

Nine framings to carry through:

1. **The rule is about source-code dependencies, not runtime direction.** Infrastructure calls the domain at runtime constantly. What it must not do is make the domain's *compilation* depend on it.
2. **Boundaries, not layers.** A layer is one way to cut a system, and usually the least interesting one: it groups things that change for different reasons and separates things that change together.
3. **Every abstraction is a bet on a change that hasn't happened.** The premium is indirection, paid every day. Ask what you're insuring against and what the claim would be worth.
4. **Volatility decides direction, not importance.** Policy is what changes for business reasons; detail is what changes for technical ones. Depend on the stable thing.
5. **The composition root is the privileged place.** Exactly one location knows every concrete type. Everything else stays ignorant. If a second location starts knowing, you have two roots and no architecture.
6. **A leaky port is a rename, not a boundary.** `IQueryable`, `IDbContext`, `DbSet<T>`, `SqlException`, `HttpResponseMessage` crossing a boundary means the boundary isn't there.
7. **Testability is the honest justification.** "We could swap the database" is a claim almost nobody cashes. "We can run 4,000 domain tests in 900 ms with no container" is cashed every single commit.
8. **Granularity is per use case, not per application.** In one system: a CRUD screen that deserves a query and an endpoint, a workflow that deserves a transaction script, and a pricing engine that deserves a full domain model. Applying the same ceremony to all three is the actual mistake.
9. **Architecture that isn't enforced by a machine is a preference.** Project references, analyzers, and arch tests are the architecture. The diagram is a picture of it.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | The dependency rule | Source-code dependencies point inward; nothing inner names anything outer |
| 2 | Policy vs. detail | Policy = rules that would exist without computers; detail = how they're delivered |
| 3 | Volatility, not importance | You depend on the *stable* thing; stability is about reasons to change |
| 4 | Crossing a boundary | Control flows out, dependency points in — via an interface owned by the inner side |
| 5 | DIP vs. DI vs. IoC | A design principle, a technique, and a control-flow property — three different things |
| 6 | Ports and adapters | Driving (primary) ports are called by the outside; driven (secondary) ports are called by the inside |
| 7 | The lineage | BCE (1992) → Hexagonal (2005) → Onion (2008) → Clean (2012) — one idea, four emphases |
| 8 | Layers as a special case | Strict vs. relaxed layering; a layer is a boundary drawn along a technology axis |
| 9 | Screaming architecture | The top-level structure should name the business, not the framework |
| 10 | What it actually buys | Test speed, deferred decisions, enforceable ownership, replaceable edges — name which one |
| 11 | What it costs | Indirection, ceremony, mapping, navigation cost, and a higher floor for small changes |
| 12 | Component principles | ADP (no cycles), SDP (depend toward stability), SAP (stable ⇒ abstract) |
| 13 | The four projects | Domain, Application, Infrastructure, Presentation — and who may reference whom |
| 14 | References are the enforcement | The `.csproj` graph is the architecture; everything else is documentation |
| 15 | Domain contents | Entities, value objects, domain services, domain events, invariants, domain errors |
| 16 | Application contents | Use cases, ports, DTOs, orchestration, the transaction boundary, authorization policy |
| 17 | Infrastructure contents | Adapters: persistence, messaging, HTTP clients, blob, email, identity, clock, secrets |
| 18 | Presentation contents | Protocol: routing, binding, serialization, status codes, auth handshake, view models |
| 19 | The composition root | One place knows all concrete types; it is the host, and it may reference everything |
| 20 | Registration ownership | `AddInfrastructure()` lives in Infrastructure; the host calls it — that's not a cycle |
| 21 | Plugin-style roots | Runtime assembly loading buys true decoupling and costs you compile-time safety |
| 22 | Project vs. folder | A project is a compile-time wall; folders are a suggestion — pay for walls you need |
| 23 | `internal` by default | Assembly-scoped visibility plus `InternalsVisibleTo` is free enforcement |
| 24 | Build-level controls | `Directory.Build.props`, central package management, banned-API analyzers |
| 25 | Naming and structure | Folder-per-use-case inside Application; name things after the business |
| 26 | The Aspire complication | AppHost wires resources; your composition root still owns the object graph |
| 27 | Repository: what it's for | An aggregate-shaped collection illusion, not a wrapper for EF |
| 28 | Unit of work | The transaction boundary is an application-layer decision, implemented in infrastructure |
| 29 | Persistence ignorance | Survives three compromises; past that you need a separate persistence model |
| 30 | The separate-model option | Explicit domain↔persistence mapping: the pure choice, and what it costs |
| 31 | `IApplicationDbContext` | The honest analysis of the most popular shortcut in the .NET ecosystem |
| 32 | `IQueryable` leakage | Exporting `IQueryable` exports the ORM's semantics, laziness, and failure modes |
| 33 | Specifications | A named query object; worth it when criteria genuinely repeat, ceremony otherwise |
| 34 | The read side | Queries don't need the domain model; a query is allowed to talk to the database |
| 35 | CQRS-lite as layering | Two paths through the architecture, only one of which pays for a domain model |
| 36 | Aggregates and load size | The aggregate defines the transaction and therefore the repository method |
| 37 | Migrations ownership | Schema is infrastructure; the domain's shape constrains it but doesn't own it |
| 38 | Mapping | How many models you really need, and why manual mapping is the sane default now |
| 39 | Validation, three kinds | Input (edge), application (context), invariant (domain) — different homes, different failures |
| 40 | Errors across boundaries | Exceptions vs. result types; the rule is "expected ⇒ value, exceptional ⇒ exception" |
| 41 | Mapping errors to protocol | `ProblemDetails` + `IExceptionHandler`; the domain never names an HTTP status code |
| 42 | Time, IDs, randomness | `TimeProvider` is the port now; IDs belong to the domain, not the database |
| 43 | Configuration | `IOptions<T>` at the edge, validated at startup, plain records inward |
| 44 | Anti-corruption layers | The adapter translates vocabulary *and* failure modes, not just types |
| 45 | Domain vs. integration events | One is in-process and inside the transaction; the other is a published contract |
| 46 | Background work | A worker is another driving adapter; the use case must not know who called it |
| 47 | Cross-cutting behaviours | Pipeline/decorator placement, and why logging is an accepted exception |
| 48 | Seams not worth building | Logging, DI, serialization, LINQ, `HttpClient`, the framework's own abstractions |
| 49 | The fake test | If you wouldn't write a fake for it, you don't need an interface for it |
| 50 | Async and cancellation | `CancellationToken` crosses every boundary; the domain stays synchronous where it can |
| 51 | Vertical slice architecture | Optimizes coupling *within* a feature; the honest claim and the honest cost |
| 52 | Coupling/cohesion reframing | Layers put the seam where change doesn't happen; features put it where change does |
| 53 | The hybrid most teams land on | Clean shell, sliced Application layer, shared domain where the domain is real |
| 54 | Transaction script | Fowler's pattern, still correct for most CRUD; it is not a failure |
| 55 | The complexity ladder | CRUD → transaction script → domain model, chosen per use case |
| 56 | Functional core, imperative shell | Same rule without interfaces: pure decisions inside, effects at the edge |
| 57 | Modular monolith | Modules first, layers second; Module 21's subject, previewed |
| 58 | When Clean is the wrong default | Thin domain, small team, short life, data pipelines, prototypes, true CRUD |
| 59 | Evolutionary vs. up-front | Start with the seams that are expensive to add later, not all of them |
| 60 | The "we could swap it" fiction | Substitutability is rarely cashed; say what you're actually buying |
| 61 | Enforcement ladder | References → visibility → analyzers → arch tests → review → ADR |
| 62 | Architecture tests | What to assert first, and the ways these tests pass when they shouldn't |
| 63 | Banned-API analyzers | Ban `DateTime.Now`, EF types, `HttpContext` per project — compile-time, not CI-time |
| 64 | The testing pyramid this buys | Domain tests with no doubles; use-case tests with fakes; adapter tests with containers |
| 65 | Fakes vs. mocks | Don't mock what you don't own; a fake repository is 30 lines and never lies |
| 66 | Testing the composition root | `WebApplicationFactory`, container validation, and the startup smoke test |
| 67 | Fitness functions | Drift detection as a scheduled job, with a number attached |
| 68 | Brownfield introduction | Seam → use case → port → domain, in that order, under a strangler |
| 69 | Collapsing layers | Removing a layer that isn't paying is a senior move, not an admission |
| 70 | The performance bill | Mapping allocations, indirection, async chains, DI resolution — measured, not feared |
| 71 | Human metrics | Files-per-change, PR diff size, onboarding time, build time as architecture signals |
| 72 | The wrong abstraction | Duplication is cheaper than the wrong abstraction; how to reverse one |
| 73 | How to talk about it | Boundaries, volatility, cost — never "because Clean Architecture" |
| 74 | Four questions before a boundary | What changes? Who owns it? What's the test? What's the cost if I'm wrong? |
| 75 | Critiquing a template | Fair, specific, current — and knowing which parts are the template's fault |
| 76 | Layers ≠ tiers | Logical layering says nothing about processes, machines, or network hops |
| 77 | Conway's law | Boundaries that don't match ownership erode; boundaries that do become real |
| 78 | Documenting it | ADRs, C4 component level, a dependency diagram that's generated, not drawn |
| 79 | Cost framing | Translate structure into delivery risk, onboarding, and change cost |
| 80 | Anti-pattern catalogue | The twelve things reviewers actually look for |
| 81 | Defaults by team size | What you'd actually start with at 2, 8, and 40 engineers |
| 82 | The one-minute answer | The compressed version you give when the interviewer wants a position, fast |

---

# Part A — The rule: what Clean Architecture actually claims

---

## Concept 1 — The dependency rule, stated precisely

The whole of Clean Architecture is one sentence, and almost every argument about it comes from people paraphrasing it badly:

> **Source-code dependencies must point only inward, toward higher-level policy.**

Three words in that sentence do all the work.

**"Source-code."** This is a statement about `using` directives, project references, and what the compiler needs in order to build an assembly. It is *not* a statement about what calls what at runtime. At runtime, your HTTP adapter calls your use case, which calls the domain, which is called *back* by the ORM's materializer, which is infrastructure. Control flow crosses the boundary in both directions constantly. What never crosses inward-to-outward is the *reference*.

**"Only."** The rule is asymmetric and absolute at the boundary it protects. Outer knows inner; inner does not know outer, does not know that outer exists, and cannot name a single type from it.

**"Toward higher-level policy."** Direction is defined by level, not by importance or by how much code there is. Concept 2 defines level.

The practical test, and the one to use in an interview because it's checkable: **delete the outer project from the solution. Does the inner one still compile?** If Domain compiles with Infrastructure deleted, the rule holds between those two. If it doesn't, you have a picture, not an architecture.

The second test, which catches the subtler violation: **does the inner project's public API mention any type it doesn't own?** A `Task<List<OrderDto>> GetOrders()` on a domain interface is fine (`Task` and `List` are language infrastructure). A `IQueryable<Order> Query()` is not, because `IQueryable`'s contract is "an expression tree that some unnamed provider will translate," and that is a database detail wearing a BCL costume.

---

## Concept 2 — Policy and detail, defined by reason-to-change

Robert Martin's definition of *level* is the distance from the inputs and outputs of the system. The more useful operational definition, and the one you should use out loud:

- **Policy** is the part of the system that would still exist if you delivered it with paper forms and a telephone. Interest accrues monthly. An order can't ship before it's paid. A seat can't be double-booked. A refund is bounded by the original payment.
- **Detail** is everything about *how* that policy is delivered: HTTP, SQL, JSON, Kafka, Blazor, Azure, the retry count, the connection string, the fact that it's a web application at all.

Policy changes when the business changes its mind. Detail changes when engineering changes its mind, or when a vendor changes theirs.

That distinction is why the dependency rule has a direction at all. If policy depended on detail, then a vendor's decision — Microsoft deprecating an API, a database changing its isolation semantics, a payment provider versioning its webhook — would force you to edit the rules of your business. The direction of the arrow is an attempt to make **technology churn unable to reach the part of the codebase that encodes decisions humans argued about in a meeting**.

Two corrections to the usual telling:

- **"The domain is the most important code" is not the argument.** It's often the smallest and least interesting code in the system. The argument is that it's the most *expensive to re-derive*, because the knowledge in it came from people, not documentation.
- **Not every system has policy.** A service that accepts a JSON payload, validates three fields, writes a row, and publishes a message has *no* policy worth protecting. Concept 58 takes this seriously, because pretending otherwise is where most of the wasted ceremony in .NET codebases comes from.

---

## Concept 3 — Volatility decides direction

Sharpen Concept 2 into a rule you can apply mechanically.

For any two components A and B, ask: **which one has more reasons to change?** The one with fewer reasons is more stable. Dependencies should point from volatile toward stable.

This produces the same answer as "policy inward" in the normal case, and it produces a *better* answer in the abnormal cases, which is why it's the version to carry:

- A domain model for a settled regulatory regime is extremely stable. Your HTTP API, with its versioning and clients and deprecations, is volatile. Arrow points inward. Standard.
- A startup's pricing rules change weekly while its Postgres schema hasn't changed in a year. Volatility says the *pricing rules* are the volatile part. This doesn't invert the architecture, but it does tell you where to spend your isolation budget: the pricing rules want to be behind a seam of their own, testable in milliseconds, possibly data-driven — not merely "in the domain project."
- A third-party SDK is volatile *for reasons you don't control*, which is the worst kind. That's the strongest possible case for an adapter, even in a codebase with no other abstractions (Concept 44).

Stability here is Martin's positional stability: a component depended on by many and depending on few is hard to change, because changing it breaks everyone. That's the *Stable Dependencies Principle* — depend in the direction of stability (Concept 12). The reason it rhymes with the dependency rule is that they're the same observation at different scales.

---

## Concept 4 — How control flows outward while dependency points inward

This is the mechanism, and it is the one thing in Part A worth being able to draw on a whiteboard in fifteen seconds.

The use case needs to save an order. Saving is a detail (SQL, network, transactions). The use case is policy. Policy must not name detail. But policy must *cause* the save.

The resolution is **polymorphism with ownership of the interface on the inner side**:

```csharp
// Application layer (inner). It DEFINES the port it needs.
public interface IOrderRepository
{
    Task<Order?> FindAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Application layer (inner). It DEPENDS on its own interface.
public sealed class PlaceOrderHandler(IOrderRepository orders, IUnitOfWork uow)
{
    public async Task<OrderId> HandleAsync(PlaceOrder command, CancellationToken ct)
    {
        var order = Order.Place(command.CustomerId, command.Lines);
        await orders.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Infrastructure (outer). It DEPENDS on the inner interface and implements it.
internal sealed class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> FindAsync(OrderId id, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task AddAsync(Order order, CancellationToken ct) =>
        db.Orders.AddAsync(order, ct).AsTask();
}
```

At runtime, `PlaceOrderHandler` calls into EF Core, which calls into `Microsoft.Data.SqlClient`, which calls a socket. Control flows outward. At compile time, `Application.csproj` references nothing; `Infrastructure.csproj` references `Application.csproj`. The dependency points inward. The interface is the hinge, and **the critical detail is that the inner layer owns the interface** — it is declared in Application, shaped by Application's needs, named in Application's vocabulary.

This is the difference between an abstraction and a wrapper. If the interface were declared in Infrastructure and merely *consumed* by Application, Application would reference Infrastructure and the whole thing collapses. Half the broken "Clean Architecture" repositories you'll see in a code review have exactly this shape: `Application` references `Infrastructure.Abstractions`, which references `Infrastructure.Contracts`, which references EF Core "just for the types."

**Interface ownership is the single most diagnostic question you can ask about a boundary: who would have to change this interface, and for what reason?** If the answer is "the database team, when we change providers," it belongs to the wrong side.

---

## Concept 5 — Dependency inversion, dependency injection, and inversion of control are three different things

Interviewers conflate these constantly, and conflating them back is a mid-level tell.

| Term | What it is | Where it lives |
|---|---|---|
| **Dependency Inversion Principle (DIP)** | A *design principle*: high-level modules shouldn't depend on low-level modules; both should depend on abstractions, and abstractions shouldn't depend on details | Your type design — who declares which interface |
| **Dependency Injection (DI)** | A *technique* for supplying a component's collaborators from outside rather than having it construct them | Constructors, and a container if you want one |
| **Inversion of Control (IoC)** | A *property* of a program where the framework calls you rather than you calling the framework | The Hollywood principle; ASP.NET Core calling your handler is IoC |

The clarifying pair of facts:

- **You can have DI with no inversion at all.** `public Sender(SmtpEmailClient client)` injects a dependency and inverts nothing — the high-level type still names the low-level one. This is extremely common and mostly harmless, but it is not DIP.
- **You can have DIP with no container.** Constructing the whole object graph by hand in `Program.cs` — Mark Seemann's "Pure DI" — satisfies the principle completely and gives you compile-time verification of the graph, which a container does not. In .NET the container usually wins for ergonomics and lifetime management (Module 18, Part E), but "we use `IServiceCollection`" is not an answer to "how do you invert dependencies."

The third distinction worth having ready: **a DI container is not an architectural boundary.** Registering `ISomething` in the container doesn't create a seam; declaring the interface on the inner side and never referencing the implementing assembly does. The container is how the seam gets filled at runtime — nothing more.

---

## Concept 6 — Ports and adapters: driving and driven

Alistair Cockburn's Ports & Adapters (2005) is the same rule with a better vocabulary, and it's the vocabulary you should use when you want to sound precise rather than trendy. His own framing: the application should be equally usable by users, programs, automated tests, or batch scripts, and equally able to run against a real database or a test double.

A **port** is a hole in the application's boundary, defined in the application's own terms. An **adapter** plugs into it and speaks a technology.

Ports come in two flavours, and the distinction resolves more real arguments than any other idea in this module:

- **Driving ports (primary, left side).** The application's own API — the use cases. *The outside calls in.* Adapters: an HTTP controller, a minimal API endpoint, a gRPC service, a CLI verb, a message-bus consumer, a test. The port is the handler's signature or the use-case interface; the adapter translates a protocol into a call.
- **Driven ports (secondary, right side).** Interfaces the application *needs someone else to implement.* *The inside calls out.* Adapters: repositories, HTTP clients, mail senders, publishers, blob stores.

Why this matters practically:

1. **Only driven ports need dependency inversion.** For driving ports the dependency already points the right way — the controller references the handler; the handler references nothing. This kills a huge amount of pointless ceremony: an `ICreateOrderUseCase` interface with exactly one implementation and exactly one caller is an interface you wrote so the arrow would look symmetrical. (There is one good reason to keep it — decorating the use case — see Concept 47.)
2. **A message consumer is a driving adapter, not infrastructure that "calls into" the app from the side.** This resolves the perennial "where do my `BackgroundService`s go?" argument (Concept 46).
3. **"Hexagonal" is not about six.** The hexagon was drawn with six sides purely to leave room for several ports around the edge; it carries no meaning. Cockburn has spent twenty years saying so, and the 2024 book *Hexagonal Architecture Explained* exists largely because of the misreadings.

---

## Concept 7 — The lineage, and why four names exist for one idea

You'll get asked "what's the difference between Clean, Onion, and Hexagonal?" The wrong answer is a taxonomy. The right answer is a sentence about emphasis plus the honest admission that they're the same rule:

| Name | Origin | The emphasis it adds |
|---|---|---|
| **Boundary–Control–Entity (BCE)** | Ivar Jacobson, *Object-Oriented Software Engineering*, 1992 | Three stereotypes per use case: boundary (interaction), control (coordination), entity (state/rules). The ancestor of "use case as a first-class unit" |
| **Ports & Adapters / Hexagonal** | Alistair Cockburn, 2005 | *Symmetry*: the UI and the database are both just outside. Driving vs. driven. Testability as the headline benefit |
| **Onion Architecture** | Jeffrey Palermo, 2008 | *Concentric coupling*: couple inward only; the number of layers is free. Explicitly aimed at .NET's N-tier orthodoxy, where data access was the bottom layer everything sat on |
| **Clean Architecture** | Robert C. Martin, 2012 blog post; 2017 book | *Generalization and naming*: one dependency rule, plus the boundary-crossing mechanism (interfaces + DTOs + the "humble object" pattern), plus the component principles |

Mark Seemann's 2013 post "Layers, Onions, Ports, Adapters: it's all the same" is the canonical statement of the equivalence, and quoting that position (with attribution) is a strong move in an interview because it signals you've read past the marketing.

What the differences actually buy you when choosing vocabulary:

- Say **ports and adapters** when the conversation is about testing, substitutability, or multiple entry points. The driving/driven split does real work.
- Say **onion** when the conversation is about .NET specifically and someone is defending a bottom-up data-access layer. That's the argument Palermo was having, and it's the one still being had in enterprise .NET shops.
- Say **clean architecture** when you need a name everyone recognizes, and then immediately say which boundaries you actually intend to build, because the name alone means "four projects" to most listeners.
- Say **BCE** if you want to make the point that "use case" is older than all of this and that the interesting unit has always been the use case, not the layer.

---

## Concept 8 — Layers as a special case of boundaries

A layer is a boundary drawn along a *technology* axis: everything that talks HTTP here, everything that talks SQL there. That's one possible cut. It is not the only one, and it has a specific, well-understood weakness.

Two definitions worth having:

- **Strict layering:** a layer may only reference the layer directly beneath it. Presentation → Application → Domain, and Presentation may not touch Domain.
- **Relaxed layering:** a layer may reference any layer beneath it. Presentation may use Domain types directly.

Clean Architecture is *relaxed* in this sense — the web layer routinely names domain value objects, and pretending otherwise produces the classic four-DTO-per-concept tax. The rule constrains direction, not adjacency.

The structural weakness of technology layering, which is the whole basis of the vertical-slice argument (Concept 51): **a feature is a vertical thing and a layer is a horizontal thing.** Adding a field to an order touches the entity, the configuration, the migration, the repository, the DTO, the mapper, the validator, the handler, the response model, and the endpoint. Ten files, nine of which contain one line each, across four projects. Meanwhile the layer boundary you paid for sits between things that always change together, and the things that *don't* change together — orders and invoicing, say — are smeared across every layer.

Martin Fowler's "Presentation Domain Data Layering" is the balanced statement of this: layering is a reasonable first cut for a small system and a poor *only* cut for a large one, at which point the primary decomposition should be by module/feature with layering *inside*. That's Concept 57 and Module 21.

Hold both facts: the dependency rule is good; "layers" is merely the most common and least imaginative way to satisfy it.

---

## Concept 9 — Screaming architecture

Martin's 2011 point, and the cheapest structural improvement available in most codebases: **the top level of your source tree should tell a reader what the system does, not which framework it was built with.**

Compare:

```
src/
  Controllers/        src/
  Services/             Billing/
  Repositories/         Catalog/
  Models/               Ordering/
  DTOs/                 Shipping/
```

The left tree says "this is an ASP.NET application." The right says "this is an order-management system." Only one of those was a decision worth encoding in a directory structure; the other is visible from the `.csproj`.

In practice this shows up *inside* the Application project even when the outer shape is layered:

```
src/Application/
  Orders/
    PlaceOrder/           PlaceOrder.cs (command + handler + validator)
    CancelOrder/
    GetOrderSummary/
  Invoicing/
    IssueInvoice/
```

rather than `Commands/`, `Queries/`, `Handlers/`, `Validators/`. This costs nothing, needs no libraries, and gets you most of the navigational benefit people go to vertical slices for. It's the single highest-ratio recommendation in this module, and both reference templates have converged on it.

---

## Concept 10 — What it actually buys you (name one, with evidence)

When you justify the architecture, do not say "maintainability," "separation of concerns," or "loose coupling." Those are category names, not benefits. Name the specific good and the evidence that you're receiving it:

| Benefit | What it looks like when you're actually getting it | Evidence it's real |
|---|---|---|
| **Test speed and determinism** | Thousands of domain tests, no container, no clock, no network; use-case tests with in-memory fakes | Test suite wall-clock time; number of tests that need Docker |
| **Deferred decisions** | You started building before choosing the queue/provider/host, and the choice arrived late without a rewrite | Count of decisions you actually deferred; date the choice was made |
| **Enforceable ownership** | A team owns a module and the compiler stops other teams reaching into it | Cross-module PR counts; failed arch-test builds |
| **Replaceable edges** | You swapped a payment provider / mail vendor / blob store without touching policy | The diff of the last vendor swap |
| **Comprehensibility under churn** | A new engineer finds the rules of the business in one place and they aren't interleaved with framework code | Time-to-first-meaningful-PR; where a reviewer looks to answer "what happens when X?" |

The first one is the one that's cashed continuously. A domain model with no infrastructure dependencies gives you tests that run in single-digit milliseconds and never flake, and that is a compounding benefit measured in *engineer-hours per week*, every week, forever. The fourth is the one people *promise* and rarely cash (Concept 60).

In an interview: pick the benefit that matches the system you're being asked about, and say which one you're buying. "For this service I'd invert the payment gateway and the notification sender, because those are vendor decisions we'll revisit and because I want deterministic tests for the retry logic. I would not invert the database, because we've committed to Postgres and the abstraction would cost us query expressiveness." That sentence, or one like it, is the whole module.

---

## Concept 11 — What it costs (quote the price honestly)

Every benefit above has an invoice. Being able to itemize it is what separates "I like Clean Architecture" from "I've run it."

1. **Indirection tax on reading.** Following a request from endpoint to database crosses 4–6 files and 2–4 assemblies. Every "go to definition" lands on an interface. For engineers new to the codebase this is the single most-reported complaint, and it's legitimate.
2. **Ceremony floor on small changes.** Adding a nullable string to a response can genuinely touch eight files. The floor on a trivial change rises, which in a CRUD-heavy system is most changes.
3. **Mapping cost, in code and in cycles.** Every boundary you cross with a different model needs a mapper, and every mapper needs maintenance and allocates (Concept 70).
4. **Expressiveness loss at the persistence seam.** The moment you hide the ORM behind aggregate-shaped methods, you give up ad-hoc projections, joins across aggregates, and database features — which is exactly why the read side usually gets an exemption (Concept 34).
5. **Abstraction drift.** Interfaces with one implementation accumulate. Nobody deletes them. In year three you have 300 interfaces and 300 implementations and the mocking framework is load-bearing.
6. **The onboarding story becomes the architecture's story.** New engineers must be taught the structure before they can ship, which is a real, recurring cost — and a real, recurring benefit if the structure is the *right* one.
7. **It can hide a missing design.** Four projects can make a system that has no domain model *look* like one that does. An anemic `Order` with public setters, wrapped in a repository, injected into a handler, is layered CRUD with extra steps — and the layering makes it harder to notice.

The senior framing: **these costs are fixed and paid daily; the benefits are contingent and paid on events.** That asymmetry is why "apply it everywhere by default" is a bad policy and "apply it where the event is likely" is a good one.

---

## Concept 12 — The component principles (ADP, SDP, SAP)

Martin's component-coupling principles predate Clean Architecture and are the part of *Clean Architecture* (the book) that people skip. They're what makes the dependency rule generalize past four projects, and they're where "architecture" stops being a diagram and becomes something you can compute.

**ADP — Acyclic Dependencies Principle.** The component dependency graph must have no cycles. In .NET this is enforced for you at the project level: `csc` refuses a circular `ProjectReference`. That is a *feature*, and it's why splitting into projects is the cheapest architectural control available (Concept 22). Inside a project there's no such enforcement, which is why namespace-level cycles proliferate silently.

When you find a cycle, there are exactly two fixes, and knowing both is the signal:
- **Dependency inversion:** A needs something from B; declare the interface in A and have B implement it. The reference flips.
- **Extract a new component:** the shared part moves to C, and both A and B depend on C. (This is where `SharedKernel` comes from, and Concept 22 covers how it rots.)

**SDP — Stable Dependencies Principle.** Depend in the direction of stability. Instability I = (outgoing dependencies) / (incoming + outgoing): 0 means maximally stable (everyone depends on you, you depend on nobody), 1 means maximally volatile. Every dependency should point from higher I to lower I. Your Domain project should have I ≈ 0; your Web project I ≈ 1. If your Domain has an I of 0.4 because someone added a package reference, you can see it in a metric rather than an argument.

**SAP — Stable Abstractions Principle.** A stable component should be abstract, in proportion to its stability — otherwise it's rigid: hard to change *and* concrete. This is the formal justification for the domain being mostly interfaces and pure types rather than machinery. Martin's "main sequence" plots abstractness against instability and calls the two failure corners the *zone of pain* (stable and concrete — a database schema, a shared DTO library everyone references) and the *zone of uselessness* (abstract and depended on by nobody).

You don't need to compute these in an interview. You need them to be able to say: *"The shared DTO package everyone references is in the zone of pain — it's concrete and nothing can change without breaking eight services. That's the thing to fix first,"* which is an architect sentence that a diagram can't produce.

---

# Part B — The .NET shape: projects, references, and the composition root

---

## Concept 13 — The four projects, and the reference rules

The canonical .NET solution, with the reference graph stated as rules rather than drawn as circles:

```
src/
  Company.Product.Domain/           → references: nothing (ideally not even a NuGet package)
  Company.Product.Application/      → references: Domain
  Company.Product.Infrastructure/   → references: Application (and therefore Domain)
  Company.Product.Web/              → references: Application, Domain, Infrastructure*
tests/
  Company.Product.Domain.Tests/           → Domain
  Company.Product.Application.Tests/      → Application (+ fakes)
  Company.Product.Infrastructure.Tests/   → Infrastructure (+ Testcontainers)
  Company.Product.Web.Tests/              → Web (WebApplicationFactory)
  Company.Product.Architecture.Tests/     → all of them, deliberately
```

\* The asterisk is the interesting part and Concept 19 is entirely about it.

The rules, stated so they can be tested:

1. **Domain references nothing in the solution.** Ideally no NuGet packages either; in practice a small number are defensible (Concept 15).
2. **Application references Domain only.** It declares the ports; it does not know who fills them.
3. **Infrastructure references Application.** It implements Application's ports. It may reference Domain because it must materialize domain objects.
4. **Web references Application** (to call use cases) **and Domain** (to name types in responses, if you choose relaxed layering) **and Infrastructure** — but only from the composition root file(s).
5. **Nothing references Web.** Except tests.

Four notes that distinguish someone who has run this from someone who has read about it:

- **"Application" is a bad name and everyone uses it.** It means "use cases." If you're starting fresh, `UseCases` is clearer, and the ardalis template uses exactly that.
- **Three projects is often the right number.** Domain and Application merge cleanly in systems where the domain is small: you get `Core` (entities + use cases + ports), `Infrastructure`, `Web`. You lose the ability to assert "use cases don't reference each other's internals," which you may not have been asserting anyway.
- **More than four is usually a smell** unless the extra projects are *modules* (Concept 57) rather than more layers. `Application.Contracts`, `Application.Abstractions`, `Domain.Shared`, `Infrastructure.Persistence.Abstractions` — each of these is a compile-time wall someone erected to solve a problem that was probably a naming problem.
- **The test-project shape is part of the architecture.** If you can't draw the test projects, you haven't finished the design — because the pyramid in Concept 64 is most of what you're buying.

---

## Concept 14 — The `.csproj` graph *is* the architecture

The most important sentence in Part B: in .NET, **the architecture is enforced by the compiler if and only if you express it as project references.** Everything else — folder names, namespace conventions, documentation, the diagram in Confluence, the onboarding deck — is a preference that survives exactly as long as the person who cares about it.

This gives .NET a genuine advantage over languages where module boundaries are conventional. Use it:

```xml
<!-- Company.Product.Domain.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
  <!-- No ProjectReference. No PackageReference. That emptiness is the design. -->
</Project>
```

The corollary is a review rule worth adopting: **a pull request that adds a `ProjectReference` or a `PackageReference` to Domain or Application is an architecture change and gets reviewed as one.** In practice, tagging those two files in a `CODEOWNERS` entry does more for architectural integrity than any amount of documentation.

The second corollary: because the compiler refuses circular project references, **splitting into projects buys you free cycle detection (ADP)**. That is the single best reason to pay the solution-noise cost of multiple projects, and it's the reason "just use folders" is weaker than it sounds (Concept 22).

---

## Concept 15 — What belongs in Domain

Contents:

- **Entities** — objects with identity and a lifecycle: `Order`, `Customer`, `Shipment`. Behaviour, not just data; they enforce their own invariants.
- **Value objects** — objects defined entirely by their values: `Money`, `EmailAddress`, `Address`, `DateRange`, `OrderId`. In modern C#, `readonly record struct` for small ones, `sealed record` for larger ones.
- **Aggregates and aggregate roots** — the consistency boundaries (Module 22 does this properly; Concept 36 does the layering consequence).
- **Domain services** — operations that don't belong to a single entity: `PricingService`, `TransferService`. Stateless, pure, named after a business verb.
- **Domain events** — facts that occurred, in past tense: `OrderPlaced`, `PaymentCaptured`. Plain records, raised by entities, dispatched elsewhere (Concept 45).
- **Domain exceptions / errors** — `InsufficientStockException`, or an error enum/result type. Named in domain vocabulary, never carrying HTTP or SQL concepts.
- **Enums, constants, and specifications** that encode business rules.

Not in Domain: DTOs, validators for input shapes, `DbContext`, attributes from ORMs or serializers, `ILogger`, `HttpClient`, anything `async` that implies I/O, and — the one people always sneak in — `DateTime.UtcNow`.

**Packages in Domain.** The purist position is zero. The defensible exceptions, roughly in order of how easy they are to justify:

| Package | Verdict |
|---|---|
| None | The ideal, and achievable more often than people think |
| A guard-clause helper (e.g. `Ardalis.GuardClauses`) | Defensible; it's a syntax convenience with no semantics of its own |
| A result type (`ErrorOr`, `OneOf`, `Ardalis.Result`) | Defensible, but it's now in your domain's public vocabulary forever — pick once |
| `System.ComponentModel.DataAnnotations` attributes | Weak. It's validation-for-a-framework leaking into invariants |
| `Newtonsoft.Json` / `System.Text.Json` attributes | No. Serialization is a detail of a boundary the domain doesn't have |
| EF Core (for `[Owned]`, navigation conventions) | No — and this is precisely the compromise Concept 29 negotiates |
| `MediatR.Contracts` (so entities can raise `INotification`) | Common, and now a licensing question; a plain `IDomainEvent` you own costs nine lines |

The test that settles arguments: **could this project compile and its tests run on a machine with no network, no database, no container runtime, and no `ASPNETCORE_ENVIRONMENT`?** If yes, the domain is clean enough.

---

## Concept 16 — What belongs in Application

The Application layer is the system's **use cases**: one type per thing a user or another system can ask the application to do. Contents:

- **Commands and queries** (the request DTOs) and their **handlers** — whether dispatched by a mediator or called directly.
- **Ports (driven interfaces)**: `IOrderRepository`, `IPaymentGateway`, `IEmailSender`, `IEventPublisher`, `IUnitOfWork`, `IFileStore`, `ICurrentUser`.
- **Application DTOs**: the shapes the use cases return. Not the domain entities, and not the HTTP response models (although in practice the second distinction is often collapsed deliberately — Concept 38).
- **Input validation** for the command shape (Concept 39, kind 2).
- **Orchestration**: load aggregate → call domain behaviour → persist → publish. This is deliberately thin. If a handler is 200 lines, either your domain is anemic or that use case is really a workflow that deserves its own explicit shape.
- **The transaction boundary** (Concept 28) — a policy decision about what must commit together.
- **Authorization policy in business terms**: "only the order's owner or a support agent may cancel it." The *mechanism* (JWT, claims, policies) is Presentation's; the *rule* is the application's.
- **Cross-cutting behaviours** if you're using a pipeline (Concept 47).

What must not be here: `HttpContext`, `ControllerBase`, `IActionResult`, status codes, `DbContext`, SQL, connection strings, `HttpClient`, retry policies, JSON settings, `Assembly.GetExecutingAssembly()` outside of registration code.

The most useful single shape to have memorized, because you'll write it on a whiteboard:

```csharp
public sealed record CancelOrder(OrderId OrderId, string Reason) : ICommand<Result>;

internal sealed class CancelOrderHandler(
    IOrderRepository orders,
    IUnitOfWork uow,
    ICurrentUser user,
    TimeProvider time) : ICommandHandler<CancelOrder, Result>
{
    public async Task<Result> HandleAsync(CancelOrder cmd, CancellationToken ct)
    {
        var order = await orders.FindAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();
        if (!user.CanAct(order.CustomerId)) return Result.Forbidden();

        var outcome = order.Cancel(cmd.Reason, time.GetUtcNow());   // all the rules live in here
        if (outcome.IsFailure) return Result.Invalid(outcome.Error);

        await uow.SaveChangesAsync(ct);
        return Result.Success();
    }
}
```

Read what that handler *doesn't* do: it doesn't decide whether cancellation is allowed (the entity does), it doesn't know what a 404 is (it returns a `NotFound` *result*), it doesn't call `DateTime.UtcNow`, and it doesn't publish anything — `order.Cancel` raised a domain event and the save dispatches it (Concept 45).

---

## Concept 17 — What belongs in Infrastructure

Everything that talks to something outside the process, plus everything that is a framework's problem rather than a business one:

- **Persistence**: `DbContext`, entity configurations, migrations, repositories, query implementations, Dapper queries, the outbox table.
- **Messaging**: Service Bus / Kafka / RabbitMQ publishers and (arguably — Concept 46) consumers.
- **HTTP clients** for third parties, with their typed clients, resilience handlers, and DTO translation.
- **Identity**: token validation plumbing, user store, `ICurrentUser` implementation sourced from claims.
- **Storage/other**: blob, file system, email, SMS, PDF generation, feature flags, caching (`HybridCache` from Module 10), secrets.
- **`TimeProvider` registration** and any remaining system-clock plumbing.

Structural conventions that pay off:

- **Implementations are `internal`.** Nothing outside Infrastructure should be able to name `EfOrderRepository` — that's what `internal` plus an `AddInfrastructure()` extension is for (Concepts 20, 23).
- **One folder per adapted technology**, not one folder per pattern: `Persistence/`, `Messaging/`, `Payments/`, `Email/`. Naming the vendor is fine here and only here: `Payments/Stripe/`.
- **Infrastructure may hold its own private domain concepts.** A Stripe adapter is allowed to have `StripeCustomerRef` types — that's the anti-corruption layer doing its job (Concept 44), and it's a feature that these types never reach the application.
- **Splitting Infrastructure into several projects** (`Infrastructure.Persistence`, `Infrastructure.Messaging`) is worth it exactly when different teams own them or when you need one without the other in a different host. Otherwise it's solution noise.

---

## Concept 18 — What belongs in Presentation

The Presentation project's job is **protocol translation**, and the sharper you hold that, the smaller and more boring it stays:

- Routing, model binding, content negotiation, serialization settings.
- Authentication handshake and the mapping of claims → `ICurrentUser`.
- HTTP status codes, headers, caching directives, `ProblemDetails` shaping (Concept 41).
- Request/response contracts *if* you version your API separately from your use cases (Concept 38).
- Framework middleware: CORS, rate limiting, compression, exception handling, OpenAPI.
- The composition root (Concept 19).

The quality test: **an endpoint should be readable in one screen and contain no business conditionals.**

```csharp
app.MapPost("/orders/{id:guid}/cancel", static async (
        Guid id,
        CancelOrderRequest body,
        ICommandDispatcher dispatcher,
        CancellationToken ct) =>
    {
        var result = await dispatcher.SendAsync(new CancelOrder(new OrderId(id), body.Reason), ct);
        return result.ToMinimalApiResult();     // one place maps Result → HTTP
    })
   .WithName("CancelOrder")
   .RequireAuthorization();
```

If an endpoint contains `if (order.Status == OrderStatus.Shipped) return BadRequest(...)`, a business rule has escaped into the protocol layer and will now be enforced in exactly one entry point — which is the bug that shows up the day someone adds a message consumer for the same operation.

Two specifics worth knowing for a .NET interview:

- **Minimal APIs vs. controllers is not an architecture decision** (Module 18 covered the trade). Both are driving adapters. The relevant architectural property is that the *handler* is reachable and testable without either.
- **FastEndpoints (used by the ardalis template) is the REPR pattern** — Request-Endpoint-Response — one class per endpoint. It's a driving-adapter organization choice that happens to align neatly with per-use-case folders.

---

## Concept 19 — The composition root

Mark Seemann's definition: **the composition root is the single location, as close as possible to the application's entry point, where the object graph is composed.** In ASP.NET Core it is `Program.cs` plus whatever registration extension methods it calls.

The reference question everyone trips on: **why is Web allowed to reference Infrastructure? Doesn't that break the dependency rule?**

No, and the reason is precise: **the composition root is not a layer.** It is the place where the abstract graph becomes concrete, and it is *definitionally* allowed to know every type in the system — that's its only job. The dependency rule constrains *policy* from depending on detail. `Program.cs` contains no policy. It contains wiring.

The shape:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddDomainServices()                        // usually nothing — domain types are new'd, not injected
    .AddApplication()                           // handlers, validators, pipeline behaviours
    .AddInfrastructure(builder.Configuration)   // the only place that knows EF, Azure, Stripe exist
    .AddWeb();                                  // controllers/endpoints, auth, OpenAPI, problem details

var app = builder.Build();
```

Three properties to insist on:

1. **Exactly one.** If a second location resolves services from the container to build graphs — a static `ServiceLocator`, a factory that takes `IServiceProvider` and news up strategies — you have two roots and the container has become a global variable. (Module 18's captive-dependency rules are the runtime consequence.)
2. **It is the only place `IServiceProvider` appears**, apart from genuine factory scenarios (`IServiceScopeFactory` in a worker, `IDbContextFactory` from Module 19).
3. **It is verified at startup, not at first request.** This is cheap and almost nobody does it:

```csharp
builder.Host.UseDefaultServiceProvider((ctx, options) =>
{
    options.ValidateScopes = true;   // catches scoped-into-singleton in every environment, not just Dev
    options.ValidateOnBuild = true;  // catches unresolvable graphs at Build(), not at 3am
});
```

If you want to eliminate the Web→Infrastructure reference entirely — some teams do, so that an engineer working in Web cannot accidentally `new` an EF repository — the options are Concept 21. The usual answer is: don't bother; enforce it with an architecture test that says "no type in Web except `Program`/`DependencyInjection` may reference Infrastructure," which gets you the benefit without the machinery.

---

## Concept 20 — Registration ownership: `AddInfrastructure()` lives in Infrastructure

A detail that resolves a surprising amount of confusion. The extension method that registers Infrastructure's types **lives in the Infrastructure project**:

```csharp
// Infrastructure/DependencyInjection.cs
namespace Company.Product.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<AppDbContext>(o => o.UseSqlServer(config.GetConnectionString("Default")));
        services.AddScoped<IOrderRepository, EfOrderRepository>();
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
        services.AddSingleton(TimeProvider.System);
        services.AddHttpClient<IPaymentGateway, StripePaymentGateway>()
                .AddStandardResilienceHandler();
        return services;
    }
}
```

Why this isn't a violation:

- Infrastructure referencing `Microsoft.Extensions.DependencyInjection.Abstractions` is Infrastructure referencing a *framework*, which is what Infrastructure is for.
- The implementing types stay `internal`; only the extension method is `public`. The rest of the solution literally cannot name `EfOrderRepository`.
- The composition root calls one method per assembly and stays legible at 40 lines instead of 400.

The failure mode to recognize in a review: **registration code that lives in Application**. `services.AddDbContext<AppDbContext>()` inside an `AddApplication()` method means Application now references EF Core, and the entire inward-pointing graph is gone — usually by accident, usually because someone wanted "all the registration in one file."

The related pattern for large solutions is a **module registration convention**: each module exposes `IModule` with `RegisterServices(IServiceCollection)`, and the host discovers them by assembly scanning. That's a modular-monolith technique (Module 21); at four projects it's over-engineering.

---

## Concept 21 — Plugin-style roots, and why you usually shouldn't

If you genuinely need the composition root not to reference Infrastructure at compile time, the options are:

1. **Runtime assembly loading**: `AssemblyLoadContext` loads `*.Adapters.dll` from a directory; a discovered `IModule` registers its own services. Real plugin architectures (Orchard Core, ABP, dev tools) do this.
2. **Late binding by configuration**: `Type.GetType(config["RepositoryImplementation"])`, which trades compile-time safety for a string in a config file.
3. **Source generation / conditional compilation**: the composition is generated per build configuration.

What you gain: the deployable artifact for the app truly does not contain Infrastructure, so a *different* Infrastructure can be dropped in without recompiling.

What you pay: no compile-time verification that the graph is satisfiable, no "find all references," reflection at startup (Module 17's cold-start budget), no trimming, no Native AOT, and a whole class of runtime-only failures. Also, in practice, **nobody swaps the plugin** — which is the same claim Concept 60 makes about databases.

The senior position: a Web→Infrastructure project reference confined to the composition root, enforced by an architecture test, achieves 95% of the benefit at 2% of the cost. Say that, and mention that you'd reach for genuine plugin loading only when third parties write the adapters.

---

## Concept 22 — Project or folder? Buy the walls you need

Every project costs you: build time, solution noise, a `.csproj` to maintain, a NuGet surface to keep aligned, more places for a package version to drift. Every project buys you: a compile-time wall, cycle detection (ADP), an `internal` scope, and a nameable unit of ownership.

A decision rule:

| Create a separate project when… | A folder is enough when… |
|---|---|
| You want the compiler to forbid a dependency | The rule is a convention you're happy to enforce in review |
| Different people/teams own the two halves | One person owns both |
| One half will be consumed by another host/artifact | Everything ships together, always |
| You want `internal` to mean something | Everything is public anyway |
| You need to detect cycles you can't see | The area is small enough to eyeball |

Two failure modes with names:

- **`SharedKernel` rot.** A project everyone references, which therefore can never change, which therefore accumulates everything anyone needed twice. It starts as `Entity`, `ValueObject`, `Result`. Eighteen months later it has a `DateTimeExtensions`, an `EmailValidator`, and a `TenantContext`. This is Martin's *zone of pain*: maximally stable, maximally concrete. Treat it like a published library — small, versioned, reviewed harder than anything else, and deliberately made **abstract** rather than useful.
- **Abstraction-project proliferation.** `Application.Abstractions` exists because someone needed Infrastructure to see an interface without seeing the handlers. Sometimes legitimate. Usually it means the interfaces were in the wrong place. Ask what would break if they merged.

.NET specifics: the `.slnx` solution format makes the solution file small enough to review in a PR, which matters when the project graph is the architecture. And **solution folders are not namespaces** — a `src/Modules/Ordering/` folder groups projects for humans and changes nothing about compilation.

---

## Concept 23 — `internal` by default is free enforcement

The most under-used architectural control in .NET: **make implementation types `internal` and export only what the boundary needs.**

```csharp
// Infrastructure
internal sealed class EfOrderRepository(AppDbContext db) : IOrderRepository { … }

// Application
public sealed record PlaceOrder(...) : ICommand<Result<OrderId>>;   // the contract: public
internal sealed class PlaceOrderHandler : ICommandHandler<…>        // the implementation: internal
```

Consequences:

- Nothing outside the assembly can name the type, so nothing outside can depend on it. That's a real boundary with zero infrastructure.
- Tests in the same solution get in via:
  ```xml
  <ItemGroup>
    <InternalsVisibleTo Include="Company.Product.Application.Tests" />
  </ItemGroup>
  ```
  (MSBuild's `InternalsVisibleTo` item, which generates the attribute — cleaner than hand-writing it in `AssemblyInfo.cs`.)
- Handlers registered by assembly scanning work fine when `internal`, because reflection ignores accessibility. If you register explicitly, `services.AddScoped<ICommandHandler<PlaceOrder, …>, PlaceOrderHandler>()` needs the type to be visible to the registration code — which is why the registration extension lives *inside* the same assembly (Concept 20).

Two related habits: `sealed` by default (it's a small JIT devirtualization win per Module 17 and it stops accidental inheritance across a boundary), and `file`-scoped types (C# 11+) for helpers that genuinely belong to one file.

---

## Concept 24 — Build-level controls: props, central packages, banned APIs

Three mechanisms that turn architectural intent into build failures. Together they're cheaper and more reliable than any amount of review discipline.

**1. `Directory.Build.props`** — one file at the repo root setting language version, nullable, warnings-as-errors, analysis level for every project. Architecture benefit: it makes "the rules are the same everywhere" a fact rather than a hope, and it's the place to enable analyzers globally.

**2. Central Package Management** — `Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>`, where every version is declared once and individual projects reference packages without versions. Architecture benefit: one place to see the whole dependency surface, one PR to review when it changes, and no silent drift where Infrastructure is on EF 10.0.3 and the test project pins 10.0.1.

**3. Banned API analyzers** — `Microsoft.CodeAnalysis.BannedApiAnalyzers` with a `BannedSymbols.txt` per project. This is the trick that impresses in a code-review round, because it enforces things architecture tests can only catch after the fact:

```
# src/Domain/BannedSymbols.txt
P:System.DateTime.Now;Use TimeProvider, passed in from the application layer.
P:System.DateTime.UtcNow;Use TimeProvider, passed in from the application layer.
P:System.DateTimeOffset.UtcNow;Use TimeProvider.
M:System.Guid.NewGuid;Ids are generated by the caller or by a port, not by the domain.
T:System.Random;Non-determinism belongs behind a port.
```

```
# src/Application/BannedSymbols.txt
N:Microsoft.EntityFrameworkCore;The application layer talks to ports, not to an ORM.
N:Microsoft.AspNetCore.Http;Use cases don't know about HTTP.
T:System.Net.Http.HttpClient;Wrap it in an adapter behind a port.
```

Set `<WarningsAsErrors>RS0030</WarningsAsErrors>` (the banned-symbol diagnostic) and the violation fails the build in the IDE, not in CI twenty minutes later. **The feedback-loop difference is the whole point:** an architecture test tells you that you violated the rule; a banned-symbol analyzer tells you as you type it.

---

## Concept 25 — Naming and internal structure

Conventions that cost nothing and pay continuously:

- **Folder per feature, then per use case, inside Application** (Concept 9). `Orders/PlaceOrder/`, not `Commands/Orders/`.
- **One file per use case where it fits.** Command record + handler + validator in `PlaceOrder.cs` is *more* readable than three files, and it's the one genuinely good idea vertical slices contributed to mainstream .NET practice. Split when it exceeds a screen or two.
- **Name use cases as the business names them.** `ApproveExpenseReport`, not `ExpenseReportUpdateService`. If the business says "void", your type says `Void`, not `Cancel`.
- **Name ports for what the application needs, not for what implements them.** `IOrderRepository`, not `ISqlOrderRepository`. `INotificationSender`, not `ISendGridClient`. The name is part of the abstraction — a port named after a vendor has already failed.
- **Avoid the `I`-prefix debate by having a rule.** .NET convention is `IThing`; keep it. What matters more is not creating `IOrderService` — a name that means nothing and attracts everything.
- **Suffix conventions are architecture tests waiting to happen.** If every handler ends in `Handler` and lives under `Application`, you can assert it (Concept 62).

---

## Concept 26 — The Aspire complication: two kinds of "composition"

Aspire (13.x by 2026) introduced a second thing that looks like a composition root and isn't, and both current templates now assume it, so it's live interview surface.

- **The AppHost project** composes *resources*: "this solution consists of a Postgres container, a Redis cache, a Service Bus emulator, an API project and a worker; here's how they discover each other; here are the connection strings." It's a distributed-application manifest that runs your topology locally and emits deployment artifacts.
- **Your application's composition root** composes *objects*: which class implements `IOrderRepository`, what the DI lifetimes are, which behaviours decorate the pipeline.

They are different jobs at different levels, and the architectural rules for each differ:

- The AppHost **references your service projects** (as orchestration targets, not as libraries). That is not an architecture violation; it is outside the layered graph entirely.
- **ServiceDefaults** (the shared project Aspire scaffolds for telemetry, health checks, service discovery and resilience) is referenced by every service. Treat it as infrastructure-level shared plumbing, and keep it ruthlessly free of anything business-shaped — it's a `SharedKernel` with the same rot risk (Concept 22).
- Connection strings and endpoints arrive through configuration, which means **your Infrastructure layer's registration code is unchanged** whether Aspire, Kubernetes, or `appsettings.json` provided them. If adopting Aspire required editing your Application layer, something was already wrong.

The one-line interview answer: *"Aspire composes the topology; my composition root composes the object graph. Aspire doesn't change the dependency rule — it changes where connection strings come from and gives me a local environment that matches the deployed one."*

---

# Part C — The persistence seam: where layering meets the database

This is the part of the module Module 19 deferred, and the part interviewers push hardest on, because it's where the diagram meets a schema and something has to give.

---

## Concept 27 — What a repository is actually for

The pattern's original definition (Evans, and Fowler's *PoEAA*) is precise: **a repository mediates between the domain and data mapping layers, acting like an in-memory collection of aggregate roots.** Note the three constraints hiding in that sentence:

1. **Of aggregate roots.** One repository per aggregate, not one per table and certainly not one per entity. `IOrderRepository`, not `IOrderLineRepository` — order lines are reached through their order.
2. **Like a collection.** `Add`, `Remove`, `FindById`, plus a small number of domain-meaningful finders. Collections don't have `Update` — you mutate the object you got and the unit of work notices (Module 19, Concept 25).
3. **Mediates.** It hides *mapping*, which in EF Core is mostly already hidden. This is why the honest argument about repositories over EF is different from the argument about repositories over ADO.NET.

What a repository is **not** for:

- It is not for "abstracting the database so we could swap it." (Concept 60.)
- It is not for "making it testable" as a primary goal — `DbContext` is already fake-able with SQLite or a container, and mocking a repository that returns whatever you say proves very little (Concept 65).
- It is not a place for query composition on behalf of arbitrary callers. That's Concept 32.

The shape that survives review:

```csharp
public interface IOrderRepository
{
    Task<Order?> FindAsync(OrderId id, CancellationToken ct);
    Task<Order?> FindWithLinesAsync(OrderId id, CancellationToken ct);       // load size is explicit
    Task<IReadOnlyList<Order>> FindOverdueAsync(DateTimeOffset asOf, int max, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    void Remove(Order order);
}
```

Every method returns **materialized aggregates or nothing**. Load size is part of the method name, because the aggregate's boundary *is* the load. There is no `IQueryable`, no `Expression<Func<T,bool>>` escape hatch, no `GetAll()`.

And the honest counterpoint, which you should volunteer before the interviewer does: **`DbSet<T>` already is a repository, and `DbContext` already is a unit of work.** Microsoft's own EF documentation says so. So the repository has to justify itself on something else: *aggregate shaping and vocabulary*, i.e. making the set of legal persistence operations small, named in domain terms, and impossible to bypass. If it isn't doing that, it's a layer of indirection over another layer of indirection.

---

## Concept 28 — The unit of work and where `SaveChanges` belongs

The transaction boundary is **an application-layer decision** — it's a statement about what must be consistent together, which is business semantics — **implemented by infrastructure**.

Concretely:

```csharp
// Application (port)
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// Infrastructure: AppDbContext implements it, and that's the whole implementation
internal sealed partial class AppDbContext : DbContext, IUnitOfWork { }
services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
```

The rules that follow, all of which Module 19 justified mechanically:

- **Repositories don't call `SaveChanges`.** A repository that saves makes multi-aggregate use cases impossible to make atomic and turns every call into its own transaction.
- **The handler calls it once, at the end.** One use case, one unit of work, one commit.
- **The transaction is per use case, not per request.** They usually coincide in a web app; the principle is the consistency requirement (Module 19, Concept 37).
- **If you need an explicit transaction** (multiple `SaveChanges` calls, or EF plus Dapper on one connection), that's still an application-layer intent with an infrastructure implementation — a port like `ITransactionScope`/`IAtomicOperation`, or a pipeline behaviour that opens one around a command (Concept 47). And with `EnableRetryOnFailure`, the whole thing must be inside `strategy.ExecuteAsync` (Module 19, Concept 43) — a nasty detail that belongs in infrastructure and must not leak.

A shortcut worth knowing and being careful with: making `IUnitOfWork` a marker for a **pipeline behaviour** that calls `SaveChanges` after every command means handlers stop calling it at all. It's clean, it's used in production widely, and it costs you the ability to see the transaction boundary by reading the handler. Name the trade-off if you use it.

---

## Concept 29 — Persistence ignorance, and exactly how far it survives

Here's the promised answer to the question Module 19 left open.

**The claim:** the domain model should be unaware of how it's stored. **The reality with EF Core:** you can get about 90% of the way with three compromises, and past that you're choosing between purity and a mapping layer you'll maintain forever.

The three compromises that a mature team accepts:

1. **A parameterless (or EF-usable) constructor and settable-by-reflection state.** EF needs to materialize. It can use a private constructor and set private fields, so this stays invisible to callers — but it means your entity has a constructor that exists for EF's sake, usually `private Order() { }`. Conventionally marked with a comment, sometimes with `#pragma`. It's a smell you accept.
2. **Collection exposure as `IReadOnlyList` over a backing field.** EF maps the field; the domain exposes an immutable view. This is straightforwardly good design that EF happens to support:
   ```csharp
   public sealed class Order
   {
       private readonly List<OrderLine> _lines = [];
       public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
       private Order() { }   // EF
   }
   ```
   Configured with `builder.Metadata.FindNavigation(nameof(Order.Lines))!.SetPropertyAccessMode(PropertyAccessMode.Field);`
3. **Shadow state for persistence-only concerns.** Concurrency tokens, `CreatedAt`/`ModifiedAt` audit columns, soft-delete flags, discriminators. These are *persistence* facts, not domain facts, and EF's shadow properties let them exist entirely in the model configuration. Use them; a `RowVersion` property on a domain entity is a database concept in your business vocabulary.

The things that go beyond compromise into contamination, and the cost of avoiding each:

| Pressure | The impure fix | The pure fix | Cost of purity |
|---|---|---|---|
| Value objects in the schema | `[Owned]`/complex type configured in Infrastructure | Same, but configured via fluent API | None — configure in Infrastructure and the domain stays clean |
| Enum stored as string | `HasConversion<string>()` in configuration | Same | None |
| ID types (`OrderId` wrapping a `Guid`) | Value converter in configuration | Same | None, plus a query-filter gotcha (converted keys and `CAST`, Module 19) |
| Aggregate references | Navigation property to another aggregate root | Store the foreign key as a typed ID and load through its repository | Slightly more code; substantially better boundaries |
| Lazy loading | Virtual navigations + proxies | Explicit loads / aggregate-shaped methods | You must decide load size deliberately — which is the point |
| Inheritance mapping | TPH discriminator visible on the type | Shadow discriminator | None |
| A constructor EF can't use | A private ctor | A separate persistence model | High (Concept 30) |

**The position to state in an interview:** *"Persistence ignorance is a spectrum, not a property. I'll keep the domain free of EF attributes, EF packages and EF types, and I'll configure everything through `IEntityTypeConfiguration` in Infrastructure. I accept a private constructor and field-mapped collections. If the schema is mine, that's enough. If I'm mapping to a legacy schema I don't control, or the impedance mismatch is fighting me on every aggregate, I'll introduce a separate persistence model — and I'll budget for the mapping."*

That answer is complete, honest, and has a decision rule in it. It beats "the domain must never know about EF" by a wide margin.

---

## Concept 30 — The separate persistence model

The pure option: domain types that nothing maps, plus *persistence entities* that EF owns, plus explicit translation in the repository.

```csharp
// Infrastructure/Persistence/Models/OrderRecord.cs — EF owns this shape entirely
internal sealed class OrderRecord
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = default!;
    public byte[] RowVersion { get; set; } = default!;
    public List<OrderLineRecord> Lines { get; set; } = [];
}

internal sealed class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> FindAsync(OrderId id, CancellationToken ct)
    {
        var record = await db.Orders.Include(o => o.Lines)
                                    .FirstOrDefaultAsync(o => o.Id == id.Value, ct);
        return record is null ? null : OrderMapper.ToDomain(record);
    }
}
```

**What you gain:** a genuinely unconstrained domain model — real constructors, no parameterless ctor, no field-mapping requirements, immutability if you want it, and total freedom to reshape the schema without touching the domain (and vice versa). This is the norm in functional-leaning codebases and in systems where the schema is legacy or shared.

**What you pay, and it's more than people expect:**

- Mapping code in both directions for every aggregate, maintained forever.
- **You lose change tracking.** EF diffs the *record*, and the record you loaded is not the one you mutated. You must either re-load-and-copy, or track manually, or mark everything modified — which is Module 19's `Update(graph)` anti-pattern with extra steps. This is the single biggest practical cost and the one candidates forget.
- Concurrency tokens, generated values, and identity all need explicit round-tripping.
- Performance: two object graphs per operation, more allocations (Module 17).

**When it's right:** a legacy/shared schema you don't own; a domain model that genuinely can't be expressed in EF's constraints (deep immutability, unusual collection semantics); event sourcing (Module 24), where the "persistence model" is an event stream and this mapping is inherent; or a team that has decided purity is worth the bill and is willing to keep paying it.

**When it's over-engineering:** a greenfield schema you control, with a team of five, and an aggregate design that EF maps comfortably. Then the three compromises of Concept 29 buy you 90% of the value for 5% of the cost.

---

## Concept 31 — `IApplicationDbContext`: the honest analysis

The most popular shortcut in .NET Clean Architecture, popularized by the jasontaylordev template: Application declares an interface exposing `DbSet<T>` and `SaveChangesAsync`, Infrastructure's `DbContext` implements it, handlers use LINQ directly.

```csharp
// Application
public interface IApplicationDbContext
{
    DbSet<Order> Orders { get; }
    DbSet<Customer> Customers { get; }
    Task<int> SaveChangesAsync(CancellationToken ct);
}
```

**The case for it** — and it's stronger than purists admit:

- No repository layer to write or maintain; no method explosion as query needs multiply.
- You keep EF's full expressiveness in handlers: projections, `Include`, split queries, `ExecuteUpdate`.
- Handlers stay testable against a real database via Testcontainers, which is the test that actually tells the truth (Module 19, Concept 70).
- For CRUD-ish use cases — most use cases in most systems — it's honest about what's happening.

**The case against**, stated precisely rather than dogmatically:

- **It is not a boundary; it's a rename.** `Application` now references `Microsoft.EntityFrameworkCore` for `DbSet<T>` — the dependency rule is broken at the package level, no matter what the diagram says. Run the Concept 1 test: delete EF Core, does Application compile? No.
- **`DbSet<T>` is `IQueryable<T>`**, so every problem in Concept 32 applies: deferred execution crossing the boundary, ORM-specific translation failures surfacing as runtime exceptions in application code, and test doubles that cannot faithfully implement it.
- **It leaks the aggregate boundary.** Nothing stops a handler loading `Customers` and mutating an `Order` reached through a navigation. All the invariant protection you designed into aggregates becomes advisory.
- **It leaks the transaction boundary**, since any handler can call `SaveChangesAsync` on whatever it happens to have tracked.

**The position to take:** it's a *deliberate trade of architectural purity for developer velocity*, and it's a defensible one in systems whose "domain" is mostly data. What is not defensible is doing it and claiming the domain is persistence-ignorant. A very common mature compromise:

- **Writes** go through aggregate-shaped repositories — small number, invariant-protecting, transaction-owning.
- **Reads** go through a separate read abstraction (or straight to Dapper/SQL) that is explicitly allowed to know the database.

That's Concept 35, and it makes both halves honest.

---

## Concept 32 — `IQueryable` leakage, in detail

Returning `IQueryable<T>` from a repository is the most common way a .NET codebase keeps the shape of Clean Architecture while losing all of its properties. Five distinct problems, worth being able to list:

1. **It exports an unspecified contract.** `IQueryable` means "an expression tree someone will translate." *Which* provider, supporting which methods, with which semantics, is invisible in the type. `.Where(o => MyHelper.IsEligible(o))` compiles everywhere and throws at runtime on EF (Module 19, Concept 15), silently client-evaluates on some other provider, and works fine on a `List<T>`-backed fake — three different behaviours from one signature.
2. **Execution timing crosses the boundary.** The caller decides when the query runs, by which point the `DbContext` may be disposed. This is how `ObjectDisposedException` appears in a controller for a repository call made three layers down.
3. **Aggregate boundaries evaporate.** If callers can compose, they can reach anywhere in the object graph. Your carefully designed consistency boundary is a suggestion.
4. **Performance becomes unattributable.** The SQL is determined by a composition of fragments from several files. Nobody owns the query, so nobody owns the index (Module 19, Concept 47).
5. **Fakes lie.** `List<T>.AsQueryable()` implements every method in memory. Your tests pass; production throws. This is the single most dangerous property, because the test suite is actively giving you false confidence.

**What to do instead**, in escalating order of ceremony:

- Return materialized results from named methods (Concept 27). Default.
- Accept a **specification** and apply it inside the repository (Concept 33), when criteria genuinely repeat.
- Accept a **paging/sort descriptor** — your own small record, not an `Expression` — when the caller needs to vary paging: `record Page(int Number, int Size, SortField Field, bool Descending)`.
- For genuinely dynamic search (a grid with 14 optional filters), **don't pretend**: give the read side a dedicated query service that owns the query and is tested against a real database (Concept 34).

The subtlety worth naming: `IAsyncEnumerable<T>` is *not* in the same category. Returning `IAsyncEnumerable<Order>` for a streaming export exports laziness but not a query language — the query is already decided. It still holds a connection while the caller enumerates (Module 19, Concept 50), so document it, but it is not a leaked abstraction in the way `IQueryable` is.

---

## Concept 33 — Specifications: a named query object

The specification pattern turns a query criterion into a first-class, testable, named object.

```csharp
public sealed class OverdueOrdersSpec : Specification<Order>
{
    public OverdueOrdersSpec(DateTimeOffset asOf)
    {
        Query.Where(o => o.Status == OrderStatus.Placed && o.DueDate < asOf)
             .Include(o => o.Lines)
             .OrderBy(o => o.DueDate);
    }
}

var overdue = await repository.ListAsync(new OverdueOrdersSpec(now), ct);
```

**What it buys:** criteria live in the Application/Domain layer with business names, they're reusable across call sites, they're unit-testable against in-memory collections, and repositories collapse from twenty methods to four generic ones.

**What it costs, and what to watch:** the `Query.Where(...)` builder is an expression DSL, so you've re-imported expression trees into the inner layer under a nicer name — a specification that includes `Include(...)` is naming an ORM concept. Ardalis's `Specification` library is explicitly EF-shaped; using it means Application depends on a package that depends on EF Core's abstractions. Be clear-eyed that this is a *smaller* leak than `IQueryable`, not zero.

**When it's worth it:** when the same criteria genuinely appear in three or more places, or when you want business rules like "what counts as overdue" to exist in one named place (which is a real domain concern that otherwise gets duplicated in five queries with slightly different definitions). **When it's ceremony:** when each spec has exactly one caller. Then it's a method on the repository, spelled worse.

---

## Concept 34 — The read side is allowed to know about the database

The single most freeing idea in this module, and the one that dissolves most repository arguments:

**A query does not need a domain model.** Domain models exist to protect invariants during *changes*. A read changes nothing, so there is nothing to protect. Loading an aggregate, rehydrating its object graph, and mapping it to a DTO in order to display it is pure cost.

So the mature layering has two paths:

| | Write path | Read path |
|---|---|---|
| Entry | Command handler | Query handler |
| Model | Aggregates, invariants, domain events | Flat DTOs shaped for the screen |
| Persistence | Aggregate repository + unit of work | Direct SQL / Dapper / EF projection, no tracking |
| Transaction | Yes, one per use case | None |
| Testing | Domain unit tests + handler tests with fakes | Integration tests against a real database |
| Abstraction | `IOrderRepository` (a port) | `IOrderQueries` (a port whose implementation is unapologetically SQL) |

```csharp
// Application: the port says WHAT, in application vocabulary, with no query language in sight
public interface IOrderQueries
{
    Task<OrderSummaryDto?> GetSummaryAsync(OrderId id, CancellationToken ct);
    Task<PagedResult<OrderListItemDto>> SearchAsync(OrderSearch criteria, CancellationToken ct);
}

// Infrastructure: Dapper, EF projection, a view, a stored procedure — the adapter's business
internal sealed class OrderQueries(IDbConnectionFactory factory) : IOrderQueries { … }
```

Note that the dependency rule is **fully satisfied**: Application declares the port, Infrastructure implements it, and the domain model isn't involved because it isn't needed. This is CQRS in its cheapest, most valuable form — no event sourcing, no separate database, no eventual consistency, just two paths through the same architecture (Module 23 goes further).

This also resolves Concept 31's tension cleanly: if the read side has an honest, SQL-aware home, the pressure to expose `DbSet<T>` to handlers mostly disappears.

---

## Concept 35 — CQRS-lite as a layering decision

Worth naming explicitly because interviewers use "CQRS" to mean anything from "two methods" to "event-sourced projections over Kafka."

The layering-level version is: **commands and queries take different routes through the architecture, and only commands pay for the domain model.** No separate database, no separate process, no eventual consistency, no new infrastructure. The only "CQRS" in it is the acknowledgement that reads and writes have different requirements.

Two consequences worth stating:

- **Your DTO inventory shrinks.** The read model *is* the DTO. You stop mapping entity → application DTO → response DTO, because the query already produced the response shape.
- **Your repository interface shrinks to the write side**, which is exactly where the aggregate rule makes sense. Most of the "repository method explosion" complaint is caused by using repositories for reads.

The failure mode to watch for: **read DTOs become the de facto domain model.** Once `OrderSummaryDto` exists, someone will add `TotalIncludingTax` to it and compute it in the query. Now the tax rule lives in SQL. The rule is: **calculations that are business rules belong to the domain even if they're only ever displayed** — either compute and store them at write time (usual answer), or call a domain function from the query's mapping step. What you must not do is let the same rule exist twice.

---

## Concept 36 — Aggregates decide the load, and therefore the repository

The layering consequence of aggregate design (Module 22 does the design itself):

- **An aggregate is the unit of consistency and therefore of loading and saving.** `IOrderRepository.FindAsync` returns the whole order with its lines, because the invariant "the order total equals the sum of its lines" needs all of them to be checkable.
- **Therefore repository method names encode load size**, and "load a partial aggregate for performance" is a signal that your aggregate boundary is wrong (or that the operation is really a read — see Concept 34).
- **References between aggregates are by ID, not by navigation.** `Order.CustomerId`, not `Order.Customer`. This is a layering property, not just a DDD nicety: it means loading an order can never accidentally drag in a customer graph, and it keeps the transaction confined to one aggregate.
- **One transaction, one aggregate** is the default. When a use case must change two, you have three options and should be able to name them: expand the aggregate (usually wrong), accept a multi-aggregate transaction inside one database (pragmatic and common), or go asynchronous with a domain event and eventual consistency (Module 11's outbox; correct when the aggregates are in different modules or services).

The "how big should an aggregate be?" answer that lands in interviews: *as small as the invariants allow.* Every extra entity inside the boundary is extra loading, extra contention, and extra concurrency conflicts (Module 19, Concept 33).

---

## Concept 37 — Schema ownership and migrations

A short but frequently-asked point of layering hygiene:

- **Migrations are infrastructure artifacts.** They live with the `DbContext`, in the Infrastructure project, and they are deployment steps — not application startup code (Module 19, Concept 57: idempotent scripts or bundles applied by a pipeline, not `Database.Migrate()` in `Program.cs`).
- **The domain constrains the schema but doesn't own it.** The domain says "an order has lines and a status"; the schema decides column types, indexes, partitioning, and whether status is a `tinyint` or an `nvarchar(32)`.
- **Each bounded context/module owns its own tables and its own `DbContext`** (Module 19, Concept 61), with its own migration history table. Schema ownership is the real module boundary — Module 21's central claim — and if two modules write the same table, they are one module with extra folders.
- **The EF model configuration is the mapping layer**, which is why `IEntityTypeConfiguration<Order>` classes living in Infrastructure are the mechanism that makes Concept 29 work: all the persistence knowledge about `Order` is in one file that the domain never sees.

---

# Part D — Cross-boundary decisions: the ten arguments every team has

---

## Concept 38 — Mapping: how many models do you actually need?

The theoretical maximum is five models for one concept: HTTP request → application command → domain entity → persistence record → HTTP response. Each boundary you take seriously adds a model and a mapper.

The reality:

| Model | Keep it separate when… | Collapse it when… |
|---|---|---|
| **HTTP request/response** vs. **application command/DTO** | Your API is versioned independently, has public consumers, or its shape is driven by UI concerns | It's an internal API and the command *is* the request — most services |
| **Application DTO** vs. **domain entity** | Always. Serializing entities exposes internals and couples your API to your model | Never (this is the one boundary that always earns its keep) |
| **Domain entity** vs. **persistence record** | Legacy/shared schema, or purity is a hard requirement | Greenfield schema you own — use Concept 29's compromises |

So the honest default for a typical internal service is **two models plus a read DTO**: the command/query (shared by HTTP and the use case), the domain entity (write side only), and the read DTO produced directly by the query. The five-model version is for public APIs over legacy schemas.

**On mapping libraries, as of 2026.** AutoMapper v15+ is commercial (Community tier free under the revenue/capital thresholds; see Orientation). This forced a lot of teams to revisit a decision they'd made by reflex, and the revisiting was overdue:

- **Manual mapping** — an extension method or a `static ToDto()` — is the default now. It's explicit, debuggable, allocation-visible, refactor-safe, and free. A mapping that's 15 lines of assignments is *not* a problem to solve.
- **Source generators** (e.g. Mapperly) give you generated code you can read, compile-time errors for unmapped members, no reflection, and AOT-friendliness (Module 17). This is the good middle.
- **Reflection-based mappers** cost startup time, hide errors until runtime, make "find all usages" useless, and now cost money at scale.

The architectural point, independent of library: **the mapper belongs to the outer side of the boundary.** `OrderDto.From(order)` in Application, `OrderResponse.From(dto)` in Web. The inner type never knows the outer type exists — which means the domain never has a `ToDto()` method on it.

---

## Concept 39 — Three kinds of validation, three homes

The single most common muddle in layered .NET codebases. There are three distinct things called "validation" and they belong in three places, with three different failure behaviours:

**1. Input validation (shape).** "Quantity must be a positive integer." "Email must look like an email." "Name is at most 200 characters." This is about whether the *request is well-formed*, requires no business knowledge and no I/O, and fails as **400 Bad Request**.
Home: the edge. Data annotations, FluentValidation (still Apache-2.0, 12.x), or ASP.NET Core 10's built-in minimal-API validation. Running it as an Application pipeline behaviour is also fine and is what the templates do.

**2. Application validation (context).** "This SKU exists." "This customer isn't already registered." "You have permission to cancel this order." Needs I/O, needs the current user, needs the database. Fails as **404 / 409 / 403** depending.
Home: the use-case handler, using ports. Not in the domain (the entity can't query the database, and shouldn't) and not at the edge (the edge shouldn't have repositories).

**3. Invariants (rules).** "An order can't be cancelled after it ships." "A transfer can't overdraw an account." These are not validation at all — they are **the rules the domain exists to enforce**, and they must be impossible to bypass.
Home: inside the entity, enforced in the constructor and in every method. They fail by *refusing to produce an invalid object* — either by throwing a domain exception or by returning a failed result.

```csharp
public Result Cancel(string reason, DateTimeOffset now)
{
    if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
        return Result.Failure(OrderErrors.CannotCancelAfterShipping);    // invariant, in the domain
    if (string.IsNullOrWhiteSpace(reason))
        return Result.Failure(OrderErrors.ReasonRequired);               // also an invariant, not input validation
    Status = OrderStatus.Cancelled;
    Raise(new OrderCancelled(Id, reason, now));
    return Result.Success();
}
```

The diagnostic question for any check: **"if a different entry point performed this operation — a message consumer, a batch job, an admin tool — would this check still need to run?"** If yes, it's an invariant and belongs in the domain. If it's only about this request's syntax, it belongs at the edge. That single question resolves almost every real argument about validation placement.

And the uncomfortable truth to volunteer: **some checks are duplicated on purpose.** The UI checks quantity > 0 for a good error message, the edge checks it to reject garbage early, and the domain checks it because it's a rule. That's not a DRY violation; those are three different requirements that happen to look alike.

---

## Concept 40 — Errors across boundaries: exceptions or results?

The rule that actually works, independent of religion: **expected outcomes are return values; unexpected ones are exceptions.**

- "Order not found," "insufficient stock," "already cancelled" are *expected* — they're part of the use case's specification, they occur in normal operation, and the caller must handle them. Model them as values: `Result<T>`, `ErrorOr<T>`, `OneOf<Success, NotFound, Conflict>`, or a discriminated-union-shaped return.
- "The database is unreachable," "the payload didn't deserialize," "a null reference" are *exceptional* — no caller can meaningfully handle them locally, and you want the stack trace and the middleware.

Why this matters for layering specifically: **exceptions cross boundaries invisibly.** A `DbUpdateConcurrencyException` thrown in Infrastructure and caught in a controller means the controller knows about EF Core, defeating the whole structure. So the adapter's job includes **translating failure modes** (Concept 44): infrastructure exceptions become either domain-meaningful results or a single wrapped exception type.

Costs of the result-type approach, which you should name rather than being sold on it:

- Every call site must propagate, and C# has no `?` operator for it — so you get `if (result.IsFailure) return result;` everywhere, or a fluent `.Bind(...)`/`.Map(...)` chain that some teams find unreadable.
- Your error type becomes public vocabulary across the whole system. Choose it once, and expect it in your domain project forever (Concept 15).
- Exceptions still exist for the genuinely exceptional, so you have two mechanisms regardless.

For the domain specifically, there's a defensible split that experienced teams use: **constructors and invariant-protecting methods throw** (because producing an invalid object is a programming error that must be impossible), while **use-case-level outcomes return results** (because "not found" is a normal Tuesday). Be able to defend whichever you choose.

---

## Concept 41 — Mapping errors to the protocol

The domain must never name an HTTP status code, and the way you honour that in ASP.NET Core is one small piece of Presentation:

```csharp
// Web: one place translates application outcomes into protocol
internal sealed class DomainExceptionHandler(IProblemDetailsService problems) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is not DomainException domain) return false;    // let the pipeline handle the rest

        ctx.Response.StatusCode = StatusCodes.Status422UnprocessableEntity;
        return await problems.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = ctx,
            ProblemDetails = new ProblemDetails
            {
                Type = $"https://errors.example.com/{domain.Code}",
                Title = "The operation violated a business rule.",
                Detail = domain.Message
            }
        });
    }
}
```

Points worth knowing:

- `IExceptionHandler` (since .NET 8) with `AddProblemDetails()` is the current idiom; several handlers can be chained and each returns whether it handled the exception. It replaces the hand-rolled exception middleware in every older template.
- **RFC 9457 `ProblemDetails`** is the standard error shape — `type`, `title`, `status`, `detail`, `instance`, plus extensions. Use the `type` URI as your stable error code; that's what clients should switch on, not the message text.
- For result types, the equivalent is a single extension method — `result.ToMinimalApiResult()` / `.ToActionResult()` — mapping error categories to `TypedResults`. Write it once. The moment it appears in two places it will diverge.
- **Never leak infrastructure detail into the response.** SQL error numbers, EF messages and stack traces are diagnostics, not contract. They go to logs and traces (Module 18's telemetry), with a correlation id in the problem details so support can join them.

---

## Concept 42 — Time, identity and randomness

The three classic non-determinism sources, and the current .NET answers:

**Time.** `TimeProvider` (BCL since .NET 8) is the port. Inject it, register `TimeProvider.System`, and use `FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing` in tests — it also controls `Task.Delay`, timers and `PeriodicTimer` when you pass it through, which is what makes testing retry/timeout logic (Module 13) possible without `Thread.Sleep`. **Delete your `IDateTime` interface.**

The layering nuance: the domain shouldn't *inject* `TimeProvider` into entities — it should **receive the time as a parameter**: `order.Cancel(reason, now)`. That keeps entities pure functions of their inputs, which is what makes domain tests trivial. The handler gets `TimeProvider` and passes the value in.

**Identity.** IDs belong to the caller or the domain, not to the database, for one architectural reason: **if the database assigns the ID, you cannot publish a domain event about an entity until after you've saved it**, which breaks the "raise the event where the decision was made" property and makes the outbox awkward. Generate IDs in the application:

- `Guid.CreateVersion7()` (.NET 9+) gives you a time-ordered UUID — client-generated *and* index-friendly, which removes the classic clustered-index fragmentation objection to GUID keys (Module 19, Concept 6).
- Wrap them: `readonly record struct OrderId(Guid Value)` with an EF value converter in Infrastructure. Typed IDs cost 20 lines and eliminate an entire class of "passed the customer id where the order id goes" bug.

**Randomness.** Anything non-deterministic — random selection, shuffling, sampling, token generation — goes behind a port, or takes its value as a parameter, for exactly the same reason time does. Ban `System.Random` in the domain via `BannedSymbols.txt` (Concept 24).

---

## Concept 43 — Configuration

Rules:

1. **`IOptions<T>` and `IConfiguration` are framework types and belong at the edge.** Infrastructure adapters may take `IOptions<StripeOptions>`; Application handlers should not, because a use case depending on `IOptions<T>` means the use case has a framework dependency for no benefit.
2. **If a use case genuinely needs a business setting** — "free shipping threshold," "maximum refund window" — that's a *policy value*, not configuration plumbing. Give it a port (`IPricingPolicy`, `IRefundPolicy`) or pass it as a plain record from the composition root. The clue is whether a product manager would change it, and whether it should be testable and auditable.
3. **Validate at startup, not at first use:**
   ```csharp
   services.AddOptions<StripeOptions>()
           .Bind(config.GetSection("Stripe"))
           .ValidateDataAnnotations()
           .ValidateOnStart();       // fail the deployment, not the first payment
   ```
4. **Secrets never reach the inner layers at all.** A connection string is an argument to an adapter's constructor, provided by the composition root from Key Vault / managed identity / Aspire.

---

## Concept 44 — Anti-corruption layers: translate the vocabulary *and* the failures

The adapter for an external system has three jobs, and most implementations do only the first:

1. **Translate types** — their JSON shape into yours.
2. **Translate vocabulary** — their model into your ubiquitous language. Their `subscription.status = "past_due"` is your `SubscriptionState.Delinquent`. If their word appears anywhere inside your boundary, the corruption got in.
3. **Translate failure modes** — a `429`, an `HttpRequestException`, a timeout, and a `200 OK` containing `{"error": …}` all become *your* error vocabulary: `PaymentResult.Declined`, `PaymentResult.TemporarilyUnavailable`, or an exception you defined.

```csharp
// Application: the port speaks your language entirely
public interface IPaymentGateway
{
    Task<PaymentOutcome> ChargeAsync(PaymentRequest request, CancellationToken ct);
}

public abstract record PaymentOutcome
{
    public sealed record Captured(string Reference) : PaymentOutcome;
    public sealed record Declined(DeclineReason Reason) : PaymentOutcome;
    public sealed record Unavailable(TimeSpan? RetryAfter) : PaymentOutcome;
}
```

Notice what the port doesn't contain: no `HttpResponseMessage`, no status codes, no vendor id format, no `StripeException`. A handler written against this can be tested with three lines of fake and reads like the business process it implements.

The retry/circuit-breaker question (Module 13, Module 25) lands here too: **resilience is the adapter's business.** The use case expresses intent once; the adapter decides whether that means one HTTP call or five with jittered backoff. If your handler contains a retry loop, the policy has leaked inward — and worse, it will now double up with the `HttpClient`'s standard resilience handler.

---

## Concept 45 — Domain events vs. integration events

A distinction that is pure layering, and a favourite interview probe.

| | Domain event | Integration event |
|---|---|---|
| Meaning | "Something happened inside this model" | "Something happened that other systems should know about" |
| Scope | In-process, same transaction, same bounded context | Cross-process or cross-module; a published contract |
| Shape | Rich, uses domain types (`OrderId`, `Money`) | Flat, primitives, versioned, serializable |
| Who defines it | The domain | The publishing module's public contract (often a separate `Contracts` package) |
| Delivery | Synchronous dispatch, in the unit of work | The outbox → broker → at-least-once → idempotent consumers |
| Failure | Rolls back with the transaction | Retries; consumers must be idempotent |

The mechanics in .NET (Module 19, Concepts 35–36 built this): entities raise domain events into a list; a `SaveChangesInterceptor` drains them during `SaveChanges`; handlers run inside the same transaction; any handler that needs to tell the outside world writes an **integration event to the outbox table in the same transaction** (Module 11's dual-write fix). A background worker publishes.

The layering consequences that people miss:

- **A domain event handler that sends an email is wrong**, because your transaction now depends on SMTP and a rollback can't unsend it. It should write an integration event or an outbox row; infrastructure does the sending later.
- **Integration event contracts are not domain types.** Publishing `OrderPlaced { Order = <entity> }` exports your model to every consumer, and now your aggregate's internals are a public API you can't change.
- **Domain events are past tense and immutable.** `OrderPlaced`, not `PlaceOrder`. If a handler can "reject" the event, it isn't an event, it's a command in disguise.

---

## Concept 46 — Background work is a driving adapter

The perennial question: *"my `BackgroundService` needs to process the outbox / poll a queue / run nightly billing — which layer does it go in?"*

The answer from Concept 6: a worker is a **driving adapter**, exactly like an HTTP endpoint. It sits in the outer ring, it translates a trigger (a timer, a message, a queue lease) into a use-case invocation, and it owns none of the logic.

```csharp
// Worker host (outer): a driving adapter
internal sealed class ExpireHoldsWorker(IServiceScopeFactory scopes, TimeProvider time, ILogger<ExpireHoldsWorker> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1), time);
        while (await timer.WaitForNextTickAsync(ct))
        {
            using var scope = scopes.CreateScope();                       // Module 18: a scope per unit of work
            var handler = scope.ServiceProvider.GetRequiredService<ExpireHoldsHandler>();
            await handler.HandleAsync(new ExpireHolds(time.GetUtcNow()), ct);
        }
    }
}
```

Three rules that fall out:

1. **The use case must not know who called it.** `ExpireHoldsHandler` is identical whether triggered by a timer, an admin endpoint, or an integration test. If it takes a `BackgroundService`-shaped parameter or reads a schedule, the adapter has leaked inward.
2. **A scope per unit of work, not per worker lifetime** — otherwise the `DbContext` lives for days and Module 19's identity-map growth kills the pod.
3. **Where the worker project sits is a hosting decision, not a layering one.** Same process as the API or a separate deployable, it references Application and Infrastructure exactly like the web host does — and it has its own composition root.

---

## Concept 47 — Cross-cutting behaviours and where to put them

Logging, validation, transactions, caching, authorization, retries, metrics: each has a natural home, and "wrap everything in a pipeline" is not automatically it.

| Concern | Natural home | Why |
|---|---|---|
| Request/response logging, correlation | Presentation middleware | It's protocol-level and already exists in ASP.NET Core |
| Telemetry spans for use cases | Application pipeline behaviour or the dispatcher | The span you want is "PlaceOrder", not "POST /orders" |
| Input validation | Edge, or an Application behaviour | Either works; pick one and be consistent |
| Transaction / `SaveChanges` | Application behaviour *or* explicit in handler | Behaviour = less code; explicit = visible boundary |
| Authorization (business rule) | Application (handler or behaviour) | It's a rule about the operation, not about the URL |
| Authentication | Presentation | It's a protocol handshake |
| Caching | Infrastructure decorator, or the query handler | Module 10: caching is a replica with weak consistency; that's a data decision |
| Retry / circuit breaker | Infrastructure adapter | It's about a specific unreliable dependency |

Two mechanisms in .NET, and the ability to compare them is a senior signal:

- **A mediator pipeline** (MediatR's `IPipelineBehavior`, or your own 30-line dispatcher): behaviours run for *every* request, ordered by registration. Powerful, and it makes the cross-cutting concerns visible in one list. Costs: an extra indirection you can't "go to definition" through, and — since v13 — a licence conversation.
- **Decorators** (Scrutor's `Decorate<,>`, or hand-written): applied per interface, explicit, and they work fine without a mediator. Costs: registration code gets fiddlier; ordering is implicit in registration order.

You can run this architecture with **no mediator at all**: inject handlers directly, wrap them in decorators where needed. A senior candidate should be able to say why the mediator exists (uniform pipeline, decoupled dispatch, one registration convention) and why it isn't required (it's a dispatcher, not an architecture — and calling `ISender.Send` from a controller is not more decoupled than calling a handler, it's just less typed).

And the accepted exception: **`ILogger<T>` in the domain.** Purists forbid it; pragmatists allow it because logging is ambient, its interface is stable, and a domain method that needs to explain a decision is more valuable than one that's abstractly pure. If you allow it, allow *only* `Microsoft.Extensions.Logging.Abstractions`, which is a tiny package with no framework pull. Most domains don't need it, because domain methods return results that the handler logs.

---

## Concept 48 — The seams that are negative value

The list that demonstrates judgment. **Do not abstract these:**

| Thing | Why not |
|---|---|
| **`ILogger<T>`** | It's already an abstraction, in a package with no dependencies, with a working no-op. Wrapping it gains nothing and loses structured logging |
| **The DI container** | An abstraction over the thing that resolves abstractions is a service locator with better PR |
| **`System.Text.Json`** | Serialization happens *at* a boundary, not across one. The adapter owns it |
| **LINQ-to-objects** | It's the language |
| **`HttpClient`** | Don't wrap it — *replace* it with a port that speaks your domain (Concept 44). `IHttpClientFactory` handles lifetime; an `IHttpClientWrapper` handles nothing |
| **Framework abstractions generally** | `IMemoryCache`, `IDistributedCache`, `HybridCache`, `TimeProvider`, `IOptions<T>` — these *are* the ports, written by people with more context than you |
| **Your own database, "to swap it later"** | See Concept 60 |
| **A single-implementation use-case interface** | `IPlaceOrderUseCase` with one implementation and one caller exists so the diagram looks symmetric. Keep it only if you decorate it |
| **Mappers behind interfaces** | `IMapper<TFrom,TTo>` for a pure function is indirection over a static method |
| **Value objects behind interfaces** | `IMoney` is not a thing |

The shared reason: **an abstraction is justified by a second implementation you actually need — a fake that makes a test possible, a vendor you might change, or a module boundary you're enforcing.** "It might be useful someday" has a well-documented failure rate.

---

## Concept 49 — The fake test

A one-line heuristic that resolves most "should this be an interface?" arguments:

> **If you wouldn't write a fake for it, you don't need an interface for it.**

Apply it:

- `IPaymentGateway` → yes, you want a fake that returns `Declined` on command. **Interface.**
- `IOrderRepository` → yes, an in-memory dictionary-backed fake makes handler tests instant. **Interface.**
- `ILogger<T>` → you'd use `NullLogger<T>` or a capturing logger, both of which already exist. **No new interface.**
- `IOrderMapper` → your fake would be… a mapper. **No interface.**
- `IApplicationDbContext` → your fake would have to implement `DbSet<T>`, which no honest fake can (Concept 32). **This is the tell that it isn't a real abstraction.**

The corollary is Concept 65: the reason to write a *fake* rather than a mock is that a fake has behaviour and can therefore be wrong in the same ways the real thing is wrong, whereas a mock returns exactly what the test author assumed. An in-memory repository that enforces uniqueness catches a bug; `mock.Setup(r => r.FindAsync(...)).ReturnsAsync(order)` cannot.

---

## Concept 50 — Async, cancellation, and the shape of port signatures

Small, concrete, and frequently probed because it's where layering meets Module 15.

- **Every port method that might do I/O is `Task`/`ValueTask`-returning and takes a `CancellationToken`.** No exceptions, including the ones that "will always be fast" — the point of the port is that the implementation might not be.
- **The token crosses every boundary**, from `HttpContext.RequestAborted` through the handler into the repository into EF. Module 18 covered the mechanics; the layering rule is that no layer may swallow it or substitute `CancellationToken.None`.
- **The domain stays synchronous wherever possible.** `order.Cancel(reason, now)` returning `Result` — not `Task<Result>` — is a design statement: this is a pure decision about in-memory state. If a domain method needs `async`, it's usually because it's secretly doing I/O, which means it's really a domain *service* that should take its data as a parameter, or a use case in the wrong project.
- **Ports return domain/application types, never framework ones.** `Task<Order?>`, not `Task<ActionResult<Order>>`; `Task<PaymentOutcome>`, not `Task<HttpResponseMessage>`.
- **`IAsyncEnumerable<T>` is legitimate for streaming ports**, with the connection-lifetime caveat from Concept 32.

---

# Part E — The alternatives, and when they win

An architect who can only advocate is a salesperson. This part is what makes the advocacy credible.

---

## Concept 51 — Vertical slice architecture: the honest claim

Jimmy Bogard's argument, stated fairly: layered architectures apply **one uniform shape to every request**, but requests are not uniform. Most are simple; a few are complex. Forcing the simple ones through Controller → Service → Repository → Entity produces abstraction for its own sake — "many abstractions around concepts that really shouldn't be abstracted" — and produces mock-heavy tests that verify wiring rather than behaviour. Instead, **treat each request as its own use case and give it exactly the code it needs**: minimize coupling *between* slices, maximize cohesion *within* one.

A slice is a folder (or a file) containing everything for one request: the endpoint, the request/response types, the validation, the handler, and the data access — whatever shape that data access deserves.

```
Features/
  Orders/
    PlaceOrder.cs        // endpoint + command + validator + handler + EF/Dapper, all here
    CancelOrder.cs
    GetOrderSummary.cs   // query straight to SQL, no aggregate, no repository
```

**What it genuinely optimizes:** the change. Adding a field touches one file. Deleting a feature deletes one file. The blast radius of a change equals the feature you changed, which is the coupling property you actually wanted from "loose coupling."

**What it genuinely costs**, and Bogard says this himself: it assumes a team that understands code smells and refactoring, because the shared logic that should be extracted *won't be extracted by the structure* — you have to notice and do it. Consequences reported consistently in practice:

- **Duplication that isn't harmless.** Three slices each compute order totals slightly differently, and one of them is wrong.
- **Inconsistency across slices.** One uses Dapper, one uses EF, one uses a repository; a new engineer doesn't know which convention applies.
- **The invariant problem.** If every slice reaches the database directly, nothing enforces aggregate rules. A rich domain and unconstrained per-slice data access are in tension, and it's a real tension, not a stylistic one.

**Bogard's own answer to the duplication objection is the important one:** refactor the common code to a shared location; just don't call one feature from another. In other words, vertical slices don't abolish shared code — they abolish *premature* shared code and make the extraction an explicit decision.

---

## Concept 52 — The reframing: where do you want your coupling?

This is the sentence that makes the whole debate tractable, and it's the one to deploy in an interview:

> **Layers decouple along the technology axis. Slices decouple along the change axis. You get to pick which axis matters more for this system.**

Consequences of the choice:

- If your churn is **"we're changing the database / moving to a queue / adding a second front end"**, layering pays: the technology axis is where your change is.
- If your churn is **"we ship five new features a fortnight and each one touches nine files"**, slices pay: the feature axis is where your change is.
- If your churn is **"the business rules themselves change constantly and correctness matters"**, neither structure alone is the answer — a real domain model is, and both structures can host one.

The second-order observation, which is where mature teams land: **the two are orthogonal.** Vertical slices tell you how to organize *the application layer*. The dependency rule tells you which way references point. Nothing stops you having both, which is Concept 53.

---

## Concept 53 — The hybrid most teams actually run

The shape you'll see in competent .NET codebases in 2026, and the one to propose as a default:

```
src/
  Domain/                      ← real aggregates, only where invariants exist
  Application/
    Orders/
      PlaceOrder/              ← slice: command + validator + handler (+ endpoint, if you like)
      CancelOrder/
      GetOrderSummary/         ← read slice: Dapper/projection, no domain model
    Invoicing/…
  Infrastructure/              ← adapters: persistence, messaging, external services
  Web/                         ← composition root + protocol
```

The rules that make it work:

1. **The dependency rule holds at the project level.** Domain knows nothing; Application knows Domain; Infrastructure implements Application's ports.
2. **Inside Application, organization is by feature, not by pattern.** No `Commands/`, `Queries/`, `Handlers/`, `Validators/` folders.
3. **Slices don't call each other.** Cross-feature needs go through the domain, a shared application service, or an event — never `PlaceOrderHandler` invoking `SendInvoiceHandler`.
4. **Complexity is chosen per slice.** A read slice projects straight to a DTO. A CRUD write slice may use the context directly. A rules-heavy slice loads an aggregate and calls domain behaviour. **This is the single most important sentence in Part E**, because it's the thing junior architectures get wrong in both directions.
5. **Shared logic is extracted when it's shared, to the layer that owns the concept** — domain rule to Domain, technical helper to Infrastructure — not to a `Common` project by reflex.

Notice that the ardalis template now ships both `clean-arch` and `min-clean` (single-project vertical slice), which is the ecosystem's acknowledgement that the answer is "it depends on the system," from the person who wrote the most-used Clean Architecture template in .NET.

---

## Concept 54 — Transaction Script and Table Module are not failures

Fowler's *PoEAA* catalogues three domain-logic patterns, and Clean Architecture discourse tends to forget two of them:

- **Transaction Script** — one procedure per business transaction, top to bottom: validate, load, decide, save. Logic lives in the procedure, data in dumb structures.
- **Table Module** — one class per table, containing the logic for operating on that table's records. (The DataSet-era .NET pattern.)
- **Domain Model** — an object graph with behaviour and invariants.

Fowler's own guidance is about the *complexity curve*: transaction scripts are cheaper up to a point and get worse quickly past it; a domain model costs more to set up and scales better with complexity. The crossover is real, and it is **per use case**, not per system.

A well-written transaction script is a perfectly good handler:

```csharp
public async Task<Result> HandleAsync(MarkInvoicePaid cmd, CancellationToken ct)
{
    var invoice = await invoices.FindAsync(cmd.InvoiceId, ct);
    if (invoice is null)             return Result.NotFound();
    if (invoice.Status == "Paid")    return Result.Conflict("Already paid.");
    if (cmd.Amount != invoice.Total) return Result.Invalid("Amount mismatch.");

    invoice.Status = "Paid";
    invoice.PaidAt = time.GetUtcNow();
    await uow.SaveChangesAsync(ct);
    return Result.Success();
}
```

There is nothing wrong with this for an invoice whose lifecycle is three states and whose rules fit on a Post-it. The mistake isn't writing it — it's writing it and calling it a domain model, or writing forty of them and never noticing that the same three rules now appear in fourteen places (the signal to promote to a domain model).

**Anemic domain model** is the name for the failure case: entities with public getters and setters and no behaviour, with all logic in "services." Fowler's objection isn't aesthetic — it's that you've paid for the object graph, the mapping, and the ceremony, and received none of the benefit. An honest transaction script over DTOs is *better* than an anemic domain model, because it doesn't pretend.

---

## Concept 55 — The complexity ladder, applied per use case

The decision procedure to state out loud when someone asks "should we use DDD here?":

| Rung | Shape | Use when |
|---|---|---|
| **1. Direct CRUD** | Endpoint → EF/Dapper → response. No handler, no domain | Reference data, admin screens, settings, anything where the rule is "save what they typed" |
| **2. Transaction script** | Handler with procedural logic over simple records | A handful of conditions, one aggregate, rules that rarely change |
| **3. Domain model** | Aggregates, invariants, value objects, domain events | Rules that are numerous, interacting, argued about, or regulated |
| **4. Explicit workflow / state machine** | A modelled process with states and transitions | Long-running, multi-step, compensating (Module 11's sagas; Module 24's event sourcing) |

Two rules about the ladder:

- **You may run all four rungs in one codebase.** A system with 60 use cases might have 35 on rung 1, 18 on rung 2, 6 on rung 3, and 1 on rung 4. That's healthy. Uniformity is not a virtue here.
- **Promotion is cheap if the boundary exists; demotion is easy always.** Because every rung sits behind a use-case boundary, moving a use case from rung 2 to rung 3 changes one file's internals. This is the strongest argument for keeping the *use-case seam* even when you skip everything else.

The interview line: *"I'd pick the simplest rung that enforces the rules, per use case, and I'd expect the mix to shift over time. What I wouldn't do is apply rung 3 to all sixty."*

---

## Concept 56 — Functional core, imperative shell

The same dependency rule expressed without a single interface — worth knowing because it's the sharpest way to demonstrate that the rule is about *dependencies*, not about DI containers.

Gary Bernhardt's formulation (the "Boundaries" talk): **a functional core of pure functions that make decisions, wrapped in a thin imperative shell that performs effects.** The core takes data, returns data (including a description of what should happen), and does no I/O. The shell does the I/O and calls the core.

```csharp
// Core: pure. No ports, no interfaces, no async, no mocks in its tests.
public static class Pricing
{
    public static PricingDecision Price(Basket basket, PriceList prices, DateOnly today)
        => …;   // returns totals + a list of discounts applied + any warnings
}

// Shell: impure. Fetches, calls the core, persists, publishes.
public async Task<Result<Quote>> HandleAsync(QuoteBasket cmd, CancellationToken ct)
{
    var basket = await baskets.FindAsync(cmd.BasketId, ct);
    var prices = await priceList.CurrentAsync(ct);

    var decision = Pricing.Price(basket, prices, DateOnly.FromDateTime(time.GetUtcNow().DateTime));

    await quotes.AddAsync(decision.ToQuote(), ct);
    await uow.SaveChangesAsync(ct);
    return Result.Success(decision.ToQuote());
}
```

Why it's worth having in your vocabulary:

- **The tests for the core need nothing.** No container, no mocks, no fakes, no DI. Property-based testing becomes available, which for pricing/tax/scheduling logic is a substantially stronger form of verification.
- **It answers "do I need an interface for this?" with "no, you need a parameter."** Mark Seemann's "Dependency rejection" makes this argument in full: many dependencies are better removed than inverted — pass the data in, return the decision out.
- **It composes with everything else here.** The functional core *is* your domain; the shell *is* your use case. It's Clean Architecture with fewer nouns.

Where it strains in C#: deep immutability is verbose (records help), and "return a description of the effects" gets awkward when the decision depends on data you'd have to load conditionally. The pragmatic version — **push decisions into pure methods, keep I/O at the edges of handlers** — gets most of the benefit at no cost, and is worth adopting even in an otherwise conventional codebase.

---

## Concept 57 — Modules before layers (Module 21 preview)

The scaling correction: in a system past a certain size, **the first cut should be by business capability, and layering should happen inside each module.**

```
src/
  Modules/
    Ordering/      → Ordering.Domain, Ordering.Application, Ordering.Infrastructure
    Billing/       → Billing.Domain, …
    Shipping/      → …
  Shared/          → SharedKernel (tiny), Contracts (integration events)
  Host/            → composition root, one process
```

Why this ordering is right, in one line: **a layer groups things that change for different reasons; a module groups things that change for the same reason.** Fowler's "Presentation Domain Data Layering" makes exactly this point — layering first is fine for small systems and wrong for large ones.

The properties that make a module real, all of which Module 21 develops:

- **It owns its schema.** No other module reads its tables (Module 19, Concept 61). This is the boundary that actually holds.
- **It exposes a public contract and hides everything else** — `internal` plus an explicit public API surface.
- **Cross-module communication is by contract**: a published interface call, or an integration event. Never a direct entity reference, never a join.
- **It could be extracted into a service** without a redesign — which is the property that makes "modular monolith now, microservices maybe later" a real strategy rather than a slogan.

---

## Concept 58 — When Clean Architecture is the wrong default

The list that most distinguishes a senior answer, because it requires you to argue against the thing you were asked about:

1. **The domain is a thin veneer over CRUD.** Content management, admin tooling, reference-data services, most internal line-of-business forms. There is no policy to protect, so the layers protect nothing and cost everything.
2. **The system is short-lived or exploratory.** A prototype, a migration tool, a one-off pipeline. The benefit of deferred decisions requires a future in which decisions are made.
3. **The team is one or two people.** The "enforceable ownership" benefit needs multiple owners, and the indirection cost is felt immediately.
4. **The system's complexity is in the data, not the rules.** ETL, reporting, analytics, ML pipelines. Layers don't help; pipeline stages and schema contracts do.
5. **Functions/serverless with tiny units.** You still want a testable core, but four projects for a 200-line function is a cold-start and cognitive tax (Module 17).
6. **The "domain" is someone else's.** An integration service whose job is to translate between two systems is an adapter all the way down. Model it as pipelines and mappings, and spend your effort on the anti-corruption layer.
7. **You're writing a library.** A library's public API *is* its boundary. Internal layering is your business; ports and adapters don't apply.

And the corresponding statement of when it *is* right: **a long-lived system with real business rules, multiple entry points, a team big enough for ownership boundaries, and technology choices that you expect to outlive.** That is a specific and defensible set of conditions, and saying it is how you demonstrate you're choosing rather than defaulting.

---

## Concept 59 — Evolutionary architecture: buy the seams that are expensive later

The refinement of Concept 58, because "don't use it" isn't actionable when you're at the start of a system and don't yet know how big the domain will get.

Rank the seams by **cost to add later**:

| Seam | Cost to add later | Verdict |
|---|---|---|
| **A use-case boundary** (logic in a handler, not a controller) | Low-medium — mechanical extraction | Adopt from day one anyway; it's nearly free |
| **A port for an external vendor** | Low — the calls are already localized if they're in one adapter | Adopt from day one; vendors change |
| **A domain model** | Medium — logic is scattered across handlers but it's *your* code | Defer until rules justify it; promote per use case |
| **Module boundaries and schema ownership** | **Very high** — untangling shared tables is the hardest refactor in this list | Adopt early, even in a small system |
| **A separate persistence model** | High — touches every aggregate | Defer; start with Concept 29's compromises |
| **Splitting a project into two** | Low — move files, add a reference | Defer freely |

The strategy that falls out: **start with one project per module (not per layer), folders per feature, logic in handlers, ports for external vendors, and one schema per module.** Add layers *within* a module when the module earns them. This costs almost nothing on day one and preserves every expensive option.

The complementary discipline is a **fitness function** (Concept 67): whatever structure you choose, encode the invariant you care about as an automated check the day you decide it, because architectural decisions that aren't checked decay at a rate proportional to team size.

---

## Concept 60 — The "we could swap the database" fiction

Address this head-on, because a lot of candidates offer it as their headline benefit and good interviewers treat it as a tell.

The claim: because the domain talks to `IOrderRepository`, we could replace SQL Server with Cosmos DB / MongoDB / anything by writing a new adapter.

Why it's usually false:

- **The abstraction encodes assumptions of the original store.** Transactions, joins, unique constraints, optimistic concurrency, ordering, read-your-writes. Swapping to a store without them doesn't require a new adapter — it requires a new *design* (Modules 7, 8, 12 are about precisely this).
- **The read side never fitted through the seam anyway** (Concept 34), so half the data access was already store-specific.
- **Migration is the hard part, and the abstraction doesn't help.** Moving 400 million rows with zero downtime is a data-engineering project (Module 19, Part F). The repository interface contributes nothing to it.
- **Empirically, teams don't.** Providers get swapped at the *start* of a project, or during a rewrite, and almost never in between.

What *is* true, and what you should say instead:

- **You can swap the smaller edges, and you do.** Mail providers, SMS, payment gateways, blob stores, feature-flag services, search — these change every few years, and the adapter genuinely absorbs it.
- **You can defer the decision,** which is different from reversing it. Building for three months with an in-memory repository while procurement argues about the database is a real benefit that teams really collect.
- **You can test without the store,** which is the benefit you're actually cashing every day (Concept 10).
- **You can run two implementations simultaneously**, which is the underrated one: dual-write during a migration, a cache-backed implementation alongside the real one, a stub for a load test, a recording adapter for contract tests.

The phrasing that lands: *"I don't build the persistence port so we can switch databases — we won't. I build it so the use cases are testable in milliseconds and so the aggregate boundary is enforced. The vendor-swap benefit is real for the payment gateway, not for Postgres."*

---

# Part F — Enforcement, testing, and evolution

---

## Concept 61 — The enforcement ladder

Architecture that isn't mechanically enforced is a preference with a diagram. Five rungs, cheapest and fastest feedback first:

| Rung | Mechanism | Feedback | Catches |
|---|---|---|---|
| 1 | **Project references** | Instant (compiler) | Layer violations across assemblies |
| 2 | **`internal` + `InternalsVisibleTo`** | Instant | Reaching into another assembly's implementation |
| 3 | **Analyzers** (`BannedApiAnalyzers`, custom Roslyn) | Instant (as you type) | Forbidden types/members inside a layer |
| 4 | **Architecture tests** | Minutes (CI / test run) | Namespace rules, naming, sealing, per-type dependency rules |
| 5 | **Code review + ADRs** | Hours-days | Judgment: is this boundary the right one at all |

Rules of thumb:

- **Push each rule as far up as it will go.** If the compiler can enforce it, don't write a test for it. If an analyzer can, don't rely on review.
- **Rungs 1–3 are almost free and almost nobody uses 3.** Concept 24's banned-symbols file is the highest-leverage twenty minutes in this module.
- **Rung 5 doesn't scale but can't be skipped**, because it's the only rung that can evaluate whether a *new* boundary is wise.

---

## Concept 62 — Architecture tests: what to assert, and how they lie

The tooling as of September 2026 (Orientation): `NetArchTest.Rules` is frozen at 1.3.2 (2021) and still works; `NetArchTest.eNhancedEdition` (MIT, 1.4.5, June 2025) is the maintained fork with a near-identical API; `ArchUnitNET` is the actively developed .NET port of Java's ArchUnit, with a richer fluent model.

```csharp
public class ArchitectureTests
{
    private static readonly Assembly Domain      = typeof(Order).Assembly;
    private static readonly Assembly Application = typeof(PlaceOrder).Assembly;
    private static readonly Assembly Infra       = typeof(AppDbContext).Assembly;
    private static readonly Assembly Web         = typeof(Program).Assembly;

    [Fact]
    public void Domain_has_no_dependency_on_anything()
    {
        var result = Types.InAssembly(Domain)
            .ShouldNot()
            .HaveDependencyOnAny("Company.Product.Application",
                                 "Company.Product.Infrastructure",
                                 "Company.Product.Web",
                                 "Microsoft.EntityFrameworkCore",
                                 "Microsoft.AspNetCore")
            .GetResult();

        Assert.True(result.IsSuccessful, Explain(result));
    }

    [Fact]
    public void Application_does_not_know_about_infrastructure_or_the_web()
    {
        var result = Types.InAssembly(Application)
            .ShouldNot()
            .HaveDependencyOnAny("Company.Product.Infrastructure", "Company.Product.Web")
            .GetResult();

        Assert.True(result.IsSuccessful, Explain(result));
    }

    [Fact]
    public void Only_the_composition_root_may_reference_infrastructure()
    {
        var result = Types.InAssembly(Web)
            .That().DoNotHaveName("Program").And().DoNotResideInNamespace("Company.Product.Web.Configuration")
            .ShouldNot().HaveDependencyOn("Company.Product.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful, Explain(result));
    }

    [Fact]
    public void Handlers_are_internal_and_sealed()
    {
        var result = Types.InAssembly(Application)
            .That().ImplementInterface(typeof(ICommandHandler<,>))
            .Should().BeSealed().And().NotBePublic()
            .GetResult();

        Assert.True(result.IsSuccessful, Explain(result));
    }

    private static string Explain(TestResult r) =>
        r.IsSuccessful ? "" : "Violations:\n" + string.Join('\n', r.FailingTypeNames ?? []);
}
```

The rules worth writing **first**, in order:

1. Domain depends on nothing (including packages).
2. Application depends on Domain only.
3. Nothing outside the composition root references Infrastructure.
4. No project references Web.
5. Modules don't reference each other's internals (once you have modules).
6. Naming/sealing conventions that other tooling assumes.

**How these tests lie** — the part that makes the answer senior:

- **They only see compiled metadata.** If a type is referenced only inside a method body and the compiler inlines or erases the reference, some tools miss it. Coverage is good but not total.
- **A passing suite proves the arrows point the right way, not that the boundaries are meaningful.** `IApplicationDbContext` passes every rule above while defeating the architecture — unless you add `Microsoft.EntityFrameworkCore` to the banned list, which is exactly why that entry matters.
- **String-based rules rot silently.** Rename a namespace and a rule can quietly match nothing and pass forever. Defend against it: assert that the rule *found types to evaluate* (`Types.InAssembly(x).That()…` returning an empty set should fail the test).
- **They run late.** CI, not the IDE. That's why rung 3 exists.

---

## Concept 63 — Banned-API analyzers as architecture

Covered mechanically in Concept 24; the architectural framing is worth its own concept because it's the enforcement most candidates have never used.

An architecture test says "Application must not depend on EF Core." A banned-symbol analyzer says the same thing **while you're typing the `using`**, with a message you wrote explaining why. The difference in a team of fifteen is the difference between a rule that's understood and a rule that's resented.

The set worth shipping on day one:

- **Domain:** ban `DateTime.Now`/`UtcNow`, `DateTimeOffset.UtcNow`, `Guid.NewGuid`, `System.Random`, `Task.Run`, and any ORM/web namespace.
- **Application:** ban `Microsoft.EntityFrameworkCore`, `Microsoft.AspNetCore.Http`, `System.Net.Http.HttpClient`, `System.Data.SqlClient`/`Microsoft.Data.SqlClient`.
- **Everywhere:** ban whatever your team has agreed is a footgun — `Thread.Sleep`, `.Result`, `.Wait()` (Module 15), `Assembly.Load`, `DateTime.Parse` without a culture.

Each entry carries a message, and the message is documentation delivered at exactly the moment it's needed: `N:Microsoft.EntityFrameworkCore;The application layer talks to ports, not to an ORM. See ADR-0007.`

---

## Concept 64 — The testing pyramid this architecture buys you

This is the concrete payoff, and you should be able to describe all four tiers with numbers:

| Tier | What it tests | Doubles | Typical count | Speed |
|---|---|---|---|---|
| **Domain unit tests** | Invariants, state transitions, calculations | **None.** Construct objects, call methods, assert | Thousands | Microseconds each; whole tier in < 1s |
| **Use-case tests** | Orchestration, authorization, error paths | In-memory **fakes** for ports | Hundreds | Milliseconds |
| **Adapter integration tests** | Repositories, queries, migrations, HTTP clients | Real dependencies via **Testcontainers**; WireMock for HTTP | Dozens–low hundreds | Seconds |
| **End-to-end / API tests** | Wiring, serialization, auth, status codes | Real app via `WebApplicationFactory`, real database | Tens | Seconds–minutes |

Three properties to point out when explaining it:

1. **The bottom tier needs no doubles at all.** That's the tell that your domain is actually pure. If domain tests need mocks, the domain is doing I/O.
2. **The second tier uses fakes, not mocks** (Concept 65). Tests assert on outcomes — what's in the fake repository afterwards, what event was raised — not on which methods were called.
3. **The third tier is where truth lives** for anything involving the database (Module 19, Concept 70: InMemory lies; SQLite half-lies; Testcontainers tells the truth).

The trap to name: a layered architecture makes it *easy* to write a fourth kind of test — the **wiring test**, which mocks every dependency of a handler and asserts that the handler called them. It tests that the code is the code. It breaks on every refactor and catches nothing. Vertical-slice advocates cite these as the characteristic disease of layered codebases, and they're right that it's common; the fix is fakes and outcome assertions, not abandoning the layers.

---

## Concept 65 — Fakes beat mocks at architectural boundaries

```csharp
internal sealed class FakeOrderRepository : IOrderRepository
{
    private readonly Dictionary<OrderId, Order> _orders = [];

    public Task<Order?> FindAsync(OrderId id, CancellationToken ct) =>
        Task.FromResult(_orders.GetValueOrDefault(id));

    public Task AddAsync(Order order, CancellationToken ct)
    {
        if (!_orders.TryAdd(order.Id, order))
            throw new InvalidOperationException("Duplicate order id.");   // behaves like the real constraint
        return Task.CompletedTask;
    }

    public void Remove(Order order) => _orders.Remove(order.Id);
    public IReadOnlyCollection<Order> All => _orders.Values;              // for assertions
}
```

Thirty lines, written once, used by every use-case test. Why it beats `Mock<IOrderRepository>`:

- **It has behaviour**, so it can catch you doing something wrong (adding twice, removing something absent). A mock returns what you told it to and therefore can only confirm your assumptions.
- **Tests assert on state, not interactions.** `Assert.Single(repo.All)` survives refactoring; `mock.Verify(r => r.AddAsync(It.IsAny<Order>(), default), Times.Once)` does not.
- **It documents the port's contract**, which is useful when someone writes the second implementation.
- **"Don't mock what you don't own"** applies with full force to types like `DbSet<T>`, `HttpClient`, and cloud SDK clients — which is another reason those shouldn't be your ports.

Where mocks still earn their place: verifying that something *was not* called, asserting on interactions that genuinely are the specification ("the gateway must be called exactly once, even on retry"), and stubbing an outcome you can't otherwise produce.

---

## Concept 66 — Testing the composition root

The layer people forget to test, and it's where the most embarrassing production failures come from — everything compiles, every test passes, and the app throws on startup because a registration is missing.

```csharp
[Fact]
public void Container_resolves_every_registered_service()
{
    using var factory = new WebApplicationFactory<Program>()
        .WithWebHostBuilder(b => b.UseSetting("ConnectionStrings:Default", "Server=(local);Database=none"));

    // ValidateOnBuild/ValidateScopes are on, so Build() itself is the assertion.
    using var scope = factory.Services.CreateScope();

    foreach (var handlerType in typeof(PlaceOrder).Assembly.GetTypes()
                 .Where(t => t is { IsAbstract: false, IsInterface: false } &&
                             t.GetInterfaces().Any(i => i.IsGenericType &&
                                  i.GetGenericTypeDefinition() == typeof(ICommandHandler<,>))))
    {
        Assert.NotNull(scope.ServiceProvider.GetService(handlerType));
    }
}
```

What this catches: unregistered handlers, missing options bindings, captive dependencies (with `ValidateScopes`), and unresolvable graphs (with `ValidateOnBuild`) — the four failure modes from Module 18's Part E, caught in CI rather than at 3 a.m.

`WebApplicationFactory` is also where the architecture's end-to-end claim gets verified: one test that posts a request, hits a real database in a container, and asserts the row exists proves that every boundary you built actually connects.

---

## Concept 67 — Fitness functions and drift

A fitness function is any automated check on an architectural characteristic, run continuously, with a number or a pass/fail. Architecture tests are one kind; the useful set is broader:

- **Dependency direction** — the arch tests above.
- **Forbidden-API usage** — banned symbols, per layer.
- **Package surface** — a CI check that Domain's transitive package closure is empty (or on an allow-list). This catches the transitive case arch tests miss.
- **Cyclomatic/coupling metrics** — efferent/afferent coupling per project, tracked over time (NDepend if you have it; `dotnet` analyzers and simple `dotnet list package --include-transitive` parsing if you don't).
- **Test-tier ratios** — number of tests requiring Docker vs. total. If the fast tier stops growing while the slow tier does, your domain is leaking into infrastructure.
- **Build and test wall-clock time** — a leading indicator of both structure and developer experience.
- **Schema ownership** — a check that no module's migrations touch another module's tables (Module 21).

The operational point: **a fitness function without an owner and a trend is theatre.** Put them in the same pipeline as the tests, fail the build on the hard ones, and chart the soft ones somewhere a human sees monthly.

---

## Concept 68 — Introducing this into a brownfield system

The scenario every architect interview eventually reaches: a large, working, profitable application whose logic is in controllers, static helpers and stored procedures. The wrong answer is "rewrite." The right answer is an ordered, incremental plan where every step ships.

**Order of operations, each step independently valuable:**

1. **Add the harness before touching anything.** Characterization tests at the API level for the highest-value flows; a container-backed database; a CI pipeline that runs them. You cannot refactor what you cannot verify. This is often 20–30% of the total effort and it's the step people skip.
2. **Create the seam, not the layers.** Introduce a use-case class for *one* endpoint: move the controller's body into `PlaceOrderHandler`, leave everything else alone. The controller becomes three lines. Repeat opportunistically — every time someone touches an endpoint, it gets extracted (the "boy scout" version of a strangler fig).
3. **Invert one volatile dependency.** Pick the vendor integration that hurts most. Define the port in application terms, wrap the existing calls, write the fake, write the tests. This delivers immediate value (testability) and proves the pattern to the team.
4. **Pull rules into a domain model where the rules actually are.** Not everywhere — find the entity with the most bugs and the most `if` statements scattered across handlers, and give it behaviour and invariants. One aggregate at a time.
5. **Establish schema ownership.** Draw the module boundaries on the *tables*, find the cross-boundary reads, and replace them with contracts. This is the slowest and most valuable step, and it's the one that makes later extraction possible.
6. **Install the enforcement** (Concepts 61–63) the moment a boundary exists, so it doesn't erode while you work on the next one.
7. **Strangle the remainder.** New features go in the new structure; old code gets migrated when touched; a sunset date for the legacy path with a tracked percentage.

**What to say about stored procedures specifically**, because .NET brownfield interviews always have them: don't rewrite them wholesale. Classify them — reporting (leave), CRUD (replace when touched), and business logic (this is the one to extract, because it's the logic nobody can test). Wrap each one behind a port first, so the call site is stable, then reimplement behind the port with a characterization test comparing outputs.

**Metrics to commit to**, because an architect who proposes a twelve-month programme without numbers gets no budget: percentage of endpoints with a use-case seam, percentage of business logic covered by fast tests, mean time to add a field, incident count in the touched areas, and build time.

---

## Concept 69 — Collapsing layers is a senior move

Removal deserves as much thought as addition, and proposing it in an interview is strong signal because it demonstrates you're evaluating rather than defending.

Signals a layer isn't paying:

- Every type in it is a pass-through: `Service.Do()` calls `Repository.Do()` with no decision in between.
- Its interfaces have exactly one implementation and exactly one caller, and no fakes exist for them in tests.
- Changes to features always touch it, but nobody has ever changed it *for its own reasons*.
- Nobody can name the thing it protects you from.

How to collapse safely:

1. **Prove the abstraction is unused as an abstraction** — one implementation, no test doubles, no second consumer.
2. **Inline mechanically.** Modern IDE refactors do most of it; the arch tests and compiler catch the rest.
3. **Keep the *use case* seam even if you remove the layer.** Merging Application into Domain is usually fine; deleting the handler and putting logic back in the controller usually isn't, because the handler is what makes multiple entry points possible.
4. **Write the ADR** recording why it was removed, so the next engineer doesn't re-add it from a blog post.

---

## Concept 70 — The performance bill, measured

Module 17's discipline applied to this module's decisions. The honest numbers:

| Cost | Magnitude | When it matters |
|---|---|---|
| **DI resolution** per request | Microseconds for a typical graph | Never, unless you're resolving thousands of transients per request |
| **An extra interface call** | A virtual dispatch; often devirtualized by the JIT on `sealed` types | Never at web-request scale; possibly in a tight loop |
| **Mapping allocations** | One object per mapped instance, per boundary | **Yes**, at high throughput: 3 boundaries × 1,000 items = 3,000 objects per request, straight into Gen 0/1 pressure (Module 14) |
| **Reflection-based mapping** | Startup cost + per-map overhead + no AOT | Yes — this is the one to replace with source generation |
| **Mediator dispatch** | Dictionary lookup + pipeline delegates | Negligible per call; the pipeline's *work* is what costs |
| **Loading whole aggregates** | Extra rows, extra materialization, extra tracking | **Yes** — this is the real one (Module 19, Concepts 16–20) |
| **Extra assemblies** | Load time, JIT | Measurable in cold start (Module 17, Concept 54); irrelevant when warm |

The rule: **the architecture's overhead is dominated by the data access decisions it encourages, not by the indirection itself.** If a layered app is slow, it is almost never because of the layers — it is because the repository loads full aggregates to render a list, which is Concept 34's problem and has an architectural fix, not a tuning one.

The measurement habit to state: benchmark the *handler*, not the endpoint, when you're evaluating an architectural cost; and profile allocations at the boundaries where you map, because that's where the garbage is.

---

## Concept 71 — The human metrics

Architecture is a claim about how a team will experience a codebase, so measure that:

- **Files touched per typical change.** Sample the last 20 feature PRs. If a one-field change touches nine files, say so out loud and decide whether it's worth it.
- **PR diff size distribution.** Bimodal (tiny + huge) usually means the structure forces ceremony for small work and people batch to avoid it.
- **Time to first meaningful PR** for a new hire. The single best measure of whether the structure is comprehensible rather than merely correct.
- **"Where do I put this?" frequency.** If experienced team members still ask, the structure isn't communicating. Usually a naming problem, not a layering one.
- **Build and test time.** Directly traded against project count.
- **Where reviewers look.** If answering "what happens when an order is cancelled?" requires opening five files, your domain logic isn't cohesive regardless of how clean the arrows are.

Bringing a number to this conversation is what separates an architect from someone with opinions about folders.

---

## Concept 72 — The wrong abstraction, and how to reverse it

Sandi Metz's rule — **"duplication is far cheaper than the wrong abstraction"** — is the most useful single sentence about the failure mode of this whole module, because Clean Architecture's tooling makes creating abstractions easy and removing them socially difficult.

The mechanism of the failure: someone notices duplication, extracts an abstraction, and it fits. Requirements change; the abstraction *almost* fits, so a parameter gets added. Repeat four times. Now you have a method with six boolean parameters that nobody understands and everybody is afraid of, and the original duplication would have been three independent pieces of code that each did one thing.

How to reverse one, in order:

1. **Inline it back** into each call site. Yes, you now have duplication. That's the point — you've recovered the information about what each caller actually needs.
2. **Delete the parameters each caller doesn't use.** The copies diverge, and the divergence is the requirement you'd been hiding.
3. **Re-extract only what's genuinely identical**, if anything is.

The prophylactic: **wait for the third occurrence**, and prefer a *name* to a *parameter*. Two things that look alike but change for different reasons (Module 20's whole theme) are not duplication — they're coincidence, and coupling them is how you get a change to invoicing breaking checkout.

---

# Part G — The architect's view

---

## Concept 73 — How to talk about it

The vocabulary shift that changes how an interview goes:

| Don't say | Say |
|---|---|
| "We use Clean Architecture" | "The domain has no outward dependencies; here's which boundaries I chose to build and why" |
| "For separation of concerns" | "So that a vendor change touches one adapter and no policy" |
| "It's more maintainable" | "A one-field change touches three files; a payment-provider change touches one project" |
| "We follow SOLID" | "The pricing rules have one reason to change, which is why they're not in the handler" |
| "Best practice" | "The trade I'm making is X for Y, and here's when I'd make the other one" |
| "We abstracted the database" | "The write side goes through aggregate repositories; the read side talks SQL on purpose" |

The structure of a good answer to any architecture question in this space is four moves, in order: **what changes (volatility) → who owns it (organization) → what it costs (indirection, ceremony, latency) → how it's enforced (compiler, analyzer, test).** Do those four and the interviewer stops probing, because you've covered the ground they were going to probe.

---

## Concept 74 — The four questions before drawing a boundary

Use these live, out loud, while designing on a whiteboard. They make your reasoning visible, which is most of what's being scored.

1. **What do I expect to change here, and how often?** If the answer is "nothing, ever," don't build a seam. If it's "the vendor, every two years," build one.
2. **Who owns each side?** A boundary that matches an ownership line becomes real because someone defends it. One that cuts across a team's work becomes a formality they route around.
3. **What test does this boundary make possible?** If you can't name a test that gets faster, more deterministic, or possible-at-all, you're buying indirection with no receipt.
4. **What does it cost if I'm wrong in each direction?** Building a seam you didn't need: some indirection, deletable later. *Not* building one you needed: a change that touches 200 files. Asymmetric costs justify asymmetric caution — which is exactly why Concept 59 ranks seams by cost-to-add-later.

---

## Concept 75 — Critiquing a template, fairly and currently

A frequent exercise: "here's our solution — it's based on a well-known template. What would you change?" The skill is to be specific, fair, and current, and to separate the template's fault from the team's.

**Things a template can't decide for you, so don't blame it:**
- Whether your domain deserves a domain model (Concept 55).
- Whether you needed four projects (Concept 22).
- Whether your aggregates are the right size.

**Legitimate, specific critiques of the common .NET Clean Architecture setup:**

1. **`IApplicationDbContext` isn't a boundary** — Application references EF Core (Concept 31). Defensible as a velocity trade; not defensible as "persistence-ignorant."
2. **MediatR and AutoMapper are now licensing decisions** (Orientation). The jasontaylordev template deliberately keeps both and surfaces the licence warning; you need a position on that before it reaches procurement.
3. **`IDateTime` (or equivalent) is obsolete** — `TimeProvider` has been in the BCL since .NET 8 (Concept 42).
4. **Generic repositories, where present, do nothing** — `IRepository<T>` over `DbSet<T>` is a rename (Module 19, Concept 66).
5. **Reads and writes share a path** that only the write side needs (Concept 34).
6. **Folder-by-pattern inside Application** — `Commands/`, `Queries/`, `Behaviours/` — rather than folder-by-feature (Concept 9). Both templates have largely fixed this; older codebases based on them haven't.
7. **The four projects are a starting point, not a target state.** A system with three modules wants three modules, not four layers (Concept 57).

**The fair counter-argument to hold simultaneously:** templates are teaching artifacts and shared vocabulary. Their value is that a new engineer recognizes the shape instantly, and that value is real. The critique is not "don't use it" — it's "know which parts are decisions and which are defaults, and re-decide the decisions."

---

## Concept 76 — Layers are not tiers

A precise distinction that catches people out, and it's a legacy of N-tier vocabulary that .NET shops carry:

- **Layer** = a logical grouping of code. Compile-time. Free.
- **Tier** = a physical deployment boundary — a separate process, container, or machine. Runtime. Expensive.

Clean Architecture is entirely about layers. It says nothing about processes. A four-layer application usually deploys as **one process**, and should.

Why it matters:

- Every tier adds a network hop (hundreds of microseconds to milliseconds — Module 5's latency numbers), a serialization boundary, a partial-failure mode (Module 13), and a deployment coordination problem.
- The classic 2000s enterprise mistake was deploying "the business layer" on its own tier for "scalability" and receiving only latency (Module 6: you scale a *bottleneck*, not a diagram).
- The classic 2020s version is one microservice per layer — an "API service" calling a "domain service" calling a "data service." That's a distributed monolith with extra hops, and it's a fast way to fail a design round.

The line to have ready: *"Layers are compile-time; tiers are runtime. I'd deploy this as one process until there's a scaling, isolation, or ownership reason to split it — and then I'd split along a module boundary, not a layer boundary."*

---

## Concept 77 — Conway's law and boundaries that survive

Boundaries erode unless someone has an interest in defending them. The interest usually comes from ownership.

- A module boundary that matches a team boundary becomes real, because cross-boundary changes require a conversation — which is friction, and friction is enforcement.
- A boundary inside one team's code survives only as long as the tooling enforces it (Concepts 61–63) and the team keeps caring.
- **The inverse manoeuvre (the "inverse Conway") is a legitimate architect tool**: choose the boundaries you want, then organize teams to match. Just know that you're proposing an org change, and say so, because the failure mode is proposing an architecture that the current org will route around within two quarters.

The practical question to ask in an interview when given an architecture problem: **"how many teams, and who owns what?"** It's a senior question because it recognizes that the diagram is downstream of the org chart.

---

## Concept 78 — Documenting it so it survives

Three artifacts, each with a different job (Module 31 covers ADRs and C4 properly):

1. **ADRs** — one short document per decision: context, decision, consequences, status. The critical property is that they capture *why*, including the options rejected. "We use `IApplicationDbContext` instead of repositories because our domain is mostly CRUD and we want EF's expressiveness in handlers; we accept that Application references EF Core" is one paragraph that prevents a year of re-litigation.
2. **A C4 component diagram** for the one or two containers that matter. Not a class diagram — the useful level is components and their dependencies.
3. **A generated dependency diagram.** Hand-drawn diagrams are aspirational; generated ones are true. `dotnet list reference`, an arch-test that emits the graph, or a tool that renders the `.csproj` graph into the README, refreshed by CI.

The rule that makes documentation survive: **if it isn't generated or executable, assume it's wrong within six months.** Which is the argument for architecture tests as the primary documentation and prose as commentary on it.

---

## Concept 79 — The cost conversation

Translating structure into language a non-engineering stakeholder acts on — the skill that most distinguishes the architect track (Module 33 goes deeper):

| Engineering statement | Executive translation |
|---|---|
| "The domain has no infrastructure dependencies" | "We can verify business rules in seconds, so regression risk on rule changes is low" |
| "The payment provider is behind an adapter" | "Switching providers is a two-week project, not a two-quarter one" |
| "We'd need a separate persistence model" | "Three engineer-months up front and ~10% ongoing overhead, in exchange for schema independence" |
| "The modules own their schemas" | "We can extract or replace a capability without a rewrite" |
| "This is a distributed monolith" | "Every release requires coordinating four teams; that's why lead time is six weeks" |
| "We should collapse this layer" | "We can remove a step from every change; expect a measurable drop in cycle time" |

Two habits: **always give a range and a unit** (engineer-weeks, percentage overhead, incidents per quarter), and **always name the option you're not taking** — executives trust people who show them the rejected alternative.

---

## Concept 80 — The anti-pattern catalogue reviewers look for

The twelve things that get flagged in a real code review of a "Clean Architecture" .NET solution:

1. **Domain references a framework** — EF attributes, `DbContext`, `IOptions<T>`, JSON attributes, `HttpClient`.
2. **`IQueryable` escaping a repository** (Concept 32).
3. **Anemic entities**: public setters everywhere, all logic in "services" (Concept 54).
4. **Generic `IRepository<T>`** over `DbSet<T>` with no aggregate shaping.
5. **`SaveChanges` inside repositories**, so no multi-aggregate operation can be atomic.
6. **Business conditionals in controllers/endpoints** — a rule enforced at exactly one entry point.
7. **Registration code in the wrong layer** — `AddDbContext` inside `AddApplication()`.
8. **A service locator**: `IServiceProvider` injected outside the composition root.
9. **Interfaces with one implementation, one caller, and no fake** (Concept 48).
10. **HTTP concepts inside the domain** — status codes, `ActionResult`, `[FromBody]`, `ProblemDetails`.
11. **Mapping in both directions in the inner layer** — `Order.ToDto()` on the entity.
12. **`DateTime.UtcNow` anywhere inside Domain or Application** (Concept 42).

Bonus tells: `Thread.Sleep` in a handler; `.Result`/`.Wait()` (Module 15); `Database.Migrate()` in `Program.cs` (Module 19); tests that mock `DbSet<T>`; a `Common`/`Shared` project containing seventeen unrelated things.

---

## Concept 81 — What you'd actually start with

Have concrete defaults ready, because "it depends" without a default reads as evasion.

**2–3 engineers, new product, uncertain domain**
One project (plus tests). Folders per feature. Handlers for logic, called directly from minimal-API endpoints. EF Core directly in handlers. Ports only for external vendors. `TimeProvider`. Banned-symbols file from day one. Schema owned by this service. *Rationale: the expensive-later seams (use case, vendor ports, schema ownership) are in place; nothing else is.*

**~8 engineers, established product, real business rules**
Three or four projects: Domain, Application, Infrastructure, Web. Folder-per-use-case inside Application. Aggregate repositories on the write side, query services on the read side. A small set of ports. Architecture tests + banned symbols in CI. ADRs for the decisions that will be questioned. *Rationale: enough people that ownership and enforcement matter; enough rules that a domain model pays.*

**~40 engineers, multiple capabilities, one deployable (for now)**
Modular monolith: one folder and project set per module, each internally layered, each owning its schema, with public contracts and integration events between them (Module 21). One host, one composition root, per-module registration. Arch tests enforcing module isolation. *Rationale: the boundary that matters at this size is the capability, not the layer — and this is the shape that makes later extraction a deployment change rather than a rewrite.*

In all three: **the use-case seam and schema ownership are non-negotiable; everything else is a dial.**

---

## Concept 82 — The one-minute answer

When an interviewer says "tell me about Clean Architecture," this is the compressed version — a position, not a definition:

> "It's one rule: source-code dependencies point inward, toward policy. The mechanism is that the inner layer declares the interfaces it needs and the outer layer implements them, so control flows outward while references point inward. Hexagonal, onion and clean are the same idea with different emphases — ports and adapters is the most precise vocabulary, because the driving/driven distinction tells you which seams actually need inversion.
>
> What I actually buy with it is test speed and deterministic tests for business rules, plus the ability to replace volatile edges — payment providers, mail, search — without touching policy. What I don't buy is swapping the database; almost nobody cashes that. What it costs is indirection, mapping, and a higher floor on small changes, and those are paid daily whereas the benefits are paid on events — which is why I apply it where the events are likely, not everywhere.
>
> In .NET I'd enforce it with the project graph, `internal` types, a banned-symbols analyzer per layer, and a handful of architecture tests — because an architecture that isn't checked by a machine is a preference. And I'd organize the application layer by feature rather than by pattern, which gets most of what vertical slices offer while keeping the dependency rule. The parts I'd argue about in any given system are whether the domain model is warranted, and whether reads should bypass it — my default is that they should."

---

# Putting it together

---

## Worked example 1 — "Design the structure for a new order service."

**Start with questions, not a diagram.** How many engineers? How long-lived? Are the rules complex or is it mostly workflow? How many entry points — API only, or API plus worker plus admin? Who owns the database? Is there a second team?

**Assume the answers:** six engineers, multi-year product, real rules (pricing, holds, cancellation windows, partial refunds), three entry points (REST API, message consumer, nightly job), one team, own schema.

**The narration** (this is what the interviewer is scoring):

> "Three entry points means the use case has to be the unit, not the controller — that's the seam I want most, because otherwise the cancellation rules get enforced in the API and forgotten in the consumer. Real pricing and refund rules means a domain model earns its keep for those aggregates, though I'd expect half the use cases to stay procedural.
>
> So: `Domain` with `Order` and `Refund` as aggregates, `Money` and `OrderId` as value objects, and domain events for the facts other parts care about. `Application` with a folder per use case — `Orders/PlaceOrder`, `Orders/CancelOrder`, `Orders/GetOrderSummary` — declaring the ports it needs: `IOrderRepository`, `IPaymentGateway`, `IOrderQueries`, `IUnitOfWork`. `Infrastructure` with EF Core, the entity configurations, the Stripe adapter, the outbox. Then three hosts, or one host with three adapters: minimal-API endpoints, a Service Bus consumer, and a `BackgroundService` — all three thin, all three calling the same handlers.
>
> Two deliberate splits. **Writes** go through aggregate repositories so the invariants can't be bypassed. **Reads** go through `IOrderQueries` implemented with Dapper, because the order-list screen wants a flat projection across four tables and forcing that through the aggregate would be both slower and less expressive.
>
> For consistency: one transaction per use case; domain events dispatch inside it; anything leaving the process becomes an outbox row in the same transaction, published by a worker with at-least-once delivery and idempotent consumers.
>
> Enforcement: Domain has no package references and a banned-symbols file that blocks `DateTime.UtcNow` and `Guid.NewGuid`; Application's banned list blocks `Microsoft.EntityFrameworkCore`; four architecture tests in CI; container validation at startup.
>
> What I'm *not* doing: no separate persistence model — the schema is ours and EF maps these aggregates fine with private constructors and field-mapped collections; no mediator unless we want a uniform pipeline, and if we do, I'd price the MediatR licence or write the 30-line dispatcher; no repository on the read side."

**What makes that answer senior:** every structural choice has a stated reason, two choices are explicitly declined with reasons, the enforcement mechanism is named, and the consistency story is complete.

---

## Worked example 2 — "Review this solution."

You're shown a solution with `Domain`, `Application`, `Infrastructure`, `Web` and asked what you'd change. Here's a realistic set of findings and how to phrase them — ordered by severity, not by how clever they make you look.

```
Domain/
  Entities/Order.cs           public class Order { public int Id {get;set;} public string Status {get;set;} … }
  Entities/OrderLine.cs
Application/
  Interfaces/IApplicationDbContext.cs
  Interfaces/IDateTime.cs
  Common/Behaviours/ValidationBehaviour.cs
  Orders/Commands/CreateOrder/CreateOrderCommand.cs
  Orders/Commands/CreateOrder/CreateOrderCommandHandler.cs
  Orders/Queries/GetOrders/GetOrdersQuery.cs
Infrastructure/
  Persistence/ApplicationDbContext.cs
  Repositories/Repository.cs  public class Repository<T> : IRepository<T> { IQueryable<T> Query() => _ctx.Set<T>(); }
Web/
  Controllers/OrdersController.cs
```

1. **`Order` has no behaviour** — public setters on `Status` mean the cancellation rules live somewhere else, probably in two places. *"This is an anemic model: we're paying for a Domain project and a mapping layer and getting none of the invariant protection. I'd start by making `Status` private-set and moving the transition rules onto the entity."* (Concept 54.) **Highest severity: this is the one that causes bugs.**
2. **`Repository<T>.Query()` returns `IQueryable`** — *"This exports EF's semantics into every caller, and it means no aggregate boundary is enforced. I'd replace it with aggregate-shaped methods on the write side and a query service on the read side."* (Concepts 27, 32, 34.)
3. **`IApplicationDbContext` in Application** — *"Application references EF Core, so the dependency rule is broken at the package level. That can be a deliberate trade, but the ADR should say so, and we shouldn't describe the domain as persistence-ignorant."* (Concept 31.)
4. **`IDateTime`** — *"Delete it; `TimeProvider` has been in the BCL since .NET 8, with `FakeTimeProvider` for tests."* (Concept 42.)
5. **`int Id` with database-generated identity** — *"This means we can't raise domain events referencing the order until after `SaveChanges`. `Guid.CreateVersion7()` in the application gives us client-side IDs that still index well."* (Concept 42.)
6. **Folder-by-pattern** — `Commands/`, `Queries/` — *"Small thing, big daily payoff: folder per use case. It also makes the arch tests easier to write."* (Concept 9.)
7. **No enforcement** — *"Nothing here stops the next `using Microsoft.EntityFrameworkCore;` in Domain. Four arch tests and two banned-symbols files, half a day."* (Concepts 61–63.)

Then the fairness move, which is part of the score: *"To be clear, this structure is fine as a skeleton and the team will recognize it. Items 1 and 2 are the ones I'd fix this sprint; the rest I'd fix as we touch things."*

---

## Worked example 3 — "The domain needs a price from an external API mid-decision. Where does that go?"

A pure layering problem, and a good one because the naive answers are all slightly wrong.

**The wrong answers:**
- *Inject `IPricingApi` into the entity.* Now the domain does I/O, entities can't be constructed without a container, and every domain test needs a mock.
- *Make `Order.Place()` async.* Same problem with a different shape, and it signals that the entity is orchestrating.
- *Call the API from the repository.* The repository now has a hidden network dependency and unpredictable latency inside a transaction.

**The right shapes, in order of preference:**

1. **Fetch first, pass the data in.** The handler calls the port, gets a `PriceList` (a domain value object), and passes it to the domain method: `order.ApplyPricing(priceList, now)`. The domain stays pure and synchronous; the decision is still made by the domain. *This is Concept 56's functional core, and it's the answer 80% of the time.*
2. **Pass a domain service that's already loaded.** If the calculation needs lookups the handler can't know in advance, fetch a bounded set and wrap it: `PricingService` in Domain, constructed from data the handler fetched.
3. **Double dispatch** — pass a narrow, domain-owned interface into the method (`order.ApplyPricing(IPriceSource source)`) where `IPriceSource` is declared in Domain and implemented by a handler-side adapter over the port. Legitimate when the domain must decide *which* lookups to do. Costs: the domain method becomes async or the adapter must be pre-loaded; use sparingly.
4. **Split the use case.** If pricing is itself a business process with failure modes and retries, it's not a step inside `Place` — it's a use case that produces a `Quote`, which `Place` then consumes.

The sentence that wins the follow-up: *"The domain decides; the application supplies. If the domain has to fetch, I've usually drawn the aggregate boundary in the wrong place or the operation is really two use cases."*

---

## Worked example 4 — "We're adding a worker. What changes?"

**Answer: nothing in Domain or Application, and that's the whole point of the exercise.**

What actually changes:
- A new host project (or a new `BackgroundService` in the existing one) — a **driving adapter** (Concept 46).
- Its own composition root calling the same `AddApplication()` / `AddInfrastructure()`.
- A scope per unit of work from `IServiceScopeFactory` (Module 18), not one scope for the worker's lifetime (Module 19).
- Graceful shutdown wired to the `CancellationToken` so in-flight work finishes or aborts cleanly (Module 13).
- Deployment/observability: its own liveness and readiness signals, its own dashboards, its own scaling rules.

**The follow-up you should pre-empt:** *"If adding a second entry point required changing a use case, that use case had HTTP assumptions baked into it — and the most common form of that is authorization done in the controller rather than in the handler. That's the thing I'd check first."*

---

## Worked example 5 — "A one-line change touches six files. Defend it or fix it."

Don't defend it reflexively — measure it, then decide. The senior answer has three parts.

**1. Characterize the change.** Which files, and is each one carrying information?

Adding an optional `GiftMessage` to an order might touch: the entity, the EF configuration, a migration, the command, the validator, the handler, the request DTO, the response DTO, the mapper, the endpoint. Ten files. Of those:
- The entity + configuration + migration are irreducible — you're adding state to a persisted model.
- The command and the request DTO are the *same information twice* if the API isn't independently versioned.
- The response DTO and the mapper are the same information twice again.
- The validator and the endpoint are one line each.

**2. Fix the duplication, not the layering.**
- **Collapse the request DTO into the command** if the API isn't versioned separately (Concept 38). Two files gone.
- **Let the query project directly into the response shape** (Concept 34). Two more gone.
- **Consider one file per use case** so the command, validator and handler live together — same number of types, one file to open (Concept 25).

From ten files to five or six, of which three are irreducible. That's a legitimate answer, and it's better than either "layers are bad" or "that's the cost of clean code."

**3. Say what you'd do if it were still too expensive.** *"If most of our changes look like this — additive fields on CRUD-ish entities — then the domain model isn't earning its keep for those use cases, and I'd move them down the ladder to rung 1 or 2 and keep the domain model for the ones with rules."* (Concept 55.)

---

## Worked example 6 — "400k lines, logic in controllers and stored procedures, twelve months. Go."

Structure the answer as a programme with phases, each shipping value, each with a metric. (Concept 68 is the detail; this is the delivery.)

**Months 1–2 — Make change safe.** API-level characterization tests for the top 20 flows. Containerized database in CI. Pipeline that runs them on every PR. Baseline metrics: build time, test time, lead time for a typical change, incidents per month in the target areas.
*Deliverable: a safety net. Metric: % of revenue-critical flows under test.*

**Months 2–4 — Seams without layers.** Extract use-case handlers from controllers, starting with the endpoints that change most often (pull that from git history — it's the single best prioritization signal). Controllers become three lines. No new projects yet.
*Metric: % of endpoints with a use-case seam.*

**Months 3–6 — Invert the painful dependencies.** The vendor integration and the two or three stored procedures with real business logic. Port + adapter + fake + tests. Characterization tests comparing old and new outputs before switching.
*Metric: number of fast tests; number of flows testable without the mainframe/vendor sandbox.*

**Months 5–9 — Draw module boundaries on the schema.** Map tables to capabilities. Find cross-boundary reads (usually dozens). Replace the worst with contracts — a published interface or an integration event. This is the expensive, unglamorous, load-bearing work.
*Metric: number of cross-module table reads remaining.*

**Months 6–12 — Domain model where it pays.** One aggregate at a time, chosen by bug density. Pull the rules out of the procedures and handlers and into entities with invariants.
*Metric: defect rate in the touched areas; % of rules covered by millisecond tests.*

**Throughout — enforcement and communication.** Arch tests and banned symbols the moment a boundary exists. One ADR per significant decision. A monthly one-page update with the metrics, because a twelve-month architecture programme dies politically, not technically.

**The risks to name before you're asked:** the safety net is underestimated and it's the step everyone skips; module boundaries will cut across team boundaries and that's an org conversation (Concept 77); the team will want to rewrite rather than strangle; and feature delivery cannot stop, so every phase must be interleaved with product work at a ratio you've agreed in advance.

---

## Common questions and what a strong answer contains

**"What is Clean Architecture?"** One rule — source dependencies point inward toward policy — plus the mechanism that makes it possible: the inner layer declares the interface, the outer implements it, so control flows out while references point in. Then, immediately, what you use it for and what it costs. A definition alone reads as recall (Concepts 1, 4, 82).

**"What's the difference between Clean, Onion, and Hexagonal?"** Same rule, different emphases and different eras: BCE (1992) gave us the use case, Hexagonal (2005) gave us driving/driven symmetry and the testability argument, Onion (2008) was the .NET-specific rebuttal to data-access-as-the-bottom-layer, Clean (2012) generalized and named it. Seemann's "it's all the same" is the honest summary (Concept 7).

**"Does the dependency rule mean infrastructure can't call the domain?"** No — control flow crosses in both directions constantly. The rule is about *source-code* dependencies: which project references which. EF materializing a domain entity is infrastructure calling inward at runtime, and it's fine (Concepts 1, 4).

**"Why is the interface in the inner layer?"** Because ownership decides the dependency direction. If Infrastructure declared `IOrderRepository`, Application would reference Infrastructure and the graph inverts. The interface is shaped by the *needs of the caller*, which is why it's named in the caller's vocabulary (Concept 4).

**"Is DI the same as DIP?"** No. DIP is a design principle about who declares the abstraction; DI is a technique for supplying collaborators; IoC is a control-flow property. You can inject a concrete class (DI, no inversion) and you can invert with no container at all (Pure DI) (Concept 5).

**"Where does the composition root go, and why can Web reference Infrastructure?"** The composition root is the entry point — `Program.cs` plus registration extensions — and it's not a layer; it's the place where abstractions become concrete, so it's allowed to know everything. Constrain it with an architecture test that forbids any other type in Web from referencing Infrastructure (Concepts 19, 21).

**"Do I need a repository over EF Core?"** `DbSet` is already a repository and `DbContext` is already a unit of work, so the repository must justify itself on aggregate shaping and vocabulary: a small, named set of legal operations that returns materialized aggregates. If it's a generic `IRepository<T>` returning `IQueryable`, it's a rename (Concepts 27, 32; Module 19, Concept 66).

**"Can the domain know about EF Core?"** It shouldn't reference the package. In practice you accept three compromises — a private constructor, field-mapped collections, and shadow properties for persistence-only state — and configure everything else in Infrastructure with `IEntityTypeConfiguration`. Past that, a separate persistence model, which costs mapping in both directions *and* change tracking (Concepts 29, 30).

**"What's wrong with `IApplicationDbContext`?"** It's not a boundary: Application now references EF Core, `DbSet<T>` is `IQueryable<T>` with all that implies, and aggregate and transaction boundaries become advisory. It's a defensible velocity trade in a CRUD-ish system; it's not persistence ignorance (Concept 31).

**"Why not return `IQueryable` from repositories?"** It exports an unspecified provider contract, moves execution timing across the boundary, destroys the aggregate boundary, makes query performance unattributable, and — worst — lets in-memory fakes behave differently from the real provider, so your tests lie (Concept 32).

**"Where does validation go?"** Three kinds, three homes: shape at the edge (400), context in the handler using ports (404/409/403), invariants in the entity (refuse to produce an invalid object). The test is whether a different entry point would need the same check (Concept 39).

**"Exceptions or result types?"** Expected outcomes are values; unexpected ones are exceptions. The layering reason is that exceptions cross boundaries invisibly, so infrastructure exceptions must be translated by the adapter or your controller ends up catching `DbUpdateConcurrencyException` (Concepts 40, 44).

**"How do domain events differ from integration events?"** Domain events are in-process, inside the transaction, use domain types, and can roll back. Integration events are published contracts, flat and versioned, delivered at-least-once via the outbox, and consumed idempotently. Never publish a domain entity as an integration event (Concept 45; Module 11).

**"Where do background services go?"** They're driving adapters, same as controllers. The use case must be identical whether triggered by HTTP, a message, or a timer. One scope per unit of work, not per worker lifetime (Concept 46).

**"What would you *not* abstract?"** `ILogger`, the DI container, serialization, LINQ, `HttpClient` (replace it with a domain-shaped port instead of wrapping it), and framework abstractions that are already ports — `TimeProvider`, `HybridCache`, `IOptions<T>`. Heuristic: if you wouldn't write a fake for it, you don't need an interface for it (Concepts 48, 49).

**"Vertical slices or Clean Architecture?"** They're orthogonal. Layers decouple along the technology axis, slices along the change axis. The mature answer is both: dependency rule at the project level, feature folders inside Application, and complexity chosen per use case (Concepts 51–53, 55).

**"When would you not use Clean Architecture?"** Thin domain over CRUD, short-lived systems, one- or two-person teams, data-shaped complexity (ETL/reporting/ML), tiny serverless units, integration services that are adapters end-to-end, and libraries. Say which seams you'd still keep — the use-case seam, vendor ports, and schema ownership are cheap and expensive-to-retrofit (Concepts 58, 59).

**"Doesn't this let you swap the database?"** Rarely, and I wouldn't sell it that way: the abstraction encodes the original store's guarantees, the read side never fitted through it, and migration is the hard part anyway. What you actually get is test speed, deferred decisions, replaceable *edges* like payment and mail, and the ability to run two implementations during a migration (Concept 60).

**"How do you stop it eroding?"** Project references, `internal` types, banned-API analyzers with messages, a handful of architecture tests, and ADRs for the judgment calls. Push every rule to the fastest feedback loop that can enforce it — the analyzer catches it as you type; the arch test catches it in CI; review catches only what neither can (Concepts 61–63).

**"How do you test this?"** Four tiers: domain tests with no doubles at all; use-case tests with hand-written in-memory fakes, asserting on outcomes not interactions; adapter tests against real dependencies in Testcontainers; a thin end-to-end tier through `WebApplicationFactory`. Plus a container-validation test, since `ValidateOnBuild` and `ValidateScopes` turn wiring bugs into build failures (Concepts 64–66).

**"What does it cost at runtime?"** Almost nothing structural — DI resolution and virtual calls are microseconds. The real cost is data access the structure encourages: loading whole aggregates for list screens, and mapping allocations at every boundary. Fix the first with a read path that bypasses the domain; fix the second with source-generated or manual mapping (Concept 70).

**"What's changed in this space recently?"** MediatR 13+ and AutoMapper 15+ are commercial under Lucky Penny Software (dual-licensed RPL-1.5/commercial, free Community tier under revenue and capital thresholds), so the standard template stack is now a procurement question; `TimeProvider` in the BCL retired hand-rolled clock interfaces; Microsoft archived `eShopOnWeb` in January 2025 and it's community-maintained at NimblePros while the e-book stays on Learn; both major templates moved to .NET 10, and ardalis now ships a vertical-slice `min-clean` template alongside the full one; NetArchTest is frozen with a maintained fork (`eNhancedEdition`) and ArchUnitNET as alternatives; Aspire moved resource composition into an AppHost beside your composition root (Orientation).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Defines it by drawing four circles | States the one rule, the mechanism, the benefit being bought, and the price |
| "Dependencies point inward" with no mechanism | Explains interface ownership: the inner layer declares the port, the outer implements it |
| Confuses runtime call direction with source dependency | "Control flows outward, references point inward" — and gives EF materialization as the example |
| Uses DI, DIP and IoC interchangeably | Separates principle, technique and control-flow property; mentions Pure DI |
| Sells "we can swap the database" | "We won't. What I'm buying is millisecond tests and replaceable vendor edges" |
| Adds four projects by default | Chooses seams by cost-to-add-later; keeps use-case seam and schema ownership, defers the rest |
| Generic `IRepository<T>` returning `IQueryable` | Aggregate-shaped repositories returning materialized aggregates; a separate read path |
| Claims persistence ignorance while referencing EF in Application | Names the three compromises, or chooses a separate persistence model and quotes its cost |
| Puts all validation in one place | Three kinds, three homes; tests placement with "would another entry point need this?" |
| Catches `DbUpdateConcurrencyException` in a controller | Translates infrastructure failures in the adapter into application vocabulary |
| Ships an anemic model with a full layer stack | Notices the entities have no behaviour and fixes that before anything else |
| `IDateTime` / `IClock` interfaces | `TimeProvider` and `FakeTimeProvider`; passes `now` into domain methods as a parameter |
| Mocks `DbSet<T>` in tests | Fakes for ports, Testcontainers for adapters, no doubles at all in domain tests |
| Asserts on interactions (`Verify(...)`) | Asserts on outcomes — what's in the fake repository, which event was raised |
| Treats layers as deployment units | "Layers are compile-time, tiers are runtime" — one process until there's a reason |
| Thinks vertical slices are the opposite of Clean | Orthogonal axes; runs feature folders inside a layered shell, complexity per use case |
| Has no enforcement | Project graph, `internal`, banned-API analyzers, arch tests, ADRs — each at the fastest feedback loop |
| Never proposes removing anything | Names the signals that a layer isn't paying and how to collapse it safely |
| Can't cost it | Files-per-change, build time, onboarding time, engineer-weeks for a vendor swap |
| Unaware of current ecosystem state | MediatR/AutoMapper licensing, `TimeProvider`, archived eShopOnWeb, NetArchTest's status, Aspire's AppHost |

---

## Practice exercises

**Exercise 1 — Build the skeleton and break it on purpose (1.5 hrs).** Create the four-project solution. Add a single use case end to end. Then deliberately violate the architecture four ways — reference EF from Application, return `IQueryable` from a repository, call `DateTime.UtcNow` in the domain, reference Infrastructure from a controller. Write the arch tests and banned-symbols files that catch each. Note which violations the analyzers catch instantly and which only fail in CI.

**Exercise 2 — The reference-template audit (1.5 hrs).** Install both templates: `dotnet new install Clean.Architecture.Solution.Template` (jasontaylordev) and `dotnet new install Ardalis.CleanArchitecture.Template` plus `Ardalis.MinimalClean.Template`. Generate one solution from each. Then, for each: list every project reference, every package in the inner projects, and every place the dependency rule is bent. Write a one-page comparison with your own recommendation. This is directly rehearsable as a code-review round.

**Exercise 3 — `IQueryable` leakage, demonstrated (1 hr).** Write a repository that returns `IQueryable<Order>`. Write a handler that composes `.Where(o => SomeStaticHelper.IsEligible(o))` onto it. Run it against (a) a `List<T>.AsQueryable()` fake, and (b) EF Core with a real database in Testcontainers. Watch the test pass and production throw. Then rewrite with a materialized method and confirm both behave identically. Write three sentences on why this is the strongest argument in Concept 32.

**Exercise 4 — Persistence ignorance, priced (2 hrs).** Model an `Order` aggregate with private setters, a private constructor, a field-mapped collection, a typed `OrderId`, a `Money` value object and a shadow concurrency token. Get it mapping with fluent configuration only — zero attributes, zero EF references in Domain. Then implement the same aggregate with a *separate* persistence model and hand-written mapping. Measure: lines of code, mapping code, and — the important one — what you have to do to make an update persist without change tracking.

**Exercise 5 — Fakes vs. mocks (1 hr).** Write use-case tests for a handler twice: once with `Mock<IOrderRepository>`, once with a hand-written in-memory fake. Then change the handler's implementation (e.g. reorder the calls, or add an existence check) without changing behaviour. Count how many tests break in each version. Write down what that tells you about what each style is actually testing.

**Exercise 6 — The complexity ladder (1.5 hrs).** Take three use cases from a system you know: one pure CRUD, one with a handful of rules, one rules-heavy. Implement each at the rung you think it deserves, in the same codebase, sharing the same ports. Then implement the CRUD one at rung 3 (full domain model) and compare line counts and test counts. Write the paragraph you'd use to justify the mixed approach to a teammate who wants uniformity.

**Exercise 7 — Vertical slices, side by side (2 hrs).** Implement the same three use cases twice: once in the layered shape, once as self-contained slices. Then apply the same change to both — add an optional field that appears in the request, the entity, the database and the response. Count files touched, lines changed, and time taken. Then apply a second change: add a rule that applies to *all three* use cases. Count again. The second measurement is the one people forget.

**Exercise 8 — Enforcement from scratch (1 hr).** Add `Microsoft.CodeAnalysis.BannedApiAnalyzers` with per-project `BannedSymbols.txt` files, promote the diagnostic to an error, and confirm the violation appears in the IDE. Then write the six architecture tests from Concept 62 with both `NetArchTest.eNhancedEdition` and `ArchUnitNET`. Compare the APIs, the failure messages, and how each handles a renamed namespace (make one rule match zero types and see whether it still passes).

**Exercise 9 — The composition-root test (45 min).** Turn on `ValidateOnBuild` and `ValidateScopes`. Deliberately introduce a captive dependency (scoped into singleton) and an unregistered handler. Confirm both fail at `Build()`. Then write the `WebApplicationFactory` test that resolves every handler in the assembly, and confirm it catches the unregistered one.

**Exercise 10 — Read path, measured (1.5 hrs).** Build an order-list screen two ways: (a) load aggregates through the repository and map to DTOs, (b) a Dapper/EF projection straight to the DTO. For 1,000 orders with 10 lines each, measure rows returned, objects materialized, bytes allocated, and elapsed time. Then write the two-sentence justification for the read/write split that you'd use in a design round.

**Exercise 11 — The ports-and-adapters ACL (1.5 hrs).** Pick a real third-party API with a messy model (a payment or shipping provider's sandbox). Write the port in *your* vocabulary, the adapter that translates types, vocabulary and failure modes, and a fake. Then write handler tests covering success, decline, rate-limit and timeout — all without touching the network. Note how many of their concepts leaked into your port on the first attempt.

**Exercise 12 — Brownfield seam extraction (2–3 hrs).** Take any controller with 100+ lines of logic (yours or an open-source app's). Write characterization tests at the API level first. Extract a use-case handler without changing behaviour. Introduce one port for its most annoying dependency. Measure test wall-clock time before and after. Write the ADR.

---

## Free resources

### Primary sources — read these first, they're short

| Resource | What it covers | Why read it |
|---|---|---|
| [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) — Robert C. Martin, 2012 | The original post: the circles, the dependency rule, crossing boundaries | It's ~1,500 words. Most of what's argued about online isn't in it, which is the point |
| [Screaming Architecture](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html) — Martin, 2011 | Structure should announce the domain, not the framework | Concept 9, and the cheapest improvement in this module |
| [A Little Architecture](https://blog.cleancoder.com/uncle-bob/2016/01/04/ALittleArchitecture.html) — Martin, 2016 | Architecture as the art of deferring decisions | The best short statement of "what it buys you" |
| [Hexagonal Architecture (Ports & Adapters)](https://alistair.cockburn.us/hexagonal-architecture/) — Alistair Cockburn, 2005 | The original pattern, driving vs. driven, the intent | The vocabulary that makes Concept 6 precise; also read it to see how little the hexagon means |
| [The Onion Architecture, Part 1](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/) — Jeffrey Palermo, 2008 (parts 1–4) | Coupling toward the centre, aimed squarely at .NET N-tier | This is the .NET-specific argument, and it's the one your interviewer probably grew up on |
| [Layers, Onions, Ports, Adapters: it's all the same](https://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/) — Mark Seemann, 2013 | The equivalence argument | The single most useful thing to be able to cite in this module |
| [Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html) — Martin Fowler | When layering helps, when modules should come first | Concept 57's foundation, from the most balanced writer on the subject |
| [On the Criteria To Be Used in Decomposing Systems into Modules](https://www.win.tue.nl/~wstomv/edu/2ip30/references/criteria_for_modularization.pdf) — David Parnas, 1972 (PDF) | Information hiding; decompose by *likely change*, not by processing step | The grandparent of this entire module. Four pages. Volatility-based decomposition was settled science in 1972 |

### Ports, adapters, and composition

| Resource | What it covers |
|---|---|
| [Composition Root](https://blog.ploeh.dk/2011/07/28/CompositionRoot/) — Seemann | The definition and the rules; why exactly one (Concept 19) |
| [Pure DI](https://blog.ploeh.dk/2014/06/10/pure-di/) — Seemann | Dependency injection with no container, and why it's a legitimate choice (Concept 5) |
| [From Dependency Injection to Dependency Rejection](https://blog.ploeh.dk/2017/01/27/from-dependency-injection-to-dependency-rejection/) — Seemann | The argument that many dependencies should be *removed*, not inverted (Concept 56) |
| [Functional architecture is Ports and Adapters](https://blog.ploeh.dk/2016/03/18/functional-architecture-is-ports-and-adapters/) — Seemann | The same rule arrived at from a functional direction |
| [Boundaries](https://www.destroyallsoftware.com/talks/boundaries) — Gary Bernhardt | Functional core, imperative shell (Concept 56) — the clearest 30 minutes on the idea |
| [DDD, Hexagonal, Onion, Clean, CQRS… How I put it all together](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/) — Herberto Graça | A synthesis of every architecture in this module into one coherent picture, with a diagram worth stealing |

### The counter-arguments and the alternatives

| Resource | What it covers |
|---|---|
| [Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/) — Jimmy Bogard | The original .NET statement of the case against uniform layering (Concept 51). Read the comments too — the duplication objection and Bogard's answer are in them |
| [Vertical Slice Architecture in ASP.NET Core](https://blog.ndepend.com/vertical-slice-architecture-in-asp-net-core/) — NDepend blog | A balanced treatment with real dependency graphs, including where slices degrade consistency |
| [Vertical Slice vs. Clean Architecture](https://nadirbad.dev/vertical-slice-vs-clean-architecture) and [VSA in .NET 10](https://nadirbad.dev/vertical-slice-architecture-dotnet) | A practitioner comparison with migration strategies and folder-structure options |
| [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html) · [Domain Model](https://martinfowler.com/eaaCatalog/domainModel.html) — Fowler, *PoEAA* catalogue | The complexity-curve argument in its original form (Concept 54) |
| [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html) — Fowler | The failure mode that layering makes easy to hide (Concept 54) |
| [The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) — Sandi Metz | Why duplication beats a bad abstraction, and how to reverse one (Concept 72) |
| [CUPID — for joyful coding](https://dannorth.net/cupid-for-joyful-coding/) — Dan North | A deliberate alternative to SOLID-derived architecture advice; useful as a counterweight |
| [CQRS](https://martinfowler.com/bliki/CQRS.html) — Fowler | The definition, and the warning that it's narrower than people use it (Concepts 34–35) |

### Microsoft Learn — architecture guidance

| Resource | What it covers |
|---|---|
| [Architecting Modern Web Applications with ASP.NET Core and Azure](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/) | Microsoft's own Clean Architecture e-book (Steve Smith). Free, still maintained on Learn |
| [Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures) | The chapter that names Clean Architecture and lays out the .NET project structure |
| [Architectural principles](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles) | Separation of concerns, encapsulation, DIP, explicit dependencies, persistence ignorance |
| [.NET microservices: DDD/CQRS patterns](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/) | The DDD-oriented microservice chapter — layering, aggregates, domain events, the ordering sample |
| [Designing the infrastructure persistence layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design) | Microsoft's own take on repositories vs. `DbContext` — including "`DbSet` is already a repository" |
| [Anti-corruption Layer pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) | Concept 44, in Azure Architecture Center's pattern format |
| [Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/) | The full catalogue; Gateway, Strangler Fig and Ambassador all relate to boundaries |

### Microsoft Learn — the .NET mechanics

| Resource | What it covers |
|---|---|
| [Dependency injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) · [in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) | Lifetimes, validation options, keyed services (Concepts 19–20; Module 18) |
| [Options pattern](https://learn.microsoft.com/en-us/dotnet/core/extensions/options) | `IOptions<T>`, binding, `ValidateOnStart` (Concept 43) |
| [`System.TimeProvider`](https://learn.microsoft.com/en-us/dotnet/api/system.timeprovider) | The clock port that's now in the BCL (Concept 42) |
| [Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling) · [Web API error handling](https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors) | `IExceptionHandler`, `AddProblemDetails`, RFC 9457 shapes (Concept 41) |
| [Central Package Management](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management) | `Directory.Packages.props` as a dependency-surface control (Concept 24) |
| [MSBuild `Directory.Build.props`](https://learn.microsoft.com/en-us/dotnet/core/project-sdk/msbuild-props) | Repo-wide settings; where warnings-as-errors and analyzer levels belong |
| [EF Core backing fields](https://learn.microsoft.com/en-us/ef/core/modeling/backing-field) · [shadow properties](https://learn.microsoft.com/en-us/ef/core/modeling/shadow-properties) | The two mechanisms that make Concept 29's compromises work |
| [Integration tests in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) | `WebApplicationFactory` (Concepts 64, 66) |
| [Testing ASP.NET Core apps](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/test-asp-net-core-mvc-apps) | The e-book's testing chapter, aligned to this architecture |
| [Aspire documentation](https://aspire.dev) | AppHost, service defaults, resource composition (Concept 26) |

### Reference solutions and templates (with their current state)

| Resource | State as of September 2026 | What to look at |
|---|---|---|
| [ardalis/CleanArchitecture](https://github.com/ardalis/CleanArchitecture) | .NET 10; two templates — `clean-arch` (full) and `min-clean` (single-project vertical slice) | The Core/UseCases/Infrastructure/Web split, FastEndpoints as the driving adapter, and the fact that a vertical-slice template now ships alongside |
| [jasontaylordev/CleanArchitecture](https://github.com/jasontaylordev/CleanArchitecture) · [docs](https://cleanarchitecture.jasontaylor.dev) | .NET 10; template 10.8.0 (March 2026); Aspire-based tests; built-in OpenAPI + Scalar | `IApplicationDbContext`, MediatR pipeline behaviours, and the ADRs in the repo — read those before critiquing it |
| [Template 10.8.0 release notes](https://jasontaylor.dev/clean-architecture-template-10-8-0-released/) | March 2026 | A worked example of a template evolving: what got removed and why |
| [NimblePros/eShopOnWeb](https://github.com/NimblePros/eShopOnWeb) · [site](https://nimblepros.github.io/eShopOnWeb/) | Community-maintained since Microsoft archived the original in January 2025 | The sample behind the Microsoft e-book; a full Clean Architecture monolith you can run |
| [dotnet/eShop](https://github.com/dotnet/eShop) | Current Microsoft reference app; Aspire-hosted | Less useful for layering, very useful for how the pieces deploy (Modules 21, 26) |
| [kgrzybek/modular-monolith-with-ddd](https://github.com/kgrzybek/modular-monolith-with-ddd) | Community reference | The best free example of modules-first structure with layering inside (Concept 57; Module 21) |

### Enforcement and architecture testing

| Resource | What it covers |
|---|---|
| [BenMorris/NetArchTest](https://github.com/BenMorris/NetArchTest) | The original fluent architecture-test library. Frozen at 1.3.2 (2021) but still works |
| [NetArchTest.eNhancedEdition](https://www.nuget.org/packages/NetArchTest.eNhancedEdition) | The maintained MIT fork (1.4.5, June 2025) with a near-identical API |
| [TNG/ArchUnitNET](https://github.com/TNG/ArchUnitNET) | The actively developed .NET port of Java's ArchUnit; richer model, more verbose |
| [Writing ArchUnit-style tests for .NET](https://www.ben-morris.com/writing-archunit-style-tests-for-net-and-c-for-self-testing-architectures/) — Ben Morris | The author's rationale, including why type-level dependency detection is the hard part |
| [Architecture tests with NetArchTest](https://code-maze.com/csharp-architecture-tests-with-netarchtest-rules/) — Code Maze | A practical walkthrough with current package status |
| [Architectural Tests in .NET](https://mareks-082.medium.com/architectural-tests-in-net-1bd5d19b0ba8) — Marek Sirkovský | Compares the libraries and covers convention tests that neither can express |
| [BannedApiAnalyzers documentation](https://github.com/dotnet/roslyn-analyzers/blob/main/src/Microsoft.CodeAnalysis.BannedApiAnalyzers/BannedApiAnalyzers.Help.md) | `BannedSymbols.txt` syntax — the highest-leverage enforcement in Concept 63 |
| [dotnet/roslyn-analyzers](https://github.com/dotnet/roslyn-analyzers) | Where to look if you want to write a custom rule for a project-specific convention |

### Testing

| Resource | What it covers |
|---|---|
| [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) — Fowler | The classical-vs-mockist distinction; the theory behind Concept 65 |
| [Test Double](https://martinfowler.com/bliki/TestDouble.html) — Fowler | The taxonomy: dummy, fake, stub, spy, mock |
| [testcontainers-dotnet](https://github.com/testcontainers/testcontainers-dotnet) | Real dependencies in tests — the third tier of Concept 64 |
| [Enterprise Craftsmanship](https://enterprisecraftsmanship.com/) — Vladimir Khorikov | Deep, opinionated posts on domain-model purity, testing without mocks, and where to draw the line |

### Libraries in this space (check the licence before you adopt)

| Library | What it does | Licence note (September 2026) |
|---|---|---|
| [MediatR](https://github.com/LuckyPennySoftware/MediatR) · [mediatr.io](https://mediatr.io/) · [licensing FAQ](https://luckypennysoftware.com/faq) | In-process dispatch and pipeline behaviours | **v13+ dual-licensed** (RPL-1.5 or commercial). Free Community tier under revenue/capital thresholds. `MediatR.Contracts` stays Apache-2.0 |
| [AutoMapper](https://www.jimmybogard.com/automapper-and-mediatr-commercial-editions-launch-today/) | Convention-based object mapping | **v15+ same model as MediatR** |
| [martinothamar/Mediator](https://github.com/martinothamar/Mediator) | Source-generated mediator | A free, AOT-friendly alternative worth knowing by name |
| [riok/Mapperly](https://github.com/riok/mapperly) | Source-generated mapping — compile-time errors, no reflection | The current default recommendation over reflection-based mappers (Concept 38) |
| [FluentValidation](https://github.com/FluentValidation/FluentValidation) · [docs](https://docs.fluentvalidation.net/) | Input validation as composable rules | Still Apache-2.0 (12.x) |
| [ardalis/Specification](https://github.com/ardalis/Specification) | The specification pattern over EF Core | Read Concept 33 first — it's EF-shaped by design |
| [ardalis/Result](https://github.com/ardalis/Result) · [ErrorOr](https://github.com/amantinband/error-or) · [OneOf](https://github.com/mcintyre321/OneOf) | Result types / discriminated-union-ish returns | Pick one; it becomes permanent vocabulary (Concept 40) |
| [ardalis/GuardClauses](https://github.com/ardalis/GuardClauses) | Guard helpers for invariants | The one package most teams accept in Domain |
| [khellang/Scrutor](https://github.com/khellang/Scrutor) | Assembly scanning and `Decorate<,>` for `IServiceCollection` | Decorators without a mediator (Concept 47) |

### Documentation and decision records

| Resource | What it covers |
|---|---|
| [adr.github.io](https://adr.github.io/) | The ADR ecosystem: templates, tooling, examples |
| [joelparkerhenderson/architecture-decision-record](https://github.com/joelparkerhenderson/architecture-decision-record) | A large collection of ADR templates, from one-paragraph to full MADR |
| [The C4 model](https://c4model.com/) — Simon Brown | Context / Container / Component / Code. The *component* level is the one this module produces |
| [Package by feature, not by layer](https://phauer.com/2020/package-by-feature/) — Philipp Hauer | The clearest write-up of the structural argument behind Concept 9 |

### Books (not free, listed for completeness)

- **Clean Architecture: A Craftsman's Guide to Software Structure and Design** — Robert C. Martin. Parts III–IV (the component principles) are the half people skip and the half that generalizes.
- **Hexagonal Architecture Explained** — Alistair Cockburn & Juan Manuel Garrido de Paz (2024; updated 1st edition 2025). The pattern author's own correction of twenty years of misreadings.
- **Patterns of Enterprise Application Architecture** — Martin Fowler. Transaction Script, Domain Model, Repository, Unit of Work, Data Mapper — the vocabulary everything here inherits.
- **Dependency Injection Principles, Practices, and Patterns** — Seemann & van Deursen. The definitive treatment of composition roots, lifetimes, and anti-patterns in .NET.
- **Domain-Driven Design** — Eric Evans; **Implementing Domain-Driven Design** — Vaughn Vernon; **Learning Domain-Driven Design** — Vlad Khononov. Module 22's reading; Khononov is the most approachable entry point.
- **Unit Testing: Principles, Practices, and Patterns** — Vladimir Khorikov. The best argument for the test pyramid in Concept 64, and against mock-heavy tests.
- **Building Evolutionary Architectures** — Ford, Parsons, Kua, Sadalage. Fitness functions (Concept 67) come from here.
- **Fundamentals of Software Architecture** / **Software Architecture: The Hard Parts** — Richards & Ford. Architecture styles compared with trade-off analysis; the second is the better book for Phase 7.
- **Monolith to Microservices** — Sam Newman. The decomposition and strangler-fig material behind Concept 68 and Module 21.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Clean Architecture in one sentence | Source dependencies point inward toward policy; control still flows outward |
| The mechanism | Inner layer declares the interface, outer implements it — interface ownership is the hinge |
| The compile test | Delete the outer project: does the inner one still build? |
| Policy vs. detail | Policy would exist with paper and a telephone; detail is how it's delivered |
| Direction, generalized | Depend toward stability; volatility decides, not importance |
| Clean vs. Onion vs. Hexagonal | Same rule, different emphases — ports/adapters is the most precise vocabulary |
| Driving vs. driven ports | Driving = outside calls in (no inversion needed); driven = inside calls out (inversion needed) |
| Why the hexagon | Six sides mean nothing; it's room for ports around the edge |
| Layers vs. boundaries | A layer is a boundary along the technology axis; usually the least interesting cut |
| Strict vs. relaxed layering | Clean is relaxed — the rule constrains direction, not adjacency |
| Screaming architecture | Top-level folders name the business, not the framework |
| Component principles | ADP (no cycles), SDP (depend toward stability), SAP (stable ⇒ abstract); zone of pain = stable + concrete |
| The four projects | Domain (nothing) ← Application (Domain) ← Infrastructure (Application) ← Web (all, at the root) |
| What enforces it | The `.csproj` graph; everything else is documentation |
| Domain contents | Entities, value objects, domain services, domain events, invariants, errors — and no packages |
| Application contents | Use cases, ports, DTOs, orchestration, transaction boundary, business authorization |
| Infrastructure contents | Adapters: persistence, messaging, HTTP, blob, mail, identity, clock |
| Presentation contents | Protocol only: routing, binding, serialization, status codes, auth handshake |
| Composition root | One place, at the entry point, allowed to know every concrete type; not a layer |
| Why Web → Infrastructure is OK | Composition isn't policy; constrain it with an arch test on every other type |
| Registration location | `AddInfrastructure()` lives *in* Infrastructure; implementations stay `internal` |
| `internal` + `InternalsVisibleTo` | Free boundary enforcement; tests get in via an MSBuild item |
| Build-level controls | `Directory.Build.props`, Central Package Management, `BannedSymbols.txt` per project |
| Aspire | AppHost composes the topology; your root composes the object graph |
| Repository, correctly | Aggregate-shaped, collection-like, returns materialized aggregates, no `Update` |
| `DbSet`/`DbContext` | Already a repository and a unit of work — so yours must add aggregate shaping and vocabulary |
| Unit of work | Transaction boundary = application decision; repositories never call `SaveChanges` |
| Persistence ignorance | Survives three compromises: private ctor, field-mapped collections, shadow persistence state |
| Separate persistence model | Pure, and it costs mapping both ways **plus change tracking** |
| `IApplicationDbContext` | A velocity trade, not a boundary — Application now references EF Core |
| `IQueryable` leakage | Unspecified provider contract, deferred execution, dead aggregate boundary, lying fakes |
| Specifications | A named query object; worth it when criteria repeat, ceremony when they don't |
| The read side | Queries don't need the domain model; let the read path know SQL |
| CQRS-lite | Two paths through one architecture; only the write path pays for aggregates |
| Aggregate size | As small as the invariants allow; reference other aggregates by ID |
| Migrations | Infrastructure artifacts, applied by a pipeline, one history table per module |
| How many models | Command/query + domain entity + read DTO is the honest default |
| Mapping libraries | Manual or source-generated (Mapperly); AutoMapper v15+ is commercial |
| Validation | Shape at the edge, context in the handler, invariants in the entity |
| The placement test | "Would another entry point need this check?" → yes means domain |
| Errors | Expected ⇒ result value; exceptional ⇒ exception; adapters translate infrastructure failures |
| Protocol mapping | `IExceptionHandler` + `ProblemDetails` (RFC 9457); the domain never names a status code |
| Time | `TimeProvider` + `FakeTimeProvider`; pass `now` into domain methods as a parameter |
| IDs | Generate in the application — `Guid.CreateVersion7()`; wrap in typed IDs |
| Configuration | `IOptions<T>` at the edge, `ValidateOnStart`, plain records inward |
| Anti-corruption layer | Translate types, vocabulary **and** failure modes |
| Resilience | Belongs to the adapter; a retry loop in a handler is a leak |
| Domain vs. integration events | In-process + in-transaction vs. published contract + outbox + idempotent consumers |
| Background workers | Driving adapters; a scope per unit of work; use case can't know who called it |
| Cross-cutting placement | Protocol → middleware; use-case concerns → behaviour/decorator; dependency concerns → adapter |
| Don't abstract | `ILogger`, the container, serialization, LINQ, `HttpClient`, framework ports |
| The fake test | If you wouldn't write a fake, you don't need an interface |
| Port signatures | `Task`/`ValueTask` + `CancellationToken`; domain types only; domain stays synchronous |
| Vertical slices | Minimize coupling between slices, maximize cohesion within; assumes refactoring discipline |
| The reframing | Layers decouple by technology; slices decouple by change — pick your axis |
| The hybrid | Dependency rule at project level, feature folders inside Application, complexity per use case |
| Transaction script | Still correct for simple use cases; better than an anemic domain model |
| The ladder | CRUD → transaction script → domain model → explicit workflow, chosen per use case |
| Functional core | Pure decisions in, effects at the edge; "pass the data, don't inject the dependency" |
| Modules vs. layers | Modules group what changes together; schema ownership is the real boundary |
| When not to use it | Thin domain, tiny team, short life, data-shaped complexity, integration services, libraries |
| Evolutionary approach | Buy the expensive-later seams: use case, vendor ports, schema ownership |
| The swap-the-database claim | Don't sell it; sell test speed, deferred decisions, and replaceable edges |
| Enforcement ladder | References → `internal` → analyzers → arch tests → review/ADR |
| Arch tests, first six | Domain depends on nothing; Application → Domain only; Infra only from the root; nothing → Web; module isolation; naming |
| How arch tests lie | Metadata-only, string rules that match nothing, and they run late |
| Banned symbols | `DateTime.UtcNow`/`Guid.NewGuid`/`Random` in Domain; EF/ASP.NET in Application |
| Test pyramid | Domain (no doubles) → use case (fakes) → adapter (Testcontainers) → e2e (`WebApplicationFactory`) |
| Fakes vs. mocks | Fakes have behaviour and catch bugs; mocks confirm assumptions |
| Container validation | `ValidateOnBuild` + `ValidateScopes` turn wiring bugs into build failures |
| Fitness functions | Dependency direction, package closure, test-tier ratio, build time — with owners and trends |
| Brownfield order | Harness → use-case seam → invert one vendor → domain where rules are → schema ownership → enforce |
| Collapsing a layer | Pass-throughs, single implementations, no fakes, nobody can name what it protects |
| Runtime cost | Indirection is free; the bill is aggregate loading and mapping allocations |
| Human metrics | Files per change, PR size, time to first PR, build time, "where do I put this?" frequency |
| The wrong abstraction | Duplication is cheaper; inline it back, let the copies diverge, re-extract what's truly shared |
| Layers vs. tiers | Compile-time vs. runtime; one process until there's a reason; never a service per layer |
| Conway's law | Boundaries that match ownership survive; others need tooling or die |
| Documentation | ADRs for why, C4 component level for what, generated graphs for truth |
| Cost framing | Engineer-weeks, % overhead, lead time, incidents — and name the rejected option |
| Anti-patterns | Anemic entities, `IQueryable` escaping, generic repos, `SaveChanges` in repos, rules in controllers, service locator, `DateTime.UtcNow` inside |
| Defaults by size | 2–3: one project, feature folders, vendor ports. ~8: four projects + arch tests. ~40: modules first |
| Recent ecosystem changes | MediatR/AutoMapper commercial; `TimeProvider` in the BCL; eShopOnWeb archived → NimblePros; both templates on .NET 10; ardalis ships `min-clean`; NetArchTest frozen, fork + ArchUnitNET maintained |

---

## Progress

**Module 20 complete — Phase 5 is open.** Phase 4 gave you the runtime, the language, the measurement discipline, the web framework and the data layer. This module is the first that is purely about *judgment*: there is no CLR behaviour to memorize here, no benchmark that settles the argument, and the interviewer is scoring whether you can choose, cost, and defend a structure rather than whether you can name one.

This module closes the loops it was created to close:

- **Module 18's Concept 72** (the composition root) now has a location, a set of rules, a startup-validation test, and an answer to "why is Web allowed to reference Infrastructure."
- **Module 19's Concepts 65–68** — where the ORM stops, repositories over EF, `IQueryable` leakage, EF and DDD — are now situated: Part C decides the seam, Concept 29 gives the three compromises that make persistence ignorance real, and Concept 30 prices the pure alternative including the change-tracking cost that nobody mentions.
- **Module 19's Concept 72 anti-patterns** became Concept 80's review catalogue, extended with the layering-specific ones.
- **Module 11's outbox and dual-write problem** is now the canonical example of an application port with an infrastructure implementation (Concept 45), with the domain-vs-integration-event distinction stated as a layering rule.
- **Module 13's resilience patterns** got their home: the adapter, not the handler (Concept 44).
- **Module 17's measurement discipline** was applied to the architecture itself (Concept 70) — the bill is aggregate loading and mapping allocations, not indirection.
- **Module 12's storage trade-offs** are why Concept 60 can say honestly that the repository doesn't let you swap the database.

Threads left open on purpose:

- **Modular monolith vs. microservices** — modules as the first cut, schema ownership as the real boundary, and when a module should become a service — is **Module 21**. Concepts 57 and 77 are its opening.
- **DDD tactical patterns** — aggregates, invariants, value objects, domain events, and how to find the boundaries in the first place — are **Module 22**, which turns Concepts 15, 36 and 45 from layering rules into design work.
- **CQRS and MediatR** — whether the read/write split of Concepts 34–35 deserves a framework, and what the MediatR licensing change means for a real procurement decision — are **Module 23**.
- **Event sourcing**, where the "separate persistence model" of Concept 30 becomes inherent rather than optional, is **Module 24**.
- **Polly and resilience composition** — the adapter-level implementation of Concept 44 — is **Module 25**.
- **ADRs and the C4 model**, the documentation half of Concept 78, are **Module 31**.
- **Brownfield migration and the strangler fig** get their full treatment in **Module 32**; Concept 68 is the compressed version.
- **Cost, build-vs-buy and technical debt** — the executive language of Concept 79 — is **Module 33**.

Next in the curriculum: **Module 21 — Modular monolith vs. microservices**: the actual decision criteria rather than dogma, why schema ownership is the only module boundary that holds, what a network hop really costs you in latency and failure modes, and how to give an answer about distributed systems that doesn't begin with "it depends."
