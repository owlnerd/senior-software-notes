# Module 21 — Modular Monolith vs. Microservices
*Phase 5: .NET Architecture Patterns · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **modularity and distribution are two independent decisions, and almost every bad architecture in this space comes from treating them as one — you get the benefits people attribute to microservices from modularity, which is free of network cost, and you pay the price people attribute to microservices for distribution, which buys you exactly one thing that modularity cannot: independent deployment and runtime isolation per team.**

That reframing is the whole module. The naive version of this topic is a two-column table — monolith on the left, microservices on the right, "scalability" and "agility" ticked in the right column — and a mid-level candidate will recite it. A senior candidate can tell you *why* a company with 40 services and one shared database has all of the costs and none of the benefits and has a specific name (distributed monolith), why a method call and an in-datacenter RPC differ by roughly **six orders of magnitude** and what that does to a loop that was fine when it was in-process, why five services each at 99.9% availability compose to 99.5% if they're in series and what to do about it, why "we'll extract services later" is a lie unless schema ownership was enforced from day one, why the Amazon Prime Video case study everyone cites is not the story they think it is, why the "42% of organizations are consolidating microservices" statistic that is all over the internet in 2026 **does not appear in any CNCF publication** and what that tells you about citing numbers in an interview, why Conway's law means a boundary that doesn't match a team will erode no matter how many architecture tests you write, and why the correct answer to "monolith or microservices?" in a design round is never "it depends" but a two-sentence decision with a named trigger attached.

This module closes the thread **Module 20 opened by name**. Module 20's **Concept 57** ("modules before layers") gave you the shape; **Concept 77** (Conway's law) gave you why it holds; **Concept 76** (layers are not tiers) gave you the distinction this module builds on — logical separation is compile-time, distribution is runtime, and conflating them is the root error. **Module 11** gave you messaging, delivery semantics and the transactional outbox, which is the single mechanism that makes a module extractable. **Module 12** gave you distributed transactions and sagas, which is what you get *instead of* ACID once a boundary becomes a network boundary. **Module 13** gave you the reliability patterns that become mandatory rather than optional the moment you distribute. **Module 7** gave you CAP/PACELC, which is the theory behind "you just gave up your transaction." **Module 6** gave you the Scale Cube, whose Y-axis is exactly this decomposition. **Module 5** gave you the latency numbers this module spends.

It shows up in five places in an interview loop: the **design round** (nearly every system design question has a decomposition moment, and how you handle it is scored), the **architect round** ("we have a monolith and the CTO wants microservices — what do you tell him?"), the **brownfield round** ("here's a 12-year-old system, plan the next eighteen months"), the **deep technical round** ("what breaks when an in-process call becomes an HTTP call?"), and the **behavioural round** (this decision is the most common source of the "technical disagreement" story). It is also the single most common place where candidates recite trends instead of reasoning, and get marked down for having no criteria.

**Current platform state (verified September 2026).** .NET 10 (November 2025) is the production-current LTS, supported to November 2028; .NET 11 RC1 shipped 8 September 2026 ahead of GA on 10 November 2026. What changed in this module's ecosystem, and why each item matters architecturally:

- **Aspire dropped the ".NET" from its name and became a polyglot application platform.** Aspire 13.0 shipped 11 November 2025 at .NET Conf, making Python and JavaScript first-class alongside .NET; 13.1 (December 2025) added MCP integration for AI coding agents; 13.2 (March 2026) was the largest release to date (1,100+ issues closed) with a CLI built for automation, detached mode and a TypeScript AppHost in preview; 13.3 (May 2026) and 13.4 added "Aspireify anything," typed resource commands, and mature Kubernetes/AKS deployment. Documentation moved to **aspire.dev**. **Why it matters here:** Aspire is the first mainstream .NET tool that makes the multi-process local dev loop cheap, which removes one of the historical arguments *against* services — and, less obviously, it makes a modular monolith with two or three extracted services the path of least resistance rather than an awkward middle state.
- **MassTransit v9 is commercial.** v9 shipped in 2026 as a source-available, commercially licensed product (free for local development and evaluation; a paid licence for deployed use), with pricing published around \$400/month for small and medium businesses and \$1,200/month for large enterprises. **v8 remains Apache-2.0 but its official maintenance ends around the end of 2026.** The MIT-licensed alternatives are **Wolverine** (mediator + messaging + outbox + sagas in one library), **Rebus**, and **Brighter**; NServiceBus remains commercial as it always was. **Why it matters here:** the library you use for module-to-module messaging is now a procurement decision, and in a modular monolith you may not need a broker-backed bus at all.
- **MediatR 13+ and AutoMapper 15+ are commercial** (Lucky Penny Software, dual-licensed RPL-1.5/commercial, free Community tier under revenue and capital thresholds; `MediatR.Contracts` stays Apache-2.0). Covered in Module 20 and taken properly in Module 23; it matters here because the in-process dispatcher is how most .NET modular monoliths route cross-module calls.
- **The first-party .NET microservices reference material has drifted.** `dotnet-architecture/eShopOnContainers` was archived **17 November 2023** and superseded by **`dotnet/eShop`** (Aspire-hosted, service-per-bounded-context). The Microsoft Learn e-book *.NET Microservices: Architecture for Containerized .NET Applications* is **still live and still worth reading for the patterns**, but it still points at the archived sample — know that before you cite it. Separately, `dotnet-architecture/eShopOnWeb` (the monolith-with-Clean-Architecture companion) was archived by Microsoft on 13 January 2025 and is community-maintained at `NimblePros/eShopOnWeb`.
- **Dapr graduated from the CNCF on 30 October 2024** (announced at KubeCon NA, 12 November 2024) and shipped v1.16 in September 2025 with multi-application workflows and, on the .NET side, Roslyn analyzers that enforce correct SDK configuration. Worth knowing that CNCF's own project metrics show contributor and star counts declining year-over-year — a graduated project is not automatically a growing one, and that nuance is the kind of thing that separates a read-the-headline answer from a real one.
- **Service mesh adoption is genuinely declining, and the honest numbers are not the ones circulating.** CNCF's 2024 Annual Survey reported service mesh adoption falling from **50% (2023) to 42% (2024)**, attributed to operational overhead. A separate CNCF/SlashData developer-population report (12,021 developers) found service mesh use among *individual developers* falling from **18% in Q3 2023 to 8% in Q3 2025**, with 46% of developers building microservices. The 2025 CNCF Annual Cloud Native Survey (628 practitioners, fielded September 2025, published 20 January 2026) reports **82% of container users running Kubernetes in production**, 98% cloud native adoption, service mesh at 39% among the most mature cohort, and — for the first time — the top adoption barrier being **cultural (47%)** rather than technical. **Istio's ambient (sidecar-less) mode reached GA in v1.24 on 7 November 2024**, which is the project's response to exactly this overhead complaint.
- **The "42% of organizations are consolidating microservices back into larger deployable units" statistic is not verifiable in any CNCF publication.** It appears in dozens of near-identical blog posts published through 2026, all attributing it to a "2025 CNCF survey," with no primary link. The number 42% *does* appear in real CNCF data — as service mesh adoption in 2024. Treat this as a laundered statistic. It is a live risk in an interview, because your interviewer may well cite it at you, and the correct move is Concept 96.

This module has nine jobs:

1. **Separate the two axes** — modularity and distribution — and show that nearly every claimed benefit belongs to one of them specifically.
2. **Price the network boundary precisely**, in latency, availability, failure modes, consistency, debugging, testing, versioning and money, so "distributed systems are hard" becomes a number you can quote.
3. **Build the modular monolith properly in .NET** — projects, visibility, schemas, migrations, endpoints, events, enforcement — as a concrete engineering artifact rather than a slogan.
4. **Make the boundary-finding problem explicit**, because the decomposition, not the deployment, is the thing that is actually hard.
5. **Give extraction a procedure**, with triggers, an order of operations, data migration mechanics and a rollback.
6. **Cover living with services on .NET/Azure** honestly, including the platform floor you must build before the second service.
7. **Handle the industry pendulum** — the case studies that are real evidence, the ones that are misread, and the statistics that are fabricated.
8. **Turn the whole thing into a decision you can defend** with a scorecard, an ADR, and a cost conversation.
9. **Give you the 60-second answer** that lands in a design round, which is the deliverable this module exists for.

Nine framings to carry through:

1. **Modularity is free; distribution is expensive.** Every claim about "microservices make code maintainable" is actually a claim about modularity, which you can have without a single network hop.
2. **The only thing distribution uniquely buys is independent deployment and runtime isolation.** Everything else on the list — clear boundaries, team ownership, technology choice within limits, testability — is available in one process.
3. **Microservices are an organizational solution to an organizational problem.** If you don't have the organizational problem (many teams blocking each other on releases), you are buying an expensive answer to a question nobody asked.
4. **The boundary is the hard part, and it is the same problem in both architectures.** Getting it wrong in a monolith costs you a refactor; getting it wrong in a distributed system costs you a migration.
5. **Schema ownership is the only boundary that actually holds.** Code boundaries erode under deadline pressure. A module that cannot read another module's tables stays separate whether people like it or not.
6. **The distributed monolith is the modal outcome**, not a rare failure. Services that must be deployed together, share a database, and call each other synchronously in a chain have every cost and no benefit.
7. **Reversibility is asymmetric.** Monolith → services is a hard, expensive, incremental migration. Services → monolith is harder, because the data has already diverged. Choose the direction that leaves you the cheaper mistake.
8. **Quote numbers, not trends.** Latency numbers, availability composition, team counts, deploy frequency, lead time. "The industry is moving back to monoliths" is not an argument; "our deploy contention is two teams, so the premium isn't earned yet" is.
9. **Both answers are correct somewhere.** The score is for the criteria and the trigger, not the conclusion. A candidate who says "modular monolith" with no extraction trigger is as weak as one who says "microservices" with no team count.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Two axes, not one | Modularity and distribution are independent; confusing them is the root error |
| 2 | The four quadrants | Big ball of mud, modular monolith, distributed monolith, microservices |
| 3 | Definitions, precisely | Monolith = one deployable; module = one boundary; service = a module with its own process and data |
| 4 | Independent deployability | The defining property; if you can't deploy one alone, it isn't a microservice |
| 5 | "Micro" is the worst word in the name | Size is a consequence of boundary choice, never a target |
| 6 | The distributed monolith | Every cost, no benefit — and the modal real-world outcome |
| 7 | Module first, service second | A service is a hosting decision applied to a module that already exists |
| 8 | Five properties of a real boundary | Own data, own contract, hidden internals, independent tests, independent change |
| 9 | Conway's law | Architecture mirrors communication structure; boundaries that don't match ownership erode |
| 10 | Cognitive load | Team Topologies' real constraint: a team can own only so much domain |
| 11 | The microservices premium | Fowler's curve: below the complexity crossover you pay and get nothing |
| 12 | MonolithFirst vs. don't-start-with-a-monolith | Both are right; the synthesis is "modular first, with extraction pre-paid" |
| 13 | Three different units | Unit of failure, unit of scale, unit of release — decide each separately |
| 14 | Reversibility asymmetry | Splitting is expensive; merging is worse; prefer the cheaper mistake |
| 15 | The five claims, graded | Deployability (real), fault isolation (conditional), scale (rarely), tech choice (overrated), autonomy (real) |
| 16 | The org-chart test | Count the teams that block each other on release; that number decides |
| 17 | The fallacies, as a bill | Latency, bandwidth, reliability, topology, security, admin — each becomes a line item |
| 18 | Six orders of magnitude | ~1 ns method call vs. ~500 µs same-DC RPC; a loop that was free is now a timeout |
| 19 | Serialization is the hidden cost | Payload size × hops; the Prime Video lesson read correctly |
| 20 | Tail-latency amplification | Fan-out to N services makes your p99 their p99.9 |
| 21 | Availability composition | Series dependencies multiply; 5 × 99.9% = 99.5% unless you decouple |
| 22 | Partial failure | The bug class that doesn't exist in a process: timeout, retry, duplicate, ambiguity |
| 23 | Losing the transaction | ACID stops at the boundary; you inherit sagas and compensations |
| 24 | Losing the join | Data gravity, API composition, and the reporting problem nobody plans for |
| 25 | The debugging tax | Stack trace → distributed trace; correlation IDs are mandatory, not nice |
| 26 | The testing tax | Contract tests in, end-to-end suites out; the integration-test trap |
| 27 | The versioning tax | Any change crossing a boundary becomes a two-release dance, forever |
| 28 | The operational floor | What must exist before service #2: pipeline, telemetry, discovery, config, on-call |
| 29 | The cost model | Infra multiplies sub-linearly; people multiply super-linearly |
| 30 | What you actually gain | Blast radius, deploy independence, scale isolation, team autonomy — priced |
| 31 | The modular monolith defined | One deployable, strict internal boundaries, no shared data across modules |
| 32 | Solution shape | Modules/<Name>/{Domain,Application,Infrastructure}, Host, SharedKernel, Contracts |
| 33 | The public surface | `internal` by default; one small public contract assembly per module |
| 34 | Host and module registration | `AddOrderingModule()` / `MapOrderingEndpoints()`; the host knows modules, modules don't know each other |
| 35 | Schema ownership | The boundary that holds; a module owns its tables and nobody else reads them |
| 36 | One database, many schemas | The pragmatic default; when to split to separate databases |
| 37 | No cross-module joins or FKs | The single rule that keeps extraction possible |
| 38 | Cross-module reads | Public contract call, replicated read model, or a query on your own copy |
| 39 | Cross-module writes | Events, not calls; commands that cross a boundary are a design smell |
| 40 | The in-process outbox | Write events in the same transaction now, so the broker is a config change later |
| 41 | Duplication is a feature | Two modules with their own `Customer` is correct, not a DRY violation |
| 42 | The shared kernel, kept tiny | IDs, money, dates, base types — and nothing with business rules |
| 43 | Migrations per module | Separate `DbContext`, separate schema, separate history table |
| 44 | Configuration per module | Module-scoped `IOptions<T>`, validated at startup |
| 45 | Endpoints per module | Each module registers its own routes; the host composes |
| 46 | Background work per module | A module's worker lives in the module and calls only its own use cases |
| 47 | Testing a module in isolation | Module tests with fakes at the contract boundary; one container per schema |
| 48 | Enforcement | Project references, `internal`, arch tests, banned APIs — machine-checked or fictional |
| 49 | In-process messaging options | Wolverine (MIT), a 40-line dispatcher, `Channels`, MediatR 13+ (commercial) |
| 50 | Aspire and the modular monolith | AppHost composes resources; one app project, real infrastructure locally |
| 51 | The real risk | Not performance — deployment coupling and the discipline decay curve |
| 52 | Extraction-readiness checklist | Nine properties that make "we can extract later" a fact, not a hope |
| 53 | Bounded contexts are the source | The boundary is a language boundary before it is a code boundary |
| 54 | Decompose by capability | Business capability or subdomain — never by entity, layer, or noun |
| 55 | The entity-service trap | "UserService" and "OrderService" are a database schema wearing a costume |
| 56 | EventStorming | The cheapest boundary-discovery tool that exists; a day with the business |
| 57 | Change coupling | Mine git history: files that change together belong together |
| 58 | Coupling taxonomy | Domain < pass-through < common < content; temporal coupling as its own axis |
| 59 | Data coupling | The shared table is the coupling that survives every refactor |
| 60 | Cohesion metrics | Afferent/efferent coupling and instability, applied to modules |
| 61 | The two-pizza rule | It encodes communication overhead, not service size |
| 62 | Nanoservices | The over-decomposition failure and how to recognize it |
| 63 | Symptoms of wrong boundaries | Lockstep deploys, chatty calls, shared DB, one team touching every service |
| 64 | Boundaries follow ownership | An unowned boundary is a suggestion; Conway's law is a physical law |
| 65 | Extraction triggers | Deploy contention, scale asymmetry, isolation needs, compliance, tech necessity, team count |
| 66 | Anti-triggers | Resume, dogma, "the monolith is a mess," vendor advice, conference talks |
| 67 | Order of operations | Logical boundary → data boundary → process boundary. Never reverse |
| 68 | Strangler fig, applied | Facade in front, route one capability at a time, delete when traffic is zero |
| 69 | Branch by abstraction | The in-process technique that makes the switch a config flag |
| 70 | Expand/contract | Schema changes that are safe with two versions running |
| 71 | Data separation | Dual write → backfill → read switch → cutover → delete; each step reversible |
| 72 | Contracts and CDC tests | Consumer-driven contract tests replace the end-to-end suite you can't afford |
| 73 | The rollback plan | If you can't undo a step within an hour, the step is too big |
| 74 | Measuring whether it worked | Lead time, deploy frequency, change-fail rate, restore time — before and after |
| 75 | Consolidating back | Harder than splitting, because the data diverged; the procedure anyway |
| 76 | Brownfield: modularize first | The monolith-to-modular-monolith path is the highest-ROI work in most legacy systems |
| 77 | The platform floor on Azure | Pipeline, registry, config, secrets, telemetry, tracing, discovery, on-call |
| 78 | Compute choice | App Service / Container Apps / AKS / Functions — the short version (Module 26) |
| 79 | Sync vs. async between services | Every synchronous call is a shared availability; async is the decoupling |
| 80 | gRPC vs. HTTP/JSON | When the wire format actually matters, and when it's a rounding error |
| 81 | Gateway and BFF | YARP, aggregation, and why the gateway must not hold business logic |
| 82 | Service mesh | What it does, the ambient shift, and why adoption is falling |
| 83 | Dapr | The building-block alternative to a mesh, and its trade-off |
| 84 | Tracing and correlation | OpenTelemetry end to end, or you cannot operate this |
| 85 | Resilience at boundaries | Timeouts, retries with jitter, circuit breakers, bulkheads — mandatory now |
| 86 | Database per service | Non-negotiable for services; and the analytics answer that follows |
| 87 | Sagas across services | What you get instead of a transaction; orchestration vs. choreography |
| 88 | Shared libraries | The coupling you accidentally reintroduce; the one rule that keeps it safe |
| 89 | Repo strategy | Mono vs. poly is a tooling choice, not an architecture; it does affect coupling |
| 90 | Testing strategy | Unit → module → contract → a thin end-to-end smoke; Testcontainers and Aspire |
| 91 | The 60-second answer | Requirements → team shape → default → trigger → boundary → first extraction |
| 92 | The decision scorecard | Six weighted factors with thresholds you can say out loud |
| 93 | Defaults by team size and age | 1–3, 4–15, 16–50, 50+ — with what changes at each step |
| 94 | The cost conversation | Engineer-months, run rate, lead time, incident cost — the executive vocabulary |
| 95 | The ADR | Context, options, decision, consequences, and the trigger that revisits it |
| 96 | The pendulum, handled | What to say about trends and how to handle a fabricated statistic |
| 97 | Case studies that are evidence | Amazon Prime Video, Segment, Shopify, Uber, Istio — read correctly |
| 98 | The anti-pattern catalogue | Fourteen things reviewers look for |
| 99 | When the interviewer wants microservices | Disagreeing well is the thing being scored |
| 100 | The close | One paragraph that demonstrates judgment rather than preference |

---

# Part A — What is actually being decided

Twelve pages of this debate on the internet, and almost none of it establishes what the question is. Part A establishes it. The single most valuable thing you can do in an interview on this topic is to reframe the question before answering it — not as a rhetorical trick, but because the unreframed question has no answer.

---

## Concept 1 — Two axes, not one

There are two independent decisions hiding inside "monolith or microservices?":

**Axis 1 — Modularity.** How strongly is the code partitioned into units with hidden internals, explicit contracts and independent reasons to change? This is a *compile-time and design-time* property. It costs discipline and some ceremony. It costs **zero runtime overhead**.

**Axis 2 — Distribution.** How many independently deployable processes is the system made of, and do they communicate over a network? This is a *runtime* property. It costs latency, availability, failure modes, operational surface and money.

These are orthogonal. You can have any combination. And the crucial observation is this: **almost every benefit people attribute to microservices is an Axis 1 benefit.** "Clear boundaries," "easier to understand," "teams can work without stepping on each other's code," "we can replace a component," "new joiners can be productive in one area" — all modularity. Only two things are genuinely Axis 2: **independent deployment** and **runtime isolation** (a crash, a memory leak, a CPU spike or a bad deploy in one unit doesn't take down the others).

So when someone says "we need microservices because the codebase is a mess," they have diagnosed an Axis 1 problem and prescribed an Axis 2 treatment. The treatment costs an order of magnitude more, and — this is the part that gets missed — **it does not cure the disease.** Distributing a tangled codebase produces a tangled distributed codebase, which is strictly worse, because now the tangles are over HTTP and you can't refactor them with a rename.

In an interview, saying this out loud early is a strong signal: *"Before I answer, I want to separate two things, because they're usually conflated…"*

---

## Concept 2 — The four quadrants

Plot the two axes and you get four real architectures, all of which exist in production somewhere:

| | **Low modularity** | **High modularity** |
|---|---|---|
| **One deployable** | **Big ball of mud.** Everything touches everything. Changes are unpredictable. This is what "monolith" is used as a slur for. | **Modular monolith.** Strict internal boundaries, one process, one deploy. Fast, simple to operate, cheap to change. |
| **Many deployables** | **Distributed monolith.** Services that must deploy together, share a database, and call each other synchronously. Every cost of distribution, no benefit. | **Microservices.** Independently deployable, independently owned, own their data. Expensive, and occasionally worth it. |

Three things follow from this picture, and all three are interview-grade points.

First, **the horizontal move is the valuable one.** Going from low to high modularity improves your life in *either* row. It is the work that has to happen regardless. A team that wants microservices and hasn't done the horizontal move will land in the bottom-left quadrant, which is the worst of the four.

Second, **the vertical move is the expensive one**, and it's only worth making from the right-hand column. Modularize, then distribute — if you still need to.

Third, **the bottom-left quadrant is not a hypothetical.** It is where most "we did microservices" organizations actually are. Recognizing it by symptom (Concept 63) is more useful than any definition.

---

## Concept 3 — Definitions, precisely

Sloppy vocabulary is why this argument never ends. Use these:

- **Monolith.** A system deployed as **a single unit**. That's the entire definition. It says nothing about code quality, size, layering, or whether it's well-designed. A monolith can be 500 lines or 5 million. Microsoft's own architecture guide is careful about this: *"monolithic" refers to the fact that these applications are deployed as a single unit* — and notes that in most situations a single application is easier to build, deploy and debug than a collection of services while still meeting the requirements.
- **Module.** A unit of code with **hidden internals, an explicit public contract, and its own reason to change**. Parnas, 1972. A module is a design-time concept. It has no deployment implications at all.
- **Modular monolith.** A monolith whose internal structure is a set of modules with **enforced** boundaries — most importantly, no module reads another module's data.
- **Microservice.** A service that is **independently deployable**, **owns its own data**, and is **owned by one team**. Sam Newman's framing: independently deployable services modelled around a business domain.
- **Distributed monolith.** A set of services that are *not* independently deployable — usually because they share a database, share a release train, or form a synchronous call chain where a change to one requires changes to others.
- **SOA.** The older, broader family. Microservices are a specific, opinionated SOA style. Ignore anyone who says they're unrelated.
- **Self-contained system (SCS).** A larger-grained alternative: each system owns its UI, logic and data, with asynchronous integration and no shared UI framework. Worth knowing because it's a legitimate middle point that interviewers occasionally raise.

The precision pays off immediately: *"By 'monolith' do you mean a single deployable, or do you mean a codebase without internal structure? Because those are different problems with different fixes."*

---

## Concept 4 — Independent deployability is the defining property

If you take one test from this module, take this one. **Can you deploy this service to production, on a Tuesday afternoon, without coordinating with any other team and without deploying anything else?**

If no, it is not a microservice, regardless of its size, its container, its repository, or what the architecture diagram says.

Independent deployability is the property that:

- makes the network cost worth paying (you're buying release autonomy),
- forces every other discipline (versioned contracts, backward compatibility, own data),
- is the thing the organization actually wanted,
- and is the thing that is almost always quietly lost first.

It is lost in three common ways. **Shared database:** a schema change requires coordinating every service that reads those tables, so they ship together. **Lockstep contracts:** a change to a request DTO breaks the caller, so the caller and callee ship together. **Shared library with business logic:** a change to the shared package requires bumping and redeploying everything that references it.

Each of these is a two-line fix in a monolith (rename, recompile, deploy) and a multi-week programme in a distributed system. That asymmetry *is* the microservices tax.

---

## Concept 5 — "Micro" is the worst word in the name

The name has done more damage than any other term in software architecture, because it puts a **size target** where there should be a **boundary criterion**. It produces the question "how big should a microservice be?" — which has no answer — instead of "what belongs together?", which has one.

The right framing: **size is an output, not an input.** You choose a boundary (a business capability, a bounded context, a team's area of ownership). The size of the service is whatever that boundary happens to contain. Sometimes that's 2,000 lines; sometimes it's 200,000. A "Billing" service at a payments company is a large, complex, valuable piece of software and should be.

Two heuristics that are *consequences* rather than targets, and are safe to quote:

- **A service should be rewritable by its owning team in a sprint or two** — Jon Eaves' old rule. Not a rule to design by, but a useful smell test: if nobody on the team understands it end to end, the boundary probably encloses more than one thing.
- **A service should fit in one team's head.** This is Concept 10, and it's the version with actual theory behind it.

If an interviewer asks "how small should a microservice be?", the senior answer is to decline the framing: *"I'd size it by boundary rather than by lines. The constraint I'd actually apply is that one team can own it completely and no other team needs to change it to ship their own work."*

---

## Concept 6 — The distributed monolith

The dominant failure mode, and the thing you should be able to diagnose from three questions.

A distributed monolith is a system that has paid **all** the costs of distribution — network latency, partial failure, operational surface, deployment complexity, distributed debugging, eventual consistency — and received **none** of the benefits, because the services are not independently deployable.

Diagnostic questions, in order of how quickly they settle it:

1. **"Do two services write to the same tables?"** If yes, you have one database with several front ends. There is no service boundary; there is a shared mutable global variable with network latency in front of it.
2. **"When you release, how many services go out together?"** If the answer is "all of them, from one pipeline, in a fixed order," you have one deployable in several containers.
3. **"If service A is down, what happens to service B's requests?"** If the answer is "they fail," they are one availability unit. You have multiplied your failure probability without multiplying your independence.

Two further symptoms worth naming: **synchronous call chains more than two deep** (a request that traverses A → B → C → D has the latency of all four and the availability of all four multiplied together), and **a shared "common" or "core" library containing domain types** that every service references and that requires a coordinated bump.

The reason this matters for interviews is that "we have microservices" is a claim you should be prepared to gently test, in a brownfield round, without being rude about it. *"Can I ask a couple of questions about the current setup — mostly around data ownership and release coupling?"* is the professional version.

---

## Concept 7 — Module first, service second

The correct mental model, and the one that makes the whole decision tractable:

**A service is a deployment decision applied to a module that already exists.**

That ordering has three consequences.

**You cannot extract what you haven't separated.** If the code isn't already modular — if Ordering reaches into Billing's tables and constructs Billing's entities — then "extracting the Billing service" isn't an extraction, it's a rewrite with a network in the middle. This is why most microservices migrations take three times as long as estimated: the estimate was for the deployment work, and the actual work was the modularization that should have happened first.

**The modularization is valuable on its own.** If you modularize and then decide not to distribute, you have still improved the system: change is localized, tests are faster, ownership is clear, onboarding is cheaper. Nothing is wasted. This makes it a genuinely low-risk first step, which matters enormously when you're proposing it to a sceptical stakeholder.

**Distribution becomes an incremental, per-module decision rather than a big-bang architecture.** You extract the one module with the scaling problem. The other eleven stay in the monolith. This is the architecture most mature systems actually run and almost nobody draws on a slide: **a modular monolith plus two to five extracted services.**

Say this in a design round and you're immediately differentiated from the candidate who drew nine boxes.

---

## Concept 8 — The five properties of a real boundary

Whether a boundary is in-process or over a network, it is only real if it has all five of these. This list is the spine of Part C.

1. **It owns its data.** No other module reads or writes its storage. This is the one that actually holds (Concept 35).
2. **It exposes an explicit contract.** A small, deliberate, named public surface. Everything else is invisible to the outside — `internal` in .NET terms.
3. **Its internals are hidden.** You can change the entity model, the table layout, the algorithm, the library, without a single caller knowing.
4. **It can be tested independently.** You can run its tests without starting the rest of the system.
5. **It changes for its own reasons.** A business change in one capability does not require edits in another module. If every feature touches four modules, your boundaries are in the wrong place (Concept 57).

Apply these to a supposed microservice and you'll catch a distributed monolith in about ninety seconds. Apply them to a proposed module and you'll catch a bad decomposition before it costs anything.

Note what is *not* on the list: size, technology, repository location, whether it has its own container. Those are all consequences or irrelevancies.

---

## Concept 9 — Conway's law, and why it isn't optional

Melvin Conway, 1968: *organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations.* Fifty-eight years later it remains the single most reliable predictor of architectural outcomes.

The practical form: **a boundary that does not match an ownership boundary will erode.** Not "might" — will. If two teams share a module, they will both change it, they will both accommodate each other's needs in it, and within a year it will contain both teams' concerns. If one team owns four modules, the boundaries between those four will soften, because there is no organizational cost to crossing them and there is a deadline.

This gives you three tools:

- **The inverse Conway manoeuvre.** Deliberately restructure teams to produce the architecture you want. Powerful, slow, requires organizational authority — which is exactly why it comes up in architect interviews rather than senior IC ones.
- **The prediction.** Given a team structure, you can predict the architecture that will emerge and say so. *"With one team of eight, you'll get one deployable no matter what the diagram says — and that's fine, so let's make it a good one."*
- **The constraint on decomposition.** Don't propose more services than you have teams to own them. A team owning six services isn't getting autonomy; it's getting six deployment pipelines and six on-call surfaces for the same amount of autonomy it had with one.

The 2025 CNCF survey's finding that the top barrier to cloud native adoption is now **cultural (47%)** rather than technical is Conway's law showing up in survey data.

---

## Concept 10 — Cognitive load is the real constraint

*Team Topologies* (Skelton & Pais, 2019) supplies the theory that "two-pizza team" only gestures at. The core claim: **the limiting factor on a team's effectiveness is cognitive load, and the architecture should be chosen so that each team's domain fits within it.**

Three kinds of load:

- **Intrinsic** — the difficulty of the domain and the skills required. Reduce by training and hiring.
- **Extraneous** — the accidental complexity of the environment: build systems, deployment mechanics, environments, tooling. Reduce by platform work. *This is the load microservices increase.*
- **Germane** — the load of the actual business problem. This is what you want teams spending capacity on.

The architectural implication is direct: **a team should own a domain small enough that its members can hold it in their heads, and large enough that most changes stay inside it.** That's the sizing criterion Concept 5 was looking for.

It also explains something otherwise puzzling: why a team that adopts microservices often gets *slower*. They didn't add germane load; they added extraneous load (pipelines, environments, service discovery, distributed debugging) and took capacity away from the business problem. Unless a platform team absorbs that extraneous load — which is the *whole point* of the platform team pattern — the net is negative.

Interview use: *"Before choosing a topology I'd want to know the team structure, because the constraint that matters is how much each team can hold. Microservices shift load from 'understanding a big codebase' to 'operating a distributed system,' and that's only a good trade above a certain team count."*

---

## Concept 11 — The microservices premium

Martin Fowler's *MicroservicePremium* (2015) is the most useful single idea in this module, and it's one paragraph long.

The claim: microservices impose a **fixed productivity cost** — the premium — which you pay from day one. In exchange, they *slow the growth* of complexity cost as the system and organization grow. So you get two curves that cross:

- **Below the crossover** (simpler systems, smaller organizations), the monolith is more productive, and the gap is large.
- **Above the crossover**, microservices are more productive, and the gap grows.

Three consequences that a senior candidate should be able to state:

1. **The premium is paid immediately; the benefit arrives later, if at all.** A team that adopts microservices for a product that never reaches the crossover has simply been slower for its entire life.
2. **You don't know where the crossover is in advance**, which is why Fowler's own advice is to start with a monolith when you can't tell.
3. **The crossover is mostly about organizational size, not traffic.** A system serving a million requests per second with eight engineers is below it. A system serving a thousand requests per second with two hundred engineers is above it.

The related Fowler pieces are *MonolithFirst* and *Microservice Trade-Offs*, and his joint talk with Sam Newman, *When To Use Microservices (And When Not To!)*, is 30 minutes and the best free video on this subject.

---

## Concept 12 — MonolithFirst vs. "Don't start with a monolith"

There are two well-argued positions, both published on Fowler's own site, and knowing both is worth more than picking a side.

**MonolithFirst (Fowler, 2015).** Almost every successful microservices system started as a monolith that grew too big and was broken up. Almost every system built as microservices from scratch ran into serious trouble. The reason is boundary knowledge: you do not understand the domain well enough on day one to place boundaries correctly, and a wrong boundary inside a monolith is a refactor while a wrong boundary between services is a migration.

**Don't start with a monolith (Stefan Tilkov, 2015).** The counter-argument: a monolith does not naturally decompose. Its internal structure grows around the assumption that everything is in one process — shared entities, shared transactions, pervasive coupling — and the "we'll split it later" promise is rarely kept because the code actively resists it. If you *know* you need separate systems (distinct domains, distinct teams, distinct lifecycles), build them separately from the start.

**The synthesis, and the answer to give:** both are right about their own failure mode, and the reconciliation is the modular monolith. Start with one deployable — Fowler's point about boundary knowledge stands — but build it with boundaries strong enough that Tilkov's objection doesn't apply: separate schemas, no cross-module reads, contract-based communication, an outbox. That is precisely "pre-paying for extraction." It costs maybe 10–15% more up front and turns a rewrite into a deployment change.

The only case where Tilkov clearly wins outright is when the separation is known on day one for non-technical reasons — a regulated subsystem, an acquisition, a genuinely separate product with a separate team.

---

## Concept 13 — Three different units: failure, scale, release

Much of the confusion in this debate comes from bundling three questions that should be answered separately.

**Unit of failure.** What is the blast radius when something breaks? In a monolith, an unhandled exception in one feature can take down the process — but only if it's that kind of failure. Out-of-memory, thread-pool starvation (Module 15), a runaway query and an infinite loop are process-wide. A bad SQL query in module A degrades module B. If you need one capability to be unaffected by another's failure, that is a genuine argument for a process boundary.

**Unit of scale.** What do you replicate when load grows? A monolith scales as a whole: to give the image-processing endpoint more CPU, you add instances of everything. That's usually fine — instances are cheap and stateless scaling is simple (Module 6). It stops being fine when one capability's resource profile is wildly different from the rest: ten times the memory, a GPU, a 30-second average duration, or a 100× traffic spike that would require scaling the whole app to absorb.

**Unit of release.** What ships together? This is the organizational one, and it is the one that usually decides.

The senior move is to answer them separately and notice they often point different ways: *"Scale isn't the argument here — the whole app fits on three instances. The argument is release: four teams are queuing behind one pipeline. So I'd extract along team lines, not along the hot path."*

---

## Concept 14 — Reversibility is asymmetric

Every architecture decision should be evaluated on what it costs to be wrong, and this one is badly asymmetric.

**Monolith → services** is expensive but *tractable*: it is an incremental migration you can do one module at a time, with a strangler facade, with rollback at each step (Part E). Painful, months of work, but the path is known and each step is small.

**Services → monolith** is harder, and this surprises people. The problem is that the data has already diverged: two services that each own a `Customer` have two schemas, two sets of invariants, two sets of nulls and two histories of drift. Merging them means reconciling data, not just deleting HTTP clients. Add to that: the teams have reorganized around the services, the deployment pipelines assume them, and the "merge the services" project has no visible feature output to justify it to a stakeholder.

Therefore: **when you're uncertain, choose the architecture whose mistake is cheaper to undo.** That is the monolith, and it's a *decision-theoretic* argument rather than a taste one, which is why it lands well.

The related framing is Bezos's one-way/two-way doors. Splitting a modular monolith is close to a two-way door. Distributing a big ball of mud is a one-way door that you walk through backwards.

---

## Concept 15 — The five claims, graded

Here is the standard list of microservices benefits with an honest grade attached. Being able to do this out loud is a strong signal, because it shows you've actually operated these systems rather than read about them.

| Claim | Grade | The honest version |
|---|---|---|
| **Independent deployment** | **Real** | The genuine article, and the one benefit that cannot be had any other way. Worth real money above ~4–6 teams. |
| **Team autonomy** | **Real, conditional** | Real if — and only if — one team owns the service end to end, including its data and its on-call. Shared ownership gives you none of it. |
| **Fault isolation** | **Conditional** | True for process-level failures (OOM, crash, thread exhaustion). **False for synchronous dependency failures** — if A must call B to serve a request, B's outage is A's outage. You only get isolation if you also decouple asynchronously or degrade gracefully. |
| **Independent scaling** | **Usually irrelevant** | Genuinely valuable when one capability has a radically different resource profile. Otherwise adding monolith instances is cheaper than operating twelve services. Most teams that cite this have never measured which component is the bottleneck. |
| **Technology heterogeneity** | **Overrated, often harmful** | The ability to use Rust for one service and Python for another is real, and it's usually a net negative: more skills to hire for, more toolchains, more security patching, less mobility between teams. The defensible version is narrow — a specific library or runtime that only exists in one ecosystem. |

Two more that belong on the list and are usually forgotten: **independent failure domains for deployment risk** (a bad deploy takes out one capability, not the site) — real, and often the strongest practical argument; and **scaling the codebase itself** (build times, test times, IDE performance) — real above a few million lines, and increasingly addressable by other means.

---

## Concept 16 — The org-chart test

The single most decision-relevant question, and a good one to ask in a design round because it demonstrates that you know what actually drives this.

**"How many teams need to ship independently, and are they currently blocking each other?"**

Rough calibration — not laws, but defensible starting positions you can state and then justify:

- **1 team (up to ~8 engineers).** One deployable. There is nobody to be independent from. Microservices here are pure cost. Use modules inside one process.
- **2–4 teams (~10–30 engineers).** A modular monolith with clear per-team module ownership is usually still the best answer. Deploy contention exists but is manageable with a good pipeline and feature flags. Extract only where a specific trigger fires (Concept 65).
- **5–15 teams (~40–150 engineers).** The crossover zone. Deploy contention is real and expensive. The typical right answer is a modular monolith core plus extracted services along team boundaries, not a full decomposition.
- **15+ teams.** Independent deployability is now worth more than the distribution cost almost everywhere. Services along team boundaries, with a platform team absorbing the extraneous load.

The number that actually matters is not headcount but **deploy contention**: how often does a team have to wait for, coordinate with, or be blocked by another team to release? If the answer is "never," the premium is not earned no matter how many people you have. If the answer is "weekly, and it costs us days," you have your business case, in a form a manager can act on.

---

# Part B — The bill: what a network boundary costs

This is the part that makes your answer quantitative instead of aesthetic. "Distributed systems are hard" is a platitude. "This call goes from 1 nanosecond to 500 microseconds, our p99 becomes their p99.9, and our composed availability drops from three nines to two and a half" is an argument. Everything here is a line item you can put in front of a stakeholder.

---

## Concept 17 — The fallacies of distributed computing, as a bill

Peter Deutsch and James Gosling's eight fallacies (Sun Microsystems, 1994–1997) are usually presented as a list of things naive developers believe. More usefully, each is an invoice line that arrives the moment you cross a process boundary:

| Fallacy | What it costs you |
|---|---|
| The network is reliable | Retries, idempotency keys, at-least-once semantics, duplicate handling (Module 11) |
| Latency is zero | Every call adds hundreds of microseconds to milliseconds; loops become disasters (Concept 18) |
| Bandwidth is infinite | Payload size becomes a design constraint; chatty contracts become expensive (Concept 19) |
| The network is secure | mTLS, token propagation, service identity, secret rotation, a zero-trust posture |
| Topology doesn't change | Service discovery, health checks, connection management, DNS caching bugs |
| There is one administrator | Coordination across teams for any cross-cutting change |
| Transport cost is zero | Serialization, deserialization, TLS handshakes, allocation, GC pressure (Module 14/17) |
| The network is homogeneous | Version skew, protocol differences, client library differences |

The reason to memorize this framing rather than the list: it converts an abstract warning into a budget. In a design round, *"crossing that boundary means I now need idempotency on the consumer, a retry policy with jitter, and a timeout that's shorter than my caller's"* is a sentence that only somebody who has operated one of these systems says.

---

## Concept 18 — Six orders of magnitude

The numbers you should be able to produce without hesitating. These come from the estimation work in Module 5, applied here:

| Operation | Order of magnitude |
|---|---|
| Inlined/non-virtual method call | **~1 ns** or less |
| Virtual / interface dispatch (DI-resolved) | ~1–2 ns |
| Main memory reference | ~100 ns |
| In-process mediator dispatch with a few pipeline behaviours | ~1–10 µs |
| SSD random read | ~16 µs |
| Loopback HTTP call (same machine, keep-alive) | ~50–150 µs |
| **Round trip within the same datacenter** | **~500 µs** |
| Cross-AZ round trip | ~1–2 ms |
| Typical service-to-service HTTP/JSON call, warm, small payload | ~1–5 ms |
| Cross-region round trip (e.g. West Europe ↔ East US) | ~80–100 ms |

The headline: **a method call and an in-datacenter RPC differ by roughly six orders of magnitude** — a factor of about 500,000.

Why this matters practically, and the example to use: a loop that calls a collaborator once per item. With 1,000 items and an in-process call, that's about a microsecond in total and nobody notices. With 1,000 items and an RPC at 1 ms each, that's **one second**, serialized — and if the call is inside a request handler you've just turned a 20 ms endpoint into a 1-second endpoint and made your thread-pool situation considerably worse (Module 15). This is the N+1 problem (Module 19) with a network in the middle, and it is the single most common performance failure in a freshly split system.

The second-order effect is worse: it changes what code is *correct*. In-process, "just call the other module and get the current value" is fine. Over the network, that same line needs a timeout, a retry policy, a circuit breaker, a fallback, and a decision about what to do when it returns stale data. The line of code went from 1 line to 40, and from 1 ns to 1 ms.

---

## Concept 19 — Serialization is the hidden cost

Latency gets all the attention; serialization is what actually killed several of the famous case studies.

Every network call means: serialize the request (allocate), write to a socket, TLS-encrypt, transit, decrypt, deserialize (allocate), execute, and the whole thing again in reverse. For a small JSON payload this is tens of microseconds each way and a handful of allocations. For a large payload it is milliseconds and significant GC pressure — remember from Module 14 that anything over 85 KB lands on the Large Object Heap.

**The Prime Video case, read correctly.** In 2023 Amazon Prime Video's Video Quality Analysis team published a post about their audio/video monitoring service, reporting that moving from a distributed serverless architecture to a monolithic one **reduced infrastructure cost by over 90%** and increased scaling headroom. The internet turned this into "Amazon abandoned microservices." Here's what it actually says, and the version to give in an interview:

- It was **one component** of Prime Video (stream quality monitoring), not Prime Video itself. Prime Video remains a large distributed system.
- The original was **serverless orchestration** (AWS Step Functions + Lambda + S3), not a conventional microservices deployment. The bottleneck was Step Functions **account limits** on state transitions — the service performed multiple state transitions per second of video — and the **cost of moving video frames through S3** between components.
- They hit a hard ceiling at roughly **5% of expected load**.
- The fix was to **co-locate the processing in one process** so frames moved through memory instead of through object storage and the network.
- The team's own conclusion was explicitly **case-by-case**, not a general rule.

So the real lesson is precise and much more useful: **do not put a network boundary in the middle of a high-frequency, high-volume data path.** If two components exchange large payloads many times per unit of work, they belong in the same process. That's a data-gravity argument (Concept 24), and it generalizes to image pipelines, ETL, ML feature computation and anything else where the payload is the work.

If an interviewer cites Prime Video as evidence that microservices are over, this correction is one of the highest-value things you can say in the entire loop.

---

## Concept 20 — Tail-latency amplification

From Dean and Barroso's *The Tail at Scale* (CACM, 2013), and the reason fan-out architectures feel slow even when every service looks healthy on its own dashboard.

If a request must call **N** services in parallel and wait for all of them, the request is as slow as the **slowest** response. If each service independently has a 1% chance of exceeding its p99 latency, the probability that **at least one** of N is slow is `1 - 0.99^N`:

| Fan-out N | P(at least one call above its p99) |
|---|---|
| 1 | 1% |
| 5 | 4.9% |
| 10 | 9.6% |
| 20 | 18.2% |
| 50 | 39.5% |

Read the table the other way and it's alarming: **with a fan-out of 10, your composite p99 is roughly each dependency's p99.9.** Your services are meeting their SLOs and your users are not.

This has two design consequences. First, **fan-out is a latency multiplier and should be budgeted explicitly** — if you need a 200 ms p99 and you fan out to 8 services, each needs a p99.9 well under 200 ms, which is a much harder target than a p99. Second, the mitigations are known and worth naming: **hedged requests** (send to two replicas, take the first — Polly supports hedging, Module 25), **request timeouts tuned to the budget rather than to a round number**, **partial results / graceful degradation**, and **removing the fan-out** by precomputing or caching (Module 10).

Sequential chains are worse than parallel fan-out: a chain A → B → C → D has latency that *adds* and availability that *multiplies*, which is Concept 21.

---

## Concept 21 — Availability composition

The arithmetic that makes "more services = more resilient" false as stated.

If a request requires N services **in series**, and each is independently available with probability p, the composite availability is `p^N`:

| Services in series | Each at 99.9% | Composite | Downtime/year |
|---|---|---|---|
| 1 | 99.9% | 99.9% | 8.8 h |
| 3 | 99.9% | 99.70% | 26.3 h |
| 5 | 99.9% | 99.50% | 43.7 h |
| 10 | 99.9% | 99.00% | 87.6 h |
| 20 | 99.9% | 98.02% | 173.4 h |

So splitting a monolith with 99.9% availability into five synchronously-coupled services gives you **99.5%** — you have turned nine hours of annual downtime into forty-four. And that's before adding the network's own failure rate, which is not zero.

This is the single best counter to "microservices are more reliable." The honest statement is: **microservices change the shape of failure, and whether that's an improvement depends entirely on whether the dependencies are synchronous.**

The three ways to actually get resilience out of a decomposition, all of which are design work you must name:

1. **Asynchronous communication.** If A publishes an event and B consumes it later, B being down doesn't fail A's request; the message waits in the queue (Module 11). This converts a series dependency into no dependency at all for the write path.
2. **Graceful degradation.** If the recommendations service is down, render the page without recommendations. This converts a hard dependency into a soft one, and requires an explicit decision per dependency.
3. **Caching and fallbacks.** Serve the last known good value (Module 10). Converts an outage into staleness.

The senior formulation: *"Decomposition only improves availability if the dependencies are asynchronous or optional. Synchronous required dependencies multiply, so before I split anything I'd classify each cross-boundary call as required-sync, optional-sync, or async — and I'd want most of them in the third bucket."*

---

## Concept 22 — Partial failure: a new class of bug

In a single process, a call either returns or throws. In a distributed system there is a third outcome that has no in-process analogue: **you don't know.**

A request times out. Did the server not receive it? Receive it and crash before processing? Process it successfully and fail to return the response? Process it and the response was lost in transit? From the caller's position these are indistinguishable, and they require different reactions.

This single ambiguity generates the entire reliability toolkit:

- **Idempotency.** Since you must retry (you can't distinguish "lost request" from "lost response"), the receiver must tolerate duplicates. That means idempotency keys, deduplication windows, or naturally idempotent operations. It has to be designed in — it cannot be added later without touching every handler.
- **Timeouts everywhere.** Any call without a timeout is a call that can hang forever and exhaust your thread pool or connection pool (Modules 13, 15). A default `HttpClient` timeout of 100 seconds is far too long to be useful as a safety valve.
- **Retries with backoff and jitter.** Without jitter, retries synchronize and produce a thundering herd exactly when the downstream is weakest (Module 13).
- **Circuit breakers.** Stop calling a service that is failing, so you fail fast instead of consuming resources waiting (Modules 13, 25).
- **Bulkheads.** Isolate connection/thread pools per dependency, so one slow dependency can't consume all your capacity.
- **Distributed transaction alternatives.** Concept 23.

None of this exists in a monolith. All of it is mandatory the moment you distribute. **This, not latency, is the real price** — it is permanent extra complexity in every piece of code that crosses a boundary, and it never goes away.

---

## Concept 23 — Losing the transaction

In a monolith with one database, "reserve inventory and charge the card and create the order" is one transaction. It commits or it doesn't. The invariant is enforced by the database and you cannot get it wrong.

Split those across services with their own databases and that transaction is gone. Two-phase commit across HTTP services is not a real option (Module 12: coordinator failure blocks participants, it doesn't scale, and most modern data stores don't support it across heterogeneous resources anyway). What you get instead:

- **Sagas** — a sequence of local transactions with compensating actions for rollback (Module 12, Concept 87 here). Orchestrated (a coordinator drives the steps) or choreographed (each service reacts to events).
- **The transactional outbox** — the mechanism that makes "update my data and publish an event" atomic without distributed transactions (Module 11).
- **Eventual consistency**, with all the business-visible consequences: the order exists but the inventory hasn't decremented yet; the user sees their own write but a colleague doesn't for 200 ms.
- **Compensations that aren't inverses.** You cannot un-send an email, un-charge a card without a refund record, or un-ship a package. Compensation is a business process, not a technical rollback, and it has to be designed with the business.

The interview-grade statement: *"The moment inventory and payment are separate services, I lose the ability to enforce 'never oversell' with a transaction. I now need a reservation model with a timeout and a saga with compensations — and someone from the business has to decide what happens when the compensation runs. That's not a technical detail I can defer; it's a product decision the split creates."*

That answer scores because it connects a topology decision to a product consequence, which is precisely what architect interviews are scoring for.

---

## Concept 24 — Losing the join, and data gravity

The cost nobody budgets for, and the one that generates the most surprise work in the second quarter of a migration.

In a monolith, "show me all orders over £100 from customers in Serbia who joined this year, sorted by lifetime value" is a SQL query. The database planner optimizes it, an index serves it, and it returns in milliseconds. Split Customers and Orders into separate services with separate databases and that query is now **impossible to express**. Your options, all worse:

- **API composition.** Fetch candidate customers from one service, then their orders from the other, and join in application memory. Works for small result sets. Falls apart for anything requiring a full scan, sorting across sources, or pagination — you cannot paginate a join you're performing in memory over two paginated sources without fetching everything.
- **A replicated read model (CQRS).** Each service publishes events; a read-side projection maintains a denormalized view suitable for querying (Module 11, and Module 23/24). Correct, standard, and it costs you a whole new component with its own consistency lag, its own rebuild procedure, and its own bugs.
- **A data warehouse / lakehouse.** Ship everything to an analytical store (Fabric, Synapse, Snowflake, BigQuery) and do reporting there. This is where all of it ends up eventually. It is also a separate platform with a separate team and a separate latency budget (hours, not milliseconds).

**Data gravity** is the general principle: data attracts processing. It is far cheaper to move computation to data than data to computation. Two components that share a lot of data want to be in the same process; two that share almost none can be split cheaply. So when choosing what to extract first, **pick the module with the thinnest data relationship to everything else** — that's usually notifications, document generation, search indexing, or an integration with a third party — not the module in the middle of your domain.

---

## Concept 25 — The debugging tax

In a monolith, a production failure gives you a stack trace. It names the file and line. You attach a debugger locally, reproduce it, and fix it.

In a distributed system, a production failure gives you an error in service D, which was called by C, which was called by B, which was called by A, and the actual cause is that B's connection pool to its database was exhausted because a deployment of E three minutes earlier caused a retry storm. To find that you need:

- **Distributed tracing** with a trace ID propagated across every hop (W3C `traceparent`), so you can see the whole request as one waterfall. OpenTelemetry is the answer in .NET and it is non-negotiable (Modules 11, 28). Without it you are reading five log files and guessing.
- **Structured logs with correlation IDs**, centrally aggregated and searchable across services.
- **Metrics per service and per dependency**, so you can see which hop degraded and when.
- **Clock discipline**, because reasoning about ordering across machines using wall clocks is a trap (Module 9: this is exactly what logical clocks exist for).

And even with all of it, mean time to diagnosis goes up. This is a real, permanent, per-incident tax paid by people at 3 a.m. It is also the thing most often cited by teams who consolidated back.

The point for an interview is not "therefore don't distribute." It's: *"The observability stack is a precondition, not a follow-up. If we don't already have end-to-end tracing, that's the first thing I'd build — before the first extraction, not after."*

---

## Concept 26 — The testing tax and the end-to-end trap

Testing a monolith: run the test suite. It's in one process, you can use real objects, and the whole thing runs in seconds to minutes.

Testing a distributed system is a different discipline, and the trap is obvious in hindsight: teams try to keep the confidence they had by writing **end-to-end tests that spin up every service**. This fails predictably. Such a suite is slow (minutes to hours), flaky (any one of N services being slow fails the run), expensive to maintain, and — the killer — **it requires coordination to run, which destroys the independent deployability you did all this for**. If team A cannot deploy until the shared end-to-end suite passes, and that suite covers team B's service, you've rebuilt the release train.

The correct shape, which you should be able to state:

1. **Unit tests** — pure domain logic, no I/O, thousands of them, milliseconds.
2. **Service/module tests** — one service in-process with its real database (Testcontainers) and **fakes or stubs at every outbound boundary**. This is where most of your confidence comes from.
3. **Contract tests** — the replacement for integration testing across services. Consumer-driven contracts (Pact is the canonical tool) let the consumer declare what it needs, the provider verify it can deliver, and **neither has to run the other**. This is the single most important testing technique in a distributed system and the one most teams skip.
4. **A thin end-to-end smoke suite** — a handful of critical user journeys, run against a deployed environment, treated as a canary rather than a gate.
5. **Production testing** — synthetic monitoring, canary releases, feature flags, progressive rollout. Above a certain scale this replaces pre-production confidence, which is a genuinely uncomfortable but correct conclusion.

Aspire helps materially here: its test host can start the whole application graph for integration tests locally, which makes tier 2 cheap and honest.

---

## Concept 27 — The versioning tax

In a monolith, renaming a method is `F2` in the IDE. The compiler finds every caller. It ships atomically.

Across a service boundary, that same change is a protocol negotiation with yourself, forever:

1. Add the new field/endpoint alongside the old one. Deploy the provider.
2. Wait for every consumer to migrate. This can take weeks — it's not your team.
3. Verify no traffic hits the old path (you need telemetry for this, which means you had to instrument it).
4. Remove the old one. Deploy again.

That's the **expand/contract** pattern (Concept 70) and it applies to every breaking change, forever, for every boundary. It means:

- **Additive changes are cheap; removals and renames are expensive.** Your contracts accumulate deprecated fields.
- **Backward compatibility is a design constraint on every message and every endpoint** — which is why Protobuf's field-number discipline and JSON's tolerant-reader pattern exist.
- **Cross-cutting changes are brutal.** "Add a tenant ID to every request" is a two-hour change in a monolith and a quarter-long programme across twenty services.
- **You cannot refactor across boundaries.** The single most valuable technique for improving a codebase stops working at the service edge. This is the deepest argument for getting boundaries right before distributing: after distribution, moving a boundary is a project, not a refactor.

---

## Concept 28 — The operational floor

Before you deploy your **second** service, the following must already exist, or you will build them under pressure during an incident. This list is one of the most practically useful things in the module, because it converts "are we ready?" from a feeling into a checklist.

1. **Automated deployment pipeline**, per service, with no manual steps. If deploying is manual, N services means N times the manual work, and people will batch releases — which kills independent deployability.
2. **Container registry and image strategy**, with reproducible builds and vulnerability scanning.
3. **Centralized structured logging** with correlation IDs, searchable across services.
4. **Distributed tracing** — OpenTelemetry, end to end, instrumented from the first service.
5. **Metrics and dashboards** per service: rate, errors, duration, saturation; plus SLOs where they matter (Module 28).
6. **Alerting with clear ownership.** Who gets paged for this service at 3 a.m.? If the answer is "the platform team," you don't have service ownership; you have a shared operational burden with extra network calls.
7. **Centralized configuration and secrets** — App Configuration and Key Vault on Azure — with per-service scoping.
8. **Service discovery and routing.** Container Apps and AKS supply this; in App Service you're doing it with configuration.
9. **A contract-testing approach**, agreed before the second service, not after the first breakage.
10. **A local development story.** If a developer needs eight services running to work on one, velocity dies. This is precisely the problem Aspire was built to solve, and it's the strongest practical reason to adopt it early.
11. **An on-call rotation and incident process** that assumes distributed failure.

If more than two or three of these are missing, the honest recommendation is: **build the platform floor first, stay modular in the meantime.** Saying that in an architect interview is a strong signal of operational maturity — it's the answer that has been through a real migration.

---

## Concept 29 — The cost model

Two components, and the second one is the one that gets underestimated.

**Infrastructure.** Goes up, but sub-linearly and often less than people fear. You still serve the same traffic; you're partitioning it differently. The real increases come from: minimum instance counts (a monolith at 3 instances becomes 12 services × 2 instances = 24 minimum, most of them idle), per-service overhead (each has its own runtime, connection pools, telemetry agent and base memory footprint — on .NET call it 80–150 MB per instance before your code), duplicated caches, cross-AZ and egress traffic charges, and one database per service where you had one. Expect something in the range of 1.5×–4× for equivalent functionality, depending heavily on how aggressive you are about scale-to-zero (Container Apps and Functions help; AKS node pools don't, unless carefully managed).

**People.** This is the dominant term, and it's the one to put in front of an executive. You need: a **platform capability** (someone owns the pipelines, the mesh or its absence, the observability stack, the base images) — realistically 1–3 engineers at minimum, permanently; more **on-call** surface; and a permanent per-change tax from Concepts 25–27 (slower debugging, contract versioning, coordination on cross-cutting changes).

The framing that works with a CFO or a VP Engineering: *"Below roughly five teams, decomposition costs us about one to two engineers' worth of permanent capacity and buys us release independence we aren't currently blocked by. Above it, the deploy contention costs more than that. Here's our current contention number — that's the thing I'd measure before committing."*

Be careful with published cost multipliers. Figures like "microservices cost 3.75×–6× more" circulate widely in 2026 content with no primary source attached; see Concept 96. Use your own numbers — a real instance count and a real Azure price list beat a borrowed statistic every time.

---

## Concept 30 — What you actually gain, priced

To close the bill honestly, the credit column. These are real and worth real money in the right context.

**Deployment risk isolation.** A bad deploy affects one capability, not the whole site. If you deploy 50 times a week, this compounds fast. Arguably the most underrated genuine benefit, because it shows up as *fewer full outages* rather than as a feature.

**Release independence.** Team A ships Tuesday, team B ships Wednesday, neither waits. The value is proportional to the number of teams and the frequency of releases, which is why Concept 16's contention question is the decisive one.

**Scale isolation for genuinely asymmetric workloads.** When one capability needs GPUs, or 30× the memory, or absorbs a 100× spike, running it separately is not architecture fashion — it's the obviously correct answer.

**Fault isolation for process-level failures.** An OOM or thread-pool exhaustion in the report generator no longer takes out checkout. Real, but only with async or optional dependencies (Concept 21).

**Technology fit where it genuinely matters.** A specific ML runtime, a specific driver, a specific licensing constraint. Narrow, but occasionally decisive.

**Codebase scaling.** Build times, test times, IDE responsiveness, merge conflicts, and the cognitive load of a single repository past a few million lines.

**Organizational clarity.** Ownership becomes physical rather than conventional. This partly overlaps with modularity — but a service with its own pipeline and its own pager is owned in a way a folder never is, and that is a real difference Conway's law predicts.

The summary line worth memorizing: **you are buying autonomy, and paying in consistency, latency and operational complexity.** If autonomy is not currently your bottleneck, you are paying for something you don't need.

---

# Part C — The modular monolith, built properly in .NET

Part A said modularity is the valuable axis. Part B priced the other one. Part C is the engineering: what a modular monolith actually *is* as a .NET solution, project by project, rule by rule. This is the part that turns "modular monolith" from a slogan into something you can describe at a whiteboard in enough detail that the interviewer believes you've built one.

The load-bearing idea throughout: **build it so that extraction is a deployment change, not a rewrite.**

---

## Concept 31 — The modular monolith, defined

**A modular monolith is a single deployable unit whose internal structure is a set of modules with strictly enforced boundaries, where each module owns its own data and communicates with other modules only through explicit contracts.**

Three non-negotiables. If any one is missing, you have a monolith with folders:

1. **Each module owns its data.** No module reads or writes another module's tables. No cross-module foreign keys. No cross-module joins.
2. **Each module exposes a deliberate public contract** and hides everything else. In .NET this is `internal` by default plus a small public surface.
3. **Cross-module communication goes through that contract** — a published interface call or an event — never a direct reference to an internal type, an entity, or a `DbContext`.

Kamil Grzybek's formulation, from the *Modular Monolith: A Primer* series and the `kgrzybek/modular-monolith-with-ddd` reference implementation (13k+ stars, MIT), is the canonical .NET statement of this, and his phrasing of the domain-model rule is exactly right: **all members are private by default, then internal — public only at the very edge.**

What you get: the boundaries and ownership of microservices, the simplicity and transactional integrity of a monolith, one process to deploy, one to debug, one stack trace, one transaction when you need it, and — critically — **the option to extract any module later at a known cost**.

What you don't get: independent deployment, runtime isolation, independent scaling, or independent technology choice. Concept 51 is honest about which of those absences actually bites.

---

## Concept 32 — Solution shape

The layout, and the reasoning behind each piece:

```
src/
  Modules/
    Ordering/
      Ordering.Domain/               → entities, VOs, domain events, invariants
      Ordering.Application/          → use cases, ports, module-internal contracts
      Ordering.Infrastructure/       → EF Core, adapters, module registration
      Ordering.PublicApi/            → THE public contract (tiny)
      Ordering.IntegrationEvents/    → published event contracts (tiny, versioned)
    Billing/
      Billing.Domain/ …              → same shape
    Shipping/
      …
  Shared/
    SharedKernel/                    → IDs, Money, Result, base types — no business rules
    Infrastructure.Common/           → cross-cutting host plumbing only
  Host/
    Api/                             → composition root, one process, one deploy
tests/
  Ordering.Domain.Tests/             → no doubles at all
  Ordering.Module.Tests/             → module in-process, real DB, fakes at the edges
  Architecture.Tests/                → the rules, enforced (Concept 48)
```

The reference rules, which are the architecture:

- `Host/Api` references **every module's Infrastructure** (composition root — Module 20, Concept 19) and nothing else special.
- `Ordering.Infrastructure` → `Ordering.Application` → `Ordering.Domain`. Clean Architecture *inside* each module (Module 20, Concept 57: modules first, layers second).
- `Ordering.Application` may reference **`Billing.PublicApi`** and **`Billing.IntegrationEvents`** — and *nothing else* of Billing. It must never reference `Billing.Domain`, `Billing.Application`, or `Billing.Infrastructure`.
- Every module may reference `SharedKernel`. `SharedKernel` references nothing.
- **No module references another module's Domain, Application or Infrastructure. Ever.** This is the rule an architecture test enforces (Concept 48) and it is the one that makes everything else possible.

Two practical notes. If a module is small, collapse it — `Ordering` and `Ordering.PublicApi` is a perfectly good two-project module, and four projects per module × eight modules is 32 projects and a slow build. Buy the walls you need (Module 20, Concept 22). And the `.slnx` XML solution format supported by current SDKs is genuinely useful here: when the solution file *is* the architecture, being able to review a change to it in a pull request is a control.

---

## Concept 33 — The public surface: `internal` by default

.NET gives you the single cheapest boundary-enforcement mechanism of any mainstream platform, and most teams don't use it: **assembly-level visibility**.

```csharp
// Ordering.Application — everything internal
internal sealed class PlaceOrderHandler(IOrderRepository orders, TimeProvider clock)
{
    internal async Task<Result<OrderId>> Handle(PlaceOrderCommand cmd, CancellationToken ct) { … }
}

// Ordering.PublicApi — the entire surface another module may see
public interface IOrderingModule
{
    Task<OrderSummary> GetOrderSummaryAsync(Guid orderId, CancellationToken ct);
    Task<Result<Guid>> PlaceOrderAsync(PlaceOrderRequest request, CancellationToken ct);
}

public sealed record OrderSummary(Guid Id, string Status, decimal Total, string Currency);
public sealed record PlaceOrderRequest(Guid CustomerId, IReadOnlyList<OrderLine> Lines);
```

The rules that make this work:

- **Every type in a module is `internal` unless it is part of the contract.** Default to `internal sealed`.
- **The public contract lives in its own tiny assembly** (`Ordering.PublicApi`). This is what makes the boundary *reviewable*: a pull request that adds a public type to that project is visible and arguable. A pull request that adds a `public` keyword somewhere in a 200-file module is not.
- **Contract types are plain DTOs or records — never domain entities.** If `OrderSummary` were the `Order` aggregate, Billing would now depend on Ordering's domain model, and any change to that model would ripple. This is the same rule as "never publish a domain entity as an integration event" (Module 20, Concept 45).
- **Tests get in via `InternalsVisibleTo`**, set once in the module's `.csproj`:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="Ordering.Module.Tests" />
</ItemGroup>
```

This is *free* enforcement: the compiler does it, there is no CI step, and there is no way to accidentally violate it. It is strictly better than a convention, a folder, or a code review.

---

## Concept 34 — The host and the module registration contract

The host is the composition root, and it is the only place that knows every module exists. Modules must not know about each other's registration.

Give every module the same two-method shape:

```csharp
// Ordering.Infrastructure
public static class OrderingModule
{
    public static IServiceCollection AddOrderingModule(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<OrderingDbContext>(o =>
            o.UseSqlServer(config.GetConnectionString("Default"),
                sql => sql.MigrationsHistoryTable("__EFMigrationsHistory", "ordering")));

        services.AddScoped<IOrderRepository, OrderRepository>();   // internal impls
        services.AddScoped<IOrderingModule, OrderingModuleFacade>(); // the public contract
        services.AddOptions<OrderingOptions>()
                .Bind(config.GetSection("Modules:Ordering"))
                .ValidateDataAnnotations()
                .ValidateOnStart();
        return services;
    }

    public static IEndpointRouteBuilder MapOrderingEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders").WithTags("Ordering");
        group.MapPost("/", PlaceOrderEndpoint.Handle);
        group.MapGet("/{id:guid}", GetOrderEndpoint.Handle);
        return app;
    }
}
```

```csharp
// Host/Api/Program.cs — the entire architecture, visible in twelve lines
builder.Services
    .AddOrderingModule(builder.Configuration)
    .AddBillingModule(builder.Configuration)
    .AddShippingModule(builder.Configuration);

var app = builder.Build();
app.MapOrderingEndpoints()
   .MapBillingEndpoints()
   .MapShippingEndpoints();
app.Run();
```

Why this specific shape matters:

- **Registration lives in the module**, so the module owns its own wiring and its implementations stay `internal` (Module 20, Concept 20 — this is not a dependency cycle; the host is allowed to know everything).
- **`Program.cs` is a readable index of the system.** A new engineer reads twelve lines and knows what the system is made of.
- **Extraction is now mechanical**: delete one `Add…Module()` / `Map…Endpoints()` pair from this host, create a new host project that calls only that pair, and the module runs as a service. The module's code does not change. That's the payoff for all of Part C.
- Turn on **`ValidateOnBuild` and `ValidateScopes`** in the host so wiring mistakes become startup failures rather than 3 a.m. surprises (Module 18).

---

## Concept 35 — Schema ownership: the boundary that holds

If you remember one implementation detail from this module, this is it.

**Code boundaries erode. Data boundaries don't.**

A code boundary is enforced by discipline and, if you're diligent, by a build. Under deadline pressure, someone adds a `public` keyword, someone adds a project reference "just for this one thing," and six months later the boundary is decorative. This happens in every organization, and being cynical about it is realism, not pessimism.

A data boundary is different. If `Billing` physically cannot read `ordering.Orders` — because it uses a connection whose SQL login has no permission on that schema — then it cannot, regardless of deadlines. The boundary is enforced by the database, which does not care about your sprint.

Concretely, in one SQL Server or PostgreSQL database:

```csharp
// OrderingDbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDefaultSchema("ordering");
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(OrderingDbContext).Assembly);
}
```

And then the part most teams skip, which is what makes it real:

```sql
CREATE LOGIN ordering_app WITH PASSWORD = '…';
CREATE USER ordering_app FOR LOGIN ordering_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::ordering TO ordering_app;
DENY SELECT ON SCHEMA::billing TO ordering_app;
```

Each module gets its own connection string with its own credentials scoped to its own schema. Now "Billing reads Ordering's tables" is not a code-review conversation — it's a runtime permission error in the developer's first test run, which is the cheapest possible place to find it.

This single practice is what separates a modular monolith you can extract from one you can't. It costs an afternoon to set up. It's also the highest-value thing you can say in an interview on this topic, because it demonstrates that you know which boundaries survive contact with a real team.

---

## Concept 36 — One database, many schemas — and when to split

The default: **one database instance, one schema per module, one credential per module.** Reasons:

- One backup, one restore, one HA configuration, one connection-pool budget, one cost line.
- You *can* still run a transaction across modules when you genuinely need to — and occasionally you genuinely need to, during a migration or for a data-fix job. Keeping that capability while forbidding its routine use is a pragmatic position, not a compromise.
- Schema-per-module gives you the boundary without the operational multiplication.

When to move a module to a **separate database**, before extracting it to a service:

- **Different storage engine genuinely fits better.** The catalogue wants a document store, the ledger wants a relational one, the session store wants Redis (Module 12).
- **Different scaling or availability profile.** One module needs a geo-replicated premium tier; the rest are fine on a cheap one.
- **Compliance isolation.** PII, PCI or regional data-residency requirements that are easier to satisfy with physical separation.
- **You're about to extract it**, and separating the database is step one (Concept 71).

The order of separation is always the same: **schema → database → process.** Each step is independently reversible and independently valuable, and the hard part (the data) is dealt with before the visible part (the deployment).

---

## Concept 37 — No cross-module joins, FKs, or entity references

The three specific prohibitions, and what to do instead.

**No cross-module joins.** `SELECT … FROM ordering.Orders o JOIN billing.Invoices i ON …` is forbidden even though the database would happily run it. The moment that query exists, the two modules cannot be separated without rewriting it, and nobody will find it until migration day.

**No cross-module foreign keys.** `ordering.Orders.CustomerId` does *not* get a foreign key to `customers.Customers.Id`. Instead, `CustomerId` is a plain `uniqueidentifier` — a reference by identity, with referential integrity enforced by the application at the point of use (the command handler checks the customer exists via the Customers module's contract before creating the order). This feels wrong to anyone with a DBA background, and it is exactly the trade you're making: you give up database-enforced referential integrity across modules to keep them separable. Within a module, foreign keys are normal and correct.

**No cross-module entity references.** `Order` does not have a `public Customer Customer { get; }` navigation property to the Customers module's entity. It has a `CustomerId`. This is the same rule as "reference other aggregates by ID" from Module 20 (Concept 36) and DDD generally (Module 22) — scaled up from aggregate to module.

What you do instead is Concept 38. And note the cost honestly: you have traded some query convenience and some database-enforced integrity for separability. If you will *never* separate these modules, that trade is a bad one — which is a legitimate reason to make two things one module rather than two.

---

## Concept 38 — Cross-module reads: the three legal options

When Ordering needs data owned by Customers, there are exactly three acceptable answers, in order of preference.

**1. Call the other module's public contract.**

```csharp
internal sealed class PlaceOrderHandler(ICustomersModule customers, IOrderRepository orders)
{
    public async Task<Result<OrderId>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await customers.GetCustomerAsync(cmd.CustomerId, ct);
        if (customer is null) return Result.Fail<OrderId>(OrderErrors.UnknownCustomer);
        …
    }
}
```

In-process this is a method call: nanoseconds, no serialization, no failure mode. After extraction the implementation of `ICustomersModule` becomes an HTTP or gRPC client and the handler **does not change** — which is exactly why the contract is defined in terms of your needs, in your vocabulary, rather than exposing the other module's model.

Two disciplines that make this survive extraction: **give every contract method a `CancellationToken` and an async signature even though it's synchronous today** (Module 20, Concept 50), and **never call a module's contract inside a loop**. The loop is free today and a catastrophe after extraction (Concept 18). Batch the call: `GetCustomersAsync(IReadOnlyList<Guid> ids, …)`.

**2. Keep your own replicated copy, fed by events.** Ordering subscribes to `CustomerRegistered` and `CustomerAddressChanged`, and maintains a small `ordering.CustomerSnapshot` table with only the fields it needs. Reads become local, there is no runtime coupling at all, and the data is eventually consistent — which is usually fine for a name and an address, and not fine for a credit limit you're about to enforce. This is the pattern that scales best and the one people are most squeamish about (Concept 41).

**3. A dedicated read model / projection** for genuinely cross-cutting queries — a reporting view assembled from events, owned by whoever owns the report. The one place a cross-module query is legitimate is a **read-only projection that is explicitly not part of any module's write path**, and is rebuildable from the source of truth.

What is *not* acceptable: reading the other module's tables, referencing its `DbContext`, or reaching in with an `IQueryable`. That's Module 20's Concept 32 (`IQueryable` leakage) at module scale — it exports the other module's storage semantics permanently.

---

## Concept 39 — Cross-module writes: events, not calls

Reads can go either way. **Writes should be asynchronous, via events**, and the reason is structural rather than stylistic.

If Ordering synchronously commands Billing to create an invoice, three things follow: Ordering knows Billing exists and knows what Billing does (behavioural coupling); Ordering's transaction now spans two modules, so either they share a transaction (which prevents extraction) or you have a dual-write problem *today*, in one process (Module 11); and after extraction Ordering's availability becomes Ordering × Billing (Concept 21).

If Ordering instead publishes `OrderPlaced` and Billing subscribes, then Ordering knows nothing about Billing, Billing can be added or removed without touching Ordering, each module commits its own transaction, and after extraction nothing about the flow changes except which process handles the message.

The rule of thumb: **a module publishes facts about itself; it does not issue commands to others.** `OrderPlaced` is a fact. `CreateInvoice` is a command, and a command crossing a module boundary is a signal that the boundary is in the wrong place — either the behaviour belongs to the caller, or the two modules are one module.

The exception worth naming: **when the caller needs a synchronous decision**, such as "authorize this payment before I accept the order." That genuinely requires a request/response call, and it is fine — but flag it as a hard dependency and design the failure path (Concept 21) rather than pretending it's decoupled.

---

## Concept 40 — The in-process outbox: pre-pay for extraction

The mechanism that makes "we can extract later" true rather than aspirational.

Even with everything in one process, write integration events to an **outbox table in the publishing module's own schema, in the same transaction as the state change** (Module 11):

```csharp
// Inside the module's transaction — atomic with the business change
public async Task<Result<OrderId>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Place(cmd.CustomerId, cmd.Lines, _clock.GetUtcNow());
    _db.Orders.Add(order);
    _db.OutboxMessages.Add(OutboxMessage.From(
        new OrderPlacedIntegrationEvent(order.Id, order.CustomerId, order.Total)));
    await _db.SaveChangesAsync(ct);   // one transaction, no dual write
    return order.Id;
}
```

A background dispatcher (one per module, a plain `BackgroundService`) polls the outbox and hands messages to a bus. **Today** that bus is an in-process dispatcher that invokes subscribed handlers in other modules. **After extraction** it's Azure Service Bus or RabbitMQ, and the only thing that changes is one registration line in the host.

Why this is worth the cost up front:

- It makes the publish **atomic with the state change** — no lost events, no phantom events, which is the dual-write problem solved once rather than discovered later (Module 11).
- It makes consumers **asynchronous from day one**, which means your handlers are already written to be idempotent, to tolerate out-of-order delivery, and to retry. Retrofitting that to handlers written as synchronous in-process calls is the expensive part of a real extraction.
- It gives you **at-least-once delivery semantics now**, so eventual-consistency bugs surface during development rather than after the split.
- It makes the extraction genuinely a configuration change.

The cost is a table, a background service and the discipline of idempotent consumers. That is the cheapest insurance policy in this entire module.

---

## Concept 41 — Duplication across modules is a feature

The hardest thing to accept, and the one that most reliably separates people who have built these from people who haven't.

Ordering has a `Customer` with `Id`, `Name`, `ShippingAddress`. Billing has a `Customer` with `Id`, `LegalName`, `VatNumber`, `BillingAddress`, `PaymentTerms`. Support has a `Customer` with `Id`, `Name`, `Email`, `Tier`, `OpenTicketCount`.

To a DRY instinct this is three copies of one thing and an obvious refactor. It is not. **They are three different concepts that share an identifier.** This is exactly what a bounded context is (Module 22): the same word means different things in different parts of the business, and forcing them into one model gives you a class with thirty properties, two-thirds of which are null in any given usage, that every team must change and nobody owns.

The rules:

- **Duplicate the *shape*, share the *identity*.** Everyone agrees on `CustomerId`. Nobody agrees on what a customer *is*, and they shouldn't have to.
- **One module is the source of truth per fact.** Customers owns name and email. Billing owns VAT number and payment terms. Nobody owns "the customer."
- **Replicated data flows by event** and is treated as a cache of someone else's truth — stale by definition, never authoritative, rebuildable.
- **DRY applies to knowledge, not to text.** Two identical-looking classes that change for different reasons are not duplication; they're coincidence. Module 20's Concept 72 (the wrong abstraction) is the general form of this, and Sandi Metz's line applies at module scale too: duplication is far cheaper than the wrong abstraction.

Be ready for the pushback, because it always comes: *"Why do we have three Customer classes?"* The answer is one sentence: *"Because merging them would mean every change to any team's customer concept requires agreement from all three teams — the duplication is what buys independent change."*

---

## Concept 42 — The shared kernel, kept tiny

You do need *some* shared code. The failure mode is that "Shared" becomes the place where everything that's awkward goes, and within a year every module depends on it, every change to it ripples everywhere, and you have a distributed monolith's coupling profile in one process.

**Belongs in `SharedKernel`:**
- Strongly-typed ID base types and the `Guid.CreateVersion7()` generation helper.
- `Money`, `DateRange`, `Percentage` — universal value objects with no business policy in them.
- `Result<T>` / `Error` types, if you use them (Module 20, Concept 40).
- Base classes: `Entity`, `AggregateRoot`, `ValueObject`, `IDomainEvent`.
- Marker interfaces for the module registration convention.

**Does not belong, ever:**
- Anything with a business rule. `Order`, `Customer`, `Invoice`, `PricingPolicy` — no.
- A shared `DbContext` or any EF configuration.
- A shared `enum` that encodes a business concept several modules interpret differently (`OrderStatus` is the classic — Ordering's statuses and Shipping's statuses are different vocabularies).
- Utility grab-bags: `StringExtensions`, `Helpers`, `Common`.

**The test:** if a change to `SharedKernel` could require a business conversation, it doesn't belong in `SharedKernel`. A useful guardrail is a hard size budget — a few hundred lines — enforced by an architecture test that fails when the assembly's type count crosses a threshold. It sounds crude, and it works, because it forces a conversation at exactly the moment the drift begins.

---

## Concept 43 — Migrations per module

Each module owns its schema, therefore each module owns its migrations. Three pieces:

**Separate `DbContext` per module** (already implied by schema ownership).

**Separate migrations history table per schema**, so `dotnet ef migrations` operations on one module never see or touch another's:

```csharp
options.UseSqlServer(connectionString, sql =>
    sql.MigrationsHistoryTable("__EFMigrationsHistory", "ordering"));
```

**Separate migration assemblies** so each module's migration files live with the module:

```csharp
sql.MigrationsAssembly(typeof(OrderingDbContext).Assembly.FullName);
```

Operational rules that follow (Module 19):

- **Apply migrations from the pipeline, not from application startup.** `Database.Migrate()` at startup is a race across instances and a rollback hazard; generate an idempotent script (`dotnet ef migrations script --idempotent`) and run it as a deployment step. With Aspire, the common pattern is a dedicated migration worker resource that the AppHost runs before the app.
- **Schema changes must be backward compatible** with the currently-deployed code, because you will have both versions running during a rolling deploy. This is the expand/contract discipline (Concept 70) — and practising it in the monolith is exactly the muscle you need after extraction.
- **One module's migration must never alter another module's schema.** Enforce it with the same credential scoping as Concept 35.

---

## Concept 44 — Configuration per module

Each module binds its own options section and validates at startup:

```csharp
services.AddOptions<OrderingOptions>()
        .Bind(config.GetSection("Modules:Ordering"))
        .ValidateDataAnnotations()
        .ValidateOnStart();
```

```json
{
  "Modules": {
    "Ordering": { "MaxLinesPerOrder": 200, "ReservationTimeout": "00:15:00" },
    "Billing":  { "InvoiceDueDays": 30, "Currency": "EUR" }
  }
}
```

Three benefits worth naming: a module's configuration is **discoverable** (one section, one options class), **validated at startup** rather than on first use in production, and **portable** — when the module becomes a service, the section becomes the service's whole configuration file with no restructuring. Keep connection strings module-scoped too (`ConnectionStrings:Ordering`), even if they currently point at the same server.

---

## Concept 45 — Endpoints per module

Each module registers its own routes; the host just composes them (see Concept 34's `MapOrderingEndpoints`). A few disciplines:

- **Group per module** with a consistent prefix (`/api/orders`, `/api/billing`) — which means the eventual gateway routing rule is a one-line prefix match rather than a per-endpoint mapping exercise.
- **Endpoint classes are `internal`** and take dependencies as parameters (minimal APIs bind from DI automatically — Module 18).
- **Request/response contracts belong to the module's API layer**, not to its domain and not to `SharedKernel`.
- **Cross-cutting concerns stay in the host pipeline** — authentication, correlation ID propagation, exception handling with `ProblemDetails`, rate limiting. Modules should not each implement their own.
- If two modules need to appear under one resource path for API-consumer reasons, that's a **composition endpoint in the host**, explicitly owned by nobody's domain, calling both modules' public contracts. Keep it thin and keep business logic out of it (this is a BFF in miniature — Concept 81).

---

## Concept 46 — Background work per module

A `BackgroundService` is a driving adapter (Module 20, Concept 46), so it belongs to the module whose use cases it invokes:

```csharp
// Ordering.Infrastructure — internal, registered by AddOrderingModule
internal sealed class ExpireReservationsWorker(
    IServiceScopeFactory scopes, TimeProvider clock, ILogger<ExpireReservationsWorker> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopes.CreateScope();          // one scope per unit of work
            var handler = scope.ServiceProvider.GetRequiredService<ExpireReservationsHandler>();
            await handler.Handle(ct);
            await Task.Delay(TimeSpan.FromMinutes(1), clock, ct);
        }
    }
}
```

Rules: one **scope per unit of work**, not per worker lifetime (a worker lives for the process lifetime, so a scoped `DbContext` captured at construction is a memory leak and a stale-data bug — Module 18); the worker calls the module's own use cases and **never** another module's internals; and it respects the host's cancellation token so shutdown is clean.

The extraction consequence: when the module becomes a service, its workers move with it. If a worker touches three modules, it belongs to none of them and will block your extraction — split it.

---

## Concept 47 — Testing a module in isolation

The payoff of all this structure is that a module can be tested as if it were already a service:

1. **Domain tests** — `Ordering.Domain.Tests`. No doubles, no I/O, no framework. Thousands of them, milliseconds to run (Module 20, Concept 64).
2. **Module tests** — start *only* this module's services, with a real database (Testcontainers) for its schema, and **fakes for every other module's public contract**:

```csharp
services.AddOrderingModule(config);
services.AddSingleton<ICustomersModule, FakeCustomersModule>();  // 30 lines, never lies
services.AddSingleton<IBillingModule, FakeBillingModule>();
```

This is the same test you'd write if Customers were a remote service — which means the day it becomes one, this test doesn't change.
3. **Contract tests between modules.** In-process these can be simple: a test asserting that `FakeCustomersModule` and the real `CustomersModule` satisfy the same behavioural contract. That's a cheap version of consumer-driven contract testing (Concept 72) and it stops your fakes from drifting into fiction — the failure mode Module 20's Concept 65 warns about.
4. **A thin host-level test** with `WebApplicationFactory` that boots everything and hits a few journeys, plus a container-validation test (`ValidateOnBuild` + `ValidateScopes`).

Notice what's absent: a slow, flaky, everything-at-once integration suite. The modular structure is what buys you the ability not to need one.

---

## Concept 48 — Enforcement: machine-checked or fictional

Module 20's Concept 61 gave the enforcement ladder. Applied to modules, in order of feedback speed:

**1. Project references.** The strongest and cheapest control. If `Ordering.Application` has no reference to `Billing.Application`, the coupling is impossible, not merely discouraged. Review every `.csproj` change.

**2. `internal` by default.** Free, compiler-enforced (Concept 33).

**3. Banned-API analyzers.** `Microsoft.CodeAnalysis.BannedApiAnalyzers` with a per-project `BannedSymbols.txt`, promoted to an error. Catches things at typing time, which is the cheapest possible moment:

```
# Ordering.Domain/BannedSymbols.txt
T:System.DateTime;Use TimeProvider — inject the clock
M:System.Guid.NewGuid();Use Guid.CreateVersion7() via the ID generator
T:Microsoft.EntityFrameworkCore.DbContext;Domain must not know about EF
```

**4. Architecture tests.** `ArchUnitNET` (actively developed) or `NetArchTest.eNhancedEdition` (the maintained MIT fork — the original `NetArchTest.Rules` 1.3.2 from May 2021 is effectively frozen). The six rules to write first:

```csharp
// 1. Module isolation — the rule that makes extraction possible
[Fact]
public void Modules_do_not_reference_each_others_internals()
{
    foreach (var (name, asms) in Modules)
        Types.InAssemblies(asms)
             .Should().NotHaveDependencyOnAny(OtherModulesInternalNamespaces(name))
             .Check(Architecture);
}

// 2. Cross-module contact is only via PublicApi / IntegrationEvents
// 3. Domain depends on nothing outside itself and SharedKernel
// 4. Nothing references the Host
// 5. Only Host references any module's Infrastructure
// 6. SharedKernel references nothing and stays under N types
```

**5. Database permissions** (Concept 35) — the only control on this list that cannot be bypassed by a determined engineer under deadline pressure.

**6. Review and ADRs** for the judgment calls a machine can't check.

Two warnings from Module 20's Concept 62, both of which apply doubly here: an architecture test written against a namespace string **passes silently when the namespace is renamed and the rule matches zero types** — so assert that each rule matched something; and arch tests run in CI, which is late. Push each rule down to the fastest loop that can hold it.

---

## Concept 49 — In-process messaging options, and their licences

For module-to-module dispatch inside one process, as of September 2026:

| Option | Licence | When it fits |
|---|---|---|
| **A hand-rolled dispatcher** | Yours | 40–80 lines: a dictionary of handlers resolved from DI, a pipeline of decorators. For a system with a dozen event types this is genuinely the right answer and has zero licensing, versioning or upgrade risk |
| **Wolverine** | MIT | Mediator *and* messaging in one library, with a built-in outbox, sagas and convention-based handlers. The strongest free option in 2026, and the natural fit for "in-process now, broker later" because the handler code doesn't change |
| **`System.Threading.Channels`** | BCL | The right primitive for in-process producer/consumer queues (background work, batching) rather than for request dispatch. No dependency at all (Module 15) |
| **MediatR 13+** | **Commercial** (RPL-1.5 or paid; free Community tier below revenue/capital thresholds) | Mature, ubiquitous, huge community. Now a procurement conversation — `MediatR.Contracts` remains Apache-2.0. Taken properly in Module 23 |
| **MassTransit v9** | **Commercial** (source-available; free for local dev; ≈\$400/mo SMB, ≈\$1,200/mo enterprise) | Excellent broker-backed messaging with sagas, outbox and a test harness. v8 stays Apache-2.0 but official maintenance ends around end of 2026 |
| **Rebus / Brighter** | MIT | Mature open-source service buses; smaller ecosystems, fewer transports |
| **Broker SDK directly** (`Azure.Messaging.ServiceBus`) | MIT | If you need one transport, simple consumers and no sagas, the raw SDK is less machinery than any framework |

The architectural point, not the shopping list: **your handler code should not know which of these it's running under.** If handlers implement a framework's interface and take a framework's context type, migrating costs you a rewrite. If handlers are plain classes with a `Handle(TMessage, CancellationToken)` method and the framework is adapted at the registration edge, migrating costs you an afternoon. That's the same "don't let the framework into your application layer" rule from Module 20, applied to the one dependency most likely to change licence under you — which, as 2025–2026 demonstrated, is not a hypothetical.

---

## Concept 50 — Aspire and the modular monolith

Aspire's relationship to this module is more interesting than it first appears, and it cuts both ways.

**What Aspire is.** An orchestration layer for local development and deployment composition: an **AppHost** project declares resources (your app, a SQL Server container, Redis, Service Bus emulator, a migration worker) and wires connection strings, service discovery and telemetry between them. As of 13.x it is polyglot (Python and JavaScript are first-class), has a CLI designed for automation and AI agents, deploys to Kubernetes/AKS and Azure Container Apps, and publishes Docker Compose. It is *not* your composition root — your `Program.cs` still composes the object graph (Module 20, Concept 26).

**For a modular monolith**, Aspire gives you one app project plus real infrastructure locally:

```csharp
var builder = DistributedApplication.CreateBuilder(args);
var sql   = builder.AddSqlServer("sql").AddDatabase("appdb");
var cache = builder.AddRedis("cache");
var migrator = builder.AddProject<Projects.MigrationWorker>("migrator")
                      .WithReference(sql);
builder.AddProject<Projects.Api>("api")
       .WithReference(sql).WithReference(cache)
       .WaitForCompletion(migrator);
builder.Build().Run();
```

**The strategic point**, and the thing worth saying in an interview: Aspire dramatically lowers the cost of the *local development* argument against services — the "I need eight things running to work on one" problem from Concept 28. That shifts the calculus at the margin. It does **not** change the latency, availability, consistency or versioning arithmetic of Part B, which is where the real cost lives. A tool that makes F5 work is valuable and is not an architecture decision.

The genuinely useful consequence: Aspire makes the **hybrid** shape cheap. A modular monolith plus two extracted services, all composed in one AppHost, all running locally with one command, all deployed by the same pipeline, is now an ordinary thing to do rather than an awkward halfway house. That is the architecture most mature systems want, and tooling is finally aligned with it.

---

## Concept 51 — The real risk (and it isn't performance)

Modular monoliths are sometimes dismissed on the grounds that they "don't scale." That's almost never the real risk. Two things are.

**Risk 1 — Deployment coupling.** Everything ships together. One team's bad merge blocks everyone's release. One team's memory leak takes down everyone's capability. As the team count grows, this becomes genuinely expensive, and it is the *correct* reason to extract (Concept 65). Mitigations that buy you real time: feature flags (so merging isn't releasing), trunk-based development with a fast pipeline, a canary deployment slot, and a rigorous "the build is never broken" culture. These push the crossover point out by a year or two — which is often exactly what you need.

**Risk 2 — Discipline decay.** The boundaries are inside one solution, and the only thing stopping a developer from referencing across them is a set of rules. Rules decay under deadline pressure. Three years in, you find a module that reads another's tables "temporarily for a report," a `SharedKernel` with 400 types, and a public surface nobody can summarize. This is why Concept 35 (database permissions) and Concept 48 (machine-checked rules) matter so much: **the parts of the design that a machine enforces are the parts that will still be true in three years.** Everything else is a prediction about human behaviour under pressure, and you know how those go.

Performance is a distant third and usually points the *other* way: in-process calls are effectively free, you keep real transactions, and you avoid serialization entirely. A modular monolith on a handful of instances will comfortably serve traffic volumes that most organizations never reach.

---

## Concept 52 — The extraction-readiness checklist

The list that turns "we can extract later" from a hope into a testable claim. Run it per module.

1. **Owns its schema**, with no other module reading or writing it — enforced by database permissions, not convention.
2. **No cross-module foreign keys** into or out of its tables.
3. **No cross-module joins** anywhere in its queries or anyone else's.
4. **A small, explicit public contract** in its own assembly, expressed in DTOs, in its own vocabulary.
5. **All inbound cross-module calls go through that contract** — verified by an architecture test that matches at least one type.
6. **All outbound notifications are integration events written to an outbox** in the same transaction as the state change.
7. **Its consumers are idempotent** and tolerate at-least-once, out-of-order delivery — because they already do, today.
8. **It registers itself** with a single `Add…Module()` / `Map…Endpoints()` pair, and owns its own configuration section, migrations and background workers.
9. **Its tests run against only its own services**, with fakes for every other module's contract.

If all nine hold, extracting the module is: create a host project, move the projects, point it at its own database, swap the in-process bus registration for a broker, put it behind the gateway, and route traffic. Days, not quarters.

If any of them fails, **that failure is the actual work** of your migration — and knowing precisely which of the nine is broken is what makes your estimate credible rather than a guess. In a brownfield interview round, walking this list out loud is a very strong answer, because it converts a vague "it'll take a while" into a specific, defensible plan.

---

# Part D — Finding the boundaries

The deployment decision gets all the airtime; the decomposition is the part that's actually hard. A wrong boundary inside a monolith is a refactor. A wrong boundary between services is a migration. So this part is disproportionately valuable relative to how often candidates prepare it.

---

## Concept 53 — Bounded contexts are the source

The boundary you're looking for is a **language boundary before it is a code boundary**.

Eric Evans' bounded context (DDD, 2003; Module 22 takes it properly): a boundary within which a model and its vocabulary are consistent and unambiguous. The diagnostic is beautifully simple — **find the words that mean different things to different people.**

- "Customer" to Sales is a prospect with a pipeline stage. To Billing it's a legal entity with a VAT number and payment terms. To Support it's a person with a ticket history and an entitlement tier.
- "Product" to the catalogue is a description, images and SEO metadata. To inventory it's a SKU with stock levels per location. To pricing it's a set of rules, tiers and promotions.
- "Order" to checkout is a basket being validated. To fulfilment it's a set of shipments. To finance it's a revenue event with tax implications.

**Each of those different meanings is a bounded context, and each bounded context is a candidate module.** The reason this is the right source is that a model that spans contexts has to satisfy all of them — which is why the "one Customer class" always ends up with thirty properties and three teams arguing about it.

Two practical markers. **Context maps** (how contexts relate — shared kernel, customer/supplier, conformist, anticorruption layer) tell you what integration you'll need, which is directly the cost of the boundary. And when the same word means two things, **the translation happens in an anti-corruption layer at the boundary** (Module 20, Concept 44) — which is the module's contract doing its job.

---

## Concept 54 — Decompose by business capability

The rule, stated positively: **decompose by what the business does, not by what the data is.**

A business capability is something the organization performs and could in principle staff a team around: *take an order*, *bill a customer*, *ship a parcel*, *manage the catalogue*, *authenticate users*, *calculate tax*. Chris Richardson's pattern language calls this **decompose by business capability**, with **decompose by subdomain** as the DDD-flavoured sibling; in practice they converge.

The test that makes it operational: **can one team own this end to end, and can a feature in this area be delivered without changing anything else?** If a typical feature request lands entirely inside one capability, you have a good boundary. If every feature touches four modules, you have a layering masquerading as a decomposition (Concept 57 quantifies this).

The other useful marker is **the rate and reason for change.** Two capabilities that change on completely different cadences, for completely different business reasons, driven by different stakeholders, are genuinely separate. Two that always change together are one thing pretending to be two.

Worth noting explicitly: this is the **Y-axis of the Scale Cube** from Module 6 — scaling by splitting different things — as opposed to the X-axis (cloning) and the Z-axis (partitioning by data). A candidate who connects the decomposition question back to the Scale Cube is demonstrating that the curriculum's pieces are one system in their head, which is exactly the signal a staff-level interview looks for.

---

## Concept 55 — The entity-service trap

The single most common bad decomposition, and it looks reasonable right up until it doesn't.

You have entities: User, Order, Product, Payment. You make a service for each: UserService, OrderService, ProductService, PaymentService. It's tidy, it's obvious, and it is a **database schema wearing a costume**.

Why it fails:

- **No business operation fits inside one service.** "Place an order" needs User (validate), Product (price, availability), Order (create), Payment (charge). Every use case is a distributed transaction across four services, so you get the worst possible ratio of network hops to business value.
- **The services are anaemic** — they're CRUD wrappers over tables (Module 20, Concept 54's failure mode) with no behaviour, because the behaviour lives in the *interactions*, which now happen over HTTP in some orchestrator.
- **Everything is chatty.** Page renders fan out to four services and hit Concept 20's tail-latency amplification.
- **Ownership is incoherent.** Who owns "place an order"? Nobody owns it; four teams each own a fragment.

The correction is to name boundaries with **verbs and capabilities**, not nouns: *Ordering*, *Fulfilment*, *Billing*, *Catalogue*, *Identity*. Note that the same underlying data appears in several of them, in different shapes — which is Concept 41, and is the thing the entity decomposition was trying to avoid and shouldn't have been.

A related smell: **a service named after a technical layer** — "DataService," "BusinessLogicService," "IntegrationService." That's layers-as-tiers (Module 20, Concept 76), which is the worst decomposition of all, because it guarantees every change touches every service.

---

## Concept 56 — EventStorming and boundary discovery

The cheapest and most effective boundary-discovery technique, and worth naming in an interview because it signals you've actually done decomposition with a business, not just on a whiteboard.

**EventStorming** (Alberto Brandolini): get domain experts and engineers in one room with a long wall and sticky notes. Write **domain events** in past tense on orange stickies — *Order Placed*, *Payment Captured*, *Parcel Dispatched*, *Invoice Issued* — and arrange them on a timeline. Then add commands, actors, policies, read models and external systems.

What emerges, essentially for free:

- **Clusters.** Events naturally group into clumps. The clumps are your candidate contexts.
- **The seams.** The places where the vocabulary changes, where a handoff happens, where a different person becomes responsible — these are boundary candidates, and they're visible on the wall.
- **Hotspots.** Disagreements between experts, marked in red, are exactly where the ambiguous language is, which is where the context boundary is.
- **The integration events you'll need**, already written down in the business's own words.

Related techniques worth knowing by name: **Domain Storytelling** (Hofer & Schwentner — narrate a workflow with a pictographic language; excellent with non-technical stakeholders), **Wardley mapping** (for deciding what to build vs. buy vs. outsource, which is Module 33), and **the Business Capability Map** many enterprises already have sitting in an architecture repository nobody has read.

The interview line: *"I'd run an EventStorming session with the domain experts before proposing any decomposition — a day of that produces better boundaries than a month of reading the codebase, because the boundaries are in the business's language, not in the schema."*

---

## Concept 57 — Change coupling: mine the git history

The empirical technique, and my favourite thing to bring to a brownfield discussion because it replaces opinion with data the team can't argue with.

**Files that change together belong together.** Run it over the repository:

```bash
# Commits touching more than one top-level area, ranked by frequency
git log --since="12 months ago" --name-only --pretty=format:"%H" \
  | awk '…'   # group paths by commit, emit co-change pairs, count
```

Tools that do this properly: **CodeScene** (Adam Tornhill's product; his books *Your Code as a Crime Scene* and *Software Design X-Rays* are the theory) and **Code Maat** (his free CLI, MIT). What you get:

- **Temporal coupling** — pairs of files or folders that change in the same commit far more often than chance. High coupling across a proposed boundary means the boundary is wrong.
- **Hotspots** — files with high change frequency *and* high complexity. These are where the pain is, and usually where the missing boundary is.
- **Knowledge maps** — who changes what, which is Conway's law made visible and tells you where ownership boundaries already exist informally.

Two ways to use this concretely. **Validating a proposed decomposition:** compute what fraction of commits in the last year would have touched more than one proposed module. If it's above roughly 25–30%, the boundaries are wrong — those modules change together, so they are one thing. **Finding the first extraction candidate:** the module with the *lowest* co-change with everything else is the cheapest to extract and the least likely to need a follow-up migration.

This is also a great answer to "how would you decide what to split first?" — because it's evidence-based, cheap, and something you can do in an afternoon before committing anyone to a quarter of work.

---

## Concept 58 — The coupling taxonomy, ranked

Not all coupling is equal, and having vocabulary for the kinds lets you argue precisely. Sam Newman's taxonomy, from loosest (fine) to tightest (dangerous):

1. **Domain coupling** — A needs B to do something, and asks via B's contract. Unavoidable and healthy; it's just "the business has dependencies." Minimize the *number* of these, don't try to eliminate them.
2. **Pass-through coupling** — A passes data through B to C, where B doesn't care about the data. A change to C's requirements forces a change to B. Fixable: have A talk to C directly, or have B hold the data opaquely.
3. **Common coupling** — A and B both use a shared mutable resource: a shared database table, a shared config store, a shared cache key. **This is the one that kills extraction.** Concept 35 exists to prevent it.
4. **Content coupling** — A reaches into B's internals: reads its tables, uses its entity types, depends on its implementation details. The worst. It means B cannot change anything without breaking A, and neither party knows it until runtime.

Add a fifth axis that cross-cuts all of them:

5. **Temporal coupling** — A requires B to be available *right now* to complete its work. This is the availability multiplication of Concept 21. Removed by asynchronous messaging, not by a better interface. This is why Concept 39 says writes go by event: it converts temporal coupling into none.

The sentence this earns you: *"The coupling I'd worry about isn't A calling B — that's domain coupling and it's fine. It's that they share three tables, which is common coupling, and it means neither can change its schema. That's what I'd fix first, before any conversation about services."*

---

## Concept 59 — Data coupling and the shared table

Worth its own concept because it is the boundary violation that survives every refactor and every good intention.

Two modules read and write the same table. Someone will present this as an optimization or as "it's the same data, why duplicate it?" The consequences:

- **Neither module can change the schema** without coordinating with the other, forever.
- **Neither module owns the invariants.** If two modules both write `Orders.Status`, the rules governing valid status transitions live in two places and will diverge. This is a correctness bug waiting for a race condition.
- **There is no boundary.** Whatever the code looks like, these are one module.
- **Extraction is blocked** until the table is split, and splitting a shared table is the most expensive step in any migration (Concept 71).

The three ways it sneaks in, all of which sound reasonable in the moment:

1. **"Just for reporting."** A read-only query against another module's tables for a dashboard. It's a read, so it feels safe. It isn't: the other module can now never rename a column. Use a projection or the warehouse (Concept 86).
2. **"It's a lookup table."** Reference data — countries, currencies, tax codes. Genuinely shared, genuinely stable. The safe answer: one module owns it, others hold a replicated copy, or it is genuinely static configuration deployed with the app rather than a table anyone writes to.
3. **"Just this one join, for performance."** The one that always wins the argument at the time and is discovered eighteen months later during the migration, by which point three other places have copied the pattern.

The countermeasure is Concept 35: make it a permission error, so the conversation happens at the moment the developer writes the query rather than at the moment you try to extract the module.

---

## Concept 60 — Cohesion metrics, applied to modules

Module 20's Concept 12 gave the component principles; here they become measurable module properties you can put on a slide.

- **Afferent coupling (Ca)** — how many other modules depend on this one. High Ca means it's a *stable* module and changing it is expensive.
- **Efferent coupling (Ce)** — how many modules this one depends on. High Ce means it's fragile: it breaks when others change.
- **Instability I = Ce / (Ca + Ce)** — 0 is maximally stable (everyone depends on it, it depends on nobody: `SharedKernel` should be near 0), 1 is maximally unstable (it depends on everything, nobody depends on it: the Host should be near 1).
- **The Stable Dependencies Principle:** depend in the direction of stability. A module with I = 0.1 should not depend on a module with I = 0.8.
- **Acyclic Dependencies Principle:** **no cycles between modules, ever.** A cycle means the modules in it are one unit — they must be built, tested, deployed and extracted together. An architecture test for cycles is the second-most valuable rule after module isolation.

`NDepend` (commercial), `ArchUnitNET` (free) and the plain `.csproj` graph will all give you these. The practical use is a quarterly fitness function (Module 20, Concept 67): plot instability per module over time, and when a module's Ce starts climbing, someone has started reaching across boundaries.

A quick sanity picture for a healthy modular monolith: `SharedKernel` near I ≈ 0; each module's `Domain` low; each module's `Application` moderate; `Host` at I ≈ 1; and **zero edges between any two modules' internals**.

---

## Concept 61 — The two-pizza rule, decoded

Amazon's "a team should be small enough to be fed by two pizzas" (roughly 6–10 people) is quoted constantly and understood rarely.

It is **not** a statement about service size. It's a statement about **communication overhead**: coordination paths grow as n(n−1)/2, so a team of 6 has 15 pairwise channels and a team of 20 has 190. Past roughly ten people, a team spends more time synchronizing than building.

The architectural implication runs one way only: **a service should be ownable by a team of that size** — not "a team of that size should have a service." A two-pizza team can perfectly well own one module in a modular monolith, or three services, or one large service. The rule constrains the team, and the architecture should be shaped so that ownership is clean given the teams you have.

The corollary worth saying out loud: **if you have one team, the two-pizza rule tells you nothing about your architecture.** It's an organizational scaling heuristic, and quoting it to justify decomposition in a 12-person company is a category error interviewers notice.

---

## Concept 62 — Nanoservices and over-decomposition

The failure mode at the other end, and increasingly the common one in 2026 as teams that adopted services aggressively find out what they bought.

Symptoms of over-decomposition:

- **Services with no meaningful state or behaviour** — a service that validates an email address, a service that formats a PDF, a service whose entire job is one function call.
- **More network calls than business operations.** A single user action generates 15 RPCs.
- **The service count exceeds the engineer count.** Ten engineers, forty services. Nobody owns anything; everybody is on call for everything.
- **Every feature touches five services**, which means every feature is a five-team coordination exercise — you have recreated the release train you were escaping, and added latency.
- **Operational overhead dominates.** More time spent on pipelines, dashboards and version bumps than on features.

The general principle: **every service has a fixed overhead** — a pipeline, a repository (maybe), a deployment, a dashboard, an alert route, an on-call owner, a set of dependencies to patch, a base image to update, a contract to version. Call it a meaningful fraction of an engineer, permanently. If the service does not deliver more value than that fixed cost, it should be a module or a library instead.

The correction is **consolidation**, and it is a legitimate, senior, career-safe move. Merging two services that always change together and always deploy together is not an admission of failure; it's noticing that they were one thing (Module 20's Concept 69 said the same about collapsing a layer). Do it the same way you'd do an extraction: incrementally, with a plan and a rollback.

---

## Concept 63 — Symptoms of wrong boundaries

The checklist for diagnosing an existing decomposition, in roughly the order the symptoms appear:

| Symptom | What it means |
|---|---|
| Services are deployed in lockstep | Not independently deployable. Distributed monolith (Concept 6) |
| Features routinely touch 3+ services | Boundaries cut across change, not along it (Concept 57) |
| Two services write the same tables | Common coupling; there is no boundary (Concept 59) |
| Synchronous call chains 3+ deep | Latency adds, availability multiplies (Concepts 18, 21) |
| A "core" or "common" library everyone depends on | Coupling reintroduced through the back door (Concept 88) |
| One team owns most of the services | The decomposition doesn't match the org; Conway will fix it for you, badly |
| Cross-service transactions needed for ordinary flows | The boundary splits an invariant that should be inside one unit |
| A change to a DTO requires a coordinated release | Contract versioning discipline missing (Concept 70) |
| Chatty N+1 calls across the boundary | An entity-service decomposition (Concept 55) |
| Nobody can draw the system from memory | Over-decomposition (Concept 62) |

When several of these fire, the answer is usually **not** "add a service mesh" or "improve the tooling." It's that the decomposition is wrong and needs to be fixed — by merging, by moving a capability from one service to another, or by moving data ownership. Saying that plainly is a senior signal, because the tempting alternative (buy tooling to manage the symptoms) is what most organizations do.

---

## Concept 64 — Boundaries follow ownership or they die

Closing Part D by returning to Conway's law with the operational version.

**A boundary that nobody owns is a suggestion.** An unowned module will accumulate whatever each passing team needs from it. A module owned by two teams will satisfy both and cohere with neither. A module owned by one team, with that team's name on the on-call rotation and their judgment on the pull requests, develops a point of view — and a point of view is what a boundary *is*.

Practical implications:

- **Assign an owner to every module**, by name, in `CODEOWNERS`. Not a committee. Not "the platform team." A team, with a lead.
- **Don't create more modules than you can own.** Eight modules and two teams means six modules nobody is thinking about.
- **When a boundary keeps eroding, check ownership first.** Nine times out of ten, the erosion is two teams needing the same thing and neither having authority.
- **When teams reorganize, revisit the boundaries.** The architecture will drift toward the new org chart whether you plan for it or not; planning for it is the difference between an evolution and a decay.

This is why the org-chart test (Concept 16) is the decisive question rather than a soft one, and it's why architecture at this level is inseparable from organizational design — which is precisely the distinction between the senior IC track and the architect track that Module 2 drew.

---

# Part E — Extraction: when and how

Part C built something extractable. Part D found the seams. Part E is the procedure: what makes an extraction justified, what order to do it in, how to move the data without an outage, and how to undo it.

---

## Concept 65 — The extraction triggers

Extract when a **specific, named, measurable** condition holds. Here is the honest list; each one comes with the evidence you should have in hand.

1. **Deploy contention.** Teams are blocked on each other's releases. *Evidence:* release frequency per team, time spent waiting, number of rollbacks caused by unrelated changes. This is the most common legitimate trigger and the one that scales with team count.
2. **Scale asymmetry.** One capability's resource profile is radically different — 10× the memory, a GPU, 30-second operations, or a 100× traffic spike that would require scaling everything. *Evidence:* profiling data attributing resource consumption to that capability (Module 17). "We think it's the search" is not evidence.
3. **Failure isolation requirement.** A capability must not be able to take down another. *Evidence:* an incident history where it did, or a regulatory/contractual availability requirement for one capability specifically.
4. **Compliance or data residency.** PCI scope reduction, PII isolation, a regional data-residency requirement, or an audit boundary. *Evidence:* the requirement in writing. This is a very strong trigger because it's non-negotiable and because scope reduction has direct, quantifiable cost savings.
5. **Technology necessity.** The capability genuinely requires a runtime, library or hardware that can't live in the main process — a Python ML stack, a native library with a hostile licence, a specific GPU driver.
6. **Team count crossing the threshold.** You've passed roughly 5–8 teams and the coordination cost is visible in lead time. *Evidence:* DORA metrics trending the wrong way as headcount grows.
7. **Independent lifecycle.** A capability has a genuinely different release cadence — a partner integration that changes weekly against a core that changes quarterly, or vice versa.
8. **Acquisition or divestiture.** The capability is being bought, sold, or spun out. The boundary is a commercial fact.

The senior framing: *"I'd extract when there's a specific trigger with evidence behind it, and I'd extract that one module rather than embarking on a decomposition programme. Here's the trigger I'd watch for first, and here's the metric that tells us it fired."*

---

## Concept 66 — The anti-triggers

Reasons that sound like reasons and aren't. Being able to name these — politely — is a big part of what an architect round is scoring.

- **"The monolith is a mess."** That's Axis 1 (Concept 1). Distribution doesn't clean it; it entrenches it in a form you can no longer refactor.
- **"Microservices are best practice."** Best practice for whom, at what scale, solving what problem? This is cargo cult, and the honest response is to ask what problem we're solving.
- **"We want to attract/retain engineers."** Real pressure, not a reason. Cheaper alternatives: modern .NET, good tooling, real ownership, interesting problems, conference budget.
- **"Netflix/Amazon/Uber do it."** They have thousands of engineers and specific problems. Netflix's decomposition followed a catastrophic monolith failure at a scale most companies will never reach, and they built an entire platform organization to support it. Also note that the Uber and Amazon stories both include partial reversals (Concept 97).
- **"It'll scale better."** Will it? Which component is the bottleneck? What's the current p99 and instance count? Nine times out of ten the bottleneck is the database, which decomposition makes harder, not easier.
- **"A consultant/vendor recommended it."** Worth understanding the incentive. Note the CNCF data pattern from the Orientation: service mesh adoption declining while Kubernetes adoption grows — organizations are discovering which parts of the stack earn their keep.
- **"We already started."** Sunk cost. The right question is always "from here, what's the cheapest path to the outcome we want?"

The diplomatic version, which is the one to actually use in an interview: *"I'd want to understand the driver before the design — if it's release contention I'd solve it one way, if it's a hot path I'd solve it another, and if it's that the codebase is hard to work in, splitting it would make that worse rather than better. Can we look at what's actually costing us time?"*

---

## Concept 67 — The order of operations

The most important procedural rule in this module, and the one most violated:

**Logical boundary → data boundary → process boundary. Never reverse.**

**Step 1 — Logical.** Move the code into a module with a clear contract, hidden internals, and no cross-module references. Fully reversible. No deployment change. No risk. All the value of Axis 1, immediately.

**Step 2 — Data.** Move the module's tables into their own schema with their own credentials. Remove every cross-module foreign key and join. Replace them with contract calls or replicated data. **This is the hard step** — this is where you discover the eleven reports that join across boundaries — and doing it while still in one process means you can do it with real transactions, in-process tests and instant rollback.

**Step 3 — Process.** Now extraction is mechanical: new host, own database, broker instead of the in-process bus, gateway route, traffic switch.

Teams that reverse this — standing up a new service and then trying to untangle the data — end up running dual writes across a network boundary with no transaction available to them, which is the single most dangerous state a migration can be in. All of the risk lands at once, in production, with no rollback that doesn't involve data reconciliation.

Say the ordering explicitly in an interview; it is a compressed demonstration of having done this for real.

---

## Concept 68 — Strangler fig, applied to a module

Martin Fowler's **strangler fig** (the vine that grows around a tree until the tree dies and the vine stands on its own) is the pattern for incremental replacement, and it's both a Microsoft Azure Architecture Center pattern and Module 32's main subject. The module-extraction version:

1. **Put a facade in front.** All traffic for the capability goes through one routing point — a gateway (YARP), a reverse proxy, or, inside the monolith, a single interface implementation. Nothing changes functionally; you've just created a switch point.
2. **Stand up the new service** alongside, serving the same contract, with its own data, and no traffic.
3. **Route one operation at a time.** Start with a read-only, low-risk, idempotent endpoint. Route a percentage of traffic. Compare results (shadow/dark traffic: send to both, serve the old, log the differences).
4. **Move writes last**, and only after the data migration (Concept 71) is complete and verified.
5. **Watch and hold.** Error rates, latency, business metrics. Leave both paths live and switchable for at least one full business cycle — a month-end, a payroll run, a peak day.
6. **Delete the old path** when traffic has been zero for long enough that you'd have heard about it. Then delete the facade if it's no longer needed.

The properties that make it work: **every step is small, every step is reversible with a config change, and the system is fully functional between steps.** The alternative — a big-bang rewrite — has a failure mode where you are 70% done, cannot ship, and cannot go back. Module 32 covers the organizational side; this is the mechanics.

---

## Concept 69 — Branch by abstraction

The in-process technique that makes step 3 of the strangler fig a config flag rather than a code change.

```csharp
// The abstraction the rest of the system uses — already exists if you built Part C
public interface IBillingModule { Task<Result<InvoiceId>> IssueInvoiceAsync(…); }

// Implementation 1: the in-process module (today)
internal sealed class InProcessBillingModule : IBillingModule { … }

// Implementation 2: the extracted service (during and after migration)
internal sealed class RemoteBillingModule(HttpClient http) : IBillingModule { … }

// Implementation 3: both, for verification
internal sealed class ComparingBillingModule(
    InProcessBillingModule primary, RemoteBillingModule shadow, ILogger log) : IBillingModule
{
    public async Task<Result<InvoiceId>> IssueInvoiceAsync(…)
    {
        var result = await primary.IssueInvoiceAsync(…);
        _ = Task.Run(async () => { /* fire-and-forget compare, log divergence */ });
        return result;
    }
}
```

The host picks one by configuration:

```csharp
services.AddScoped<IBillingModule>(sp => config["Billing:Mode"] switch
{
    "remote"  => sp.GetRequiredService<RemoteBillingModule>(),
    "compare" => sp.GetRequiredService<ComparingBillingModule>(),
    _         => sp.GetRequiredService<InProcessBillingModule>()
});
```

Now the migration is: deploy with `compare`, watch divergence for a week, flip to `remote`, watch, and keep `inprocess` as the rollback for a month. **The switch is a configuration change, so rollback is seconds rather than a deployment.** This is the concrete payoff for having defined a module contract in Part C, and it is a genuinely strong thing to be able to sketch on a whiteboard.

---

## Concept 70 — Expand/contract

The pattern for changing a contract or a schema when two versions must coexist — which is always, during a rolling deploy, and forever, after extraction.

**Expand.** Add the new thing alongside the old. New column (nullable, or with a default), new field, new endpoint. Deploy. Both old and new code work.

**Migrate.** Backfill data. Update writers to write both. Update readers to prefer the new and fall back to the old. Deploy. Verify.

**Contract.** When nothing reads or writes the old thing — verified by telemetry, not by grep — remove it. Deploy.

Examples worth having ready:

- **Renaming a column:** add `FullName`, write both, backfill, switch reads, stop writing `Name`, drop `Name`. Four deployments for a rename. This is the tax from Concept 27, and it's why you want to get names right *before* the boundary hardens.
- **Changing a field's type in a message contract:** add the new field with a new name, publish both, consumers migrate, remove the old. Never change a field's meaning in place; consumers you don't know about will silently misinterpret it.
- **Splitting a table across a new schema boundary:** Concept 71.

Two supporting disciplines: **tolerant readers** (ignore unknown fields, don't fail on additions — the default for `System.Text.Json` unless you opt into strict handling) and, for Protobuf, **never reuse a field number**; mark removed ones `reserved`.

Practising expand/contract in the modular monolith is preparation, not overhead: it's the same discipline you'll need permanently once a boundary is a network boundary.

---

## Concept 71 — Data separation, step by step

The hardest part of any extraction, with the procedure. Each step is independently deployable and independently reversible.

**Starting point:** `Billing` reads and writes tables in the shared schema, and other modules join to them.

1. **Inventory.** Find every read and write of those tables from outside Billing. Query logs, stored procedures, reports, ETL jobs, the BI tool, that one Excel connection somebody set up in 2019. This step always takes longer than planned, and skipping it is how migrations fail.
2. **Eliminate outside access.** For each one: replace a join with a contract call, replace a report with a projection, replace an ETL source with events. **Nothing proceeds until this is zero.**
3. **Move to its own schema** in the same database. Enforce it with credentials (Concept 35). Now you find any access you missed — as a permission error in a test environment, which is the cheap place.
4. **Move to its own database.** Now cross-database queries are physically impossible and you must have already handled step 2. Still one process, so still easy to roll back by repointing a connection string.
5. **Dual write** if the data must exist in two places during transition (for example, a denormalized copy another module needs): write to both, with the old one authoritative. Use the outbox so the second write is reliable rather than best-effort.
6. **Backfill.** Copy historical data. Do it in batches, resumable, idempotent, with progress tracking, and — crucially — **verify with row counts and checksums**, not by looking at a few records.
7. **Switch reads** to the new location behind a feature flag. Monitor. Keep the old path warm.
8. **Switch writes.** The point of no easy return; do it after the reads have been stable for a full business cycle.
9. **Stop dual-writing**, verify nothing reads the old location (telemetry, not assumption), then **delete** it. Deleting is important: an unused copy will be discovered by someone and used.

Rule of thumb: **steps 1–2 are 70% of the effort.** If someone estimates a data migration by looking at the size of the table, they haven't done one.

---

## Concept 72 — Contracts and consumer-driven contract tests

Once a boundary is a network boundary, the contract is the only thing holding the system together, and it needs its own testing discipline.

**Define the contract explicitly and version it.** OpenAPI for HTTP (ASP.NET Core's built-in OpenAPI document generation in .NET 9+, rendered with Scalar since NSwag's Swagger UI was dropped from the common templates), `.proto` files for gRPC, and a schema for events — a versioned record type in a contracts package, or a registry if you're on a platform that has one.

**Consumer-driven contract testing** is the technique that replaces the end-to-end suite you can't afford (Concept 26). The idea: the *consumer* declares what it actually needs from the provider; the provider verifies it can satisfy every consumer's expectations. Neither side runs the other. **Pact** is the canonical tool and has a .NET implementation (`PactNet`). The flow:

1. Consumer writes a test against a mock provider, expressing exactly the fields and behaviours it relies on.
2. The test produces a **pact file** — a machine-readable contract.
3. The provider's CI replays every consumer's pact against the real implementation.
4. The provider's build fails if a change would break a consumer — **before deployment**, without any coordinated environment.

Why this is the right shape: it catches breaking changes at the earliest point, it scales with the number of consumers rather than the number of service pairs, it documents what's *actually* used (so you can delete the rest with confidence), and it preserves independent deployability — which is the entire point of the exercise.

In the modular monolith, this discipline is cheap to rehearse (Concept 47, tier 3), and rehearsing it is one of the concrete things that makes an extraction go smoothly rather than surprisingly.

---

## Concept 73 — The rollback plan

The question to ask about every step of a migration: **if this goes wrong at 2 a.m., what does the on-call engineer do?**

The rule: **if you can't undo a step within an hour, the step is too big.** Break it down until you can.

What that means concretely:

- **Feature-flag every switch.** Routing, read source, write target. Flipping a flag is seconds; deploying is minutes and requires a pipeline that might itself be broken.
- **Keep the old path functional** until you've been through at least one full business cycle. Delete it deliberately, as its own change, when you're confident — never as a cleanup at the end of the migration PR.
- **Data changes need a reverse.** Additive changes are reversible for free. Deletions and in-place transformations are not; keep a copy, keep the dual-write running, or keep a restorable snapshot with a tested restore procedure.
- **Rehearse the rollback.** An untested rollback is a hypothesis. Test it in staging, with real data volumes, and time it.
- **Write the runbook before the change**, not during the incident: what the symptoms look like, what to flip, who to call, how to verify recovery.

And name the one genuinely irreversible step: **once you've deleted the old data and the old code path, you're committed.** Do that last, deliberately, as a separate decision with its own sign-off.

---

## Concept 74 — Measuring whether it worked

You proposed this. You should be able to prove it did what you said. Take the baseline *before* you start, because after the fact everyone's memory conveniently agrees with whatever happened.

**DORA metrics** (from *Accelerate*, Forsgren/Humble/Kim) — the four that matter and are widely accepted:

| Metric | What you expect from a good extraction |
|---|---|
| **Deployment frequency** | Up for the extracted team (that's the point) |
| **Lead time for change** | Down for the extracted team; watch it doesn't rise elsewhere |
| **Change failure rate** | Flat or down; a rise means the contract discipline isn't there |
| **Time to restore service** | **Watch this one carefully.** It usually goes *up* — distributed debugging is slower (Concept 25). If it rises a lot, your observability isn't ready |

Plus the ones specific to this decision:

- **Deploy contention** — how often a team waits for another. The thing you were buying. If it didn't move, the extraction didn't do its job.
- **Cross-boundary change frequency** — what fraction of changes touch more than one service. If it's high, the boundary is wrong (Concept 57) and you should say so early rather than defend it.
- **p99 latency of affected endpoints** — you added network hops; quantify them.
- **Availability of affected journeys** — Concept 21's arithmetic, measured.
- **Infrastructure run rate** — the actual Azure bill, before and after.
- **Onboarding time** — how long until a new engineer ships to this area.

Committing to this measurement up front is itself a senior signal. It converts an architecture opinion into a falsifiable claim, which is the difference between engineering and taste.

---

## Concept 75 — Consolidating back

Sometimes the right move is to merge services. It's a legitimate, senior decision, and 2025–2026 has produced a lot of teams doing it. It is also **harder than splitting**, for a reason worth understanding: splitting divides data that is currently consistent; merging must reconcile data that has already diverged.

When to do it: two services always deploy together; one team owns both and there's no ownership reason to separate; the network hop between them is on a hot path and costs real latency; the operational overhead exceeds the value; or the boundary turned out to be wrong (Concept 63).

The procedure mirrors extraction, in reverse and with extra care:

1. **Verify they really are one thing** — co-change analysis (Concept 57), deploy coupling, ownership.
2. **Merge the code first**, keeping both deployments running. Bring the code into one repository/solution as two modules. No behaviour change, no data change.
3. **Reconcile the data.** This is the hard part: two `Customer` tables with divergent rows, different nullability, different histories, different IDs for what turns out to be the same entity. Expect data-quality work and expect to need the business to decide which source wins.
4. **Route traffic to the merged deployment** behind a flag, with the old services still live.
5. **Decommission** the old deployments, pipelines, dashboards and alerts — all of them, or you'll be paying for and being paged by ghosts a year later.

And **keep them as modules**, not as a re-merged tangle. You're moving from the bottom-right or bottom-left quadrant to the top-right one (Concept 2) — that's the whole objective, and if you lose the boundaries in the merge you've given up the only thing you were keeping.

---

## Concept 76 — Brownfield: modularize first

The highest-ROI work in most legacy systems, and the answer to the most common architect-round scenario ("we have a 15-year-old monolith; what's the plan?").

**Don't start by extracting services.** Start by finding the seams, in this order:

1. **Build a characterization test harness** at the API level. You cannot refactor safely without one, and legacy systems rarely have unit tests worth trusting. Test the behaviour you have, not the behaviour you want.
2. **Get telemetry in.** OpenTelemetry, request-level tracing, per-endpoint latency and error rates, and — very useful — instrumentation that attributes database queries to endpoints. You need to know what the system actually does before you change its shape.
3. **Run the co-change analysis** (Concept 57) and the EventStorming session (Concept 56). One gives you what the code says; the other gives you what the business says. Where they agree, you have a boundary.
4. **Pick one capability** — the one with the lowest coupling to everything else — and pull it into a module *inside the existing solution*. Not a service. A module, with its own namespace, its own folder, an explicit contract, and `internal` types.
5. **Move its data into its own schema.** Fix every join and report that crosses the new line. This is the slow part and the valuable part.
6. **Enforce it** with an architecture test and database permissions, so it can't erode while you work on the next one.
7. **Repeat.** Each module is independently valuable, independently shippable, and doesn't commit you to distribution.
8. **Only then**, and only for modules with a trigger (Concept 65), extract to services.

The reason this sequencing is right is the Concept 14 asymmetry: every step is reversible, every step improves the system whether or not you continue, and you never end up 70% through a distributed rewrite unable to ship. It's also far easier to fund, because you can show value each quarter rather than asking for eighteen months of faith.

---

# Part F — Living with services on .NET and Azure

If the decision goes the other way — and sometimes it should — this is what you're signing up for. Module 26 covers compute choice properly and Module 28 covers observability; this part is the connective tissue, at the depth a design round expects.

---

## Concept 77 — The platform floor, concretely on Azure

Concept 28 listed the capabilities. Here they are with Azure specifics, so you can answer "how would you actually do this?" without hand-waving:

| Capability | Azure / .NET answer |
|---|---|
| Build & deploy | GitHub Actions or Azure DevOps; one pipeline per service; environment promotion; `dotnet publish` with container support (no Dockerfile needed since .NET 7) |
| Container registry | Azure Container Registry with Microsoft Defender vulnerability scanning; immutable tags |
| Compute | Azure Container Apps (managed, scale-to-zero, Dapr built in) or AKS (full control); Module 26 |
| Configuration | Azure App Configuration with labels per environment, feature flags built in |
| Secrets | Key Vault with managed identity — never connection strings in config |
| Service discovery | Container Apps/AKS DNS; `Microsoft.Extensions.ServiceDiscovery` for logical names in .NET |
| Ingress & routing | Azure Front Door / Application Gateway at the edge; YARP for an application-level gateway |
| Messaging | Azure Service Bus (queues, topics, sessions, dead-letter) or Event Hubs for streams (Module 11) |
| Telemetry | OpenTelemetry → Azure Monitor / Application Insights; `ActivitySource` and `Meter` in code |
| Tracing | W3C `traceparent` propagated automatically by `HttpClient` and the ASP.NET Core instrumentation |
| Health | `Microsoft.Extensions.Diagnostics.HealthChecks`; separate liveness and readiness endpoints |
| Identity | Microsoft Entra ID; managed identity service-to-service; token propagation for user context (Module 29) |
| Local dev | Aspire AppHost — the whole graph with one command |
| On-call | Azure Monitor alerts routed to a rotation, per service, with a named owner |

The point to make in an interview isn't the list — it's the ordering: **this is a prerequisite, not a follow-up.** A team that extracts its first service and then discovers it has no distributed tracing spends the next six months debugging blind.

---

## Concept 78 — Compute choice, the short version

Module 26 takes this properly. The one-paragraph version you need here, because a design round will ask where these run:

- **Azure App Service** — a handful of HTTP services, minimal ops, no container expertise needed. Perfectly respectable and unfashionable.
- **Azure Container Apps** — the default for most .NET microservices in 2026: managed Kubernetes underneath without the Kubernetes, KEDA-based scaling including scale-to-zero, built-in Dapr, managed ingress and revisions for blue/green. The right answer for most teams that don't have a platform team.
- **AKS** — when you need full Kubernetes: custom operators, complex networking, multi-tenancy, a mesh, GPU pools, or a platform team that already runs it. Aspire 13.3+ ships mature Kubernetes and AKS deployment support.
- **Azure Functions** — event-driven, spiky, short-lived work. Cheap at low volume; watch cold starts and per-execution costs at high volume (and remember the Prime Video lesson about per-transition costs in orchestration — Concept 19).
- **Azure Container Apps jobs / Kubernetes jobs** — scheduled and queue-triggered batch work.

The judgment line: *"I'd default to Container Apps unless there's a specific reason to need AKS, because the operational surface is much smaller and the scaling story covers most requirements — and the moment we need a mesh, custom CRDs or GPU scheduling, we move."*

---

## Concept 79 — Synchronous vs. asynchronous between services

The single most consequential design choice after the boundary itself, because it determines your availability arithmetic (Concept 21).

**Synchronous (HTTP/gRPC request-response).** Simple to reason about, immediate consistency, easy to debug, natural for queries. But: temporal coupling (both must be up), latency adds through the chain, failures propagate, and it invites chatty call patterns. Use it for **queries** and for **commands that genuinely need an immediate answer** (authorize a payment before accepting an order).

**Asynchronous (messaging).** No temporal coupling — the consumer can be down and the message waits. Natural load levelling under spikes. Adds consumers without touching producers. But: eventual consistency becomes business-visible, debugging is harder, ordering and duplication must be handled explicitly, and you need an answer for poison messages and dead letters (Module 11). Use it for **state-change notifications** and for anything that can complete after the user's request returns.

The default worth stating: **queries synchronous, commands asynchronous where the business allows it, state changes always as events.** And the design move that resolves most arguments: when someone insists a flow must be synchronous, ask what the user actually needs to see *now*. "Your order is confirmed" usually needs to be immediate. "Your invoice has been generated" almost never does. Making that distinction explicit with the business is architecture work, not technical work — and it's exactly the kind of thing an architect round is testing.

---

## Concept 80 — gRPC vs. HTTP/JSON

The options in .NET, and when the choice actually matters:

**HTTP/JSON.** The default. Universal, debuggable with `curl`, browser-friendly, cacheable via standard HTTP semantics, and every tool understands it. `System.Text.Json` is fast and allocation-conscious (Module 17), and source-generated serialization contexts remove reflection and work under Native AOT. OpenAPI gives you documentation and client generation.

**gRPC.** Protobuf over HTTP/2: smaller payloads, faster serialization, real streaming (client, server, bidirectional), a strongly-typed contract in a `.proto` file, and built-in deadline propagation. `Grpc.AspNetCore` is first-class in .NET. Costs: not human-readable on the wire, needs gRPC-Web or a transcoding gateway for browsers, and the tooling is less universal.

**When the choice actually matters:** high call volume between internal services, large payloads, streaming requirements, or latency budgets where tens of microseconds per call add up. **When it doesn't:** almost everything else. If you're making 50 calls per second, the serialization format is a rounding error next to the network round trip, and JSON's debuggability is worth more.

The honest framing: *"I'd use JSON at the edge for clients and consider gRPC between internal services if the call volume or payload size justifies it — but I'd want a measurement rather than a preference, because the round trip usually dominates the encoding."*

Also worth knowing: **the fastest call is the one you don't make.** Batch endpoints, coarse-grained contracts, and caching (Module 10) beat protocol optimization by an order of magnitude.

---

## Concept 81 — Gateway and BFF

**API gateway.** A single entry point that handles cross-cutting concerns: TLS termination, authentication, rate limiting, routing, request aggregation. On Azure: Front Door or Application Gateway at the edge, API Management for a full API product surface, and **YARP** (Yet Another Reverse Proxy — Microsoft's, used by Azure itself, MIT) when you want an application-level gateway you control in C#.

**Backend for Frontend (BFF).** One gateway per client type — web, mobile, partner API — each tailored to that client's needs. Mobile needs fewer, fatter responses because round trips are expensive on a cellular network; web can afford more, finer ones. The BFF is owned by the client team, which is the point: it lets the client team shape its own contract without negotiating with every service team.

The rule that keeps this healthy: **the gateway must not contain business logic.** It routes, authenticates, aggregates, and transforms shapes. The moment it starts making business decisions, you have a distributed monolith with a single point of coupling that every team must change — the worst possible place to put it, because it's on everyone's critical path and owned by nobody.

The modular-monolith preview of this is Concept 45's composition endpoint: the same idea, in-process, with the same rule about logic.

---

## Concept 82 — Service mesh: what it does and why adoption stalled

**What it is.** Infrastructure that handles service-to-service communication outside your application code: mTLS and service identity, retries, timeouts, circuit breaking, traffic splitting for canaries, and L7 telemetry — all without a line of application code. Classically implemented by injecting an Envoy **sidecar** proxy into every pod.

**Why it's attractive.** Policy in one place, uniform across languages, and zero application changes. If you have forty services in four languages, having the platform enforce mTLS and retry policy is genuinely valuable.

**Why adoption is falling, with the real numbers.** CNCF's 2024 Annual Survey put service mesh adoption at **42%, down from 50% in 2023**, attributing the decline to operational overhead. A separate CNCF/SlashData developer-population report found mesh usage among individual developers falling from **18% (Q3 2023) to 8% (Q3 2025)**. The 2025 CNCF Annual Survey shows mesh at 39% even among the most mature cohort, while Kubernetes production use hit 82%. The complaints are consistent: a sidecar per pod means memory and CPU overhead on everything, a second distributed system to operate, sidecar lifecycle problems with init containers and Jobs, and a genuinely steep learning curve.

**The project's answer.** **Istio's ambient mode reached GA in v1.24 on 7 November 2024** — no sidecars, with a per-node `ztunnel` DaemonSet handling L4 mTLS, identity (SPIFFE) and policy, and optional per-namespace **waypoint** proxies for L7 features. The key property is **incremental adoption**: you can take the L4 secure overlay — which is most of what people actually wanted — without the L7 cost. Istio graduated from the CNCF in July 2023; multi-cluster ambient went alpha in 1.27 (2025). Linkerd remains the simpler alternative; note that Buoyant changed its release model in 2024 so that stable builds are a commercial offering.

**The interview position:** *"A mesh solves a real problem at real scale — uniform mTLS and policy across many services and languages. Below maybe 15–20 services, it's more operating surface than it saves, and the library-level equivalents in .NET (Polly, `HttpClient` resilience handlers, OpenTelemetry) cover most of it. If I needed one now I'd look at ambient mode specifically, because it lets you buy the L4 half without the sidecar tax."*

---

## Concept 83 — Dapr: the building-block alternative

**Dapr** (Distributed Application Runtime) takes a different cut: rather than intercepting the network transparently, it exposes **building-block APIs** over HTTP/gRPC via a sidecar — service invocation, state management, pub/sub, bindings, actors, secrets, configuration, distributed lock, and workflow. Your application calls a local endpoint; Dapr handles the infrastructure behind it.

Current state: **graduated from the CNCF on 30 October 2024** (announced at KubeCon NA on 12 November 2024). v1.15 (February 2025) made Dapr Workflow stable after an engine rewrite and made the Scheduler service the default backend for actor reminders. **v1.16 (16 September 2025)** added multi-application workflows and, in the .NET SDK, Roslyn analyzers that enforce correct configuration. It's built into **Azure Container Apps**, which makes it unusually cheap to adopt on Azure.

**The trade.** You get portability (swap Redis for Cosmos as your state store with a config change), a consistent programming model across languages, and a genuinely good durable-workflow engine — which is a strong answer to the saga problem (Concept 87). You pay with: another runtime to operate and version, a sidecar hop on every call, an abstraction that can hide the storage semantics you actually needed to reason about (Module 12's point about the leaky ORM applies to state stores too), and a smaller ecosystem than Kubernetes-native tooling. Worth noting honestly that CNCF's own project metrics show Dapr's contributor and star counts declining year-over-year — graduated does not mean growing, and an architect should check project health rather than trusting a badge.

**The .NET-specific note:** much of what Dapr offers, .NET already has natively — `HybridCache` for caching (Module 10), `Azure.Messaging.ServiceBus` for pub/sub, Polly for resilience, `Microsoft.Extensions.ServiceDiscovery` for discovery. Dapr's real value is highest in **polyglot** environments where those libraries don't exist uniformly, or where you specifically want its workflow engine.

---

## Concept 84 — Tracing and correlation, end to end

Without this you cannot operate a distributed system. With it, you can. It is the highest-leverage investment in the whole platform floor.

**OpenTelemetry** is the answer, and in .NET it's first-class:

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("ordering-api"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddSource("Ordering")          // your own ActivitySource
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("Ordering")
        .AddOtlpExporter());
```

What you get and what to insist on:

- **Automatic W3C `traceparent` propagation** across `HttpClient` calls, so a single trace ID follows a request through every service.
- **Manual propagation through messaging.** This is the bit people miss: the broker does not propagate context for you. Put `traceparent` in the message's application properties on publish and restore it on consume, or your trace breaks at every async hop — which is exactly where you most need it. Wolverine, MassTransit and the Azure SDK all do this if configured; verify it rather than assuming.
- **Correlation IDs in structured logs**, so a log query by trace ID returns every service's view of one request.
- **`Activity` and `ActivitySource` are the BCL types** — `System.Diagnostics.Activity` *is* OpenTelemetry's span in .NET. No third-party dependency needed to instrument your own code.
- **Aspire's dashboard** consumes OTLP locally, which means you get production-shaped telemetry on your laptop — genuinely one of the best reasons to adopt it.

The interview-grade point: *"I'd instrument with OpenTelemetry before the first extraction, not after — and I'd specifically check that trace context survives the broker hop, because that's the one that silently breaks and it's the hop you most need to see."*

---

## Concept 85 — Resilience at every boundary

Everything from Module 13, now mandatory rather than advisable. In .NET 8+ the built-in resilience handlers cover the common case with almost no code:

```csharp
builder.Services.AddHttpClient<IBillingClient, BillingClient>(c =>
{
    c.BaseAddress = new Uri("https+http://billing");   // service discovery name
    c.Timeout = TimeSpan.FromSeconds(5);
})
.AddStandardResilienceHandler();   // retry + circuit breaker + timeout + rate limiter
```

`AddStandardResilienceHandler()` (from `Microsoft.Extensions.Http.Resilience`, built on Polly v8) gives you a sensible default pipeline. Module 25 takes Polly properly. The judgment points that matter here:

- **Timeouts must form a budget, not a set of round numbers.** If your caller has a 3-second budget and you call three services, each cannot have a 3-second timeout. Propagate deadlines (gRPC does this natively; with HTTP you pass a remaining-budget header or shorten the `CancellationToken`).
- **Retry only idempotent operations.** Retrying a non-idempotent `POST` after a timeout is how you double-charge a customer. This is Concept 22's ambiguity with money attached.
- **Jitter is mandatory**, or retries synchronize into a thundering herd exactly when the downstream is weakest.
- **Circuit breakers need a fallback.** Opening the circuit converts a slow failure into a fast one, which is good — but the caller still has to do something. Decide what: cached value, degraded response, queued for later, or an honest error.
- **Bulkheads per dependency.** Separate connection and concurrency limits, so one slow dependency can't consume all your capacity.
- **Resilience belongs in the adapter, not the handler** (Module 20, Concept 44). A retry loop in a use case is a leak.

---

## Concept 86 — Database per service, and the analytics answer

**Database per service is non-negotiable** for real microservices. A shared database means a shared schema, which means coordinated releases, which means it isn't a microservice (Concept 4). The moment you share, you have all the cost and none of the independence.

Which raises the question every organization hits in month four: **how do we report across services?** The options, in order of how often they're right:

1. **An analytical store fed by events or CDC.** Every service publishes events (or you use change data capture on its database); a pipeline lands them in a warehouse or lakehouse — Microsoft Fabric, Synapse, Databricks, Snowflake. Reporting queries run there. This is the standard answer and it is the right one. Cost: latency (minutes to hours), a pipeline to own, and schema evolution on the analytics side.
2. **CQRS read models** for operational queries that need to be fresh — a projection service that subscribes to events and maintains a denormalized view for a specific screen (Module 23/24). Use for a handful of high-value views, not as a general reporting strategy.
3. **API composition** — query several services and join in memory. Only for small, bounded result sets. Cannot sort or paginate across sources correctly.
4. **A read replica per service that the analytics team may read.** Pragmatic, and a slippery slope: the moment analytics depends on a service's internal schema, that service can't change it. If you do this, expose a **published view** (a stable, versioned schema contract) rather than the raw tables — the same contract discipline as an API.

What is not an option: letting the reporting tool query every service's production database directly. It's the fastest possible route back to common coupling (Concept 59), and it is how most "we have microservices" organizations quietly lost their independence.

---

## Concept 87 — Sagas across services

You lost the transaction (Concept 23). Here's what replaces it, at design-round depth.

**Choreography.** Each service reacts to events and publishes its own. Order publishes `OrderPlaced`; Payment consumes it and publishes `PaymentCaptured`; Inventory consumes that and publishes `StockReserved`. No central coordinator.
*Pros:* loosely coupled, no single point of failure, easy to add a participant.
*Cons:* the workflow exists nowhere — you cannot read the business process in any one place; debugging means reconstructing it from traces; cyclic event dependencies are easy to create accidentally.

**Orchestration.** A coordinator (a saga/process manager) explicitly drives the steps and issues commands: reserve stock, capture payment, create shipment — and on failure, runs compensations in reverse.
*Pros:* the workflow is explicit, readable, testable and monitorable; error handling and timeouts have an obvious home.
*Cons:* the orchestrator is a component to own, and it can accumulate business logic that belongs to the participants.

**The default worth stating:** orchestration for anything with more than ~3 steps or with real compensation logic; choreography for simple notification-style flows. The readability of an explicit process is usually worth more than the coupling it costs — and in an interview, "I'd want the business process to be visible in one place" is a strong, experience-flavoured justification.

**.NET options:** MassTransit sagas and state machines (commercial from v9), Wolverine sagas (MIT), **Dapr Workflow** (stable since 1.15, with multi-app workflows in 1.16), **Azure Durable Functions**, or Azure Logic Apps for integration-shaped workflows. All of them solve the same hard part: **durable state for a long-running process that must survive crashes.**

And the design constraint that isn't optional: **compensations are business decisions, not technical rollbacks.** You cannot un-ship a parcel. Someone from the business has to say what happens instead — and surfacing that requirement early is architecture work.

---

## Concept 88 — Shared libraries: coupling through the back door

The most common way teams accidentally rebuild the monolith while believing they've escaped it.

A shared NuGet package containing domain types, a shared client library, a shared "common" utilities assembly. Everyone references it. Now a change to it requires bumping the version in every service and redeploying them — and you have **lockstep releases**, which means you don't have microservices (Concept 4).

The rules that keep it safe:

- **Shared libraries may contain technical concerns only.** Logging setup, telemetry configuration, a JSON serialization policy, an HTTP handler, an authentication helper. **Never domain types, never business rules.**
- **Never share entity or aggregate types.** Each service owns its model (Concept 41).
- **Contract packages are the exception, with a rule:** a `Billing.Contracts` package containing the event and DTO records is legitimate — it's the contract, and it should be **versioned, additive-only, and owned by the publisher**. Consumers upgrade on their own schedule. That's a genuine contract, not shared code.
- **Avoid the "everyone must be on the latest version" pattern.** If a security fix forces an emergency coordinated release, that's acceptable. If a feature does, your boundary is wrong.
- **Prefer duplication.** Twenty lines of the same helper in two services is cheaper than a shared package that couples their release cycles. This is Concept 41 again, and it is the correct instinct in a distributed system even though it feels wrong.

---

## Concept 89 — Repo strategy: mono vs. poly

A tooling choice that is frequently mistaken for an architecture choice. State that first, because interviewers sometimes bait it.

**Monorepo.** One repository, all services. Atomic cross-service commits, easy refactoring across boundaries, one version of the tooling, easy code sharing. Google, Meta and Microsoft run at enormous scale this way. Costs: needs real tooling to scale (selective builds, a build graph, sparse checkout), and — the architectural risk — **it makes cross-service coupling easy, which erodes the boundaries you're paying for**.

**Polyrepo.** One repository per service. Hard boundaries, clear ownership, independent CI, simple access control. Costs: cross-cutting changes are painful, shared code needs packaging, and tool/version drift across repos is real.

**The honest position:** repository layout doesn't determine architecture, but it does shape incentives. A monorepo with strict build-level boundaries (a per-service build graph, no cross-service project references) gives you the best of both. A monorepo without them quietly becomes a distributed monolith, because making a cross-service change is one commit away and nobody's build stops them.

In .NET specifically, a monorepo with `Directory.Build.props`, **Central Package Management** (one `Directory.Packages.props`), per-service solution filters or `.slnx` files, and architecture tests enforcing the boundaries is a very workable setup — and it's also exactly the setup you already have if you started as a modular monolith, which is another reason that path is smooth.

---

## Concept 90 — Testing strategy across services

The pyramid from Concept 26, now with tooling:

1. **Unit tests** — domain logic, no I/O. Thousands, milliseconds. `xUnit`/`NUnit`, no mocking framework needed for pure logic.
2. **Service tests** — one service in-process, real database via **Testcontainers** (`Testcontainers.MsSql`, `Testcontainers.PostgreSql`, `Testcontainers.Azurite`), fakes at every outbound boundary. `WebApplicationFactory` for the HTTP surface. **This is where most confidence lives.**
3. **Contract tests** — `PactNet` for consumer-driven contracts (Concept 72). The provider's build fails if it would break a consumer.
4. **A thin end-to-end smoke suite** — a handful of critical journeys against a deployed environment. A canary, not a gate. **Aspire's test host** (`Aspire.Hosting.Testing`) can start the full application graph, which makes a realistic integration tier cheap; `jasontaylordev/CleanArchitecture` now ships Aspire-based functional tests as its default.
5. **Production verification** — synthetic monitoring, canary/progressive deployment (Container Apps revisions make traffic splitting a configuration value), feature flags, and real SLO alerting.

The rule that keeps independent deployability alive: **no test that requires another team's service to be running may gate your deployment.** The moment it does, you have a coordinated release, and you've spent the microservices premium without receiving the goods.

---

# Part G — The architect's view

Everything above is input. This part is the deliverable: how you actually answer, decide, cost, document and defend.

---

## Concept 91 — The 60-second answer

When an interviewer asks "monolith or microservices?", this is the shape. Six moves, about a minute:

1. **Reframe.** *"There are two decisions here — how modular the code is, and how many processes we deploy. I'd want strong modularity either way; the question is really whether to distribute."*
2. **Ask for the deciding input.** *"The thing that decides it for me is team structure: how many teams need to ship independently, and are they blocking each other today?"*
3. **State a default.** *"For a team of this size, I'd start with a modular monolith — one deployable, modules by business capability, each owning its own schema."*
4. **Name the trigger.** *"I'd extract a service when one of a few specific things happens: deploy contention between teams, a capability with a radically different scaling profile, a hard failure-isolation or compliance requirement, or a technology we can't host in-process."*
5. **Name the first candidate and why.** *"Given what you've described, the first thing I'd extract is X — it has the thinnest data relationship to everything else and the most distinct scaling profile."*
6. **Name the cost you're accepting.** *"The price of staying in one deployable is that everything ships together, so I'd invest in feature flags and a fast pipeline to push that pain out as far as possible."*

That answer scores because it has a position, a criterion, a trigger and an acknowledged cost — which is what "judgment" means on a rubric. Compare it with "it depends," which reads as either evasion or lack of experience.

---

## Concept 92 — The decision scorecard

For the architect round, when you need to justify it rather than assert it. Score each factor, state the threshold, say what it implies.

| Factor | Points toward monolith | Points toward services | Weight |
|---|---|---|---|
| **Teams needing independent release** | 1–4 | 5+ | **Highest** |
| **Deploy contention today** | Rare | Weekly, costly | **Highest** |
| **Operational maturity** | No tracing, manual deploys | Full platform floor exists | **High** |
| **Scaling asymmetry** | Uniform profile | One capability 10×+ different | High |
| **Domain clarity** | Still discovering boundaries | Boundaries stable for 12+ months | High |
| **Failure-isolation requirement** | Whole-system SLO | One capability needs its own | Medium |
| **Compliance/residency isolation** | None | PCI/PII/regional scope reduction | Medium (decisive when present) |
| **Technology necessity** | One stack fits | A capability needs a different runtime | Medium |
| **System lifespan** | < 2 years | Long-lived, strategic | Low |
| **Budget for platform capability** | None | 1–3 engineers available permanently | High |

The thresholds to say out loud: **if teams ≤ 4 and deploy contention is rare, the premium isn't earned — modular monolith.** **If operational maturity is low, build the platform floor first regardless of everything else.** **If domain clarity is low, do not distribute — you'll bake in a boundary you'll have to migrate later.**

And the framing that makes this land: this is not a scorecard that produces a number. It's a scorecard that produces a *conversation* in which the weights are explicit, which is the thing an architect is actually paid to do.

---

## Concept 93 — Defaults by team size and system age

What you'd actually start with, said plainly. Interviewers like this because it's falsifiable and specific.

**1–3 engineers.** One project, feature folders, one database. Modules are folders, not projects. Don't over-engineer: you have no coordination problem and every hour spent on structure is an hour not spent finding product-market fit. Keep one thing: **don't let the data model become a tangle you can't separate later** — even here, think in terms of which tables belong to which capability.

**4–15 engineers (1–2 teams).** Modular monolith, properly. Modules as projects, `internal` by default, schema per module, an outbox, architecture tests. This is the sweet spot for the full Part C treatment and the configuration that serves most companies for years.

**16–50 engineers (3–6 teams).** Modular monolith with module ownership assigned per team, plus **1–3 extracted services** where a trigger fired. Build the platform floor now, because you'll need it. Invest heavily in feature flags and pipeline speed to delay deploy contention.

**50+ engineers (7+ teams).** Services along team boundaries, with a platform team absorbing the extraneous load (Concept 10). A modular monolith core often survives at the middle of this — and that's fine, not a failure.

**Cross-cutting by system age:** a system in its first year has unstable boundaries — do not distribute. A system that's been stable for three years has boundaries you can trust — the git history will tell you where they are (Concept 57). A fifteen-year-old system needs Concept 76's sequencing regardless of headcount.

---

## Concept 94 — The cost conversation

Translating architecture into the language executives act on. This is a Module 33 skill, previewed here because this decision is where it most often comes up.

**Don't say:** "Microservices add complexity." (Unquantified, easily dismissed, sounds like preference.)

**Do say:** *"Splitting into services costs roughly N engineer-months of migration work, plus about one to two engineers of permanent platform capacity, plus an estimated X% increase in infrastructure run rate. In exchange we remove the release contention that currently costs each of our four teams about a day a week. At four teams that's roughly break-even; at eight it's clearly positive. My recommendation is to modularize now — which is about a third of the cost, is reversible, and delivers most of the maintainability benefit — and revisit extraction when we cross six teams or when the checkout path's scaling profile diverges."*

The elements that make it work:

- **A number for the cost**, even a rough one with the assumptions stated.
- **A number for the benefit**, in time or money, tied to something the business already feels.
- **A break-even condition**, so the decision has a trigger rather than being permanent.
- **A cheaper intermediate option**, which is almost always available and almost always right.
- **An explicit revisit point**, which is what turns a decision into a plan.

And the reframe that defuses the most common executive pressure: when "we need microservices" comes from above, the question to ask is *"what outcome are we trying to get?"* — because the answer is usually faster delivery, and there are cheaper routes to it.

---

## Concept 95 — The ADR

Write this decision down. Module 31 covers ADRs properly; this is the shape for this specific decision, and it's a realistic take-home or design-document exercise.

```markdown
# ADR-014: Modular monolith with selective extraction

## Status
Accepted — 2026-09-24. Revisit at 6 teams or 2027-06, whichever first.

## Context
- 18 engineers across 3 teams; one deployable today.
- Deploy contention: roughly 2 incidents/month of a team blocked by another.
- Domain boundaries: 4 candidate contexts from EventStorming (Aug 2026);
  co-change analysis shows 31% of commits span Ordering/Billing — that pair
  is not yet a clean boundary.
- No distributed tracing; deployments are semi-automated.
- Media processing shows a 12× memory profile vs. the rest of the app.

## Options considered
1. Status quo — monolith with no enforced internal boundaries.
2. Modular monolith with enforced module boundaries.   ← chosen
3. Full decomposition into ~6 services.
4. Modular monolith + extract media processing now.

## Decision
Option 2, with option 4 as the planned next step once the platform floor exists.
Modules: Ordering, Billing, Catalogue, Media. Each owns its schema with
scoped DB credentials. Cross-module: PublicApi contracts + outbox events.
Enforced by project references, `internal` visibility, ArchUnitNET rules and
per-schema database permissions.

## Consequences
+ Boundaries and ownership without the distribution premium.
+ Extraction becomes a deployment change (see extraction checklist).
+ Ordering/Billing coupling is now visible and will be addressed before any split.
− Everything still deploys together; mitigated by feature flags + pipeline work.
− No runtime isolation; the media memory profile stays a shared risk until extracted.

## Revisit triggers
- Team count reaches 6, OR
- Deploy contention exceeds 1 incident/week, OR
- Media processing forces the whole app onto a larger SKU.
```

The details that make an ADR good rather than ceremonial: **numbers in the context**, **options that were genuinely considered and rejected with reasons**, **negative consequences stated honestly**, and **explicit revisit triggers**. An ADR without the last item is a decision with no expiry date, which is how organizations end up defending a 2019 choice in 2026.

---

## Concept 96 — The pendulum, handled

You will be asked about the trend. Here is how to handle it without sounding like a content farm.

**The real, documentable situation as of September 2026:**

- Kubernetes adoption is *up* — 82% of container users run it in production (CNCF Annual Survey 2025, 628 practitioners, published January 2026), up from 66% in 2023. The platform is not in retreat.
- **Service mesh adoption is genuinely down** — 50% → 42% among organizations (CNCF 2024), and 18% → 8% among individual developers between Q3 2023 and Q3 2025 (CNCF/SlashData, 12,021 developers). The cited reason is operational overhead. Istio's response was ambient mode, GA November 2024.
- The top barrier to cloud native adoption is now **cultural (47%)**, not technical — which is Conway's law in survey form.
- There is a real, visible industry mood shift toward "modular first, extract selectively," reflected in tooling (Aspire making hybrid topologies cheap) and in prominent teams publishing consolidation stories.

**And the part that will make you stand out:** the widely circulated claim that *"a 2025 CNCF survey found 42% of organizations are consolidating microservices back into larger deployable units"* **does not appear in any CNCF publication I can find.** It shows up in dozens of near-identical 2026 blog posts, none with a primary link. The number 42% *does* appear in CNCF data — as service mesh adoption in 2024. Related figures in the same posts ("microservices cost 3.75×–6× more," "Gartner says 60% of teams regret it") have the same provenance problem.

So if an interviewer cites it, the move is neither to accept it nor to correct them bluntly:

> *"I've seen that figure a lot this year — I went looking for the primary source and couldn't find it in CNCF's published surveys; the 42% in their data is service mesh adoption in 2024. The trend it's pointing at is real though: mesh adoption genuinely dropped from 50% to 42%, and there's a visible move toward modular-first. I'd rather argue it from our own numbers than from that one."*

That answer demonstrates three things at once: you read primary sources, you're not contrarian for its own sake, and you know which numbers you'd actually use. It's one of the highest-leverage twenty seconds available in this topic.

---

## Concept 97 — Case studies that are actually evidence

The ones worth knowing, with the correct reading of each.

**Amazon Prime Video (2023).** One component — the Video Quality Analysis monitoring service — moved from AWS Step Functions + Lambda + S3 to a single process on EC2/ECS, **cutting infrastructure cost by over 90%** and raising the scaling ceiling. The bottlenecks were Step Functions state-transition account limits and the cost of shuttling video frames through S3; they'd hit a wall at about 5% of target load. The team's own conclusion was explicitly case-by-case. **The lesson is about data gravity and high-frequency data paths (Concept 19), not about microservices being over.** Prime Video remains a large distributed system.

**Segment (2018, "Goodbye Microservices").** Went from a monolith to 140+ microservices and back to a monolith. The driver was that each destination integration became a service with its own queue, its own repository, its own dependency versions — so operational overhead scaled with integration count while the code was nearly identical across them. **The lesson: don't create a service per instance of a thing; create one per capability (Concept 62).**

**Shopify (2019, "Deconstructing the Monolith").** Explicitly evaluated microservices and **chose a modular monolith instead**, citing distributed-systems complexity as not worth it for their case. Introduced the "design payoff line" — the point at which poor design starts costing more than the fix. Still one of the best-argued public statements of this position, and a very useful one to cite because it's a company operating at genuine scale choosing the modular monolith on purpose.

**Uber (2020, DOMA).** After thousands of microservices produced unmanageable complexity, Uber introduced **Domain-Oriented Microservice Architecture** — grouping services into domains with gateways and explicit layering. Sometimes described as "microservices to macroservices." **The lesson: extreme decomposition has a cost, and the correction is coarser boundaries, not a return to a monolith.**

**Istio ambient mode (GA 2024).** A whole mesh ecosystem responding to "the sidecar overhead isn't worth it" by re-architecting the data plane. Useful as evidence that the *industry* discovered the operational cost was real.

**Netflix.** The canonical pro-decomposition story, and worth citing honestly: it followed a catastrophic database corruption in 2008, took years, and was accompanied by building an entire platform organization and open-source ecosystem (Eureka, Ribbon, Hystrix, Zuul) because the tooling didn't exist. **The lesson is that the cost is real and Netflix paid it deliberately for reasons most companies don't have.**

Using two or three of these *with the correct reading* is worth more than listing ten.

---

## Concept 98 — The anti-pattern catalogue

What reviewers actually look for. Fourteen items:

1. **Distributed monolith** — services that deploy together (Concept 6).
2. **Shared database** across services (Concept 59).
3. **Entity services** — UserService, OrderService (Concept 55).
4. **Nanoservices** — more services than engineers (Concept 62).
5. **Synchronous chains** 3+ deep (Concepts 18, 21).
6. **No idempotency** on message consumers (Concept 22).
7. **Shared domain library** requiring coordinated bumps (Concept 88).
8. **Business logic in the gateway** (Concept 81).
9. **End-to-end test suites gating deployment** (Concepts 26, 90).
10. **No distributed tracing** — operating blind (Concept 84).
11. **Distributed transactions** attempted with 2PC across services (Concept 23).
12. **Chatty N+1 calls** across a network boundary (Concept 18).
13. **Extraction before modularization** — reversed order of operations (Concept 67).
14. **No extraction trigger** — "we're going microservices" as a programme rather than a response to a condition (Concept 65).

And the modular-monolith-specific ones: a `SharedKernel` containing business logic (Concept 42); cross-module joins "just for a report" (Concept 59); no per-schema database credentials so the boundary is advisory (Concept 35); modules that aren't owned by anyone (Concept 64).

---

## Concept 99 — When the interviewer wants microservices

Sometimes the interviewer — or the hypothetical CTO in the scenario — has already decided. This is a test, and what's being scored is **whether you can disagree well**, which is a Module 34/35 skill applied to a technical decision.

What not to do: cave immediately (no judgment), or dig in and lecture (no collaboration).

The shape that works:

1. **Take the goal seriously, not the solution.** *"Let me make sure I understand what we're optimizing for — is this mainly about teams being able to ship independently, or about a specific scaling problem?"*
2. **Agree with what's right about it.** *"If it's release contention at eight teams, I agree that's exactly what decomposition solves and I'd support it."*
3. **Name the precondition, not the objection.** *"The thing I'd want to establish first is the platform floor — tracing, per-service pipelines, on-call ownership. Without those the first extraction is going to hurt."*
4. **Offer the path that gets them there.** *"I'd propose we modularize and extract the two capabilities with the clearest triggers this quarter, rather than a full decomposition — same direction, much lower risk, and we learn what the real cost per service is before committing to twelve of them."*
5. **Say what would change your mind, and commit.** *"If the deploy contention numbers are worse than I'm assuming, that changes the sequencing and I'd move faster. And if we decide to go all in, I'll build it properly — I'd just want the ADR to record what we're accepting."*

That's disagree-and-commit with a plan attached, and it is very close to what senior and architect rubrics are literally scoring for.

---

## Concept 100 — The close

The paragraph to leave in the interviewer's head:

> *"The way I think about this: modularity and distribution are separate decisions. Modularity is nearly free and always worth it — clear boundaries, hidden internals, and each module owning its own data. Distribution is expensive: it costs me latency, availability, transactions, and a permanent operational tax, and it buys me exactly one thing — teams shipping independently. So my default is a modular monolith built so that extraction is a deployment change rather than a rewrite: schema per module, contracts in their own assembly, an outbox from day one, and the boundaries enforced by database permissions and architecture tests rather than by good intentions. Then I extract a specific module when a specific trigger fires — deploy contention, a scaling profile that's genuinely different, a compliance boundary — and I measure whether it worked. Most systems I've seen end up as a modular monolith plus two to five services, and that's not a compromise; it's usually the right answer."*

Six things are on display in that paragraph: a framework, a default, a mechanism, a trigger, a measurement, and an honest conclusion that isn't dogmatic. That is the full set of things this topic is scored on.

---

# Putting it together

Six worked examples in the shape these questions actually arrive.

---

## Worked example 1 — "Design the backend for a B2B invoicing SaaS. Monolith or microservices?"

**Don't answer yet. Get the deciding inputs** (three questions, thirty seconds):

> *"Three things would change my answer: how many engineers and teams, whether there's a capability with a very different scaling or compliance profile, and whether there's an existing platform — tracing, pipelines, on-call."*

Say the interviewer answers: 12 engineers in 2 teams, greenfield, no platform yet, PDF rendering is CPU-heavy and spiky at month-end.

**The answer:**

> *"With two teams and no platform, I'd start with a modular monolith. Modules by capability from an EventStorming pass — I'd expect roughly Identity, Customers, Invoicing, Payments, Documents and Notifications. Each owns its schema with its own database credential, so cross-module data access is a permission error rather than a code review. Cross-module reads go through a small public contract per module; state changes are integration events written to an outbox in the same transaction. Enforced with project references, `internal` by default and a handful of architecture tests.*
>
> *One deliberate exception: PDF rendering. It's CPU-heavy, spiky at month-end, and it has almost no data relationship to the rest of the domain — it takes an invoice model and returns bytes. That's the cleanest possible extraction candidate, so I'd build Documents as a module but give it a queue-based interface from day one, and extract it to its own Container App with KEDA queue-length scaling the first time month-end forces us onto a bigger SKU for everything.*
>
> *The trigger list for anything else: team count past five, deploy contention past about one blocked release a week, or a compliance requirement — payments data is the likely candidate there. The cost I'm accepting is that everything else ships together, so I'd put real effort into feature flags and pipeline speed."*

**Why this scores:** a default with a reason, a specific exception with a specific justification, a named trigger, and an acknowledged cost. It also demonstrates the scale-asymmetry trigger being applied correctly rather than invoked generically.

---

## Worked example 2 — "We have 40 services, 15 engineers, and everything is slow and fragile. What do you do?"

The over-decomposition scenario (Concept 62). The instinct is to propose tooling; the right answer is to propose consolidation — but only after evidence.

> *"Forty services and fifteen engineers means roughly three services per person, so nobody owns anything and everybody is on call for everything. Before proposing anything structural I'd want three pieces of data, and I can get all of them in about two weeks:*
>
> *First, the deploy graph: which services actually ship independently, and which always go out together? Any group that always ships together is one service wearing several hats.*
>
> *Second, co-change analysis on the git history: which services change in the same commit or the same PR? Those belong together.*
>
> *Third, the call graph from tracing — if we don't have distributed tracing, that's the first thing I build, because we can't diagnose this system at all without it.*
>
> *My expectation is that those three overlap heavily and point at maybe six to eight real capabilities. The plan is then consolidation: merge the services in each cluster into one deployable per capability, keeping them as modules inside it so we don't lose the boundaries. Merge code first with both deployments still live, reconcile data second, switch traffic third, decommission last — and decommission properly, including the pipelines, dashboards and alerts.*
>
> *I'd sequence it by pain: start with the cluster causing the most incidents, ship it, measure MTTR and lead time, and use that result to fund the rest. And I'd be explicit with the team that this isn't an admission that microservices were wrong — it's that the boundaries were at the wrong granularity, and coarser ones fit fifteen people."*

**Why this scores:** evidence before action, a sequencing that produces early proof, honesty about the reverse migration being real work, and attention to the human framing — which is what separates an architect answer from an engineer answer.

---

## Worked example 3 — "Walk me through extracting the payments module from your monolith."

> *"First I'd check extraction readiness, because the answer determines whether this is weeks or quarters. Nine things: does Payments own its schema; are there cross-module foreign keys into it; are there joins across it; does it have a small public contract; does everything inbound go through that contract; are its outbound notifications going through an outbox; are the consumers idempotent; does it own its configuration, migrations and workers; and do its tests run with fakes for everything else. Whichever of those fails is the actual work.*
>
> *Assuming it's mostly there, the order is logical, then data, then process — never reversed.*
>
> *Data first: inventory every read and write of the payments tables from outside the module — including reports, ETL and the BI tool, which is always where the surprises are. Replace each with a contract call or a projection. Then move it to its own schema with its own credential, which surfaces anything I missed as a permission error in a test environment. Then its own database.*
>
> *Then the process. New host project that calls only `AddPaymentsModule()` and `MapPaymentsEndpoints()` — the module code doesn't change. Swap the in-process bus registration for Service Bus. Put it behind the gateway with a prefix route.*
>
> *The switch is branch-by-abstraction: the `IPaymentsModule` interface already exists, so I add an HTTP-backed implementation and a comparing implementation that calls both and logs divergence. Deploy in compare mode, watch for a week including a month-end, flip to remote by configuration, keep the in-process implementation as a rollback for a month.*
>
> *Payments specifically needs extra care on two things: idempotency, because retrying a capture after a timeout can double-charge — so every command carries an idempotency key and the service deduplicates; and compensations, because refunding isn't un-charging, it's a business process with its own record. I'd want the finance team in the room for that part, not just engineering.*
>
> *Measuring it: lead time and deploy frequency for the payments team, p99 on checkout before and after, composed availability of the checkout journey, and the infra delta."*

---

## Worked example 4 — "The CTO read that microservices are dead and wants us to merge our 8 services back. We have 60 engineers."

The reverse of Concept 99, and the test is whether you resist a bad idea from above as readily as a bad idea from below.

> *"Sixty engineers across — I'd guess — six to eight teams, with eight services, is roughly one service per team. That's the shape the whole industry converged on, and it's probably close to right. I'd want to check that before doing anything.*
>
> *The check is three questions: do the services deploy independently, or in lockstep? Does each have one owning team? And does each own its own data? If yes to all three, we have working microservices at appropriate granularity and merging them would remove release independence from six teams to buy operational simplicity we don't especially need at that size.*
>
> *If some answers are no, then we have a distributed monolith in places, and the fix is targeted rather than wholesale — usually splitting a shared database or merging two services that always ship together.*
>
> *On the trend itself: the consolidation story is real, and it's mostly about teams that over-decomposed — a service per integration, or forty services for fifteen people. It's not a finding that one service per team is wrong. I'd also gently flag that the specific statistic circulating this year — 42% of organizations consolidating — doesn't seem to trace back to a CNCF publication; the 42% in their data is service mesh adoption.*
>
> *What I'd offer instead: if the underlying concern is operational cost or incident load, let me measure those directly. If the platform overhead is genuinely too high for eight services, the cheaper fix is usually to invest in the platform, not to merge. And if two specific services are chatty on a hot path, I'll merge those two — that's a real win and a much smaller bet."*

---

## Worked example 5 — "Two modules need to be consistent. How do you handle that once they're separate services?"

The consistency question, and the answer that demonstrates Module 12's material in this context.

> *"First I'd push back on 'need,' because it's doing a lot of work. There's a real difference between an invariant that must never be violated and a business expectation that things converge quickly. 'Never oversell inventory' is the first kind. 'The order total and the invoice total should match' is usually the second.*
>
> *If it's genuinely an invariant that must hold atomically, that's strong evidence the two things belong in the same service — the transaction boundary and the aggregate boundary should line up. I'd rather move the boundary than distribute the invariant. That's the cheapest answer and it's the one people skip.*
>
> *If they must be separate, I'd model it as a reservation with a timeout: Inventory reserves stock with a TTL, Ordering completes within the window, and an expiry process releases unclaimed reservations. That converts a distributed invariant into a local one plus a compensating timeout, which is a pattern the business can actually understand.*
>
> *For the multi-step case, an orchestrated saga: a process manager drives reserve → capture → confirm and runs compensations on failure. Orchestration rather than choreography, because with three or more steps I want the business process readable in one place. Durable state via Dapr Workflow or Durable Functions, so it survives a crash mid-flight.*
>
> *And the part that isn't technical: compensations are business decisions. We can't un-ship a parcel. Someone from operations has to decide what happens when payment fails after dispatch — and that conversation is part of the design, not a follow-up."*

---

## Worked example 6 — "Here's our architecture diagram. Critique it."

Twelve boxes, arrows between most of them, one database icon at the bottom connected to several services.

> *"A few questions before I critique, because the diagram won't tell me the things that matter most.*
>
> *That database at the bottom with several lines into it — is that one database that several services share, or is it one icon representing several? If it's shared, that's the thing I'd fix first, and it means these aren't independently deployable regardless of how they're deployed.*
>
> *Second, the arrows: are they synchronous calls or events? The diagram draws them the same way, which hides the most important property. I can count at least one path four hops deep — if that's synchronous, it has the latency of four calls and the availability of four services multiplied, which for three-nines components is about 99.6%.*
>
> *Third, ownership: how many teams, and does each box have one? Twelve boxes and three teams means nine of them are nobody's first priority.*
>
> *Fourth, I don't see a gateway, an identity boundary or anything representing observability. If there's no distributed tracing, an incident on that four-hop path is going to be very expensive to diagnose.*
>
> *What I like: the boxes have capability names rather than entity names, so the decomposition is by business capability, which is the right instinct. And the notification service on the edge with one inbound arrow is exactly the shape a well-bounded service should have.*
>
> *If the shared database is real, my recommendation is to stop extracting and fix data ownership first — split schemas, remove cross-service joins, enforce with credentials. That's unglamorous and it's the work that determines whether the rest of this is an architecture or a diagram."*

**Why this scores:** questions before verdicts, the two highest-signal questions asked first, specific arithmetic applied, something genuinely positive identified, and a recommendation with a sequencing rationale.

---

## Common questions and what a strong answer contains

**"Monolith or microservices?"** Reframe into two axes, ask for team count and deploy contention, give a default with a trigger and an acknowledged cost. Never "it depends" unqualified (Concepts 1, 91).

**"What is a modular monolith?"** One deployable; internal modules with enforced boundaries; each owns its data; cross-module contact only by contract or event. Then the payoff: extraction becomes a deployment change rather than a rewrite (Concepts 31, 52).

**"What's the difference between a modular monolith and microservices?"** Deployment topology, and that's it. Same boundaries, same ownership discipline, same contracts — one process vs. many. Which means the benefits people attribute to microservices are mostly available without the network (Concepts 1, 15).

**"When would you use microservices?"** Five or more teams blocking each other on release; a capability with a radically different scaling profile; a hard failure-isolation or compliance boundary; a technology that can't be hosted in-process; an acquisition. Plus the precondition: a platform floor that already exists (Concepts 65, 28).

**"What's a distributed monolith and how do you spot it?"** Services that can't deploy independently. Three questions: do two services write the same tables, do they deploy together, does one being down fail the other's requests (Concept 6).

**"What does a network call actually cost?"** ~1 ns in-process vs. ~500 µs same-datacenter — six orders of magnitude. Plus serialization and allocation, plus the permanent code cost: timeout, retry, idempotency, circuit breaker, fallback (Concepts 18, 22).

**"Do microservices improve availability?"** Only if the dependencies are asynchronous or optional. Five services at 99.9% in series compose to 99.5% — from 9 to 44 hours a year. Decomposition improves *blast radius* for process-level failures; it worsens availability for synchronous required dependencies (Concept 21).

**"How do you handle transactions across services?"** You don't get them. Sagas with compensations, orchestrated when there are more than a couple of steps; transactional outbox for reliable publishing; reservations with timeouts to turn a distributed invariant into a local one. And compensations are business decisions (Concepts 23, 87).

**"How do you find service boundaries?"** Bounded contexts via EventStorming with domain experts, validated against git co-change analysis. Decompose by business capability, never by entity. Then check the boundary against the org chart, because unowned boundaries erode (Concepts 53–57, 64).

**"What's wrong with a UserService and an OrderService?"** That's the entity-service trap: no business operation fits inside one of them, so every use case becomes a distributed transaction across four chatty CRUD wrappers. Name boundaries after capabilities (Concept 55).

**"How do you share data between services?"** You don't share storage. Three legal options: call the owner's contract, keep an event-fed replica of just the fields you need, or build a projection. Duplication across boundaries is correct, not a DRY violation (Concepts 38, 41).

**"How do you report across services?"** An analytical store fed by events or CDC — Fabric, Synapse, Databricks, Snowflake. CQRS read models for a handful of fresh operational views. Never let the BI tool query production service databases directly (Concept 86).

**"How do you test this?"** Unit → service tests with Testcontainers and fakes at every outbound boundary → consumer-driven contract tests (Pact) → a thin end-to-end smoke suite → production verification. The rule: no test requiring another team's service may gate your deploy (Concepts 26, 90).

**"How do you migrate a monolith to services?"** Logical boundary → data boundary → process boundary, never reversed. Strangler fig with a facade, branch by abstraction so the switch is a config flag, expand/contract for every schema and contract change, dual write → backfill → read switch → cutover for data. Every step reversible in under an hour (Concepts 67–73).

**"What's the hardest part?"** The data. Steps one and two of a data separation — finding every cross-boundary read and eliminating it — are about 70% of the effort, and the discoveries are always in reports and ETL jobs nobody documented (Concept 71).

**"Do you need a service mesh?"** At 40 services in several languages, probably — uniform mTLS and policy is real value. Below 15–20, it's more operating surface than it saves, and .NET's own libraries cover most of it. If you do need one, look at Istio's ambient mode: it lets you take the L4 half without the sidecar cost (Concept 82).

**"What about Dapr?"** Building-block APIs over a sidecar; CNCF graduated October 2024; strongest where you're polyglot or want its durable workflow engine, and it's built into Azure Container Apps. Weakest argument in a pure .NET shop, where `HybridCache`, the Service Bus SDK, Polly and service discovery already cover the same ground (Concept 83).

**"What's the biggest risk with a modular monolith?"** Not performance — deployment coupling and discipline decay. Mitigate the first with feature flags and a fast pipeline; mitigate the second by making the rules machine-enforced, especially per-schema database credentials, because that's the only one a deadline can't override (Concept 51).

**"Isn't it slower to develop with all those boundaries?"** Marginally, for changes inside one module — and dramatically faster for changes that would otherwise ripple. The measurable version is files-per-change and PR size, which is what I'd track rather than arguing about it (Concepts 11, 74).

**"Isn't the industry moving back to monoliths?"** Partly, and specifically away from *over*-decomposition and from service meshes — mesh adoption fell from 50% to 42% between 2023 and 2024 in CNCF's data. Kubernetes adoption meanwhile rose to 82% in production. The convergent position is modular-first with selective extraction. I'd also be careful with the "42% consolidating" figure that's everywhere this year — I can't find it in a CNCF publication (Concept 96).

**"What about the Amazon Prime Video case?"** One component, originally serverless rather than microservices, bottlenecked by Step Functions state-transition limits and S3 data transfer for video frames; consolidating into one process cut cost by 90%. The lesson is about high-frequency data paths and data gravity, not about microservices generally — Prime Video is still a distributed system, and the team said case-by-case (Concept 19, 97).

**"How does this change with Aspire?"** It makes the multi-process local dev loop cheap, which removes one real argument against services and makes the hybrid — modular monolith plus two or three services in one AppHost — the easy default. It doesn't change the latency, availability or consistency arithmetic, which is where the real cost lives (Concept 50).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Answers "it depends" and stops | Reframes into two axes, asks for team count, gives a default with a named trigger |
| Treats modularity and distribution as one decision | Separates them explicitly and notes that most claimed benefits belong to modularity |
| Lists benefits without costs | Quotes the latency, the availability arithmetic and the permanent operational tax |
| "Microservices scale better" | Asks which component is the bottleneck and what the instance count is; notes the DB usually is |
| "Microservices are more resilient" | Notes that synchronous series dependencies multiply: 5 × 99.9% = 99.5% |
| Decomposes by entity (User, Order, Product) | Decomposes by business capability; names the entity-service trap |
| Ignores data | Treats data ownership as the boundary that actually holds, and the migration's real cost |
| "We'll extract later" with no mechanism | Names the nine-item extraction-readiness checklist and the outbox as pre-payment |
| Proposes extraction before modularization | Insists on logical → data → process, and says why reversing it concentrates all the risk |
| Talks about code structure only | Asks about teams, ownership, on-call and deploy contention (Conway's law) |
| Ignores the platform floor | Names tracing, pipelines, config, secrets, discovery and on-call as preconditions |
| Big-bang migration plan | Strangler fig, branch by abstraction, a config-flag switch, rollback in under an hour |
| Cites trends and viral statistics | Cites primary sources, flags the unverifiable ones, and prefers the team's own numbers |
| Misreads Prime Video as "microservices are over" | Gives the accurate version: one component, serverless, Step Functions limits, data gravity |
| No way to measure success | Commits to DORA metrics plus deploy contention, p99 and run rate, baselined beforehand |
| Accepts the interviewer's premise uncritically | Asks what outcome is wanted, agrees with what's right, names preconditions, offers a lower-risk path |
| Treats consolidation as failure | Treats merging two always-co-deployed services as a legitimate senior correction |
| Says boundaries are enforced by review | Enforces with project references, `internal`, analyzers, arch tests and DB permissions |

---

## Practice exercises

**Exercise 1 — The two-axis audit (45 min).** Take a system you know well. Place it in the four-quadrant grid (Concept 2) and justify the placement with three pieces of evidence, not impressions: how many things deploy together, whether any two components share tables, and what fraction of recent commits touched more than one component. Then write the two-sentence description of the move you'd make first.

**Exercise 2 — Co-change analysis (1.5 hrs).** Run `git log --name-only` over twelve months of a real repository (yours or a large open-source .NET project) and compute co-change frequency between top-level folders. Rank the pairs. Propose a module decomposition. Then compute what fraction of commits would have crossed your proposed boundaries — if it's over 30%, redo the decomposition and say what you learned.

**Exercise 3 — Build a modular monolith skeleton (3 hrs).** Three modules, each with Domain/Application/Infrastructure/PublicApi, a host, a `SharedKernel`. Everything `internal` except the contracts. One `DbContext` per module with `HasDefaultSchema` and a per-schema migrations history table. Then deliberately violate a boundary — reference another module's Application project — and confirm you can still build. Notice that nothing stopped you. That's Exercise 4's motivation.

**Exercise 4 — Enforcement (1.5 hrs).** Add `ArchUnitNET` and write the six rules from Concept 48. Confirm each one *fails* when you introduce the violation from Exercise 3. Then make one rule match zero types (rename a namespace) and confirm it still passes — that's the silent-failure mode Module 20 warned about. Add an assertion that each rule matched at least one type. Finally, add `BannedApiAnalyzers` with a per-module `BannedSymbols.txt` and promote it to an error.

**Exercise 5 — Database-enforced boundaries (1 hr).** Create a SQL Server (or PostgreSQL) database with three schemas. Create a login per schema with `GRANT` on its own and `DENY` on the others. Wire each module's `DbContext` to its own connection string. Write a test that attempts a cross-schema read and assert that it throws. This is the single highest-value hour in this list.

**Exercise 6 — The outbox (2 hrs).** Add an outbox table to one module. Write the event inside the business transaction. Write a `BackgroundService` dispatcher that polls and dispatches to an in-process handler in another module. Make the consumer idempotent with a processed-message table. Then kill the process between the commit and the dispatch and confirm the event is still delivered on restart. Finally, swap the in-process dispatcher for Azure Service Bus (or the emulator) and confirm the only change is in the host's registration.

**Exercise 7 — Branch by abstraction (2 hrs).** Take one module's public contract. Write three implementations: in-process, remote (HTTP to a second host running only that module), and comparing (calls both, serves the first, logs divergence). Switch between them with configuration. Introduce a deliberate behavioural difference in the remote implementation and confirm the comparing implementation detects it.

**Exercise 8 — The latency experiment (1.5 hrs).** Write a benchmark with BenchmarkDotNet (Module 17) measuring: (a) a direct method call, (b) a DI-resolved interface call, (c) an in-process mediator dispatch with two behaviours, (d) a loopback HTTP call with JSON, (e) a loopback gRPC call. Then simulate the N+1 problem: 1,000 iterations of each. Write down the numbers. Carry them into your next interview — you will use them.

**Exercise 9 — Availability arithmetic (30 min).** Build a small spreadsheet or script: given N services in series and a per-service availability, compute composite availability and annual downtime. Then compute the tail-latency amplification table for fan-out N at a given per-service p99. Produce the two tables from Concepts 20 and 21 yourself, so you can regenerate them under pressure.

**Exercise 10 — Contract tests (2 hrs).** Using `PactNet`, write a consumer test for one module's contract, generate the pact file, and verify it against the provider. Then make a breaking change to the provider and confirm the verification fails. Then make an additive change and confirm it passes. This is the discipline that makes independent deployment real.

**Exercise 11 — Write the ADR (1 hr).** Take a real system (yours, or one from a job description you're targeting) and write the Concept 95 ADR for it. Include real numbers in the context, four options, the negative consequences honestly stated, and explicit revisit triggers. Then have someone try to argue you out of it and see whether the ADR holds.

**Exercise 12 — The 60-second answer, recorded (45 min).** Record yourself answering "monolith or microservices for a system like ours?" in under 90 seconds, hitting all six moves from Concept 91. Play it back. Count how many times you said "it depends," "obviously," or a trend claim you can't source. Re-record until the answer contains a default, a trigger, a first candidate and a cost.

**Exercise 13 — The critique (1 hr).** Find a public microservices architecture diagram (search any vendor's reference architecture). Apply Concept 63's symptom table and Worked Example 6's question list. Write down the two questions you'd ask first and why those two.

**Exercise 14 — Extraction dry run (3 hrs).** Take your Exercise 3 skeleton, pick a module, and run the full Concept 52 checklist against it. For each item that fails, do the work. Then actually extract it: new host project, own database, Service Bus instead of the in-process dispatcher, YARP in front routing by path prefix. Time the whole thing. That number is your credibility in a brownfield round.

---

## Free resources

Roughly ninety free resources, grouped. Current-state caveats are noted where a repository or sample has been archived or superseded, so you don't cite something in an interview that moved.

### Primary sources — the canonical arguments, and most of them are short

| Resource | What it covers | Why read it |
|---|---|---|
| [Microservices](https://martinfowler.com/articles/microservices.html) — James Lewis & Martin Fowler, 2014 | The definition that named the style: nine characteristics, including decentralized data management and "smart endpoints, dumb pipes" | The origin document. Most arguments online are about things it explicitly doesn't claim |
| [MicroservicePremium](https://martinfowler.com/bliki/MicroservicePremium.html) — Fowler, 2015 | The productivity curve and the crossover point | **Concept 11.** Three paragraphs, and the single most useful idea in this module |
| [MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html) — Fowler, 2015 | Why successful microservice systems almost always started as monoliths | **Concept 12**, one half of the argument |
| [Don't start with a monolith](https://martinfowler.com/articles/dont-start-monolith.html) — Stefan Tilkov, 2015 | The rebuttal: monoliths resist decomposition, so "split later" rarely happens | **Concept 12**, the other half. Read both, then hold the synthesis |
| [Microservice Trade-Offs](https://martinfowler.com/articles/microservice-trade-offs.html) — Fowler, 2015 | Benefits and costs side by side, honestly | The most balanced short piece on the subject |
| [MicroservicePrerequisites](https://martinfowler.com/bliki/MicroservicePrerequisites.html) — Fowler | Rapid provisioning, basic monitoring, rapid deployment, DevOps culture | **Concept 28**, from the source |
| [Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html) — Fowler | When layering helps, and why modules should come first at scale | The bridge from Module 20 to this one |
| [Conway's Law](https://martinfowler.com/bliki/ConwaysLaw.html) — Fowler | The law, the inverse manoeuvre, and why it's not optional | **Concept 9** |
| [On the Criteria To Be Used in Decomposing Systems into Modules](https://www.win.tue.nl/~wstomv/edu/2ip30/references/criteria_for_modularization.pdf) — David Parnas, 1972 (PDF) | Information hiding; decompose by likely change, not by processing step | Four pages. The grandparent of every idea in this module |
| [Sacrificial Architecture](https://martinfowler.com/bliki/SacrificialArchitecture.html) — Fowler | Designing something you expect to throw away | Useful counterweight to over-investing in boundaries too early |
| [The Twelve-Factor App](https://12factor.net/) | Config, backing services, disposability, logs as streams | The operational baseline distributed services assume |
| [Self-Contained Systems](https://scs-architecture.org/) | The coarser-grained alternative: own UI, logic and data, async integration | The middle position between monolith and microservices, worth knowing by name |

### The modular monolith specifically

| Resource | What it covers |
|---|---|
| [Modular Monolith: A Primer](https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer) — Kamil Grzybek | The definition, precisely stated, and the properties a module must have. Start here |
| [Modular Monolith: Architectural Drivers](https://www.kamilgrzybek.com/blog/posts/modular-monolith-architectural-drivers) — Grzybek | When this architecture is the right one, driven by quality attributes rather than fashion |
| [Modular Monolith: Architecture Enforcement](https://www.kamilgrzybek.com/blog/posts/modular-monolith-architecture-enforcement) — Grzybek | **Concept 48** — how to make the boundaries real rather than aspirational |
| [Modular Monolith: Integration Styles](https://www.kamilgrzybek.com/blog/posts/modular-monolith-integration-styles) — Grzybek | **Concepts 38–40** — how modules talk without coupling |
| [Modular Monolith: Domain-Centric Design](https://www.kamilgrzybek.com/blog/posts/modular-monolith-domain-centric-design) — Grzybek | Layering inside each module; the bridge to Module 20 and Module 22 |
| [kgrzybek/modular-monolith-with-ddd](https://github.com/kgrzybek/modular-monolith-with-ddd) | **The** .NET reference implementation. MIT, 13k+ stars, production-shaped: modules, schema isolation, CQRS, domain events, an architecture decision log, integration tests. Read the README end to end even if you never run the code |
| [Deconstructing the Monolith](https://www.shopify.com/partners/blog/monolith-software) — Kirsten Westeinde, Shopify | Why Shopify evaluated microservices and chose a modular monolith; the "design payoff line" |
| [Under Deconstruction: The State of Shopify's Monolith](https://shopify.engineering/shopify-monolith) — Shopify Engineering | The follow-up, years later: what actually happened, including what didn't work |
| [How Shopify Migrated to a Modular Monolith](https://www.infoq.com/news/2019/07/shopify-modular-monolith) — InfoQ | A concise summary of the Shopify talk if you want the argument in five minutes |
| [Some benefits of simple software architectures](https://www.wave.com/en/blog/simple-architecture) — Wave | A fintech at real scale explaining why they run a monolith on purpose. Unusually concrete about the trade-offs |
| [awesome-monolith](https://github.com/canerbasaran/awesome-monolith) | A curated link list for the monolith side of the argument — useful for finding primary sources fast |

### .NET reference implementations and templates

| Resource | What it covers and its current state |
|---|---|
| [dotnet/eShop](https://github.com/dotnet/eShop) | Microsoft's **current** first-party reference microservices app: Aspire-hosted, service per bounded context, integration events, gRPC. This is the one to read in 2026 |
| [dotnet-architecture/eShopOnContainers](https://github.com/dotnet-architecture/eShopOnContainers) | **Archived 17 November 2023**, superseded by `dotnet/eShop`. Still the sample the Microsoft e-book refers to, so know the relationship |
| [NimblePros/eShopOnWeb](https://github.com/NimblePros/eShopOnWeb) | The monolith-with-Clean-Architecture companion. Microsoft archived its own copy on 13 January 2025; this is the community-maintained continuation |
| [ardalis/CleanArchitecture](https://github.com/ardalis/CleanArchitecture) | On .NET 10. Ships two templates: `clean-arch` (full) and `min-clean` (single-project vertical slice). The `min-clean` template is a good starting point for a small modular system |
| [jasontaylordev/CleanArchitecture](https://github.com/jasontaylordev/CleanArchitecture) | On .NET 10 (template 10.8.0, March 2026), with a dedicated docs site, Aspire-based functional tests, and built-in OpenAPI + Scalar. Note the MediatR licence warning in its build output — the maintainer has publicly defended that trade-off |
| [mehdihadeli/food-delivery-modular-monolith](https://github.com/mehdihadeli/food-delivery-modular-monolith) | A practical modular monolith with vertical slices, CQRS, DDD and event-driven integration. Its sibling repo ports the same domain to microservices, which makes the pair unusually instructive |
| [dotnet/yarp](https://github.com/dotnet/yarp) | Microsoft's reverse proxy toolkit — the .NET way to build a gateway or BFF you control (**Concept 81**) |

### Microsoft Learn — architecture guidance (free e-books)

| Resource | What it covers |
|---|---|
| [.NET Microservices: Architecture for Containerized .NET Applications](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/) | The full free e-book: decomposition, data sovereignty, event-driven integration, resilience, API gateways. Still excellent on the patterns; note it references the archived eShopOnContainers |
| [Architect Modern Web Applications with ASP.NET Core and Azure](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/) | The monolith-first companion guide, and the source of the careful definition of "monolithic" used in **Concept 3** |
| [.NET application architecture guides](https://dotnet.microsoft.com/en-us/learn/dotnet/architecture-guides) | The index of all the free Microsoft architecture e-books |
| [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/) | The whole reference: architecture styles, patterns, best practices |
| [Microservices architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/microservices) | Benefits, challenges and best practices, stated more soberly than most vendor material |
| [Design a microservices architecture](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/) | The full design series: boundaries, data, communication, gateway, CI/CD |
| [Using domain analysis to model microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis) | DDD applied to service boundaries, worked through a drone-delivery example |
| [Identifying microservice boundaries](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/microservice-boundaries) | **Part D** in Microsoft's own words, with concrete heuristics |
| [Interservice communication](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication) | **Concept 79** — sync vs. async, with the failure implications spelled out |
| [Rebuild monolithic applications using microservices](https://learn.microsoft.com/en-us/azure/app-modernization-guidance/expand/rebuild-monolithic-applications-using-microservices) | Microsoft's own migration sequencing guidance |

### Azure Architecture Center — the patterns you'll be asked to name

| Pattern | Where it applies in this module |
|---|---|
| [Strangler Fig](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig) | **Concept 68** — incremental replacement |
| [Anti-Corruption Layer](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) | **Concept 53** — translating between contexts |
| [Backends for Frontends](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends) | **Concept 81** |
| [Gateway Routing](https://learn.microsoft.com/en-us/azure/architecture/patterns/gateway-routing) · [Gateway Aggregation](https://learn.microsoft.com/en-us/azure/architecture/patterns/gateway-aggregation) · [Gateway Offloading](https://learn.microsoft.com/en-us/azure/architecture/patterns/gateway-offloading) | The three gateway responsibilities, separated |
| [Saga](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga) | **Concept 87** — distributed transactions you can actually implement |
| [CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) | **Concept 86** — the read-model answer |
| [Sidecar](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar) | **Concepts 82–83** — the mesh and Dapr deployment shape |
| [Ambassador](https://learn.microsoft.com/en-us/azure/architecture/patterns/ambassador) | Client-side resilience as infrastructure |
| [Bulkhead](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) · [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) · [Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) | **Concept 85**, and Module 13 in Azure's vocabulary |
| [Publisher/Subscriber](https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber) · [Competing Consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers) | Module 11's patterns, in the Azure catalogue |

### Chris Richardson's pattern language (microservices.io) — free and exhaustive

| Resource | What it covers |
|---|---|
| [Microservice Architecture pattern language](https://microservices.io/patterns/index.html) | The whole map: decomposition, data, communication, reliability, testing, deployment, observability |
| [Decompose by business capability](https://microservices.io/patterns/decomposition/decompose-by-business-capability.html) · [by subdomain](https://microservices.io/patterns/decomposition/decompose-by-subdomain.html) | **Concept 54**, both flavours |
| [Database per Service](https://microservices.io/patterns/data/database-per-service.html) · [Shared Database](https://microservices.io/patterns/data/shared-database.html) | **Concepts 59, 86** — including an honest treatment of when the shared database is a pragmatic choice |
| [Saga](https://microservices.io/patterns/data/saga.html) · [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) · [API Composition](https://microservices.io/patterns/data/api-composition.html) | **Concepts 40, 87, 24** |
| [Strangler Application](https://microservices.io/patterns/refactoring/strangler-application.html) · [Anti-Corruption Layer](https://microservices.io/patterns/refactoring/anti-corruption-layer.html) | **Part E** |
| [Service Component Test](https://microservices.io/patterns/testing/service-component-test.html) · [Consumer-Driven Contract Test](https://microservices.io/patterns/testing/consumer-driven-contract-test.html) | **Concepts 26, 72, 90** |

### Boundaries, DDD and decomposition

| Resource | What it covers |
|---|---|
| [BoundedContext](https://martinfowler.com/bliki/BoundedContext.html) — Fowler | The shortest correct definition, plus why it's about language |
| [DDD Crew — Bounded Context Canvas](https://github.com/ddd-crew/bounded-context-canvas) | A one-page template for designing a single context: purpose, messages, ubiquitous language, business decisions. Excellent for structuring a design conversation |
| [DDD Crew — Context Mapping](https://github.com/ddd-crew/context-mapping) | The relationship patterns between contexts: shared kernel, customer/supplier, conformist, ACL, published language |
| [DDD Crew — Core Domain Charts](https://github.com/ddd-crew/core-domain-charts) | Deciding which contexts deserve investment vs. buying or outsourcing — the strategic half, and a nice bridge to Module 33 |
| [EventStorming](https://www.eventstorming.com/) — Alberto Brandolini | The official site: the method, the notation, the big-picture and design-level formats |
| [Domain Storytelling](https://domainstorytelling.org/) — Hofer & Schwentner | The complete method, free, with a free modeling tool. The best technique for working with non-technical stakeholders |
| [Code Maat](https://github.com/adamtornhill/code-maat) — Adam Tornhill | Free CLI for mining version-control history: temporal coupling, hotspots, knowledge maps. **Concept 57** |
| [Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/) — Unmesh Joshi | A free pattern catalogue of the mechanisms underneath all of this; excellent companion to Modules 8 and 9 |

### Case studies — the ones that are actual evidence

| Resource | What it covers and how to read it |
|---|---|
| [Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%](https://www.primevideotech.com/video-streaming/scaling-up-the-prime-video-audio-video-monitoring-service-and-reducing-costs-by-90) — Marcin Kolny, 2023 | The original post. Read it yourself rather than a summary: one component, Step Functions limits, S3 frame transfer, case-by-case conclusion (**Concepts 19, 97**) |
| [Reduce costs by 90% by moving from microservices to monolith](https://www.devclass.com/ci-cd/2023/05/05/reduce-costs-by-90-by-moving-from-microservices-to-monolith-amazon-internal-case-study-raises-eyebrows/1621790) — DevClass | A careful contemporaneous write-up including the Step Functions account-limit detail and the community reaction |
| [Goodbye Microservices: From 100s of problem children to 1 superstar](https://segment.com/blog/goodbye-microservices/) — Alexandra Noonan, Segment, 2018 | The consolidation story with the clearest lesson: don't create one service per integration (**Concept 62**) |
| [Introducing Domain-Oriented Microservice Architecture](https://www.uber.com/en-US/blog/microservice-architecture/) — Uber, 2020 | The correction for over-decomposition: domains, gateways and layers over thousands of services |
| [One Team At Uber Is Moving From Microservices To Macroservices](http://highscalability.com/blog/2020/4/8/one-team-at-uber-is-moving-from-microservices-to-macroservic.html) — High Scalability | The shorter version of the same story, with commentary |
| [Deconstructing the Monolith](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) — Shopify Engineering | The engineering-blog version of the Shopify decision |

### Migration, strangler fig and safe change

| Resource | What it covers |
|---|---|
| [StranglerFigApplication](https://martinfowler.com/bliki/StranglerFigApplication.html) — Fowler | The original naming and the metaphor |
| [BranchByAbstraction](https://martinfowler.com/bliki/BranchByAbstraction.html) — Fowler | **Concept 69** — the technique that makes the switch a flag |
| [ParallelChange (expand/contract)](https://martinfowler.com/bliki/ParallelChange.html) — Danilo Sato | **Concept 70** — the three phases, precisely |
| [FeatureToggle](https://martinfowler.com/articles/feature-toggles.html) — Pete Hodgson | Release toggles vs. ops toggles vs. experiment toggles, and how to avoid toggle debt. **Concept 51's** mitigation |
| [kgrzybek/modular-monolith-with-ddd — Architecture Decision Log](https://github.com/kgrzybek/modular-monolith-with-ddd) | A real, worked ADR log for exactly this kind of system — rare and worth studying (**Concept 95**) |
| [DORA / Accelerate State of DevOps](https://dora.dev/) | The four key metrics and the research behind them. **Concept 74's** measurement vocabulary |

### Enforcement and architecture testing

| Resource | What it covers |
|---|---|
| [ArchUnitNET](https://github.com/TNG/ArchUnitNET) | The .NET port of ArchUnit; actively developed. Fluent rules for layer and module dependencies (**Concept 48**) |
| `NetArchTest.eNhancedEdition` (NuGet, MIT) | The maintained fork of NetArchTest; the original `NetArchTest.Rules` 1.3.2 (May 2021) still works but is frozen |
| [Microsoft.CodeAnalysis.BannedApiAnalyzers](https://github.com/dotnet/roslyn-analyzers) | Per-project `BannedSymbols.txt` promoted to build errors — compile-time enforcement, the fastest feedback loop available |
| [Central Package Management](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management) | One `Directory.Packages.props` for the whole solution; prevents module-level version drift |
| [InternalsVisibleTo via MSBuild](https://learn.microsoft.com/en-us/dotnet/standard/assembly/internals-visible-to) | The mechanism that lets `internal` be your default without breaking tests (**Concept 33**) |

### Testing across boundaries

| Resource | What it covers |
|---|---|
| [Pact documentation](https://docs.pact.io/) | Consumer-driven contract testing, end to end, with the .NET implementation documented |
| [Testcontainers for .NET](https://github.com/testcontainers/testcontainers-dotnet) | Real databases, brokers and Azurite in integration tests — the foundation of tier-2 testing |
| [Testing Strategies in a Microservice Architecture](https://martinfowler.com/articles/microservice-testing/) — Toby Clemson | The definitive free treatment of the testing pyramid for services; diagrams worth stealing |
| [Aspire testing](https://aspire.dev/) | `Aspire.Hosting.Testing` — start the whole application graph for integration tests. See the docs site's testing section |
| [TestingPyramid / IntegrationTest](https://martinfowler.com/bliki/IntegrationTest.html) — Fowler | Why "integration test" means two different things and why that causes arguments |

### Platform: mesh, Dapr, Aspire, observability

| Resource | What it covers |
|---|---|
| [Istio ambient mode reaches GA](https://istio.io/latest/blog/2024/ambient-reaches-ga/) — 7 November 2024 | Sidecar-less mesh: `ztunnel` for L4, waypoints for L7, and the incremental-adoption argument (**Concept 82**) |
| [Istio Roadmap 2025–2026](https://istio.io/latest/blog/2025/roadmap/) | Where the project is going, and an honest read on ambient adoption |
| [Dapr](https://dapr.io/) · [Dapr docs](https://docs.dapr.io/) | The building blocks, the .NET SDK, and Dapr Workflow (**Concept 83**) |
| [Dapr v1.16 release notes](https://blog.dapr.io/posts/2025/09/16/dapr-v1.16-is-now-available/) | Multi-app workflows; .NET SDK Roslyn analyzers; framework consolidation on .NET 8/9 |
| [CNCF: Dapr graduation](https://www.cncf.io/announcements/2024/11/12/cloud-native-computing-foundation-announces-dapr-graduation/) · [Dapr project page](https://www.cncf.io/projects/dapr/) | The graduation announcement, and the LFX project-health metrics that tell a more nuanced story |
| [Aspire — What's new in Aspire 13](https://aspire.dev/whats-new/aspire-13/) | The rebrand, polyglot support, template changes, and the upgrade path from 9.x |
| [What's New in Aspire 13.3](https://devblogs.microsoft.com/aspire/whats-new-aspire-13-3/) | Aspireify, command results, Kubernetes and AKS deployment |
| [Aspire roadmap discussions](https://github.com/microsoft/aspire/discussions/15662) | The Q1 2026 update: what shipped in 13.2, what's next, and where the team says the gaps are |
| [OpenTelemetry](https://opentelemetry.io/) · [opentelemetry-dotnet](https://github.com/open-telemetry/opentelemetry-dotnet) | The instrumentation standard and the .NET implementation (**Concept 84**) |
| [Amazon Builders' Library](https://aws.amazon.com/builders-library/) | Free, deep, vendor-agnostic essays on timeouts, retries with jitter, load shedding and health checks. The jitter article alone is worth the visit |
| [Linkerd](https://linkerd.io/) | The simpler mesh alternative; note Buoyant's 2024 change to how stable releases are distributed |

### Libraries in this space — check the licence before you adopt

| Library | Licence status as of September 2026 |
|---|---|
| [Wolverine](https://wolverinefx.net/) ([repo](https://github.com/JasperFx/wolverine)) | **MIT.** Mediator + messaging + outbox + sagas. The strongest free option for in-process-now, broker-later (**Concept 49**) |
| [MassTransit](https://masstransit.io/) | **v9 is commercial** (source-available; free for local dev; ≈\$400/mo SMB, ≈\$1,200/mo enterprise). v8 stays Apache-2.0 with official maintenance ending around end of 2026 |
| MediatR | **v13+ commercial** (Lucky Penny Software; RPL-1.5 or paid; free Community tier below revenue/capital thresholds). `MediatR.Contracts` remains Apache-2.0. Module 23 takes this properly |
| AutoMapper | **v15+ commercial**, same arrangement. Consider Mapperly (source-generated) or manual mapping |
| [Rebus](https://github.com/rebus-org/Rebus) · [Brighter](https://github.com/BrighterCommand/Brighter) | **MIT.** Mature open-source service buses with smaller ecosystems |
| FluentValidation | **Apache-2.0** (12.x) — unchanged |
| [Polly](https://github.com/App-vNext/Polly) | **BSD-3.** v8 resilience pipelines, surfaced through `Microsoft.Extensions.Http.Resilience`. Module 25 |
| [YARP](https://github.com/dotnet/yarp) | **MIT**, Microsoft-maintained, used by Azure itself |

### Surveys and industry data — with provenance

| Resource | What it says, and what it doesn't |
|---|---|
| [CNCF Annual Cloud Native Survey (2025 results)](https://www.cncf.io/announcements/2026/01/20/kubernetes-established-as-the-de-facto-operating-system-for-ai-as-production-use-hits-82-in-2025-cncf-annual-cloud-native-survey/) | 628 practitioners, fielded September 2025, published January 2026. **82% of container users run Kubernetes in production**; 98% cloud native adoption; top barrier is now cultural (47%) |
| [The CNCF Annual Cloud Native Survey report](https://www.cncf.io/reports/the-cncf-annual-cloud-native-survey/) | The report landing page — go here for the actual PDF rather than relying on secondary coverage |
| [CNCF 2024 Annual Survey announcement](https://www.cncf.io/announcements/2025/04/01/cncf-research-reveals-how-cloud-native-technology-is-reshaping-global-business-and-innovation/) | **Service mesh adoption fell from 50% (2023) to 42% (2024)** on operational-overhead grounds. This is the real, citable 42% |
| [CNCF 2024 Annual Survey (PDF)](https://www.cncf.io/wp-content/uploads/2025/04/cncf_annual_survey24_031225a.pdf) | The full document with sample sizes and question wording — exactly what you want when someone quotes a number at you |
| [CNCF/SlashData: 15.6M cloud native developers](https://cloudnativenow.com/features/cncf-total-number-of-cloud-native-developers-reaches-15-6m/) | 12,021 developers surveyed. **46% build microservices**; service mesh use fell from 18% (Q3 2023) to 8% (Q3 2025). Note the denominator is developers, not organizations |
| [Voice of Kubernetes Experts 2025](https://www.cncf.io/blog/2025/08/02/what-500-experts-revealed-about-kubernetes-adoption-and-workloads/) | 500+ enterprise practitioners on workload placement — useful for the compute-choice conversation |

**A provenance warning worth internalizing (Concept 96):** the widely repeated claim that *"a 2025 CNCF survey found 42% of organizations are consolidating microservices back into larger deployable units"* does not appear in any CNCF publication linked above. Nor do the companion figures ("3.75×–6× cost," "Gartner: 60% regret it") carry primary citations in the posts that repeat them. Use the numbers above, which you can point at.

### Talks worth an hour (search by title — all free on YouTube)

| Talk | Why |
|---|---|
| **"When To Use Microservices (And When Not To!)"** — Sam Newman & Martin Fowler, GOTO | The two people most associated with this topic arguing it out. The best single hour available |
| **"Modular Monoliths"** — Simon Brown, GOTO 2018 | The talk that popularized the term in the wider community; also the source of "if you can't build a well-structured monolith, what makes you think microservices are the answer?" |
| **"Mastering Chaos — A Netflix Guide to Microservices"** — Josh Evans | The pro-decomposition case from someone who actually built one at scale, including the parts that hurt |
| **"Deconstructing the Monolith"** — Kirsten Westeinde, Shopify Unite 2019 | The modular-monolith decision from a company at genuine scale |
| **"Boundaries"** — Gary Bernhardt | Functional core, imperative shell — the same boundary thinking from a different direction (also Module 20) |
| **"Domain-Driven Design Europe"** conference channel | Years of free talks on bounded contexts, EventStorming and context mapping |

### Books (not free, listed for completeness)

- **Sam Newman — *Building Microservices*, 2nd ed.** The standard reference. The 2nd edition is significantly more balanced than the 1st, and the coupling taxonomy in **Concept 58** is from it.
- **Sam Newman — *Monolith to Microservices*.** The migration playbook; **Part E** is a compressed version of it.
- **Skelton & Pais — *Team Topologies*.** The organizational half of this decision (**Concept 10**).
- **Vaughn Vernon — *Implementing Domain-Driven Design*** and **Eric Evans — *Domain-Driven Design***. Module 22's reading; bounded contexts are the source of boundaries.
- **Adam Tornhill — *Software Design X-Rays*.** The theory behind **Concept 57**'s co-change analysis.
- **Michael Nygard — *Release It!*, 2nd ed.** Stability patterns and failure modes; the practical companion to Module 13.
- **Gregor Hohpe — *The Software Architect Elevator*.** How to have the **Concept 94** conversation with executives.
- **Forsgren, Humble & Kim — *Accelerate*.** The research behind DORA metrics.
- **Kleppmann — *Designing Data-Intensive Applications*.** Still the standard reference behind Phase 3.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Monolith or microservices, in one sentence | Two separate decisions: modularity (nearly free, always do it) and distribution (expensive, buys deploy independence) |
| The reframe | Most claimed microservices benefits are modularity benefits; only deployment and runtime isolation require the network |
| The four quadrants | Big ball of mud / modular monolith / distributed monolith / microservices — the horizontal move is the valuable one |
| Definition of a monolith | One deployable unit. Says nothing about code quality |
| Definition of a microservice | Independently deployable, owns its data, owned by one team |
| The defining test | Can you deploy it alone, on a Tuesday, without coordinating? If no, it isn't one |
| "Micro" | Size is an output of boundary choice, never a target |
| Distributed monolith | Every cost, no benefit. Three questions: shared tables, lockstep deploys, hard sync dependencies |
| Module vs. service | A service is a hosting decision applied to a module that already exists |
| Five properties of a boundary | Owns data, explicit contract, hidden internals, independently testable, changes for its own reasons |
| Conway's law | Boundaries that don't match ownership erode. Predict the architecture from the org chart |
| Cognitive load | Microservices shift extraneous load onto teams; only a platform team makes that a good trade |
| The premium | Fixed cost paid immediately, benefit arrives above the crossover — which is about org size, not traffic |
| MonolithFirst vs. Tilkov | Both right; the synthesis is modular-first with extraction pre-paid |
| Three units | Unit of failure, unit of scale, unit of release — decide separately, they often disagree |
| Reversibility | Splitting is expensive; merging is worse because data diverged. Prefer the cheaper mistake |
| The five claims graded | Deployability real, autonomy real-if-owned, fault isolation conditional, scaling rarely, tech choice overrated |
| The decisive question | How many teams need to ship independently, and are they blocking each other? |
| Latency, in-process vs. network | ~1 ns vs. ~500 µs same-DC — six orders of magnitude |
| The loop | 1,000 in-process calls ≈ 1 µs; 1,000 RPCs at 1 ms ≈ 1 second |
| Serialization | Payload × hops; Prime Video was data gravity, not a verdict on microservices |
| Prime Video, correctly | One component, serverless, Step Functions state-transition limits + S3 frame transfer, 5% of load ceiling, case-by-case |
| Tail latency | Fan-out to 10 makes your p99 their p99.9; 1 − 0.99^N |
| Availability | Series multiplies: 5 × 99.9% = 99.5% → 9 h becomes 44 h/year |
| Getting resilience anyway | Async, graceful degradation, cached fallbacks — classify every dependency |
| Partial failure | You can't distinguish lost request from lost response → idempotency, timeouts, retries, breakers, bulkheads |
| Losing the transaction | ACID stops at the boundary → sagas, outbox, compensations that are business decisions |
| Losing the join | API composition, event-fed read models, or the warehouse. Data gravity decides what to extract first |
| Debugging tax | Stack trace → distributed trace; OTel is a precondition, not a follow-up |
| Testing tax | Contract tests in, shared end-to-end gates out; no test needing another team's service may gate your deploy |
| Versioning tax | Expand/contract forever; additive cheap, removals expensive; you can no longer refactor across boundaries |
| Operational floor | Pipelines, registry, logs, tracing, metrics, alerting+ownership, config, secrets, discovery, contract testing, local dev, on-call |
| Cost model | Infra 1.5×–4×; people is the dominant term — 1–3 engineers of permanent platform capacity |
| Modular monolith, defined | One deployable, enforced internal boundaries, each module owns its data, contracts only |
| Solution shape | `Modules/<Name>/{Domain,Application,Infrastructure,PublicApi}` + `Host` + tiny `SharedKernel` |
| The public surface | `internal sealed` by default; contract in its own assembly; DTOs never entities; `InternalsVisibleTo` for tests |
| Module registration | `AddXModule()` + `MapXEndpoints()`; `Program.cs` is a readable index of the system |
| Schema ownership | Own schema, own DB credential, `DENY` on the others — the only boundary a deadline can't override |
| One DB, many schemas | The default; split to separate databases for storage fit, scaling profile, compliance, or imminent extraction |
| The three prohibitions | No cross-module joins, no cross-module FKs, no cross-module entity references |
| Cross-module reads | Contract call (batched, async, `CancellationToken`), event-fed replica, or a projection |
| Cross-module writes | Publish facts, don't send commands. A command crossing a boundary means the boundary is wrong |
| The outbox | Write the event in the same transaction, today — extraction becomes a registration change |
| Duplication | Three `Customer` shapes is correct; share the ID, not the model |
| SharedKernel | IDs, Money, Result, base types. Nothing with a business rule. Budget it and enforce the budget |
| Migrations | Per-module `DbContext`, schema, history table and assembly; applied by the pipeline, not at startup |
| Endpoints | Per-module route groups; cross-cutting concerns in the host pipeline |
| Background work | Belongs to one module; a scope per unit of work, never per worker lifetime |
| Testing a module | Module in-process + real DB via Testcontainers + fakes for other modules' contracts |
| Enforcement ladder | Project references → `internal` → banned APIs → arch tests → **DB permissions** → review/ADR |
| Arch tests, first six | Module isolation; contact only via PublicApi/Events; Domain depends on nothing; nothing → Host; only Host → Infrastructure; no cycles |
| How arch tests lie | A renamed namespace makes the rule match zero types and pass silently — assert the match count |
| In-process messaging | Wolverine (MIT) or a 40-line dispatcher; MediatR 13+ and MassTransit v9 are commercial |
| Aspire | Makes the hybrid cheap and local dev sane; does not change Part B's arithmetic |
| The real risk | Deployment coupling and discipline decay — not performance |
| Extraction readiness | Nine items; whichever fails is your actual migration work |
| Boundaries come from | Bounded contexts — find the words that mean different things to different people |
| Decompose by | Business capability or subdomain; never entity, never layer |
| Entity-service trap | UserService/OrderService = a schema in a costume; no use case fits inside one |
| EventStorming | A day with domain experts beats a month reading the codebase |
| Co-change analysis | Files that change together belong together; >30% cross-boundary commits means wrong boundaries |
| Coupling taxonomy | Domain < pass-through < common < content; temporal as its own axis |
| Data coupling | The shared table is the coupling that survives every refactor |
| Instability | I = Ce/(Ca+Ce); depend toward stability; **no cycles between modules, ever** |
| Two-pizza rule | It's about communication overhead, not service size |
| Nanoservices | More services than engineers; consolidation is a senior move, not an admission |
| Wrong-boundary symptoms | Lockstep deploys, 3+ deep sync chains, shared tables, features touching 3+ units, a "core" library |
| Ownership | An unowned boundary is a suggestion; `CODEOWNERS` per module, one team each |
| Extraction triggers | Deploy contention, scale asymmetry, failure isolation, compliance, technology necessity, team count, lifecycle, M&A |
| Anti-triggers | "The monolith is a mess," best practice, retention, Netflix, "it'll scale," a vendor, sunk cost |
| Order of operations | Logical → data → process. Never reversed |
| Strangler fig | Facade → new service with no traffic → one operation at a time → writes last → hold a business cycle → delete |
| Branch by abstraction | In-process / remote / comparing implementations selected by configuration; rollback in seconds |
| Expand/contract | Add new, migrate, verify with telemetry, remove old. Four deploys to rename a column |
| Data separation | Inventory → eliminate outside access → own schema → own DB → dual write → backfill → read switch → write switch → delete |
| Where the effort is | 70% is finding and removing cross-boundary reads, especially in reports and ETL |
| Contract testing | Consumer-driven (Pact): provider's build fails before deployment, no shared environment needed |
| Rollback rule | If you can't undo it within an hour, the step is too big |
| Measuring it | DORA four + deploy contention + p99 + composed availability + run rate — baselined **before** |
| Consolidating back | Legitimate and harder than splitting; merge code first, reconcile data second, keep them as modules |
| Brownfield order | Characterization tests → telemetry → co-change + EventStorming → one module → its schema → enforce → repeat |
| Platform floor on Azure | ACA/AKS, App Configuration, Key Vault, Service Bus, OTel → Azure Monitor, Entra managed identity, Aspire locally |
| Compute default | Container Apps unless you specifically need AKS |
| Sync vs. async | Queries sync; state changes always events; ask what the user actually needs to see now |
| gRPC vs. JSON | JSON at the edge; gRPC internally if volume or payload justifies it — measure, don't prefer |
| Gateway | Routes, authenticates, aggregates. **Never business logic** |
| Service mesh | Real value above ~20 services in several languages; adoption fell 50%→42% (orgs) and 18%→8% (devs); ambient GA'd Nov 2024 |
| Dapr | Building blocks over a sidecar; CNCF graduated Oct 2024; strongest polyglot or for its durable workflow |
| Tracing | OTel end to end; `Activity` *is* the span in .NET; **verify context survives the broker hop** |
| Resilience | `AddStandardResilienceHandler()`; timeout budgets not round numbers; retry only idempotent; jitter mandatory |
| Reporting across services | Analytical store fed by events/CDC; CQRS views for a few fresh screens; never BI against production services |
| Sagas | Orchestration past ~3 steps; durable state via Dapr Workflow or Durable Functions; compensations are business decisions |
| Shared libraries | Technical concerns only; contract packages versioned and additive; prefer duplication |
| Repo strategy | Tooling, not architecture — but a monorepo without build-level boundaries erodes them |
| Testing strategy | Unit → service (Testcontainers + fakes) → contract (Pact) → thin smoke → production verification |
| The 60-second answer | Reframe → ask team count → default → trigger → first candidate → cost accepted |
| Scorecard top factors | Teams needing independent release; deploy contention; operational maturity; domain clarity |
| Defaults by size | 1–3: one project. 4–15: modular monolith. 16–50: + 1–3 services. 50+: services per team + platform team |
| Cost conversation | Engineer-months + permanent platform capacity + run-rate delta vs. contention cost, with a break-even and a revisit date |
| The ADR | Numbers in context, rejected options with reasons, honest negatives, explicit revisit triggers |
| The trend question | Kubernetes up to 82%; mesh genuinely down; the "42% consolidating" stat has no CNCF primary source |
| Case studies | Prime Video (data gravity), Segment (service per integration), Shopify (chose modular), Uber (DOMA correction), Netflix (paid the cost deliberately) |
| Disagreeing well | Take the goal seriously → agree with what's right → name the precondition → offer the lower-risk path → say what would change your mind |

---

## Progress

**Module 21 complete.** Phase 5 is now half built: Module 20 gave you the structure *inside* a deployable, and this module gave you the decision about *how many deployables there are* — which is the decision that most often gets made on vibes and most often gets scored on criteria.

This module closes the loops it was created to close:

- **Module 20's Concept 57** (modules before layers) is now a full architecture: Part C gives the projects, the visibility rules, the schema ownership, the registration contract, the outbox, and the nine-item extraction checklist that makes "modules first" a strategy rather than a preference.
- **Module 20's Concept 77** (Conway's law) became the decisive input rather than a footnote — Concepts 9, 16 and 64 make team structure the thing that actually settles the decision.
- **Module 20's Concept 76** (layers are not tiers) is the root of Concept 1's two axes: logical separation is compile-time, distribution is runtime, and conflating them is the error this whole module exists to prevent.
- **Module 11's outbox** is now the mechanism that pre-pays for extraction (Concept 40), and the domain-vs-integration-event distinction became the module contract rule.
- **Module 12's sagas and distributed transactions** got their trigger: Concept 23 explains exactly what you give up at the boundary and Concept 87 says what replaces it.
- **Module 13's resilience patterns** moved from advisable to mandatory (Concepts 22, 85), with the reason stated precisely: partial failure is a bug class that does not exist in a process.
- **Module 7's CAP/PACELC** is the theory behind Concept 23's "you just lost your transaction."
- **Module 6's Scale Cube** Y-axis is Concept 54's decomposition, and Concept 13 separates the scaling question from the release question that usually dominates it.
- **Module 5's latency numbers** became Concept 18's six orders of magnitude, which is the single most useful number in a design round on this topic.
- **Module 19's schema-ownership thread** is answered: it's not just an EF Core concern, it's the only module boundary that survives organizational pressure (Concept 35).

Threads left open on purpose:

- **DDD tactical and strategic patterns** — aggregates, value objects, domain events, and the strategic work of *finding* bounded contexts that Concepts 53–56 previewed — are **Module 22**, which turns boundary discovery from a technique into a discipline.
- **CQRS and MediatR** — whether the read/write split of Concepts 38 and 86 deserves a framework, and the full MediatR licensing analysis — are **Module 23**.
- **Event sourcing**, where the event-fed replicas of Concepts 38 and 41 become the source of truth rather than a projection, is **Module 24**.
- **Polly and resilience composition** — the implementation of Concept 85 — is **Module 25**.
- **Compute choices** (Concept 78) get their proper treatment in **Module 26**: App Service vs. Container Apps vs. AKS vs. Functions, with the cost and operational models.
- **The messaging and data platform** (Concepts 79, 86) — Service Bus, Event Grid, Event Hubs, Cosmos partitioning — is **Module 27**.
- **Observability** (Concepts 84, 74) — OpenTelemetry in depth, SLOs, error budgets — is **Module 28**.
- **Security architecture across service boundaries** — token propagation, service identity, mTLS, STRIDE on a distributed system — is **Module 29**.
- **ADRs and the C4 model**, the documentation half of Concept 95, are **Module 31**.
- **Brownfield migration and the strangler fig** get their full organizational treatment in **Module 32**; Parts E and Concept 76 are the engineering half.
- **Cost, build-vs-buy and technical debt** — the executive language of Concept 94 — is **Module 33**.

Next in the curriculum: **Module 22 — DDD tactical & strategic patterns**: bounded contexts, aggregates as consistency boundaries, value objects, domain events, and the strategic design work that decides where the boundaries in this module actually go — because Part D told you that finding the boundary is the hard part, and Module 22 is where you learn to do it.
