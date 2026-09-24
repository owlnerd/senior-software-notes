# Module 22 — DDD Tactical & Strategic Patterns
*Phase 5: .NET Architecture Patterns · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **Domain-Driven Design is not a catalogue of class shapes; it is a method for making three decisions — where consistency must be immediate, where the meaning of words changes, and where the business needs you to invest design effort — and every pattern in it is either one of those decisions or a mechanism for living with its consequences.**

An aggregate is the first decision: a **consistency boundary** chosen from the business's invariants. A bounded context is the second: a **language boundary** chosen from where meaning shifts. A core/supporting/generic subdomain classification is the third: an **investment decision** chosen from business strategy. Entities, value objects, repositories, domain events, anticorruption layers, context maps and the rest are the machinery that makes those three decisions implementable and keeps them true over time.

That reframing is the whole module. The naive version of this topic is a vocabulary quiz — "an entity has identity, a value object doesn't, an aggregate has a root" — and a mid-level candidate can recite it. A senior candidate can tell you *why* an aggregate exists at all, derived from which invariants need coordination and which don't (Module 9's invariant confluence), why the aggregate is the unit of serialization and therefore has a **throughput ceiling of roughly one over its read–decide–write latency** no matter how much hardware you buy, why EF Core will happily let two concurrent requests break an aggregate's invariant if a child row changes and the root row doesn't, why "one `Customer` class" always ends with thirty properties and three teams arguing, why the anticorruption layer is a coupling-management tool and not a design-pattern flourish, why an anaemic domain model is often the *correct* choice for a supporting subdomain, why "the aggregate" is being actively challenged in 2025–2026 by Dynamic Consistency Boundaries and what that does and doesn't change for a state-stored .NET system, and why the right answer to "should we use DDD here?" is almost never yes-or-no but "here, and here, and not there."

This module closes threads that several earlier modules opened by name. **Module 12** (Concepts 32 and 47) told you that distributed transactions are usually a symptom of a boundary drawn in the wrong place and promised that aggregates would turn the transaction boundary from a storage constraint into a design method. **Module 16** gave you records, value semantics, strongly-typed IDs, "make illegal states unrepresentable," and C# 15's `closed` hierarchies and `union` types — the language features that make modern tactical DDD cheaper than it has ever been in C#. **Module 19** (Concepts 5, 33, 35, 36, 68) gave you EF Core complex types, optimistic concurrency, the outbox interceptor, the three domain-event dispatch placements, and EF's DDD mapping features. **Module 20** (Concepts 15, 29, 34–36, 44, 45) put the domain at the centre of the layering, priced persistence ignorance, split reads from writes, and distinguished domain events from integration events. **Module 21** (Concepts 41, 53–58) told you that bounded contexts are the source of module boundaries, that duplication across them is a feature, and that EventStorming is the cheapest discovery tool — and said "Module 22 takes it properly." **Module 9** (Concept 31) gave you invariant confluence, which is the first-principles justification for the aggregate. **Module 11** gave you the outbox and delivery semantics that domain events ride on.

It shows up in five places in an interview loop: the **design round** (step 4 of the 7-step framework, "data model," is where aggregates quietly decide whether your design survives concurrency), the **architect round** ("how would you split this organization's systems?" is a bounded-context and context-map question wearing a different hat), the **code-review round** (anaemic models, public setters, leaky aggregates and god classes are the most common things put in front of you), the **deep technical round** ("what happens when two requests modify the same order?"), and the **behavioural round** (disagreements with the business about what a word means, or with another team about who owns a model, are among the most common "technical disagreement" stories).

**Current platform state (verified September 2026).** .NET 10 (November 2025) is the production-current LTS, supported to November 2028; .NET 11 RC1 shipped 8 September 2026 ahead of GA on 10 November 2026. What changed in this module's ecosystem, and why each item matters:

- **C# now has real sum types — almost.** C# 14 (with .NET 10) gave you the `field` keyword, which makes invariant-guarding properties one line shorter. **C# 15 (with .NET 11) adds `closed` class hierarchies and `union` types** with compiler-checked exhaustiveness (Module 16, Concepts 31–40). **Why it matters here:** aggregate lifecycles ("Draft → Placed → Paid → Shipped"), command outcomes ("Accepted / Rejected(reason) / Conflict") and decider-style aggregates (Concept 73) are sum types by nature, and modelling them as such removes an entire category of invalid-state bugs. Until you're on .NET 11, use closed-by-convention hierarchies with a throwing discard arm.
- **EF Core 10 made complex types the recommended mapping for value objects.** Complex types have value semantics (assignment copies, comparison compares contents, `ExecuteUpdateAsync` works), EF 10 added optional complex types (currently requiring at least one required property on the type), struct support (but not collections of structs), JSON mapping, and **`ComplexCollection`**, which maps a collection of complex types — and on relational providers must map it to a single JSON column. Microsoft's own guidance says users of owned entity types for table splitting or JSON "are advised to switch." EF 11 (November 2026) extends complex types to TPT/TPC inheritance and lets keys and indexes target scalar properties nested inside complex types. **Why it matters here:** the long-standing "owned types are how you map value objects" advice is now wrong, and the reason — owned types have hidden identity and reference semantics — is exactly the entity/value-object distinction this module is built on.
- **The aggregate is being challenged, seriously, for the first time in a decade.** **Dynamic Consistency Boundary (DCB)**, introduced in Sara Pellegrini's "Killing the Aggregate" and now formalized as a specification at **dcb.events** (Bastian Waidelich, Sara Pellegrini, Paul Grimshaw), replaces per-stream optimistic locking with a *query-scoped* append condition, so the consistency boundary is chosen per decision rather than per object. Support is real: **Axon Framework 5 with Axon Server 2025.1** (initially experimental), and **Marten 9.0** — released May 2026 as part of the "Critter Stack 2026" wave alongside Wolverine 6 and Polecat 4, targeting .NET 9/10 and removing runtime code generation — ships tag-based queries, `FetchForWritingByTags`, `DcbConcurrencyException`, identity-less "boundary aggregates" and an opt-in PostgreSQL `hstore` tag-storage mode. Implementations also exist in TypeScript, PHP, Rust, Elixir and Python. **Why it matters here:** you should be able to explain what DCB changes (event-sourced systems can enforce cross-entity invariants without a fixed aggregate), what it doesn't (a state-stored EF Core system — most .NET systems — still needs aggregates), and what it costs (Concept 74).
- **Coupling finally has a measurable vocabulary.** Vlad Khononov's *Balancing Coupling in Software Design* (Addison-Wesley, 2024) and the companion site **coupling.dev** define coupling along three dimensions — **integration strength** (intrusive, functional, model, contract), **distance** and **volatility** — with the heuristic *balance = (strength XOR distance) OR NOT volatility*. **Why it matters here:** it explains, from first principles, why the context-mapping patterns work: an anticorruption layer or published language converts **model coupling** into **contract coupling**, which is what makes distance affordable (Concept 32).
- **Eric Evans is now working on DDD and LLMs.** His consultancy's stated focus is integrating LLM components into domain-rich systems; his January 2026 article *Context Mapping with an AI-based Component* argues that an LLM is itself a bounded context — its own language, a probabilistic consistency model — and that an **anticorruption layer is essential** at that seam. His DDD Europe 2026 opening keynote states the working hypothesis plainly: *domain models still matter, bounded contexts still matter, language still matters — but the models will look different.* **Why it matters here:** "does DDD still matter with AI?" is now a question you can be asked, and there's a primary-source answer (Concept 104).
- **Microsoft's guidance was refreshed — and part of it wasn't.** The Azure Architecture Center's *Use tactical DDD to design microservices* (updated February 2026) states the sizing rule worth memorizing — *design a microservice to be no smaller than an aggregate and no larger than a bounded context* — and its *domain analysis* article now recommends Khononov's *Learning Domain-Driven Design* as the modern reference. The free e-book *.NET Microservices: Architecture for Containerized .NET Applications* still has the most complete .NET-specific DDD chapter (seedwork base classes, value objects, enumeration classes, domain events), but it still references the archived eShopOnContainers and dispatches domain events through MediatR *before* `SaveChanges` — both facts worth knowing before you cite it.
- **The collaborative-modelling toolkit is current and free.** The DDD Crew's *DDD Starter Modelling Process* and *Bounded Context Canvas* were updated in May 2026. One detail that signals you've actually facilitated a session: the **EventStorming glossary now calls the big yellow sticky a "constraint"** — "aggregate" is officially a legacy word in EventStorming, because it confuses business stakeholders (Concept 35).
- **Library licensing affects DDD templates.** MediatR 13+ and MassTransit v9 are commercial (Module 21, Concept 49). A large share of .NET DDD templates dispatch domain events through MediatR's `INotification`; that is now a procurement decision rather than a default. Wolverine (MIT) and a 40-line in-house dispatcher remain the free options, and the domain model itself should not care which is used (Concept 81).

This module has nine jobs:

1. **Reframe DDD as three decisions** — consistency, language, investment — and derive every pattern from one of them rather than memorizing it.
2. **Do strategic design properly**: classify subdomains, find the core, and turn the classification into an investment and implementation strategy.
3. **Find and relate bounded contexts**, and use the context-map patterns as coupling-management tools with explicit costs.
4. **Turn business knowledge into boundaries** with discovery techniques that work in a room with non-engineers.
5. **Build the tactical building blocks in modern C#** — entities, value objects, services, factories, policies — so that the model is always valid and illegal states are unrepresentable.
6. **Design aggregates with numbers**: derive them, size them from contention and invariants, enforce them correctly under concurrency in EF Core, and handle set-based and cross-aggregate rules.
7. **Make domain events reliable** — raising, dispatching, translating to integration events, and designing the eventual consistency the business actually sees.
8. **Implement the application layer and persistence in .NET/EF Core 10** without letting the database or the framework shape the model.
9. **Take the architect's view**: when not to use DDD, what reviewers look for, how to use it in a design round without sounding like a textbook, and what AI changes.

Nine framings to carry through:

1. **An aggregate is a consistency boundary, not an object graph.** It exists to protect invariants at commit time. If there's no invariant, there's no reason for the boundary.
2. **A bounded context is a language boundary, not a deployment boundary.** It's the scope within which a word has one meaning. Whether it's a module, a service or a folder is a separate decision (Module 21's two axes).
3. **A subdomain classification is an investment decision.** Core gets your best people and your richest model; generic gets bought; supporting gets the simplest thing that works.
4. **Invariants are discovered, not invented.** The question is always "must this be true the instant the transaction commits, or is it acceptable for it to become true shortly afterwards?" — and the business, not the developer, answers it.
5. **Small aggregates, references by identity, events between them** — Vernon's rules — and the senior skill is knowing precisely when each one should be broken.
6. **Duplication across contexts is the price of independence; translation at the boundary is the mechanism.** A `Customer` in Billing and a `Customer` in Support are different concepts sharing an ID.
7. **A model is a tool for a purpose, not a copy of reality.** Its measure is whether it makes the important decisions easy to express and the invalid ones impossible — not whether it resembles the real world.
8. **Tactical DDD without strategic DDD is ceremony.** Repositories and entities in a system with no ubiquitous language and no bounded contexts is "DDD-lite," and it usually costs more than it returns.
9. **Apply DDD where the complexity lives.** In most systems, most of the code should *not* be a rich domain model — and saying so is a senior signal, not a heresy.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Three decisions, not a catalogue | Consistency, language, investment — every pattern serves one of them |
| 2 | The problem DDD exists for | Essential complexity in business rules; if the rules are trivial, DDD is overhead |
| 3 | Strategic before tactical | Tactical patterns without contexts and language is DDD-lite |
| 4 | A model is a tool | Useful for a purpose, not realistic; judged by what it makes easy and impossible |
| 5 | Ubiquitous language | One vocabulary per context, used in speech, docs, tests and code — renames are design |
| 6 | Knowledge crunching | Refactoring toward deeper insight; the breakthrough is a new concept, not a new class |
| 7 | Domain experts and the whirlpool | Scenarios → model → code probes → back to scenarios; iterative by design |
| 8 | When DDD is the wrong tool | Transaction script, active record and CRUD are correct choices in the right subdomain |
| 9 | The vocabulary map | Problem space vs solution space, strategic vs tactical, one table |
| 10 | Domain and subdomains | The problem space, partitioned by what the business does |
| 11 | Core, supporting, generic | Differentiating and complex; necessary and simple; complex and solved |
| 12 | Finding the core | "Would we win by being better at this?"; Core Domain Charts |
| 13 | Subdomains evolve | Core becomes commodity; classifications have a shelf life |
| 14 | From classification to strategy | Build/buy, team allocation, and which logic pattern each type warrants |
| 15 | Distillation | Vision statement, highlighted core, segregated generic subdomains |
| 16 | Bounded context defined | The boundary within which a model and its language are consistent |
| 17 | Subdomains ≠ bounded contexts | Problem-space partition vs solution-space boundary; the mapping is n:m |
| 18 | Language heuristics | Same word, different meaning; different words, same thing — both mark seams |
| 19 | Other boundary heuristics | Pivotal events, business decisions, rate of change, ownership; the canvas |
| 20 | Sizing a context | Team-ownable; no smaller than an aggregate, no larger than a context |
| 21 | Contexts vs modules vs services | A context is a model boundary; deployment is a separate decision |
| 22 | Context maps | The integration relationships between contexts — and the power behind them |
| 23 | Partnership | Two teams succeed or fail together and coordinate releases |
| 24 | Shared kernel | A small, jointly owned model subset — expensive, keep tiny |
| 25 | Customer–supplier | Upstream serves downstream's negotiated needs |
| 26 | Conformist | Downstream adopts upstream's model wholesale — cheap until it isn't |
| 27 | Anticorruption layer | Translate the other model at the edge so it can't leak in |
| 28 | Open host service & published language | Upstream publishes a stable, documented protocol for many consumers |
| 29 | Separate ways | Sometimes the cheapest integration is none |
| 30 | Big ball of mud & bubble contexts | Quarantine legacy; grow clean models inside a protected bubble |
| 31 | Context maps as power maps | Upstream/downstream is organizational leverage; map the current state first |
| 32 | Balanced coupling | Strength, distance, volatility — why ACLs and published languages work |
| 33 | Integration between contexts | Events as published language; RPC where decisions need answers; never shared tables |
| 34 | AI components as bounded contexts | An LLM has its own language and a probabilistic model — wrap it in an ACL |
| 35 | EventStorming, precisely | Formats, notation, and why "aggregate" became "constraint" |
| 36 | From stickies to aggregates | Command + constraint + event clusters are aggregate candidates |
| 37 | Other discovery techniques | Domain Storytelling, Example Mapping, Event Modeling |
| 38 | The DDD Starter Modelling Process | Understand → Discover → Decompose → Strategize → Connect → Organise → Define → Code |
| 39 | Validating boundaries | Walk real scenarios across the proposed contexts; count the messages |
| 40 | Why tactical patterns exist | To keep invariants true and business rules in one place |
| 41 | Entities | Identity and lifecycle; equality by ID, not attributes |
| 42 | Identity generation | Who assigns identity and when — early IDs, GUIDv7, natural keys |
| 43 | Value objects | Defined by attributes; immutable; self-validating; side-effect-free |
| 44 | Value objects in C# | `record` vs `readonly record struct` vs class; the `with` and collection traps |
| 45 | Primitive obsession & strongly-typed IDs | `OrderId`, not `Guid`; generators when there are many |
| 46 | Value objects carry behaviour | Money allocation, date-range overlap — push logic down into values |
| 47 | Mapping value objects in EF Core 10 | Complex types (and `ComplexCollection` → JSON), converters; owned types are legacy |
| 48 | Always-valid & parse-don't-validate | Construct valid or not at all; input validation ≠ invariants |
| 49 | Illegal states unrepresentable | States as types; C# 15 `closed` hierarchies for lifecycles |
| 50 | Domain services | Domain logic with no natural owner; stateless; named in the language |
| 51 | Passing collaborators | Method injection and double dispatch; the DDD trilemma introduced |
| 52 | Factories | Complex creation, creation by another aggregate, creation vs reconstitution |
| 53 | Policies and specifications | Make implicit rules explicit, named, testable objects |
| 54 | Enumerations | `enum` vs enumeration class vs closed hierarchy |
| 55 | Domain modules | Package by concept, never by pattern (`Entities/`, `ValueObjects/`) |
| 56 | Deriving the aggregate | Non-confluent invariants need serialization; the aggregate is its smallest unit |
| 57 | The aggregate defined | Root, boundary, local identity, invariants satisfied at commit |
| 58 | Finding true invariants | "Must this hold at commit?" and "whose job is it to make it consistent?" |
| 59 | Vernon's four rules | True invariants, small, reference by ID, eventual consistency outside |
| 60 | Small aggregates | Large clusters fail on loading, contention and conflicts |
| 61 | Reference by identity | IDs, not navigations — load boundaries and distribution-readiness |
| 62 | Eventual consistency outside | Events + handlers; the business decides the acceptable lag |
| 63 | Breaking the rules | UI convenience, missing mechanisms, global transactions, query performance |
| 64 | Behaviour-first design | Commands in, decisions made, events out — never nouns first |
| 65 | Sizing with numbers | Conflict probability ≈ 1 − e^(−λd); an aggregate's ceiling is ~1/d |
| 66 | Hot aggregates | Split, escrow, reserve, queue, or relax the invariant |
| 67 | Growing collections | An unbounded child collection is a separate aggregate in disguise |
| 68 | Optimistic concurrency in EF Core | A child change doesn't touch the root row — version the root explicitly |
| 69 | Resolving conflicts | Retry the decision, surface it, or merge; race conditions are business questions |
| 70 | Set-based rules and uniqueness | Constraint, reservation, registry aggregate, or compensate — and the trilemma |
| 71 | Cross-aggregate invariants | Merge, co-transact pragmatically, reserve, or orchestrate |
| 72 | Aggregates as state machines | Guarded transitions; lifecycle as types |
| 73 | The functional aggregate (Decider) | decide(command, state) → events; evolve(state, event) → state |
| 74 | Dynamic Consistency Boundary | Per-decision boundaries in event-sourced stores — power and price |
| 75 | Aggregates at runtime scale | Partition key, actor, grain — the aggregate as a unit of placement |
| 76 | The aggregate design checklist | The Aggregate Design Canvas, condensed |
| 77 | Domain events defined | A fact the business cares about, in the language, past tense |
| 78 | Kinds of events | Domain vs integration; notification vs state transfer vs sourcing |
| 79 | Designing events | Intent-revealing names, not CRUD; carry what consumers need to decide |
| 80 | Raising events | Collect on the root, return from methods, or a static raiser — trade-offs |
| 81 | Dispatching events | In-transaction, after-commit, outbox — and the cascading loop |
| 82 | What handlers may do | Same-context side effects in-transaction; everything else via the outbox |
| 83 | Translating at the boundary | Domain event → integration event is the published language being written |
| 84 | Eventual consistency, as the user sees it | "Received" vs "confirmed"; pending states are domain states |
| 85 | Processes as aggregates | Sagas and process managers are consistency boundaries for workflows |
| 86 | Repositories | One per aggregate root; the illusion of an in-memory collection |
| 87 | Repository API design | Minimal, intention-revealing, no `Update`, no `IQueryable` |
| 88 | Application services | Load → decide → save → publish; thin, transactional, no rules |
| 89 | Mapping an aggregate with EF Core 10 | Fields, private constructors, complex types, converters, tokens |
| 90 | Creation vs reconstitution | Constructors validate; materialization must not |
| 91 | Persistence ignorance, priced | The pragmatic line; separate persistence models; documents per aggregate |
| 92 | Reads bypass the aggregate | Queries project straight from storage — the model is for writes |
| 93 | Testing the model | Given/when/then on aggregates, no mocks, properties for invariants |
| 94 | Validation layering | Input, invariant, cross-aggregate, policy — each has one home |
| 95 | Errors and outcomes | Exceptions for broken programs, results for business outcomes |
| 96 | The anaemic domain model | An anti-pattern in the core, often correct in supporting subdomains |
| 97 | DDD-lite | Patterns without language or boundaries — all cost, little benefit |
| 98 | The canonical enterprise model | One model for the whole company is the opposite of DDD |
| 99 | Leaky aggregates | Public setters, exposed collections, cross-aggregate navigation, lazy loading |
| 100 | Over-modelling | Value objects for everything, events nobody reads, DDD in CRUD |
| 101 | The review catalogue | What a reviewer looks for in a domain model, in order |
| 102 | DDD in the design round | Where aggregates and contexts fit in the 7 steps, without jargon |
| 103 | DDD in the architect round | Context maps as org design; the core domain as an investment case |
| 104 | DDD and AI | What changes, what doesn't, and the primary-source position |
| 105 | The 60-second answers | What is DDD; how to design an aggregate; how to find contexts |
| 106 | The close | One paragraph that demonstrates judgment |

---

# Part A — What DDD actually is

Most DDD explanations start with entities and value objects, which is exactly backwards. The building blocks are consequences. Part A establishes what they are consequences *of* — so that when an interviewer asks "why is an aggregate small?" or "why not one Customer class?", you answer from the reason rather than from the rule.

---

## Concept 1 — Three decisions, not a pattern catalogue

Eric Evans' *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) is 500 pages and introduces several dozen patterns, and the most common way people learn it is as a catalogue: here is an Entity, here is a Value Object, here is a Repository. That's the least useful way to hold it in your head, because a catalogue gives you shapes without telling you when to use them.

Hold it instead as **three decisions**, each of which DDD gives you a method for making:

| Decision | The question | DDD's unit | What goes wrong without it |
|---|---|---|---|
| **Consistency** | Which facts must be true *together*, at the instant a change commits? | **Aggregate** | Invariants enforced by hope; race conditions; giant transactions; deadlocks |
| **Language** | Where does a word stop meaning one thing and start meaning another? | **Bounded context** | One `Customer` class with thirty properties that three teams fight over |
| **Investment** | Where does the business win or lose, and where is it merely keeping the lights on? | **Subdomain type** (core / supporting / generic) | Gold-plating the invoice PDF while the pricing engine is a stored procedure |

Everything else in DDD is either **one of these decisions made visible** or **a mechanism for living with one**:

- *Entities, value objects, domain services, factories, specifications* — the vocabulary for expressing rules *inside* a consistency boundary.
- *Repositories* — how you load and save exactly one consistency boundary at a time.
- *Domain events* — how consistency boundaries tell each other what happened, since they can't share a transaction.
- *Context maps, anticorruption layers, published languages, shared kernels* — how language boundaries integrate without their models leaking into each other.
- *Distillation, core domain charts* — how you make the investment decision explicit and keep it.

The interview use of this framing is immediate. When asked "what is DDD?", a candidate who recites building blocks sounds like a textbook. A candidate who says *"it's a method for three decisions — where consistency has to be immediate, where language changes meaning, and where the business needs us to invest — and the patterns are how you implement those decisions"* sounds like someone who has applied it.

---

## Concept 2 — The problem DDD exists for

Fred Brooks' *No Silver Bullet* (1986) separated **essential complexity** — inherent in the problem — from **accidental complexity** — introduced by our tools and choices. Most of the curriculum so far has been about accidental complexity: GC pauses, thread-pool starvation, N+1 queries, network partitions. DDD is almost entirely about the other kind.

**Essential complexity in business software lives in the rules.** Not in the data volume, not in the request rate — in the fact that an insurance claim can be reopened if new evidence arrives within 90 days unless it was settled by arbitration, in which case only the ombudsman can reopen it, and reopening re-triggers fraud scoring but not reserve recalculation unless the claim exceeds the adjuster's authority limit. That sentence is the software. Everything else is plumbing.

DDD's bet is that the most effective way to manage that kind of complexity is to:

1. **Build a model of the domain** that makes those rules explicit, named, and located in one place.
2. **Develop that model with the people who understand the domain**, in their language, continuously.
3. **Keep the model's integrity** by drawing explicit boundaries around where it applies.

The corollary, which you should say before anyone else in the room does: **if the rules are trivial, DDD is overhead.** A system whose behaviour is "store what the form says, show it back, let an admin edit it" has no essential complexity for a domain model to manage. Wrapping that in aggregates, repositories and domain events doesn't manage complexity — it creates it. That's the whole of Concept 8, and it's the most important calibration point in the module.

A useful diagnostic question for any system under discussion: *"If I gave a new engineer the database schema, how much of the system's behaviour could they reconstruct?"* If the answer is "most of it," the domain is data-centric and DDD's tactical patterns will mostly be ceremony. If the answer is "almost none — the interesting part is all in the rules about when things can change," DDD is earning its keep.

---

## Concept 3 — Strategic before tactical

DDD has two halves:

- **Strategic design** — subdomains, bounded contexts, context maps, ubiquitous language, distillation. The *large-scale* decisions: how to partition the problem, where the model boundaries go, how teams and models relate, where to invest.
- **Tactical design** — entities, value objects, aggregates, domain services, repositories, factories, domain events. The *small-scale* decisions: how to express a model in code inside one bounded context.

Evans' book presents tactical patterns first (Part II) and strategic ones much later (Part IV), and he has said in talks since that bounded contexts and context mapping deserved far more prominence than their position at the back of the book gave them: the strategic patterns are the more important and the more widely applicable. Vaughn Vernon's *Implementing Domain-Driven Design* (2013) and Vlad Khononov's *Learning Domain-Driven Design* (2021) both lead with strategy for this reason.

The industry largely learned DDD in the wrong order anyway, and the result has a name: **DDD-lite** — teams adopting repositories, entities with private setters and a `Domain` folder, with no ubiquitous language, no bounded contexts and no subdomain analysis (Concept 97). That gives you the ceremony of the tactical patterns without the thing that makes them valuable: a model that is *consistent within a boundary* and *expressed in the business's language*. Without a bounded context, "the model" is the whole company's data model with methods on it. Without a ubiquitous language, the tactical patterns are just an unusual way of writing CRUD.

The ordering principle for this module, and for any design conversation: **strategic decisions constrain tactical ones, never the reverse.** You can't size an aggregate until you know what context it's in (because "Order" means something different in Checkout and in Fulfilment). You can't decide whether a rich domain model is worth it until you know whether the subdomain is core. And you can't decide what a value object is called until you know the language of the context it lives in.

The interview signal: when a design question arrives, the senior move is to spend thirty seconds on the strategic layer — *"let me first separate the parts of this that are differentiating from the parts that are just necessary, because I'd model them very differently"* — before drawing a single class.

---

## Concept 4 — A model is a tool, not a copy of reality

The most persistent misconception about domain modelling is that the job is to *discover* the concepts that exist in the real world and translate them into classes: find the nouns, make them entities, add methods. Rebecca Wirfs-Brock — who helped popularize that approach in her 1990 book — and Mathias Verraes refute it directly in their essay *Design and Reality* (2021): the relationship between model and reality is not discovery but **invention**. A model is a deliberately simplified, deliberately partial representation that is **useful for a specific purpose**.

The classic illustration is a map — Evans uses maps to make the same point. A subway map is wildly unrealistic — distances are wrong, the river is a straight line, the streets are missing — and it is *the best possible model* for choosing which train to take. A street map is more realistic and worse for that purpose. **Usefulness for the purpose is the only criterion.**

Three practical consequences follow, and all three show up in interviews:

**1. Different purposes need different models of the "same" thing.** A shipping company's `Cargo` in the booking context is a customer commitment with a price and a delivery promise. In the routing context it's a set of legs through a transport network. In the handling context it's a physical object being scanned at ports. Trying to make one model serve all three purposes makes it worse at each — which is the entire motivation for bounded contexts (Concept 16).

**2. The best model is often not the obvious one.** The breakthroughs in Evans' book aren't found by looking harder at reality; they're *invented* by reframing the problem. His syndicated-loan example (Chapter 8) is the canonical story: the team spent months fighting a model where loan shares and facility shares were tangled together, until someone proposed a new abstraction — a "share pie" that could be applied to either — and a whole class of bugs and special cases vanished. Nobody in the business had used that concept before. It was a design choice that the domain experts then *adopted* because it made their own reasoning clearer.

**3. A model should be judged by what it makes easy and what it makes impossible.** Good model: the important business decisions are one method call with an obvious name, and the invalid states can't be constructed. Bad model: every rule is a conditional in a service class, and the types allow anything.

The line worth having ready when someone says "our model should reflect the real world": *"I'd push back gently on 'reflect' — I'd want it to be useful for the decisions this context makes. The real world has far more detail than any one purpose needs, and a model that tries to capture all of it ends up serving none of the purposes well."*

---

## Concept 5 — Ubiquitous language

The single idea in DDD with the highest return on investment, and the one most often skipped.

**A ubiquitous language is a shared, rigorous vocabulary, developed jointly by domain experts and developers, used consistently in conversation, documentation, tests and code — within one bounded context.**

Every word in that definition matters:

- **Shared** — the experts and the engineers use the *same* words. Not "the business says 'policy lapse' and we call it `IsActive = false`." When the translation happens in engineers' heads, it's lossy, it's invisible, and it's where the bugs live.
- **Rigorous** — each term has one precise meaning in this context. "Active customer" is defined (placed an order in the last 12 months? has a non-cancelled subscription? logged in?), and the definition is written down.
- **Used in code** — the class is `Policy`, the method is `Lapse(reason)`, the event is `PolicyLapsed`. Not `PolicyEntity.UpdateStatus(3)`. If a domain expert read your aggregate's public surface aloud, they should recognize it.
- **Within one bounded context** — this is the part people miss. The language is *not* company-wide. "Policy" means one thing in underwriting (a risk decision with terms) and another in billing (a schedule of premiums). Each context has its own ubiquitous language. The boundary of a language *is* the boundary of a context (Concept 16).

Why it's worth the discipline:

- **It removes a translation layer.** Every translation between "what the business said" and "what the code says" is a place where meaning is lost. A shared language removes the layer entirely.
- **It makes the model reviewable by the people who know the rules.** You can walk an expert through `claim.Reopen(evidence, receivedOn)` and they can tell you what's wrong. You cannot walk them through `ClaimService.UpdateClaimStatus(claimId, 4, dto)`.
- **It surfaces modelling problems as language problems.** When the team can't agree on a word, or the experts use two words for what the code treats as one thing, you've found a design issue — usually a missing concept or a context boundary.

Three practices that make it real:

1. **A glossary per context**, kept short, living next to the code (a `GLOSSARY.md` in the module is fine). Terms, definitions, examples, and — crucially — *terms we deliberately don't use and why*.
2. **Renames are design changes.** When the language evolves ("we don't 'approve' claims, we 'accept liability'"), rename the code. A codebase whose names drift from the business's speech is decaying, even if every test passes.
3. **Tests read like specifications.** `Accepting_liability_on_a_claim_above_adjuster_authority_requires_supervisor_referral()` is documentation that can't go stale.

The interview line: *"The first thing I'd want is the vocabulary. If the business says 'lapse' and the code says `IsActive = false`, every conversation needs a translator, and every translation is a place for a bug to hide."*

---

## Concept 6 — Knowledge crunching and refactoring toward deeper insight

Evans' term for how a model is built: **knowledge crunching** — an iterative process in which developers and domain experts together digest large amounts of domain information, try model ideas against real scenarios, discard what doesn't work, and distil what does. It is explicitly *not* a requirements-gathering phase that ends. The model is never finished, because the team's understanding is never finished.

Three ideas from this part of the book are worth being able to discuss:

**Refactoring toward deeper insight.** Ordinary refactoring improves code structure without changing behaviour. *Refactoring toward deeper insight* improves the **model** — replacing a shallow concept with a deeper one that makes the code simpler and the rules more explicit. A shipping model that represents a cargo's route as a list of ports is shallow. One that introduces `Itinerary` made of `Leg`s, each with a load and unload location and time, is deeper — and suddenly "is this cargo misdirected?" is a method on `Itinerary` rather than a 60-line procedure.

**Implicit concepts made explicit.** The signal that a concept is missing is awkwardness: a method with many flags, a conditional that keeps growing, a comment explaining "this is how we handle the case where…", a word the experts use that appears nowhere in the code. The fix is to name the thing. A rule buried in a service ("allow booking up to 110% of capacity") becomes an explicit `OverbookingPolicy` object (Concept 53) — Evans' own example.

**Breakthroughs are discontinuous.** Deeper insight tends to arrive in jumps, not increments — the "share pie" story from Concept 4. The practical implication is uncomfortable: a breakthrough may require a significant restructuring of code that works. That's a legitimate cost, and the reason to invest in the *core* domain (Concept 14) is that this is where breakthroughs are worth their cost.

For an interview, the valuable part is being able to describe the process concretely: *"I'd expect the model to change significantly in the first few months. The signal I watch for is awkwardness — flags, growing conditionals, experts using words the code doesn't have. That's usually a missing concept, and naming it is the cheapest simplification available."*

---

## Concept 7 — Domain experts and the modelling whirlpool

DDD assumes access to **domain experts**: people who understand the business rules deeply — underwriters, dispatchers, accountants, clinicians — as opposed to people who relay requirements. The approach doesn't work well through a chain of intermediaries, because each hop loses nuance and the language degrades into a project dialect.

Evans describes the working loop as the **Model Exploration Whirlpool** (on domainlanguage.com): start from a concrete **scenario**, propose a **model**, **code-probe** it (a small, fast implementation or test that exercises the scenario), **challenge** it with harder scenarios, and harvest the language that survives. Round and round, cheaply. The emphasis is on *concrete scenarios* — "a customer with a 30-day payment term places an order on the 28th of the month in a currency we don't bill in" — because abstract discussion of models converges on vague agreement, while scenarios expose disagreement.

What makes this work in practice:

- **Concrete examples beat abstract definitions.** Ask experts "walk me through the last time this went wrong," not "what are the requirements for refunds."
- **Model in front of the experts.** Whiteboards, sticky notes (Concept 35), or a test written live. Models developed in private and presented for approval get polite nods and hidden disagreement.
- **Hunt for disagreement.** When two experts describe the same process differently, don't pick one — the difference is information, often a context boundary (Concept 18) or a policy that varies by region or product.
- **Developers must learn the domain.** Not to replace the experts, but to ask the questions that expose the model's gaps. A developer who thinks of the domain as "the requirements" will build what was asked for rather than what was meant.

Two realities worth acknowledging honestly in an architect conversation. First, expert time is expensive and scarce; DDD's collaborative techniques are designed to use it efficiently (a day of EventStorming often replaces weeks of requirement documents). Second, sometimes there *is* no expert — a genuinely new product, or a domain the business is still inventing. Then the team is doing the discovery itself, and the model will change more; that's another argument for concentrating modelling effort where it pays (the core).

---

## Concept 8 — When DDD is the wrong tool

Martin Fowler's *Patterns of Enterprise Application Architecture* (2002) describes the three main ways to organize domain logic, and knowing all three is what lets you choose rather than default:

| Pattern | Shape | Right when |
|---|---|---|
| **Transaction Script** | One procedure per use case, operating directly on data | The logic per use case is simple and doesn't share much across use cases. Cheapest to write and read |
| **Table Module / Active Record** | One class per table/row, data and simple behaviour together | Behaviour maps closely onto the data structure; mostly CRUD with light validation |
| **Domain Model** | An object model of the domain, with rules on the objects, persistence separate | Rules are complex, interact, and change; many use cases share the same concepts and invariants |

Fowler's accompanying observation is a cost curve: transaction scripts are cheap to start and get expensive fast as logic grows, while a domain model costs more up front and scales much better with complexity. **The crossover depends on the complexity of the rules**, and below it the domain model is slower to build, harder to read and adds nothing. (It's the same shape as Module 21's microservice premium, and for the same reason: fixed cost now, benefit only above a complexity threshold.)

Khononov's *Learning Domain-Driven Design* turns this into a decision heuristic tied to subdomain type (Concept 14):

- **Simple logic, mostly data** → transaction script, or active record if the data structures are complex.
- **Complex business logic** → domain model.
- **Complex logic *and* a need for audit, history, or analysis of how state evolved** (money, compliance) → event-sourced domain model (Module 24).

And the explicit statement you should be willing to make in an interview: **most of a typical system should not be a rich domain model.** The admin screens, the reference-data maintenance, the notification preferences, the reporting — these are supporting or generic, and an anaemic model or plain transaction scripts are the *correct* engineering choice there (Concept 96). DDD's own strategic design tells you this; it's not a departure from DDD but an application of it.

Signals that DDD's tactical patterns are the wrong tool for a given area:

- The domain experts describe it in terms of screens and fields rather than rules and decisions.
- Most operations are create/read/update/delete with validation that fits in attributes.
- There are few invariants that span more than one field.
- It's not where the business competes.
- It can be bought (Concept 14).

The senior sentence: *"I'd apply the full tactical toolkit to the pricing engine and probably nowhere else in this system. The rest is CRUD, and wrapping CRUD in aggregates and repositories makes it slower to change without making it more correct."*

---

## Concept 9 — The vocabulary map

A single table to fix the terms in place, because interviewers frequently probe whether you know which concepts live in the problem space and which in the solution space:

| | **Problem space** (the business as it is) | **Solution space** (what we build) |
|---|---|---|
| **Strategic** | **Domain** — the business's sphere of activity · **Subdomain** — a part of it (core / supporting / generic) · **Domain expert** | **Bounded context** — a boundary of model and language consistency · **Context map** — how contexts relate · **Ubiquitous language** (per context) · **Integration patterns** (ACL, OHS, PL, shared kernel, conformist…) |
| **Tactical** | **Invariants** and **business rules** · **Business events** ("the claim was settled") · **Processes** | **Aggregate** (root, boundary) · **Entity** · **Value object** · **Domain service** · **Domain event** · **Factory** · **Repository** · **Specification / policy** · **Module** |
| **Process** | Scenarios, stories, examples | Knowledge crunching · EventStorming · Domain Storytelling · Refactoring toward deeper insight |

Three relationships that are commonly confused, stated precisely:

- **Subdomain → bounded context** is *not* 1:1 by necessity. A subdomain is a partition of the *problem*; a bounded context is a boundary you *choose* in the solution. The ideal is close to 1:1, but a large core subdomain can span several contexts, and a small context can serve two supporting subdomains (Concept 17).
- **Bounded context → module → service**: a context is a *model* boundary. It can be implemented as a module in a modular monolith or as one or more services. Deployment is a separate decision (Concept 21, and Module 21's two axes).
- **Aggregate → transaction → partition**: an aggregate is the unit of consistency, therefore the unit of transaction, therefore the natural unit of partitioning and placement (Concept 75). Those three coincide by design, and when they don't, something is about to hurt.

---

# Part B — Strategic design: the problem space

Before you draw a single boundary, you need to know what the business actually does and which parts of it matter most. Part B is the investment decision — the third of the three — and it's the part of DDD that an architect is most directly paid for, because it's where engineering effort gets allocated.

---

## Concept 10 — Domain and subdomains

**The domain** is the sphere of knowledge and activity the software serves: insurance, freight logistics, online retail, clinical scheduling. It exists independently of any software — the business did it with paper before it did it with computers.

**Subdomains** are the parts of that domain, partitioned by *what the business does*. An online retailer's domain decomposes into subdomains like catalogue management, pricing and promotions, ordering, payments, fulfilment, returns, customer support, fraud detection, marketing, and identity. Each is a coherent area of activity with its own experts, its own rules and, usually, its own vocabulary.

Two properties of subdomains that trip people up:

**Subdomains are discovered, not designed.** They describe the *problem space* — how the business is organized to create value. You don't choose them the way you choose a bounded context; you find them, by talking to the business, reading its org chart and its capability map, and watching where its experts' knowledge clusters. (Module 21's Concept 54, "decompose by business capability," is the same idea from the architecture side: a business capability is roughly a subdomain.)

**Subdomains are hierarchical and the granularity is a judgment call.** "Fulfilment" breaks into warehouse management, carrier selection, shipment tracking, and last-mile delivery. How deep you go depends on what you're deciding: for an investment decision, the level where the answer to "does this differentiate us?" changes is the right level. Pricing might be core while "promotional coupon validation" inside it is supporting.

The practical output of this step is a **subdomain map**: a list (or diagram) of the business's subdomains with a one-line description of each. It is almost always the first artifact in a strategic design engagement, and it's worth producing even for a system you'll never touch, because it's the frame in which every later decision gets justified.

---

## Concept 11 — Core, supporting, generic

The classification that turns a subdomain map into a strategy. Evans introduced it; Khononov gave it the sharpest operational definition:

| Type | Competitive advantage? | Complexity | Examples | What it deserves |
|---|---|---|---|---|
| **Core** | **Yes** — the business wins or loses here | High (usually) | A freight company's route optimization; an insurer's risk pricing; a marketplace's matching and trust system | Your best engineers, a rich model, continuous knowledge crunching, in-house build |
| **Supporting** | No — but specific to this business | Low | Back-office workflows, internal catalogue maintenance, a custom approval flow | The simplest thing that works; transaction scripts, CRUD, low-code, or outsourced build |
| **Generic** | No — every business needs it | Often high, but *solved* | Identity, payments processing, email delivery, accounting ledger, tax calculation | **Buy or adopt**: SaaS, open source, a managed service |

The two defining axes are **differentiation** (does being better at this make the business win?) and **complexity** (is the logic hard?). The combinations produce the types:

- **Complex + differentiating = core.** This is where the business's secret sauce is encoded — and it's usually complex *because* it's where the business has invested in being clever.
- **Complex + not differentiating = generic.** Hard problems that everyone has, and that someone has already solved better than you will. Payments, identity, tax. Building these in-house is the classic engineering vanity project.
- **Simple + not differentiating = supporting.** Necessary, specific to your business, but not where you compete. The danger is over-engineering.
- **Simple + differentiating** is rare and usually unstable — if something simple differentiates you, competitors will copy it quickly and it will become supporting or generic (Concept 13).

Two points that separate a thoughtful answer from a textbook one:

**"Core" is about the business, not the technology.** The most technically interesting part of a system is frequently *not* core. A team that built a clever distributed rate limiter for an e-commerce site has built something generic; the core is probably pricing, recommendations or supplier relationships — which might be a humble-looking set of rules.

**Most of any system is not core.** A realistic ratio is one or two core subdomains, a handful of generic ones, and many supporting ones. If an organization claims ten core subdomains, it hasn't done the analysis — it has labelled everything important.

---

## Concept 12 — Finding the core

Identifying the core domain is harder than the definition suggests, because everybody believes their area is core. Useful tests, in order of how often they settle the question:

1. **"If we were twice as good at this, would we win more business or make more money?"** For route optimization at a freight company — yes, directly. For the HR leave-request workflow — no.
2. **"Could we buy this off the shelf without losing anything that customers notice?"** If yes, it's generic. If the answer is "we could, but we'd have to change how we do business," that's a strong core signal — the business's way of doing it *is* the advantage.
3. **"Why do customers choose us over competitors?"** The answer, translated into capabilities, points at the core. Ask sales and product, not engineering.
4. **"Where do the domain experts disagree, and where is the knowledge scarce?"** Core domains are usually where the business has deep, contested, hard-won expertise.
5. **"What does the business's strategy document say it will invest in?"** Often it names the core explicitly, in different words.

**Core Domain Charts** (from the DDD Crew, free on GitHub) make this collaborative: plot each subdomain on two axes — **business differentiation** and **model complexity** — with the business and engineering stakeholders in the room. The plot does two useful things. It forces an explicit, comparable judgment rather than a list of labels, and it surfaces *disagreement* — when product places a subdomain in the top-right and engineering places it bottom-left, you've found a strategic conversation that needed to happen. The charts can also carry arrows showing where each subdomain is moving, which connects directly to Concept 13.

Two common misidentifications to watch for:

- **The "hidden core"**: something the business doesn't think of as special, but which turns out to be why customers stay — a particularly good claims-handling experience, a dispatch process that's faster than competitors'. Often discovered by asking customers rather than the business.
- **The "suspect core"**: something labelled core for political reasons or because a senior person built it. The test is always question 1 — does being better at it actually move the business?

In an architect interview, the question "which part of this system would you invest the most design effort in?" is precisely this. The strong answer names the core, justifies it with a business reason, and says what you'd do *less* of elsewhere as a consequence.

---

## Concept 13 — Subdomains evolve

A subdomain classification is a snapshot, and it has a shelf life.

Simon Wardley's evolution axis (from Wardley Mapping) describes how any capability moves through four stages over time: **genesis** (novel, poorly understood, uncertain) → **custom-built** (understood well enough to build, still done differently by everyone) → **product** (standardized enough to be sold) → **commodity/utility** (undifferentiated, bought by the unit). Electricity, computing, payment processing and identity management have all made that journey.

Mapped onto subdomain types:

- **Core subdomains tend to drift toward generic** as the market catches up. Online payment was a genuine differentiator for early e-commerce sites; it's now bought from a provider. Recommendation engines are heading the same way.
- **Supporting subdomains can become core** when the business discovers an advantage in them — a logistics company that realizes its internal routing tool is good enough to sell as a product.
- **Generic subdomains can become core** for a specific business whose strategy changes — a retailer that decides to compete on fraud prevention rather than buying it.

The practical consequences:

1. **Revisit the classification periodically** — annually, or when strategy changes. An investment decision based on a three-year-old classification may be protecting yesterday's advantage.
2. **Design the core so it can be "demoted."** When a core capability commoditizes, the right move is to replace the in-house model with a bought one. That's much easier if the core was a well-bounded context with a clean contract than if its concepts leaked everywhere (Concept 27's anticorruption layer and Concept 28's published language are what make this swap cheap).
3. **Don't over-invest in genesis-stage modelling.** When a subdomain is genuinely new and poorly understood, the model will change dramatically. Build for learning — small, disposable, fast to change — and invest in model depth once the concepts stabilize.

A good sentence for an architecture discussion: *"I'd treat the classification as a decision with a revisit date. Payments was core for us three years ago; today it's generic and we should be buying it."*

---

## Concept 14 — From classification to strategy

Classification is only useful if it changes what you do. The consequences, per type:

| | **Core** | **Supporting** | **Generic** |
|---|---|---|---|
| **Build or buy** | Build, in-house | Build simply, or outsource, or low-code | Buy, adopt open source, use a managed service |
| **Who works on it** | Your strongest engineers, with direct expert access | Anyone competent; good for onboarding | Integration specialists; whoever owns the vendor relationship |
| **Business-logic pattern** (Khononov) | Domain model; event-sourced domain model if history/audit matters | Transaction script or active record | Whatever the product dictates; wrap it |
| **Modelling effort** | Continuous knowledge crunching; refactoring toward deeper insight | Just enough | None — you adopt the vendor's model at the edge |
| **Integration posture** | Protect it: ACLs around everything it consumes (Concept 27) | Conformist is often fine (Concept 26) | ACL around the vendor so it can be swapped |
| **Quality bar** | High test coverage of rules, property tests for invariants | Pragmatic | Contract tests against the vendor |
| **Architecture investment** | Worth hexagonal/clean boundaries, a rich model | Keep it thin; don't impose ceremony | Keep it at the edge |

Three points to make explicitly in an interview:

**Team allocation is the most consequential line in the table.** A common organizational failure is putting senior engineers on the interesting *technical* problems (which are often generic) and junior engineers on the "boring" business rules (which are often core). DDD's strategic advice inverts this: the business's advantage lives in the rules, so that's where the strongest modellers should be.

**Buying generic subdomains is a strategic decision, not a failure of ambition.** Building your own identity provider or payment processor consumes years of engineering on a problem that makes you no more competitive, and you'll do it worse than the specialists. The architect's job is to recognize this and say it — the language executives respond to is Module 33's build-vs-buy vocabulary.

**The pattern choice follows the classification, not the team's preference.** A team that loves rich domain models will want to apply one everywhere; a team that loves thin CRUD services will want the opposite. The classification gives you a principled reason to do different things in different places — which is exactly what a reviewer wants to hear: *"Pricing gets a proper domain model with value objects and aggregates; the supplier onboarding flow is a set of transaction scripts; identity is Entra ID behind an adapter. Three different approaches, chosen deliberately."*

---

## Concept 15 — Distillation

Evans' Part IV includes a set of **distillation** patterns — techniques for separating the core from everything else so the core stays visible and gets the attention it needs. They're less frequently discussed than bounded contexts, and knowing them is a mark of having read past the tactical chapters. The four worth knowing:

**Domain Vision Statement.** A one-page description of the core domain and the value it brings — written for the team, not for marketing. It answers "what is special about our model, and why does it matter?" It's short enough to be read, and it orients the team when a design choice pits core concerns against peripheral ones.

**Highlighted Core.** Make the core *visible* in the codebase and documentation: a short document naming the core concepts, a marker in the module structure, or simply a README that says "these five types are where the business's advantage is encoded; change them with extreme care." In a large codebase the core is easily lost among supporting code; highlighting it keeps review and testing attention where it belongs.

**Generic Subdomains, segregated.** Pull generic concerns — time zones, currency conversion, unit-of-measure math, address validation — out of the core into separate modules or libraries (or replace them with off-the-shelf components), so the core model contains *only* what's specific to the business. The core gets smaller and clearer every time you do this.

**Cohesive Mechanisms.** Complex *computational* machinery that the core needs but that isn't itself domain knowledge — a graph-traversal algorithm for routing, a constraint solver for scheduling, a rules engine — gets separated into a framework-like component with an intention-revealing interface. The core says *what* ("find the cheapest route that satisfies these constraints"); the mechanism handles *how*. Your strong DS&A background lives here: the core domain decides which constraints matter; the cohesive mechanism is where the algorithm lives.

The unifying idea: **the core should contain the business's distinctive knowledge and nothing else.** Everything generic, mechanical, or merely supporting is pushed out, so that the part of the system that matters most is also the part that's easiest to understand and change.

---

# Part C — Strategic design: bounded contexts and context maps

Part B partitioned the *problem*. Part C draws the boundaries of the *solution* — the second of the three decisions. Module 21 told you that the boundary is a language boundary before it is a code boundary; this part is where that sentence becomes a method, and where the integration patterns between contexts become coupling-management tools you can price.

---

## Concept 16 — The bounded context, defined

**A bounded context is an explicit boundary within which a particular domain model — and the ubiquitous language that expresses it — is defined, consistent and applicable.**

Inside the boundary, every term has exactly one meaning, every rule is expressed once, and the model is internally consistent. Outside it, the same words may mean something else, and that's fine — because the boundary makes it explicit that they're different models.

The motivation is the failure it prevents. In any organization larger than a handful of people, **a single unified model of the whole domain is not achievable**, and attempting one produces a model that is simultaneously too complicated for every use and too vague to enforce any rule precisely. Evans' observation: large projects that try for a unified model end up with an implicit fragmentation anyway — different teams quietly interpret the same classes differently — and the result is subtle bugs where one team's assumption violates another's. Bounded contexts make the fragmentation **explicit and managed** rather than implicit and accidental.

What a bounded context gives you:

- **Freedom inside.** The team can refactor the model, rename concepts and restructure aggregates without negotiating with anyone else, because nobody else depends on the internals.
- **Precision inside.** "Active customer" means exactly one thing, so rules can be stated exactly.
- **Explicit translation at the edges.** Where two contexts must exchange information, the translation is a designed, visible artifact (Concepts 27–28), not an assumption buried in shared classes.

And what defines one in practice — the boundary should be visible in at least four ways:

1. **Language**: a glossary where each term has one meaning.
2. **Code**: a module, assembly set or service whose internals are hidden (Module 21, Concept 33).
3. **Data**: a schema that only this context reads and writes (Module 21, Concept 35).
4. **Ownership**: one team (Concept 20).

If a "context" is only visible in a diagram, it isn't one yet.

---

## Concept 17 — Subdomains are not bounded contexts

The most common strategic-design confusion, and a favourite probing question.

- A **subdomain** is part of the **problem space**: an area of the business's activity. It exists whether or not you write software.
- A **bounded context** is part of the **solution space**: a boundary you *choose* around a model you build.

In an ideal world, they align 1:1 — one context per subdomain — because then each model serves exactly one business capability with one set of experts and one vocabulary. In practice the mapping is **n:m**, and knowing *why* is the useful part:

- **One subdomain, several contexts.** A large core subdomain may be split across contexts because a single model would be too big for one team, or because parts of it have genuinely different languages. "Pricing" at a large retailer might be split into *list-price management* and *promotion evaluation*, which share little vocabulary despite serving the same business goal.
- **One context, several subdomains.** A small supporting context might serve two closely related supporting subdomains because splitting them would add integration cost for no benefit. Early in a product's life, one context often spans several subdomains that will separate later.
- **A legacy system spanning everything.** The big ball of mud (Concept 30) is one "context" that spans many subdomains with no internal boundaries — which is exactly why it's hard to change.

The guideline: **aim for alignment, accept deviation when there's a reason, and write the reason down.** A context that serves three unrelated subdomains is a monolith-in-waiting; a subdomain split across five contexts with the same language is fragmentation for its own sake.

The interview line: *"Subdomains are what the business does; bounded contexts are how we've chosen to model it. I'd aim for one-to-one, but I'd split a context if its language diverges internally, and I'd merge two if keeping them apart costs more integration than it saves."*

---

## Concept 18 — Language heuristics for finding boundaries

Module 21's Concept 53 gave the core diagnostic: find the words that mean different things to different people. Here it is as a systematic technique, because language is the most reliable boundary signal available.

**Heuristic 1 — Polysemes: the same word, different meanings.** "Policy" to underwriting is a risk decision with terms and exclusions; to billing it's a premium schedule with payment dates; to claims it's a coverage definition to check a claim against. Each meaning is a candidate context. The tell in conversation is qualification: experts say "the *billing* policy" or "well, in underwriting terms…" — the qualifier is the context name.

**Heuristic 2 — Synonyms: different words, same thing.** When sales says "prospect," marketing says "lead," and onboarding says "applicant," they may be talking about the same person at different lifecycle stages — which often means a *handoff* between contexts, with a pivotal event at the seam ("ApplicationSubmitted" turns a lead into an applicant).

**Heuristic 3 — The attribute test.** List the attributes each group of experts cares about for a concept. If the sets barely overlap — Sales cares about pipeline stage and deal size, Billing about VAT number and payment terms, Support about ticket history and entitlement tier — the concept belongs to multiple contexts, each owning its slice. Module 21's Concept 41 ("three Customer classes") is the direct consequence.

**Heuristic 4 — Rule conflict.** If two groups of experts state rules about the same concept that can't both be true in one model — "a customer can have only one active subscription" vs. "a customer can have one subscription per product line" — you're usually looking at two contexts with two definitions of "customer" or "subscription."

**Heuristic 5 — Lifecycle divergence.** When one group's version of a concept is created, changes and dies on a completely different schedule from another's — a *product* in the catalogue lives for years, the *product* in a shopping cart lives for minutes — the lifecycles are telling you the models are different.

A practical technique that makes these heuristics concrete: in a workshop, write each important noun on a card and ask each group of experts to write their definition on the back. The cards where definitions diverge are the boundaries.

---

## Concept 19 — Other boundary heuristics, and the canvas

Language is the primary signal. Five other heuristics corroborate it:

**Pivotal events.** In an EventStorming timeline (Concept 35), some events mark a transition where responsibility, language or pace changes — `OrderPlaced`, `ClaimSubmitted`, `ShipmentDispatched`. Brandolini calls them pivotal events. Boundaries often sit right beside them: before `OrderPlaced`, you're in checkout's world of baskets and validation; after, you're in fulfilment's world of picks and shipments.

**Business decisions.** Group the decisions the business makes — "approve this credit line," "set this price," "assign this driver" — and ask which decisions share data and experts. A context should own a coherent set of decisions; one that owns nothing but data-copying is a smell.

**Rate and reason of change.** Parts of the business that change for different reasons, driven by different stakeholders, on different cadences, belong in different contexts (Module 21, Concept 54). The tax rules change when legislation changes; the promotions change weekly when marketing wants.

**Ownership and people.** Which experts, which department, which team? A boundary that cuts across a single department's knowledge will be fought; one that follows it will be defended (Conway, Module 21 Concept 9).

**Swimlanes in the process.** When a workflow naturally has lanes that proceed independently after a handoff — payment and fulfilment proceed in parallel after an order is placed — those lanes are often separate contexts.

**The Bounded Context Canvas** (DDD Crew, updated May 2026) turns all this into a one-page design artifact for a single proposed context. Its sections, which double as a checklist:

- **Name** and **Purpose** — one sentence on why the context exists, in business terms.
- **Strategic classification** — domain type (core/supporting/generic), business model role (revenue, engagement, compliance, cost reduction), and evolution stage (Concept 13).
- **Domain roles** — what kind of context it is (e.g. execution, analysis, gateway, specification).
- **Inbound communication** — which messages (commands, queries, events) arrive, from whom.
- **Outbound communication** — which messages leave, to whom.
- **Ubiquitous language** — the key terms and their definitions *in this context*.
- **Business decisions** — the rules and policies this context owns.
- **Assumptions, verification metrics, open questions.**

The canvas's value in an interview or design review is that it forces the boundary to justify itself: a context whose "business decisions" section is empty and whose inbound and outbound sections are enormous is a pass-through that probably shouldn't exist.

---

## Concept 20 — How big is a bounded context?

There's no line-count answer, but there are constraints that bound it from both sides.

**Upper bound — one team must be able to own it.** A context is a unit of model consistency, and consistency requires that the people changing the model agree on it. That's feasible for one team; it breaks down across several, because the model starts accommodating each team's needs and the language diverges. The rule of thumb: **a bounded context should be owned by exactly one team; a team may own several contexts.** Team Topologies' cognitive-load constraint (Module 21, Concept 10) sets the practical ceiling.

**Upper bound — one language.** If the context's own glossary develops terms with two meanings, it has grown past its boundary.

**Lower bound — no smaller than an aggregate.** Microsoft's Azure Architecture Center states the rule for services crisply, and it applies to contexts as well: **no smaller than an aggregate, no larger than a bounded context.** A context smaller than an aggregate would split a consistency boundary across models, which means cross-context transactions for a single invariant — the worst of both worlds.

**Lower bound — coherent decisions.** A context should own a meaningful set of business decisions. Contexts that own a single entity and do nothing but CRUD (the entity-service trap, Module 21 Concept 55) are too small; the behaviour that matters is happening in the gaps between them.

In practice, a healthy context typically contains a handful of aggregates, a few dozen domain concepts in its glossary, and a clear list of business decisions — but the constraint to state in an interview is the pair above: *"no bigger than one team and one language; no smaller than the invariants it protects."*

---

## Concept 21 — Bounded contexts vs. modules vs. microservices

Three terms that are routinely conflated, and Module 21's two axes (modularity and distribution) untangle them.

- A **bounded context** is a **model** boundary: the scope of one consistent model and language. It's a *design* concept.
- A **module** (in the modular-monolith sense) is a **code** boundary: hidden internals, explicit contract, owned schema. It's a *compile-time* concept.
- A **microservice** is a **deployment** boundary: independently deployable, owns its data. It's a *runtime* concept.

The healthy mapping is:

- **One bounded context → one module** in a modular monolith. This is the default, and it's exactly what Module 21 Part C builds.
- **One bounded context → one service** (or occasionally a small cluster of services) when a context is extracted for a Module 21 trigger — deploy contention, scale asymmetry, compliance.
- **One bounded context → several services** is legitimate when a context has genuinely different runtime needs internally (a command-processing service and a read-model projection, say), *provided the model is still owned by one team and one language*. The services share a model, so they're deployed by the same team and versioned together.

The unhealthy mappings:

- **One service → several contexts.** A service whose code contains two models with conflicting meanings for the same term is a distributed big ball of mud.
- **One context → many teams' services.** A single model split across services owned by different teams guarantees language drift and coordinated releases.
- **Service per entity.** Module 21's entity-service trap — decomposition by noun instead of by context.

The sentence that shows you know the difference: *"A bounded context is a model boundary; whether it's a module or a service is a deployment decision I'd make separately — usually a module first, extracted when there's a specific trigger."*

---

## Concept 22 — Context maps

A **context map** documents the bounded contexts in a system and **the relationships between them** — which one depends on which, how their models are translated, and what kind of organizational relationship the teams have. Evans placed it at the centre of strategic design, and it's the single most useful diagram an architect can produce for a large system, because it captures both the technical integration and the *politics* that constrain it.

The map carries two kinds of information:

**1. Team relationships** — how the *teams* depend on each other (the DDD Crew's context-mapping repository frames these as three categories):

| Relationship | Meaning |
|---|---|
| **Mutually dependent** | Both teams must succeed together; changes are coordinated (partnership, shared kernel) |
| **Upstream/downstream** | One team's output is the other's input; upstream's decisions affect downstream, not vice versa (customer–supplier, conformist, ACL, OHS/PL) |
| **Free** | Changes in one context don't affect the other (separate ways) |

**2. Integration patterns** — how the *models* relate at the seam. Evans' original patterns plus two added later (partnership and big ball of mud, which appear in his 2015 *DDD Reference* along with domain events):

| Pattern | One line |
|---|---|
| Partnership | Two teams align and release together |
| Shared kernel | A small subset of the model is shared and co-owned |
| Customer–supplier | Upstream plans to meet downstream's needs |
| Conformist | Downstream adopts upstream's model as-is |
| Anticorruption layer (ACL) | Downstream translates upstream's model into its own |
| Open host service (OHS) | Upstream exposes a protocol designed for many consumers |
| Published language (PL) | A documented, shared exchange format |
| Separate ways | No integration at all |
| Big ball of mud | A marked area of unstructured legacy |

On a map, relationships are drawn as lines annotated **U** (upstream) and **D** (downstream), with the pattern at each end — e.g. `Catalogue [U, OHS/PL] → [D, ACL] Pricing`. Both ends matter: the upstream's *posture* (does it offer a published protocol?) and the downstream's *defence* (does it translate, or conform?).

The two practical rules for drawing one:

1. **Map what *is*, not what you wish.** The first map should document the current integration honestly, including the shared database nobody admits to and the team that conforms because it has no choice. A map of the aspirational architecture drawn over a reality nobody has examined is worse than no map.
2. **Every line is a cost.** Each relationship implies coordination, translation or coupling. The map is how you see where that cost concentrates — usually around one or two contexts that everyone depends on.

---

## Concept 23 — Partnership

**Two contexts whose teams succeed or fail together**, so they coordinate planning, share integration tests and release in step when the interface changes.

When it's right: two contexts that are genuinely co-dependent for a period — a new product launch where the ordering and fulfilment teams are both building towards the same deadline and the interface between them is still evolving daily.

What it costs: coordination. Joint planning, synchronized releases, shared ownership of the interface. The two teams effectively behave as one for integration purposes.

What to watch: partnership is a relationship that should usually be **temporary**. Once the interface stabilizes, the relationship should settle into customer–supplier or OHS/PL, which decouple release cycles. A permanent partnership is two teams that should perhaps be one, or a boundary that's in the wrong place (Module 21, Concept 63's "lockstep deploys" symptom).

---

## Concept 24 — Shared kernel

**A small, explicitly defined subset of the domain model that two (rarely more) contexts share and jointly own** — typically as a shared library, with changes requiring agreement from both teams and passing both teams' tests.

When it's right: when two contexts genuinely share a small, stable, high-value concept, and duplicating it would be more error-prone than sharing it — a `Money` type with the company's rounding rules, a `PolicyNumber` format with a check-digit algorithm, a set of regulatory classification codes.

What it costs: it's the tightest integration short of merging the contexts. Every change is a two-team negotiation. It couples the release cycles of both contexts for anything touching the kernel.

What to watch — and this is Module 21's Concept 42 stated as a DDD pattern: **the kernel must stay tiny.** The failure mode is universal: the kernel becomes the place where anything shared-ish goes, grows to hundreds of types, and every context depends on it, reintroducing exactly the coupling the contexts were meant to remove. The test that keeps it honest: *if a change to the shared kernel could require a business conversation, it doesn't belong in the shared kernel.* Identifiers, money, dates, units, base types — yes. `Customer`, `Order`, `PricingRule` — never.

---

## Concept 25 — Customer–supplier

**An upstream (supplier) context whose team plans to meet the needs of a downstream (customer) context**, with the downstream's requirements represented in the upstream's planning.

When it's right: the most common healthy upstream/downstream relationship inside one organization. The catalogue team supplies product data to the pricing and search teams; the downstreams tell catalogue what they need; catalogue prioritizes those needs alongside its own.

What makes it work: the downstream has **negotiating power** — its needs appear on the upstream's backlog, and there are shared acceptance tests (Module 21's consumer-driven contract tests are the modern mechanism: the downstream writes the contract, the upstream's build verifies it).

What to watch: customer–supplier is a *political* arrangement as much as a technical one. If the upstream team's management doesn't actually prioritize downstream needs, the relationship degrades into conformist (Concept 26) whatever the diagram says. That's worth diagnosing honestly in a brownfield round: *"The map says customer–supplier; the backlog says conformist."*

---

## Concept 26 — Conformist

**The downstream team adopts the upstream's model wholesale**, with no translation — it uses the upstream's concepts, names and structures directly.

When it's right:

- The upstream model is **good** — well designed, stable, and close enough to what the downstream needs.
- The downstream is a **supporting** subdomain where the cost of translation isn't worth it.
- The downstream has **no leverage** — the upstream is an external vendor, a regulator's format, or a large internal platform that won't change for you — and the upstream model is acceptable.

What it costs: nothing up front, which is why it's so common. The deferred cost is that the upstream's model — its concepts, its naming, its quirks — now shapes the downstream's code, and **every upstream change ripples into the downstream**. Conforming to a poor upstream model is how bad models propagate across an organization.

The judgment: conformist is a perfectly good choice for a supporting context integrating with a well-designed upstream. It's a bad choice for a **core** context, because it means your competitive advantage is being expressed in someone else's language. That distinction — conform in the supporting areas, translate in the core — is the heuristic to state.

---

## Concept 27 — Anticorruption layer

**A translation layer, owned by the downstream, that converts the upstream's model into the downstream's own model at the boundary**, so that the upstream's concepts never leak into the downstream's domain.

When it's right: whenever the downstream's model matters more than the cost of translating — which is always true for a **core** context consuming anything, and for any context consuming a **legacy system**, a **third-party API** or a **poorly designed upstream**.

What it contains, in the classic structure: a **facade** over the upstream's interface (simplifying it to what's needed), **adapters** that call it, and **translators** that map its data and concepts to the downstream's types. Module 20's Concept 44 described the port-and-adapter mechanics; here's the DDD purpose, with the translation doing real semantic work rather than field copying:

```csharp
// Claims context (downstream, core). The port speaks Claims' language only.
public interface IPolicyCoverage
{
    Task<CoverageDecision> CheckCoverageAsync(
        PolicyNumber policy, LossEvent loss, CancellationToken ct);
}

public abstract record CoverageDecision
{
    public sealed record Covered(Money Deductible, Money Limit) : CoverageDecision;
    public sealed record Excluded(ExclusionReason Reason) : CoverageDecision;
    public sealed record PolicyNotInForce(DateOnly LapsedOn) : CoverageDecision;
}

// Infrastructure: the ACL over the legacy Policy Administration System (upstream).
internal sealed class LegacyPasCoverageAdapter(PasSoapClient pas, TimeProvider clock)
    : IPolicyCoverage
{
    public async Task<CoverageDecision> CheckCoverageAsync(
        PolicyNumber policy, LossEvent loss, CancellationToken ct)
    {
        // Upstream model: a flat record with status codes, Y/N flags and amounts in cents.
        var pasRecord = await pas.GetPolicyAsync(policy.Value.PadLeft(12, '0'), ct);

        // Translation is semantic, not just structural:
        // PAS status "L" and "C" both mean "not in force" in Claims' language.
        if (pasRecord.StatusCode is "L" or "C")
            return new CoverageDecision.PolicyNotInForce(
                DateOnly.ParseExact(pasRecord.StatusDate, "yyyyMMdd"));

        // PAS encodes exclusions as a comma-separated list of 3-letter codes;
        // Claims models them as a closed set of reasons it can reason about.
        var exclusion = ExclusionTranslator.FindApplicable(pasRecord.ExclCodes, loss.Peril);
        if (exclusion is not null)
            return new CoverageDecision.Excluded(exclusion.Value);

        return new CoverageDecision.Covered(
            Deductible: Money.FromMinorUnits(pasRecord.DedAmtCents, pasRecord.CurrCd),
            Limit:      Money.FromMinorUnits(pasRecord.LimitAmtCents, pasRecord.CurrCd));
    }
}
```

The points a reviewer looks for:

- **The port is defined in the downstream's language** — `CoverageDecision`, `LossEvent`, `ExclusionReason` — and nothing about SOAP, status codes or cents appears in it.
- **The translation makes semantic decisions** ("L" and "C" both mean *not in force*). That knowledge — how the legacy system's concepts map to yours — is exactly what should be concentrated in one place, and it is valuable domain knowledge in its own right. Test it thoroughly.
- **The ACL is owned by the downstream.** It's the downstream's defence; the upstream doesn't know it exists.

What it costs: code to write and maintain, a translation to test, sometimes a performance cost, and — the real cost — **the translation must be kept in sync with the upstream's evolution**. It's worth it where the downstream model matters; it's overhead where it doesn't.

Two strategic uses beyond "wrapping a legacy system": an ACL is how you **isolate a vendor** so the generic subdomain can be swapped later (Concept 13), and it's how you **protect a new model growing inside a legacy system** (the bubble context, Concept 30).

---

## Concept 28 — Open host service and published language

The upstream-side counterparts to the ACL.

**Open host service (OHS):** the upstream defines a **protocol designed for consumption by many downstreams** — a deliberately stable, documented API or event stream, separate from its internal model. Instead of each downstream integrating with the upstream's internals (or the upstream building a custom integration per consumer), the upstream offers one well-designed service.

**Published language (PL):** a **well-documented, shared exchange format** in which the translation between contexts is expressed — often used by an OHS. It can be industry-standard (ISO 20022 for payments, HL7 FHIR for healthcare, ACORD for insurance, UN/EDIFACT for logistics) or organization-specific (a versioned set of event schemas in a contracts package).

The key idea, which is also Module 20's domain-vs-integration-event distinction and Module 21's contracts assembly: **the published language is not the internal model.** The upstream keeps its internal model free to change; the published language changes slowly, additively and with versioning:

```csharp
// Ordering context — internal domain event (rich, uses domain types, can change freely)
public sealed record PlacedLine(Sku Sku, Quantity Quantity, Money UnitPrice);   // immutable snapshot, not the entity

public sealed record OrderPlaced(OrderId OrderId, CustomerId CustomerId,
    Money Total, IReadOnlyList<PlacedLine> Lines, DateTimeOffset PlacedAt) : IDomainEvent
{
    public DateTimeOffset OccurredAt => PlacedAt;
}

// Ordering.Contracts — the published language (flat, primitive, versioned, additive-only)
namespace Ordering.Contracts.V1;

/// <summary>Published when a customer commits to an order. Stable contract; additive changes only.</summary>
public sealed record OrderPlacedV1(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    string Currency,           // ISO 4217
    int LineCount,
    DateTimeOffset PlacedAtUtc);
```

When OHS/PL is right: an upstream with **many consumers** (catalogue, identity, customer master data), where per-consumer integration would be unmanageable, and where consumers need stability more than they need the upstream's internal richness.

What it costs: the upstream must design, document, version and support the protocol — a real ongoing commitment (Module 21 Concept 27's versioning tax, paid deliberately). In return, the upstream's *internal* model becomes free to change, and consumers get a contract they can rely on.

The combination to name in an interview: **OHS/PL upstream, ACL downstream** — the upstream publishes a stable language, and each consumer that cares about its own model translates that language into its own concepts. That's the most decoupled integration two contexts can have while still exchanging data.

---

## Concept 29 — Separate ways

**Decide not to integrate.** Two contexts that could share data or functionality instead each do their own thing, and the duplication is accepted.

When it's right: when the integration costs more than it saves. A marketing team that needs a list of product categories for campaign tagging may be better off maintaining its own short list than integrating with the catalogue's full taxonomy service. Two contexts that each need a small "business days" calculator might each write their own twenty lines rather than depend on a shared component.

What it costs: duplicated effort and the possibility of divergence. For low-value, low-volatility concerns, that's cheap.

Why it's worth naming in an interview: it's the pattern candidates forget exists, and invoking it shows you understand that **integration is a cost, not a default**. The architect's job isn't to connect everything; it's to connect what's worth connecting. *"Honestly, I wouldn't integrate these at all — the overlap is small and stable, and a dependency would cost more than the duplication."*

---

## Concept 30 — The big ball of mud and the bubble context

**Big ball of mud** — the term comes from Brian Foote and Joseph Yoder's 1997 paper, and Evans added it to the context-mapping vocabulary in the 2015 *DDD Reference* — is a **context-map pattern for honestly marking an area with no coherent model**: mixed concepts, inconsistent language, no clear boundaries. Most brownfield systems contain at least one.

The pattern's advice is modest and correct: **draw a boundary around the mess, and don't try to apply sophisticated modelling inside it.** Accept that its model is whatever it is, and protect everything else from it.

Evans' paper *Getting Started with DDD When Surrounded by Legacy Systems* (linked from domainlanguage.com, with an implementation by DDD Hamburg on GitHub) describes four strategies for introducing a clean model into that reality — and they're the DDD-specific half of Module 21's strangler-fig discussion:

1. **Bubble context.** Create a small new context with a clean model for one specific, valuable capability, and protect it with an **ACL** that translates to and from the legacy system. The bubble reads from and writes back to the legacy database through the ACL; it has no data of its own. Cheap to start, and it lets the team practise modelling without a migration.
2. **Autonomous bubble.** The bubble gets its **own data store**, synchronized with the legacy system asynchronously (via events or change capture through a synchronizing ACL). It can now operate independently — and survive legacy downtime — at the cost of eventual consistency and a synchronization component.
3. **Exposing legacy assets as services.** Wrap legacy capabilities behind an OHS so new contexts can consume them through a published language rather than through the legacy internals.
4. **Expanding a bubble.** Grow the bubble's scope over time as it proves itself — the DDD version of the strangler fig.

The strategic point: a bubble context is a **low-risk way to start DDD in a legacy environment** — one capability, one team, an ACL, no big-bang rewrite. In a brownfield interview, it's a strong opening move: *"I wouldn't try to model the whole system. I'd pick the one capability where the business most needs change, build it as a bubble context with a clean model behind an ACL, and let it prove itself before expanding."*

---

## Concept 31 — Context maps as power maps

The part of context mapping that's easiest to miss and most valuable in an architect round: **the upstream/downstream relationship is organizational power made visible.**

An upstream team can change its model and the downstream must cope. A downstream team can only ask. So:

- **Conformist relationships usually indicate a power imbalance**, not a design choice. The downstream conforms because it can't get the upstream to change.
- **Customer–supplier relationships only exist where the downstream has leverage** — a shared manager, a formal agreement, a service-level objective.
- **ACLs are what a powerless downstream builds** to protect itself when it can't influence the upstream.
- **A context that everyone is downstream of** (a customer master, an identity service, the legacy ERP) holds disproportionate power and is where the organization's bottlenecks will form.

This matters for two reasons. First, **a context map predicts where projects will stall.** If your core context is conformist to an upstream that doesn't prioritize your needs, your core domain's evolution is controlled by another team's backlog — a strategic risk worth raising explicitly. Second, **the fix is often organizational, not technical.** Moving a relationship from conformist to customer–supplier requires a management conversation, not a code change. Knowing that — and being willing to say so — is precisely what separates the architect track from the senior IC track (Module 2).

A useful exercise before any large redesign: draw the current context map with the team relationships annotated honestly, then ask "which of these relationships would we *choose* today?" The gap between the current map and the chosen one is your integration and organizational roadmap.

---

## Concept 32 — Balanced coupling: why the patterns work

Vlad Khononov's *Balancing Coupling in Software Design* (2024) gives the context-mapping patterns a first-principles explanation, and it's worth knowing because it lets you *reason* about integration choices instead of pattern-matching.

Coupling has three dimensions:

**1. Integration strength** — how much *knowledge* two components share. From strongest to weakest:

| Level | What's shared | Example |
|---|---|---|
| **Intrusive** | Implementation details, private internals | Reading another context's tables; using its internal types via reflection |
| **Functional** | Knowledge of business *requirements* — both must implement the same rule | Two contexts each implementing "a customer is VIP if…" and needing to stay in sync |
| **Model** | The *domain model* itself | Downstream uses upstream's entities directly (conformist) |
| **Contract** | Only an integration-specific contract | Downstream consumes a published language through an ACL |

**2. Distance** — how far apart the components are: same class, same module, same service, different services, different teams, different organizations. Greater distance makes coordinated change more expensive (and ownership distance counts as much as physical distance).

**3. Volatility** — how often the shared knowledge changes. Core subdomains are volatile (they evolve as the business competes); generic ones are stable.

The balance heuristic, from coupling.dev: **BALANCE = (STRENGTH XOR DISTANCE) OR NOT VOLATILITY.** In words:

- **High strength is fine at low distance** — sharing a model *inside* a context (or inside an aggregate) is cohesion, not a problem.
- **Low strength is fine at high distance** — contract coupling between distant contexts is what healthy integration looks like.
- **High strength at high distance is the pain** — sharing a model across teams or services, or two services reading the same tables. It's "global complexity": every change requires coordinated effort far away.
- **Low strength at low distance is also a smell** — unrelated things co-located ("local complexity").
- **Low volatility forgives imbalance** — conforming to a stable, unchanging upstream (a regulator's format, a legacy system nobody will change) is cheap *because it never changes*.

Now the context-map patterns read as coupling moves:

- **ACL** and **published language** convert **model coupling** into **contract coupling**, which is exactly what makes large distance affordable.
- **Conformist** accepts **model coupling across distance** — fine only when the upstream's volatility is low.
- **Shared kernel** is **model coupling at team distance**, which is why it must be tiny and stable.
- **Separate ways** removes coupling entirely where the shared knowledge isn't worth it.
- **Reading another context's database** is **intrusive coupling at distance** — the worst combination there is, which is why Module 21 made it a permission error.

The interview-grade sentence: *"I'd use an ACL here because the core context is volatile and the upstream is owned by another team — sharing its model across that distance would be the most expensive combination of coupling there is. Converting it to a contract is what makes the distance affordable."*

---

## Concept 33 — Integration between contexts

How bounded contexts actually exchange information, in order of how loosely they couple:

**1. Events in a published language (asynchronous).** The upstream publishes integration events describing what happened; downstreams subscribe and update their own models. No temporal coupling, contract-level strength. The default for **state changes** (Module 21, Concept 39). The translation from a context's internal domain events into its published integration events is Concept 83.

**2. Queries/commands against an open host service (synchronous).** Downstream calls a stable API. Use when the downstream needs an answer *now* to make a decision (check credit before accepting the order). Contract coupling, but temporal coupling too — so it carries availability arithmetic (Module 21, Concept 21).

**3. Shared kernel (compile-time).** A tiny shared library. Model coupling, only acceptable at low volatility.

**4. Shared database.** Intrusive coupling. Not an integration pattern between bounded contexts; it's the absence of a boundary.

Three rules that keep contexts honest:

- **Never share an aggregate across contexts.** If two contexts both need to *change* the same aggregate, the aggregate is in the wrong context, or it's really two aggregates with different responsibilities (one per context), linked by ID and events.
- **Each fact has one owning context.** Customers owns the email address; Billing owns the VAT number. Other contexts hold replicated copies as *caches of someone else's truth* (Module 21, Concept 41).
- **Translate at the boundary, not in the middle.** Incoming integration events are translated into the receiving context's language *at the edge* — the handler is part of that context's ACL — so the rest of the context never sees the upstream's vocabulary.

---

## Concept 34 — AI components as bounded contexts

A current-state concept, and increasingly an interview question: *where do LLM-based components fit in a DDD design?*

Eric Evans' own answer, developed across his Explore DDD 2024 keynote, his January 2026 article *Context Mapping with an AI-based Component*, and his DDD Europe 2026 opening keynote, is that **an LLM is itself a bounded context**:

- It has **its own language** — natural-language prompts and outputs — that doesn't match your ubiquitous language.
- It has **its own consistency model** — probabilistic rather than deterministic. The same input can produce different outputs, and outputs can be fluent and wrong.
- It has **its own interface contracts**, which change when the model or prompt changes.

The consequence is a direct application of the context-mapping patterns:

- **An anticorruption layer is essential** at the seam between a deterministic domain model and a probabilistic component. The ACL constrains inputs (structured prompts built from domain concepts), validates and parses outputs into domain types ("parse, don't validate," Concept 48), and decides what happens when the output doesn't parse or fails a domain rule. An LLM should never write directly into an aggregate; its output is a *proposal* that the domain model accepts or rejects against its invariants.
- **The ubiquitous language becomes the contract** the prompt must respect. A bounded context's glossary is exactly the material an LLM component needs in order to use terms consistently — and exactly the material it will otherwise blur.
- **The component's placement on the context map matters.** An LLM that *classifies* incoming claims is upstream of the claims model and should be treated like any unreliable external supplier. An LLM that *drafts* customer letters is downstream of it and consumes a published language.

Evans' 2026 keynote framed the broader question honestly: whether AI amplifies DDD or replaces parts of it is unknown, but his working hypothesis is that **domain models, bounded contexts and language still matter — the models will look different.** He has also described experiments using LLMs to extract domain-specific vocabulary from code and compare it across contexts, as a possible way to *measure* context boundaries more objectively.

The interview answer: *"I'd treat the model as another bounded context with an unreliable, probabilistic contract. It gets an anticorruption layer that builds its input from our domain concepts and parses its output back into our types, and its output is a proposal that the aggregate validates against the same invariants as anything else. The language work matters more, not less — it's what keeps the component using our terms precisely."*

---

# Part D — Discovery: turning business knowledge into boundaries

Strategic design depends on knowledge the engineering team doesn't have yet. Part D is the toolkit for getting it out of domain experts' heads efficiently and in a form that maps onto contexts and aggregates. Module 21 (Concept 56) introduced EventStorming as a boundary-discovery tool; here it's taken to the level of detail that lets you say you've *facilitated* one, plus the techniques that complement it.

---

## Concept 35 — EventStorming, precisely

Alberto Brandolini's **EventStorming** is a collaborative workshop technique: domain experts and engineers model a business process together on a very long wall using coloured sticky notes, starting from **domain events** in the past tense and working outward. Its power is that it's cheap, fast, visual and inclusive — it gets knowledge that is distributed across a dozen people onto one wall in a few hours.

**Three formats, used at different stages:**

| Format | Scope | Participants | Output |
|---|---|---|---|
| **Big Picture** | The whole business line, end to end | Many — all the relevant experts and stakeholders | The overall flow, pivotal events, hotspots, candidate boundaries, organizational pain points |
| **Process Level** | One business process in detail | The people who run and build that process | A complete, consistent process with commands, policies, read models and actors |
| **Software Design (Design Level)** | One area, in enough detail to code | The team that will build it, with experts | Aggregates (constraints), commands, events — the tactical design |

**The notation** (as documented in the DDD Crew's EventStorming glossary and cheat sheet — the colours matter because they encode the grammar):

| Sticky | Colour | Meaning |
|---|---|---|
| **Domain event** | Orange | Something that happened that the business cares about, in the past tense: *Claim Submitted* |
| **Command** | Blue | An intention or decision that triggers events: *Submit Claim* |
| **Actor / person** | Small yellow | Who issues the command: *Policyholder*, *Adjuster* |
| **Constraint** (formerly "aggregate") | Large yellow | The thing that accepts or rejects the command based on rules: *Claim* |
| **Policy** | Lilac | A reactive rule: "*whenever* a claim is submitted, *then* run fraud scoring" |
| **Read model / information** | Green | What an actor needs to see to make the decision |
| **External system** | Pink (large) | Something outside the boundary: *Payment Provider*, *Legacy PAS* |
| **Hotspot** | Red / bright pink | A question, conflict, risk or pain point |

The detail that signals you've actually facilitated one: **the glossary now calls the big yellow sticky a "constraint," and describes "aggregate" as a legacy word in EventStorming** — because using "aggregate" with business stakeholders derails the conversation into jargon. In the room you ask "*what decides whether this command is allowed?*", not "*what's the aggregate?*" The engineers can translate afterwards.

**How a Big Picture session runs**, in outline: chaotic exploration (everyone writes events, in parallel, without discussion — speed matters more than order); enforcing the timeline (arrange events left to right, merge duplicates, argue about order — the arguments are the value); marking **pivotal events** and **swimlanes**; flagging **hotspots**; and finally drawing candidate boundaries where the language, the actors or the pace changes.

**What goes wrong:**

- **Engineers dominate.** The session becomes a design meeting in developer vocabulary. The facilitator's job is to keep the experts writing.
- **Premature solutioning.** Someone starts drawing database tables. Park it on a hotspot.
- **The wrong people.** Without the experts who actually run the process, you get the process as management imagines it. Include the people who handle the exceptions.
- **Treating the wall as the deliverable.** The output is shared understanding plus a set of decisions and questions; photograph the wall, then turn it into context maps, canvases and a backlog.

---

## Concept 36 — From sticky notes to aggregates

Design-level EventStorming is where strategic discovery turns into tactical design, and the mechanical step from the wall to code is worth being able to describe.

The grammar of the wall is: **an actor, looking at a read model, issues a command; a constraint (aggregate) accepts or rejects it based on its rules; if accepted, one or more events occur; policies react to events and issue further commands.**

```
[Adjuster] --reads--> (Claim details, policy coverage)
     |
     v
 [Accept Liability]  --->  [[ Claim ]]  --->  (Liability Accepted)  --->  <Whenever liability is accepted,
   (command, blue)       (constraint,        (event, orange)              schedule payment>  (policy, lilac)
                          large yellow)                                         |
                                                                                v
                                                                        [Schedule Payment] ---> [[ Payment ]] ...
```

Reading aggregates off the wall:

1. **Cluster commands and events around the constraint that decides them.** Every blue command needs something to accept or reject it; if the same rules decide several commands, they share a constraint. `Submit Claim`, `Add Evidence`, `Accept Liability`, `Reject Claim`, `Reopen Claim` all go through rules about the claim's state — one constraint, `Claim`.
2. **Ask what information the constraint needs to decide.** That's the aggregate's state — and it's usually *less* than the data model suggests. The `Claim` constraint needs the claim's status, the coverage decision, the amounts and the evidence list; it doesn't need the policyholder's address.
3. **Look for policies that cross constraints.** A lilac policy between two yellow constraints ("whenever liability is accepted, schedule payment") marks **eventual consistency** between two aggregates — they don't need to be in one transaction. If the business insists the two must happen atomically, that's a signal to reconsider whether they're one aggregate (Concept 58).
4. **Look for hotspots on constraints.** "What if two adjusters accept liability simultaneously?" is a concurrency question the model must answer (Concept 69).
5. **Name everything in the experts' words** — the stickies already are, which is the point.

The result isn't a finished design, but it is a set of **aggregate candidates with their commands, events and invariants** — which is precisely the input the Aggregate Design Canvas (Concept 76) wants.

---

## Concept 37 — Other discovery techniques

EventStorming is the most widely used, but not always the best fit. Three complementary techniques, and when each wins:

**Domain Storytelling** (Stefan Hofer & Henning Schwentner). Domain experts *tell stories* about concrete scenarios while a moderator draws them using a small pictographic language: **actors** (people and systems), **work objects** (documents, things, information) and numbered **activities** connecting them — "1. The customer *sends* the claim form *to* the call centre agent. 2. The agent *enters* the claim *into* the claims system…" Each story is concrete (one specific case, not a generalization). **Wins when:** stakeholders are non-technical and uncomfortable with abstract notations, when the process is document- or conversation-heavy, and for *to-be* design ("tell me how it *should* work"). The free Egon.io modelling tool supports it, and a story's sentences map almost directly onto ubiquitous language.

**Example Mapping** (Matt Wynne, Cucumber). A short, focused session per user story: a yellow card for the story, blue cards for the **rules**, green cards for **examples** that illustrate each rule, red cards for **questions**. **Wins when:** you need to nail down the rules of one specific behaviour precisely — it's the natural step between a design-level EventStorming and the first tests. The green examples become given/when/then tests on the aggregate (Concept 93) almost verbatim.

**Event Modeling** (Adam Dymitruk). A blueprint-style technique that lays out a system as a timeline of **UI wireframes, commands, events and views (read models)**, organized into vertical **slices** — each slice being one command or one view, independently implementable. **Wins when:** you want a specification the team can build from directly, with an obvious mapping to CQRS (Module 23) and event sourcing (Module 24), and a clear unit of work (the slice) for planning.

A practical sequencing that works: **Big Picture EventStorming** to find contexts and hotspots → **Domain Storytelling** for the processes where experts struggle with sticky notes → **Design-level EventStorming or Event Modeling** within a chosen context → **Example Mapping** per behaviour to get rules and examples → tests.

---

## Concept 38 — The DDD Starter Modelling Process

The DDD Crew's *DDD Starter Modelling Process* (free on GitHub, CC BY-SA 4.0, updated May 2026) is the best available answer to "we want to do DDD — where do we start?" It sequences the techniques above into eight steps, each with a recommended starting tool:

| Step | Purpose | Suggested starting technique |
|---|---|---|
| **1. Understand** | Align on the business model, user needs and goals | Business Model Canvas, Impact Mapping, User Story Mapping |
| **2. Discover** | Discover the domain visually and collaboratively | Big Picture EventStorming, Domain Storytelling |
| **3. Decompose** | Break the domain into subdomains | EventStorming with subdomains, business capability modelling |
| **4. Strategize** | Identify the core and decide on investment | Core Domain Charts, Wardley Mapping |
| **5. Connect** | Design how subdomains/contexts interact end to end | Domain Message Flow Modelling |
| **6. Organise** | Align teams with contexts | Team Topologies, context maps |
| **7. Define** | Define each bounded context's responsibilities and language | Bounded Context Canvas |
| **8. Code** | Turn the design into a domain model | Aggregate Design Canvas, design-level EventStorming |

The authors are explicit that it's not a waterfall: real projects jump between steps as insight arrives, and the order can be changed (starting with a visualization of the existing architecture is common in brownfield work). Its value in an interview is that it gives you a **defensible, named process** when asked "how would you introduce DDD to this organization?" — which is much stronger than listing patterns. SAP's open *Curated Resources for DDD* repository includes a **DDD kata** that walks the process end to end on a sample problem; it's an excellent practice exercise.

---

## Concept 39 — Validating boundaries with scenarios

Proposed context boundaries are hypotheses. The cheapest test is to **walk real business scenarios across them and count the messages.**

**Domain Message Flow Modelling** (DDD Crew) does this visually: for one concrete scenario ("a customer changes their delivery address after the order has been placed but before it ships"), draw the bounded contexts as boxes and draw every **command, event and query** that crosses between them, numbered in order.

What the diagram reveals:

- **Chatty boundaries.** A single scenario requiring ten back-and-forth messages between two contexts means their responsibilities are tangled — the boundary probably cuts through the middle of a concept.
- **Missing owners.** A step where nobody is sure which context decides ("who checks whether the address change is still allowed?") is a responsibility that's fallen between contexts.
- **Synchronous chains.** A scenario that requires Context A to query B to query C before responding is Module 21's availability-multiplication problem, visible at design time.
- **Data shipped around for no decision.** A context forwarding data it doesn't use (pass-through coupling, Module 21 Concept 58).

The practice: pick five to ten scenarios that the business considers important *or* troublesome — including at least a couple of awkward edge cases — and run each one against the proposed boundaries. Boundaries that handle all of them with few, meaningful messages are probably right. Boundaries that make half of them chatty should be moved before any code exists, when moving them is free.

This is also the cheapest possible answer to the interview question "how would you know your bounded contexts are right?": *"I'd walk the ten most important scenarios across them and count the cross-boundary messages. If a common scenario needs a chatty conversation between two contexts, the boundary is in the wrong place, and it's much cheaper to find that on a whiteboard than in production."*

---

# Part E — Tactical design: the building blocks

With the strategic decisions made, tactical design is how you express a model *inside one bounded context* so that its rules are explicit and its invariants can't be broken by accident. Part E covers the building blocks except the aggregate, which gets Part F to itself. Most of these are cheaper in C# 14/15 than they have ever been — records, `field`, collection expressions, strongly-typed IDs, EF Core 10 complex types — and a senior answer uses the modern forms while knowing their traps.

---

## Concept 40 — Why the tactical patterns exist

Every tactical pattern exists for one purpose: **to keep the business rules in one place and make it impossible (or at least very hard) for any code path to leave the model in a state the business considers invalid.**

The contrast that makes this concrete is where a rule lives. Here's "an order can't be modified after it's been placed, and can't exceed 50 lines" in the typical anaemic form:

```csharp
// Application service — rules scattered, model is a bag of properties
public async Task AddLineAsync(Guid orderId, AddLineDto dto)
{
    var order = await _db.Orders.Include(o => o.Lines).SingleAsync(o => o.Id == orderId);
    if (order.Status != "Draft") throw new InvalidOperationException("Order is not editable");
    if (order.Lines.Count >= 50) throw new InvalidOperationException("Too many lines");
    order.Lines.Add(new OrderLine { Sku = dto.Sku, Quantity = dto.Quantity, UnitPrice = dto.UnitPrice });
    order.Total = order.Lines.Sum(l => l.Quantity * l.UnitPrice);
    await _db.SaveChangesAsync();
}
```

And the rich form:

```csharp
// Domain — the rule lives with the data it protects
public void AddLine(Sku sku, Quantity quantity, Money unitPrice)
{
    EnsureDraft();
    if (_lines.Count >= MaxLines) throw new DomainException($"An order can have at most {MaxLines} lines.");
    _lines.Add(new OrderLine(NextLineNumber(), sku, quantity, unitPrice));
    RecalculateTotal();
}
```

The anaemic version works — until the *second* use case that adds lines (bulk import, re-order from history, an admin tool) forgets one of the checks, or computes the total slightly differently. The model can't defend itself because `Status`, `Lines` and `Total` are public and settable. **The rules are only as reliable as the least careful caller.**

The rich version has two properties that matter:

1. **The rule has exactly one home.** Every path that adds a line goes through `AddLine`, so the rule is enforced everywhere by construction.
2. **The state can only change through meaningful operations.** There's no `order.Status = …` anywhere, because `Status` has no public setter. The only way to change it is `Place()`, `Cancel()`, `MarkPaid()` — operations named in the ubiquitous language, each of which checks its own preconditions.

That's "**tell, don't ask**" applied to business rules: the caller tells the object what it wants (`order.AddLine(...)`), rather than asking for its data, making the decision itself, and writing the result back.

The trade-off, stated honestly: the rich form costs more design effort and some friction with ORMs and serializers (Part H). It pays off where rules are numerous, interacting and changing — the core — and is often not worth it where they aren't (Concept 96).

---

## Concept 41 — Entities

**An entity is an object defined by its identity and its continuity over time, rather than by its attributes.**

A customer who changes their name, address and email is still the same customer. A bank account whose balance changes a thousand times is the same account. Two accounts with identical balances and owners are still two accounts. That's the test: **if two instances have exactly the same attributes, are they the same thing?** If no, it's an entity.

Properties of entities:

- **Identity is stable and unique** within its scope. It's assigned once and never changes (Concept 42).
- **They have a lifecycle.** Created, modified through meaningful operations, possibly archived or deleted. Often the lifecycle *is* the interesting part of the model (Concept 72).
- **Equality is by identity.** Two references to "account 42" are the same account, regardless of whether one copy is stale.
- **They are mutable**, but only through behaviour that preserves their invariants.

Two .NET-specific points worth knowing:

**Don't make entities records.** Records have value equality over *all* fields, which is the opposite of entity semantics, and `with` produces an untracked copy that EF will never save (Module 16, Concept 21). Entities are classes.

**Be careful overriding `Equals` on EF entities.** An identity-based `Equals` override is conceptually right, but it interacts with EF's change tracker and with hash-based collections: an entity whose ID is assigned by the database has a default ID until `SaveChanges`, so two *different* new entities compare equal and collide in a `HashSet`. The clean way out is to assign identity in the constructor (Concept 42) — then identity-based equality is safe from the moment the object exists. If IDs are database-generated, prefer comparing IDs explicitly over overriding `Equals`.

**Child entities have local identity.** An `OrderLine` inside an `Order` aggregate is an entity — two lines with the same SKU and quantity are still two lines — but its identity only needs to be unique *within its order* (line 1, line 2). Nothing outside the aggregate refers to it directly (Concept 57).

---

## Concept 42 — Identity generation: who assigns it, and when

An underrated design decision, because it affects events, idempotency, testing and database performance.

| Strategy | When the ID exists | Strengths | Costs |
|---|---|---|---|
| **Database identity** (`IDENTITY`, `SERIAL`) | After `INSERT` | Compact integer keys, familiar | The domain can't know its own ID until saved; can't raise events carrying the ID before save; can't make creation idempotent from the client |
| **HiLo / sequence blocks** (`UseHiLo()` in EF Core on SQL Server) | Before save — the app reserves blocks from a DB sequence | Integer keys known early; batching-friendly | A DB round trip per block; gaps in numbering |
| **Client-generated GUID** | At construction | Known immediately; no DB round trip; enables idempotent creation (client supplies the ID); distribution-friendly | 16-byte keys; index locality depends on the GUID version *and* the database's sort order (below) |
| **Natural key** (ISBN, IBAN, VIN) | Given by the domain | Meaningful; stable across systems | Natural keys change more often than anyone expects (people change names, companies re-issue numbers); formats vary; privacy concerns |

**The modern default for aggregate roots is a client-generated, time-ordered GUID wrapped in a strongly-typed ID**, assigned in the aggregate's factory — with one database-specific catch worth knowing cold:

- On **PostgreSQL**, `uuid` compares bytewise, so .NET 9+'s `Guid.CreateVersion7()` (timestamp in the leading bytes) inserts in near-sequential order and B-tree locality is good.
- On **SQL Server**, `uniqueidentifier` has an unusual sort order that compares the *last* byte group first. A version-7 GUID's timestamp sits in the *first* bytes, so **`Guid.CreateVersion7()` is not sequential from SQL Server's point of view** and fragments a clustered index much like a random GUID. On SQL Server, either let EF Core's default client-side `SequentialGuidValueGenerator` produce SQL-Server-ordered GUIDs, generate IDs through a small domain port whose SQL Server implementation produces that ordering, or keep a non-clustered GUID key with a clustered surrogate. Saying this unprompted is a strong signal, because the common advice ("use v7 GUIDs") is database-specific.

**Business identifiers are a separate concept from technical identity.** A claim's technical ID is a GUID; its *claim number* (`CLM-2026-004211`) is a value object generated from a sequence, printed on letters and read over the phone. Keep them distinct: the claim number can have a format, a check digit and a regional prefix without any of that leaking into foreign keys.

**Early identity is what makes creation idempotent.** If the client generates the ID and sends it with the create command, a retried request after a timeout (Module 21, Concept 22) finds the existing aggregate instead of creating a duplicate. That's the cheapest idempotency mechanism there is.

---

## Concept 43 — Value objects

**A value object is an object defined entirely by its attributes, with no conceptual identity.** Two `Money(100, "EUR")` instances are interchangeable; there's no meaningful question of "which one." The test is the inverse of the entity test: **if two instances have the same attributes, are they the same thing?** If yes, it's a value object.

The properties, each of which has a practical consequence:

- **Immutable.** A value never changes; you replace it with a different value. "Changing" an address means assigning a new `Address`. Immutability makes values safe to share, cache, use as dictionary keys and pass between threads (Module 15).
- **Value equality.** Equality compares all defining attributes.
- **Self-validating.** A value object cannot exist in an invalid state: an `EmailAddress` is a valid email address, a `Percentage` is between 0 and 100, a `DateRange` has a start before its end. Validation happens once, at construction (Concept 48).
- **Side-effect-free behaviour.** Operations return new values: `money.Add(other)` returns a new `Money`; `range.Overlaps(other)` returns a `bool`.
- **Whole value.** A value object groups attributes that only make sense together — an amount without a currency is meaningless, so `Money` holds both. Ward Cunningham's term: **Whole Value**.
- **Replaceable.** Because values have no identity, the only operation on a value-typed field is replacement, which makes reasoning about state much simpler than with entities.

The heuristic that comes out of this, and that experienced DDD practitioners repeat: **prefer value objects.** Model a concept as a value unless you have a clear reason for identity. Values are easier to test (no setup, no lifecycle), easier to reason about (no aliasing, no mutation), and they're where a surprising amount of domain logic naturally lives (Concept 46). A healthy domain model typically has several times more value objects than entities.

The interview line: *"My default is a value object. I only reach for an entity when the thing has a lifecycle I need to track through changes — and even then, most of its attributes are usually values."*

---

## Concept 44 — Value objects in C#: the shapes and the traps

C# gives you three good shapes and several traps. Module 16 covered records in depth; here's the value-object-specific version.

**Shape 1 — `sealed record` class with a private constructor and a factory.** The default for most value objects, especially any containing strings or references:

```csharp
public sealed record EmailAddress
{
    public string Value { get; private init; }   // private init: EF can materialize it; outsiders can't `with` it

    private EmailAddress(string value) => Value = value;

    public static EmailAddress Parse(string raw) =>
        TryParse(raw, out var email) ? email : throw new DomainException($"'{raw}' is not a valid email address.");

    public static bool TryParse(string? raw, [NotNullWhen(true)] out EmailAddress? email)
    {
        email = null;
        if (string.IsNullOrWhiteSpace(raw)) return false;
        var normalized = raw.Trim().ToLowerInvariant();      // normalize so equality is semantic
        if (normalized.Length > 320 || !MailAddress.TryCreate(normalized, out _)) return false;
        email = new EmailAddress(normalized);
        return true;
    }

    public override string ToString() => Value;
}
```

Records give you value equality, `GetHashCode` and immutability for free; the private constructor forces all creation through validation; normalization in the factory makes equality mean what the business means (`Alice@Example.com` and `alice@example.com` are one address).

**Shape 2 — `readonly record struct`** for small, frequently created values whose `default` is acceptable or guarded: strongly-typed IDs, `Quantity`, coordinates. No allocation, value semantics, but **`default(T)` bypasses your constructor**, so a struct can't guarantee validity the way a class with a private constructor can (Module 16, Concept 20). Use a struct only when `default` is a legitimate value, or when an analyzer (Vogen's, for example) bans default construction.

**Shape 3 — a class with explicit equality components**, the pattern in Microsoft's eShop seedwork (`ValueObject` base with `GetEqualityComponents()`). Still useful when you need custom equality — collection members, case-insensitive comparison — but records have made it largely unnecessary for simple values.

**The traps a reviewer looks for:**

**Trap 1 — Positional records skip validation, and `with` skips it again.**

```csharp
public sealed record Percentage(decimal Value);          // public ctor, public init: no validation anywhere
var p = new Percentage(150m);                             // accepted
var q = validPercentage with { Value = -3m };             // accepted — `with` calls no constructor
```

Fix it either with private accessors (shape 1), or by validating in the `init` accessor using C# 14's `field` keyword so that *every* path, including `with`, goes through the check:

```csharp
public sealed record Percentage
{
    public decimal Value
    {
        get;
        init => field = value is >= 0m and <= 100m
            ? value
            : throw new DomainException($"A percentage must be between 0 and 100; got {value}.");
    }

    public Percentage(decimal value) => Value = value;
}
```

**Trap 2 — Collections inside records compare by reference.** A `record Route(IReadOnlyList<Stop> Stops)` has value equality over the *reference* to the list, so two routes with identical stops are unequal. Override `Equals(Route?)` and `GetHashCode` to compare element-wise (records allow this), or use a purpose-built immutable collection type with structural equality.

**Trap 3 — Floating point for money or measurements.** `double` in a value object whose equality matters produces `0.1 + 0.2 != 0.3`. Use `decimal` for money; for physical measurements, define a tolerance-based comparison explicitly rather than relying on record equality.

**Trap 4 — Culture-sensitive normalization.** `ToLower()` without a culture uses the current culture (the Turkish "I" problem). Use `ToLowerInvariant()` or ordinal comparisons in value objects.

**Trap 5 — Exposing a mutable member.** A value object holding a `List<T>` or a mutable class is not immutable, however it's declared.

---

## Concept 45 — Primitive obsession and strongly-typed IDs

**Primitive obsession** is representing domain concepts with language primitives — `string` for an email, `decimal` for money, `Guid` for every identifier. The costs are concrete: no validation (every `string` is a potential email), no behaviour (money arithmetic scattered and inconsistent), and no type safety (`TransferAsync(Guid fromAccount, Guid toAccount)` will happily accept arguments in the wrong order).

Module 16 (Concepts 22 and worked example 5) built strongly-typed IDs end to end — JSON converters, EF value converters via `ConfigureConventions`, `IParsable<T>` for minimal-API binding, and the `default` problem. The DDD-specific points:

```csharp
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New(IIdGenerator ids) => new(ids.NewGuid());   // DB-appropriate ordering (Concept 42)
    public override string ToString() => Value.ToString();
}
public readonly record struct CustomerId(Guid Value);

// The compiler now prevents a whole category of bugs:
public Task<Order?> GetAsync(OrderId id, CancellationToken ct);          // can't pass a CustomerId
```

- **Every aggregate root gets a strongly-typed ID.** It's the cheapest invariant-encoding upgrade available and it makes references between aggregates (Concept 61) self-documenting: `CustomerId CustomerId` says exactly what it references.
- **Domain primitives beyond IDs** — `Sku`, `PolicyNumber`, `Iban`, `Quantity`, `Percentage` — earn their keep wherever a value has a format, a range or behaviour.
- **At scale, generate them.** At three types, hand-writing is fine and teaches the mechanics. At thirty, a source generator such as **Vogen** (value objects with validation and analyzers that forbid `default` construction) or **StronglyTypedId** (lighter, ID-focused) removes the boilerplate and the bugs in it.
- **Where to stop:** not every integer needs a type. The test is whether *mixing two of them up* is a plausible bug with real consequences. `OrderId` vs `CustomerId` — yes. `RetryCount` — no.

---

## Concept 46 — Value objects carry behaviour

A value object isn't a validated wrapper; it's where a surprising amount of domain logic naturally lives. Pushing logic *down* into values is one of the most effective simplifications in tactical design, because values are trivially testable and reusable.

The canonical example is `Money`, including the allocation problem — split €100.00 three ways without losing or inventing a cent:

```csharp
public sealed record Money
{
    public decimal Amount { get; private init; }
    public string Currency { get; private init; }        // ISO 4217, upper-case

    private Money(decimal amount, string currency) { Amount = amount; Currency = currency; }

    public static Money Of(decimal amount, string currency)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        currency = currency.Trim().ToUpperInvariant();
        if (currency.Length != 3) throw new DomainException($"'{currency}' is not an ISO 4217 code.");
        if (decimal.Round(amount, MinorUnits(currency)) != amount)
            throw new DomainException($"{currency} amounts have at most {MinorUnits(currency)} decimal places.");
        return new Money(amount, currency);
    }

    public static Money Zero(string currency) => Of(0m, currency);
    public static Money FromMinorUnits(long minor, string currency)
    {
        var code = currency.Trim().ToUpperInvariant();
        return Of(minor / Scale(code), code);
    }

    public Money Add(Money other)      { EnsureSameCurrency(other); return new(Amount + other.Amount, Currency); }
    public Money Subtract(Money other) { EnsureSameCurrency(other); return new(Amount - other.Amount, Currency); }

    public Money MultiplyBy(decimal factor, MidpointRounding rounding = MidpointRounding.ToEven) =>
        new(decimal.Round(Amount * factor, MinorUnits(Currency), rounding), Currency);

    /// Splits a non-negative amount in proportion to the ratios; the parts always sum to the original.
    public IReadOnlyList<Money> Allocate(params ReadOnlySpan<int> ratios)
    {
        if (Amount < 0) throw new DomainException("Allocate a positive amount and negate the parts.");
        if (ratios.IsEmpty) throw new ArgumentException("At least one ratio is required.", nameof(ratios));

        long totalRatio = 0;
        foreach (var r in ratios)
        {
            if (r < 0) throw new ArgumentOutOfRangeException(nameof(ratios), "Ratios must be non-negative.");
            totalRatio += r;
        }
        if (totalRatio == 0) throw new ArgumentException("Ratios must not all be zero.", nameof(ratios));

        var scale = Scale(Currency);
        var totalMinor = (long)(Amount * scale);
        var parts = new long[ratios.Length];
        long allocated = 0;

        for (var i = 0; i < ratios.Length; i++)
        {
            parts[i] = totalMinor * ratios[i] / totalRatio;          // round each share down
            allocated += parts[i];
        }
        for (var i = 0; allocated < totalMinor; i++, allocated++)
            parts[i % parts.Length]++;                               // hand out the remainder one minor unit at a time

        return [.. parts.Select(p => new Money(p / scale, Currency))];
    }

    private void EnsureSameCurrency(Money other)
    {
        if (other.Currency != Currency)
            throw new DomainException($"Cannot combine {Currency} and {other.Currency} without an explicit conversion.");
    }

    private static int MinorUnits(string currency) => currency switch
    {
        "JPY" or "KRW" or "CLP" or "ISK" => 0,
        "BHD" or "KWD" or "JOD" or "OMR" or "TND" => 3,
        _ => 2,
    };

    private static decimal Scale(string currency) => MinorUnits(currency) switch { 0 => 1m, 3 => 1000m, _ => 100m };
}

// Money.Of(100m, "EUR").Allocate(1, 1, 1) → 33.34, 33.33, 33.33
```

What this buys you:

- **The rounding rule has one home.** Every place in the system that splits an amount — instalments, cost allocation across cost centres, refunds across payment methods — uses the same algorithm and never loses a cent.
- **Currency mistakes are impossible to make silently.** Adding EUR to USD throws; there is no code path that does it by accident.
- **The tests are trivial** — pure functions over values, no setup (Concept 93).

Other examples of behaviour that belongs in values: `DateRange.Overlaps`, `DateRange.Intersect` and `DateRange.Days` (booking and scheduling domains live on these); `Quantity.Add` with unit-of-measure checks; `Address.IsInSameCountryAs`; `Percentage.Of(Money)`; `PolicyTerm.Contains(DateOnly)`; `GeoPoint.DistanceTo`. Each is a small, well-tested piece of business knowledge that would otherwise be reimplemented, slightly differently, in several services.

The review heuristic: **when an entity or a service has a method that operates mostly on one of its value-typed fields, move the method into the value.**

---

## Concept 47 — Mapping value objects in EF Core 10

Module 19 (Concept 5) explained why EF Core 10 changed the recommendation; here's the DDD mapping in practice.

**Complex types are now the right default** for value objects stored alongside their owner. They have value semantics — assignment copies, comparison compares contents — which is exactly what a value object is. Owned entity types, the old recommendation, carry a hidden identity and reference semantics, which is why assigning a customer's billing address from their shipping address used to fail. Microsoft's EF 10 guidance explicitly advises owned-type users to switch.

```csharp
internal sealed class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> b)
    {
        b.ToTable("Customers", "crm");
        b.HasKey(c => c.Id);
        b.Property(c => c.Id).ValueGeneratedNever();                 // the domain assigns identity

        // Single-value VO → one column
        b.ComplexProperty(c => c.Email, e =>
            e.Property(x => x.Value).HasColumnName("Email").HasMaxLength(320));

        // Multi-attribute VO → several columns on the owner's table (table splitting)
        b.ComplexProperty(c => c.CreditLimit, m =>
        {
            m.Property(x => x.Amount).HasColumnName("CreditLimitAmount").HasPrecision(19, 4);
            m.Property(x => x.Currency).HasColumnName("CreditLimitCurrency").HasMaxLength(3);
        });

        // Optional VO (EF 10+: requires at least one required property on the complex type)
        b.ComplexProperty(c => c.BillingAddress);

        // Collection of VOs (EF 10+: ComplexCollection — must map to a single JSON column on relational providers)
        b.ComplexCollection(c => c.PreviousAddresses, a => a.ToJson());
    }
}
```

Three other mapping tools, and when each fits:

- **Value converters** for single-value VOs and strongly-typed IDs, applied by convention in `ConfigureConventions` (Module 16). A converter makes the VO look like a scalar to EF, so it works in keys, foreign keys and `WHERE` clauses. Use this for IDs and single-field domain primitives; use complex types for multi-attribute values.
- **JSON columns** (`ToJson()` on a complex property or collection) for value objects you rarely query into, or collections of values. EF 10 can query into and bulk-update JSON-mapped complex types, but indexing and filtering on JSON contents costs more than on columns — choose by query pattern.
- **Owned entity types** — still work, now legacy for this purpose. You'll meet them in existing codebases; know why to migrate.

Limitations to state if asked (current as of EF Core 10): optional complex types need at least one required property; collections of `struct` complex types aren't supported (use `record` classes for collection elements); complex collections must be JSON-mapped on relational databases. EF 11 extends complex types to TPT/TPC inheritance and allows keys and indexes on scalar properties nested inside complex types.

---

## Concept 48 — Always-valid, and parse, don't validate

The principle: **a domain object that exists is valid.** If you hold an `EmailAddress`, it's a valid email address; if you hold an `Order`, all its invariants hold. You never need to call `IsValid()` on a domain object, because an invalid one can't be constructed.

Vladimir Khorikov calls this the **always-valid domain model**, and it's the opposite of the "validate before save" approach where objects can be in any state and a validation pass runs at the end. The always-valid approach has two big advantages: rules are checked at the moment of change (so errors point at the cause), and no code downstream has to be defensive.

It pairs with Alexis King's **"parse, don't validate"** (Module 16, Concept 32): at the edge of the system, convert unstructured input into types that can only hold valid values, and let the rest of the code trust the types.

```csharp
// Edge: parse raw input into domain types; reject at the boundary with a 400
app.MapPost("/customers/{id}/email", async Task<Results<NoContent, ValidationProblem, NotFound>>
    (CustomerId id, ChangeEmailRequest body, ChangeEmailHandler handler, CancellationToken ct) =>
{
    if (!EmailAddress.TryParse(body.Email, out var email))
        return TypedResults.ValidationProblem(new Dictionary<string, string[]>
            { ["email"] = ["The email address is not valid."] });

    return await handler.Handle(new ChangeEmail(id, email), ct) switch
    {
        ChangeEmailOutcome.Changed   => TypedResults.NoContent(),
        ChangeEmailOutcome.NotFound  => TypedResults.NotFound(),
        _ => throw new UnreachableException(),
    };
});
```

Note the command `ChangeEmail(CustomerId, EmailAddress)` — it's built from domain types, so the handler and the aggregate never see a raw string.

**Input validation vs. invariants** — the distinction interviewers probe:

| | Input validation | Domain invariants |
|---|---|---|
| **Question** | "Is this request well-formed?" | "Is this change allowed by the business?" |
| **Examples** | Required fields present, string lengths, formats, types | "An order can't be placed with no lines," "credit limit can't be exceeded" |
| **Where** | API edge (endpoint filters, FluentValidation, data annotations) and value-object parsing | Value-object constructors, aggregate methods |
| **Failure** | 400 with field-level problem details | 409/422 with a domain reason (Concept 95) |
| **Depends on state?** | No — the request alone | Often yes — the aggregate's current state |

Two practical notes. First, **format validation belongs in the value object**, not only at the edge — the edge's job is to *call* the value object's parser and translate its failure into a 400, so the rule lives in one place. Second, **don't validate on load**: data already in the database was valid under the rules in force when it was written; re-validating on materialization breaks when rules tighten and turns a rule change into a data migration (Concept 90).

---

## Concept 49 — Make illegal states unrepresentable

A step beyond always-valid: design the *types* so that invalid combinations can't even be expressed. Module 16 (Concepts 31–33) developed this in general; in a domain model it's the difference between checking a status flag in every method and having the compiler do it.

The classic smell is a type with a status and a set of nullable fields whose validity depends on the status:

```csharp
public sealed class Payment
{
    public PaymentStatus Status { get; set; }          // Pending, Authorized, Captured, Failed
    public string? AuthorizationCode { get; set; }     // only meaningful when Authorized or Captured
    public DateTimeOffset? CapturedAt { get; set; }    // only when Captured
    public string? FailureReason { get; set; }         // only when Failed
}
```

Nothing stops `Status = Captured` with no `CapturedAt`, or `Failed` with an authorization code. Every consumer must know the unwritten rules.

The state-as-types alternative, using C# 15's `closed` hierarchy (Module 16, Concept 33):

```csharp
public closed record PaymentState;
public sealed record Pending : PaymentState;
public sealed record Authorized(AuthorizationCode Code, DateTimeOffset At) : PaymentState;
public sealed record Captured(AuthorizationCode Code, DateTimeOffset CapturedAt, Money Amount) : PaymentState;
public sealed record Failed(FailureReason Reason, DateTimeOffset At) : PaymentState;

public string Describe(PaymentState state) => state switch
{
    Pending          => "Awaiting authorization",
    Authorized a     => $"Authorized ({a.Code})",
    Captured c       => $"Captured {c.Amount} at {c.CapturedAt:u}",
    Failed f         => $"Failed: {f.Reason}",
    // exhaustive — adding a state produces compile errors at every switch that must handle it
};
```

Each state carries **exactly** the data that exists in that state, and the compiler checks every switch for completeness. A `Captured` payment without a capture time is not a runtime bug to be caught; it's a type that doesn't exist.

**On .NET 10 (C# 14)**, the same design works with an `abstract record` base and a throwing discard arm (`_ => throw new UnreachableException()`); you lose compile-time exhaustiveness but keep the data-per-state guarantee. The persistence mapping for a state hierarchy is covered in Concept 72 — it's the part that needs care.

Two more everyday uses of the principle in a domain model:

- **Distinct types for distinct lifecycle stages** — `UnverifiedEmail` and `VerifiedEmail`, so a method that sends marketing email takes a `VerifiedEmail` and can't be called with an unverified one.
- **Non-empty collections** — an `Order` that can't be placed without lines can take a `NonEmptyList<OrderLine>` in its `Place` method, moving the check from runtime into the signature.

The interview sentence: *"I'd rather the type system reject an invalid state than a test catch it. If a field only makes sense in one state, it belongs to that state's type, not to a nullable property on the parent."*

---

## Concept 50 — Domain services

Some domain operations don't belong naturally to any entity or value object. Evans' definition: **when a significant process or transformation in the domain is not a natural responsibility of an entity or value object, add an operation to the model as a standalone interface declared as a service**, named in the ubiquitous language, and stateless.

Recognize one by three signs:

1. **It involves several aggregates or values, and belongs to none of them.** Quoting a price for a cart involves the cart, a price list and the active promotions. Putting `Quote()` on `Cart` makes the cart know about promotions; putting it on `PriceList` makes the price list know about carts.
2. **It's a verb the experts use**, not a thing: "quote," "allocate," "route," "transfer."
3. **It's stateless** — everything it needs comes in as arguments.

```csharp
// Domain service: pure domain logic over several domain objects; no I/O, no persistence
public sealed class PriceQuoter
{
    public PriceQuote Quote(Cart cart, PriceList prices, IReadOnlyCollection<Promotion> activePromotions, DateOnly on)
    {
        var lines = cart.Lines
            .Select(l => new QuotedLine(l.Sku, l.Quantity, prices.UnitPriceOf(l.Sku, on)))
            .ToList();

        var discounts = activePromotions
            .Where(p => p.AppliesOn(on))
            .SelectMany(p => p.DiscountsFor(lines))
            .ToList();

        return PriceQuote.From(lines, discounts, prices.Currency);
    }
}
```

The important distinction, which interviewers probe because the word "service" is overloaded:

| | **Domain service** | **Application service** | **Infrastructure service** |
|---|---|---|---|
| **Contains** | Domain logic that has no natural owner | Use-case orchestration: load, call domain, save, publish | Technical capability: email, files, clocks, external APIs |
| **Knows about** | Domain types only | Domain + ports (repositories, gateways) + transactions | Frameworks, SDKs, protocols |
| **Business rules?** | Yes | **No** — delegates them | No |
| **I/O?** | No (ideally pure) | Yes, through ports | Yes |
| **Lives in** | Domain layer | Application layer | Infrastructure layer, behind a port |
| **Example** | `PriceQuoter`, `FundsTransferPolicy` | `PlaceOrderHandler` | `SmtpEmailSender`, `EcbExchangeRateClient` |

The failure mode on each side: **too many domain services** is an anaemic model in disguise — every rule extracted into a `*Service` and entities reduced to data (Concept 96). **Too few** means cross-aggregate logic ends up in application services, where it's mixed with orchestration and duplicated across use cases. The test: if you removed all application services and rebuilt them, would any business rule be lost? If yes, that rule is in the wrong layer.

---

## Concept 51 — Passing collaborators into the model

When an aggregate method needs something it doesn't own — a policy, a calculation, a lookup — how does it get it? The options, from most to least preferred:

**1. Pass the data, not the dependency.** The application service loads what the decision needs and passes values in. The aggregate stays pure:

```csharp
// Application service
var creditAvailable = await _credit.GetAvailableCreditAsync(order.CustomerId, ct);   // I/O here
order.Place(creditAvailable, _clock.GetUtcNow());                                   // pure decision here
```

**2. Pass a pure collaborator (double dispatch).** When the decision logic varies — by region, product line, contract — pass the policy object into the method:

```csharp
public void ApplyDiscount(IDiscountPolicy policy)
{
    EnsureDraft();
    _discount = policy.DiscountFor(Total, CustomerTier);   // policy is domain logic, not I/O
    RecalculateTotal();
}
```

**3. Pass a domain service interface that performs I/O.** The aggregate calls, say, `IUniquenessChecker.IsEmailTaken(...)`. This keeps the rule inside the model but makes the aggregate impure and async, and it hides database round trips inside domain methods.

**Never: inject dependencies into entities through the constructor or a service locator.** EF materializes entities without your DI container, so constructor injection doesn't work, and a static service locator makes the model untestable and its dependencies invisible.

Options 1 and 3 are two corners of what Khorikov calls the **DDD trilemma**: you can have only two of **domain-model completeness** (all domain logic in the domain layer), **domain-model purity** (no out-of-process dependencies in the domain) and **performance** (no unnecessary calls to external systems). Option 1 gives up some completeness (part of the decision — fetching the credit figure — lives in the application layer). Option 3 gives up purity. Loading everything into memory to keep both gives up performance. Khorikov's recommendation, which is also the mainstream one, is to **keep purity and accept some fragmentation** — and Concept 70 applies it to the hardest case, uniqueness.

---

## Concept 52 — Factories

**Factories encapsulate the creation of complex objects and aggregates**, so that an object is born valid and callers don't need to know how to assemble it. Three shapes, from simplest:

**Static factory methods on the aggregate** — the default. Named, intention-revealing, and they can return a result instead of throwing:

```csharp
public static Order Draft(OrderId id, CustomerId customer, string currency, TimeProvider clock)
{
    var order = new Order(id, customer, Money.Zero(currency), clock.GetUtcNow());
    order.Raise(new OrderDrafted(id, customer));
    return order;
}
```

A named factory communicates more than a constructor (`Order.Draft` vs `Order.ImportFromLegacy`), and you can have several without overload ambiguity.

**Factory methods on another aggregate** — when creation *depends on the state* of an existing aggregate. If only an active customer with no overdue invoices can start an order, `customer.StartOrder(...)` checks that, then creates the `Order`. This expresses a real business rule ("customers start orders") and keeps the precondition in the aggregate that knows the relevant state. Note that it creates a *separate* aggregate, referenced by ID, and saved in its own right — it doesn't make the order part of the customer aggregate.

**Standalone factories** — a separate class, when creation is genuinely complex: assembling an aggregate from several sources, choosing among implementations, or translating from an external representation (an ACL's translator is a kind of factory).

**Creation vs. reconstitution** — the distinction that matters in .NET. *Creation* makes a new object: it validates, assigns identity, and may raise a domain event ("OrderDrafted"). *Reconstitution* rebuilds an existing object from storage: it must **not** re-validate (rules may have changed since the data was written), must **not** raise events (nothing happened), and must **not** generate a new ID. EF Core handles reconstitution through a private parameterless constructor or constructor binding plus backing fields (Concept 90) — which is precisely why domain constructors that validate and raise events should be separate from the path EF uses.

---

## Concept 53 — Policies and specifications

Two related patterns for **making implicit rules explicit** — the knowledge-crunching move from Concept 6 turned into code.

**Policy (strategy).** A rule that varies, or that's important enough to name, becomes an object. Evans' own example is overbooking: a shipping line accepts bookings up to 110% of a voyage's capacity, because some cargo always cancels. Buried in a method, it's `if (bookedSize + cargo.Size > voyage.Capacity * 1.1m)` — a magic number with an invisible business reason. Made explicit:

```csharp
public interface IOverbookingPolicy
{
    bool Allows(Voyage voyage, CargoSize additional);
}

public sealed class PercentageOverbookingPolicy(Percentage allowance) : IOverbookingPolicy
{
    public bool Allows(Voyage voyage, CargoSize additional) =>
        voyage.BookedSize.Plus(additional) <= voyage.Capacity.IncreasedBy(allowance);
}
```

Now the rule has a name the experts recognize, a home, a test suite, and a way to vary (a different allowance per trade lane, a policy that forbids overbooking for hazardous cargo) without touching `Voyage`.

**Specification.** Eric Evans and Martin Fowler's 1997 paper *Specifications* describes a predicate object that answers "does this object satisfy the criteria?" and can be combined with `And`, `Or`, `Not`. It has three uses:

1. **Validation** — does this object meet a business standard? (`CreditWorthyCustomerSpecification`)
2. **Selection** — find the objects that satisfy it (a query).
3. **Building to order** — describe what must be created (rarer).

In .NET, the *selection* use is the one most people know — Module 19 (Concept 67) covered query specifications over `IQueryable` (Ardalis.Specification), with the caution that they're worth it only for genuinely reused query criteria. The *domain* use is different and simpler: a named, testable business predicate evaluated in memory:

```csharp
public sealed class EligibleForExpressShipping
{
    public bool IsSatisfiedBy(Order order, Address destination) =>
        order.Lines.All(l => !l.IsHazardous)
        && order.TotalWeight <= Weight.Kilograms(30)
        && destination.Country.SupportsExpress;
}
```

Don't conflate them: a **business specification** encodes a rule the experts would state; a **query specification** is a reusable query shape. Both are fine; they're different patterns with the same name.

---

## Concept 54 — Enumerations: enum, enumeration class, or closed hierarchy

Domain models are full of closed sets — order status, claim type, customer tier — and C# offers three representations:

| Shape | Strengths | Weaknesses | Use when |
|---|---|---|---|
| **`enum`** | Simple, fast, persists as int or string easily | No behaviour, no per-case data; any integer is a valid value; switches are never exhaustive | Plain labels with no behaviour; persist as **strings**, not ints, so reordering and inserting cases is safe |
| **Enumeration class** (Microsoft's eShop `Enumeration` base — a class with static instances, an `Id` and a `Name`) | Behaviour per value, can hold data, easy to persist by `Id` or `Name` | Boilerplate; not exhaustive; equality and serialization to wire up | Legacy codebases; when you need behaviour per case on .NET 10 |
| **Closed hierarchy** (C# 15) | Exhaustive switches, per-case data, shared members, behaviour | Needs .NET 11; persistence needs a discriminator mapping | States and outcomes with per-case data; new code on .NET 11+ |

The review points: an `enum` that's grown a companion `switch` in six places (`GetDisplayName`, `CanTransitionTo`, `IsTerminal`...) wants to be a richer type; and an `enum` persisted as an integer is a latent data-corruption bug the day someone inserts a value in the middle.

---

## Concept 55 — Domain modules: package by concept

DDD's **module** pattern (not to be confused with a modular-monolith module, which is closer to a bounded context) is about organizing code *within* a context so the structure tells the story of the domain.

The rule: **package by domain concept, never by pattern type.**

```
// Tells you nothing about the domain; every change touches every folder
Claims.Domain/
  Entities/          Claim.cs, Evidence.cs, Payment.cs
  ValueObjects/      ClaimNumber.cs, Money.cs, LossEvent.cs
  Services/          LiabilityAssessor.cs, ReserveCalculator.cs
  Events/            ClaimSubmitted.cs, LiabilityAccepted.cs
  Repositories/      IClaimRepository.cs

// Tells you what the domain is; related things change together
Claims.Domain/
  Submission/        Claim.cs, ClaimNumber.cs, LossEvent.cs, ClaimSubmitted.cs, IClaimRepository.cs
  Liability/         LiabilityAssessor.cs, LiabilityDecision.cs, LiabilityAccepted.cs
  Reserves/          Reserve.cs, ReserveCalculator.cs, ReserveAdjusted.cs
  Settlement/        Settlement.cs, SettlementOffer.cs, ClaimSettled.cs
  Shared/            Money.cs, PolicyNumber.cs
```

This is the same principle as Module 20's "screaming architecture" and feature folders, applied inside the domain layer: **cohesion by concept, low coupling between concepts.** Namespaces become part of the ubiquitous language, and a newcomer can find the rules about reserves without knowing which pattern each class implements.

---

# Part F — Aggregates

The aggregate is the pattern everyone names and few design well. It's the first of the three decisions — **where consistency must be immediate** — and it's the one with the most direct consequences for correctness, concurrency, throughput and distribution. This part derives it, defines it, sizes it with numbers, and then works through the cases that actually cause production incidents: concurrency in EF Core, hot aggregates, uniqueness and cross-aggregate rules — ending with the functional and "dynamic" alternatives that are reshaping the conversation in 2026.

---

## Concept 56 — Deriving the aggregate from first principles

Most explanations start with Evans' definition. Start instead from the problem, because the definition then follows by necessity.

**Step 1 — Business rules are invariants over data.** "An order's total equals the sum of its lines." "An order has at most 50 lines." "An account's balance never falls below its overdraft limit." "No two customers share an email address." "Every order references an existing customer."

**Step 2 — Some invariants survive concurrency without coordination; some don't.** Module 9 (Concept 31) gave you the tool: **invariant confluence**. An invariant is I-confluent if merging any two valid states produces a valid state. "Every order references an existing customer" is confluent on insert — two concurrent orders for the same customer can't jointly violate it. "At most 50 lines" is not — two concurrent requests can each see 49 lines and each add one. "Balance ≥ limit" is not — two concurrent withdrawals can each pass the check.

**Step 3 — Non-confluent invariants require serialization.** The only way to keep a non-confluent invariant true under concurrency is to ensure that the operations that could violate it are **serialized** with respect to each other: one at a time, each seeing the other's result. That means a transaction (or a single writer) covering **all the data the invariant spans**.

**Step 4 — Serialization is expensive, so the serialized unit should be as small as possible.** Every piece of data inside a serialization unit contends with every other. The bigger the unit, the lower the throughput and the higher the conflict rate (Concept 65).

**Conclusion: the aggregate is the smallest set of data that must be changed atomically and serially to keep a non-confluent invariant true.** Everything else is outside it and can be updated independently, and eventually.

Two independent traditions arrive at the same object, which is a good sign it's real:

- **Pat Helland's *Life beyond Distributed Transactions*** (CIDR 2007; revised in ACM Queue 2016) argues that scalable systems must assume transactions cannot span machines, so the application must be built from **entities** — each with a unique key, each living on one machine at a time, and each the scope of an atomic transaction — communicating by messages. Helland's entity is DDD's aggregate, arrived at from the infrastructure side.
- **Module 12's** conclusion that "distributed transactions are usually a symptom of a boundary drawn in the wrong place" is the same statement from the database side: get the aggregate right and the transaction boundary takes care of itself.

This derivation gives you the consequences for free, and they're worth stating as a chain because it's what interviewers probe:

**aggregate = consistency boundary = transaction boundary = concurrency unit = unit of loading and saving = natural partition key.**

When those coincide, the system is simple. When they don't — a transaction spanning several aggregates, a partition splitting one, a load that pulls half of another — something is about to hurt.

---

## Concept 57 — The aggregate, defined

Evans' definition: **an aggregate is a cluster of associated objects that we treat as a unit for the purpose of data changes.** Each aggregate has a **root** — a single entity through which all access happens — and a **boundary** that defines what's inside.

The rules that make it work (paraphrasing Evans' Chapter 6):

1. **The root entity has global identity** and is ultimately responsible for checking invariants.
2. **Entities inside the boundary have local identity**, unique only within the aggregate (order line 3 of order X).
3. **Nothing outside the boundary may hold a reference to anything inside it except the root.** The root may hand out transient references to internal objects (for reading), but outsiders must not keep them, and must not modify internals directly.
4. **Only aggregate roots can be obtained directly from the database.** Everything else is reached by traversal from a root. That's why there is one repository per aggregate root, not per entity (Concept 86).
5. **Objects within an aggregate can hold references to *other* aggregate roots** — by identity (Concept 61).
6. **A delete removes everything within the boundary at once.**
7. **When a change to any object within the boundary is committed, all invariants of the whole aggregate must be satisfied.**

Rule 7 is the one that matters most and the one that most implementations quietly break: it means the aggregate's invariants are guaranteed **at commit**, which in turn means the **whole aggregate must be the unit of concurrency control**. Concept 68 shows how EF Core breaks this if you're not careful.

A small concrete shape to keep in mind:

```
Order  (root; global identity: OrderId)
 ├── Status, CustomerId (reference to another aggregate, by ID), Total : Money
 ├── OrderLine #1  (child entity; local identity: line number)
 ├── OrderLine #2
 └── ShippingAddress : Address (value object)

Invariants: Total == Σ line totals; ≤ 50 lines; lines only change while Draft;
            can't be Placed with zero lines.
```

---

## Concept 58 — Finding true invariants

The hardest and most important step in aggregate design, because the boundary should enclose **true invariants and nothing else**. Most oversized aggregates come from treating "related data" as "data that must be consistent."

**Question 1 — "What happens if this rule is violated for a few seconds and then corrected?"**

- "We'd oversell stock we don't have" / "the account would be overdrawn and we'd owe the regulator an explanation" → true invariant, must hold at commit, belongs *inside* one aggregate.
- "The customer's order count on their profile would be stale for a moment" → not an invariant; it's derived data that can be updated eventually, *outside* the aggregate.

**Question 2 — Vaughn Vernon's "whose job is it?"** From *Effective Aggregate Design*: when a use case changes something, ask whether it's **the job of the user executing this use case** to make the data consistent. If yes, aim for transactional consistency within one aggregate. If it's **another user's job, or the system's job**, allow it to be eventually consistent across aggregates. A customer placing an order is responsible for the order being internally consistent; they're not responsible for the warehouse's pick list being updated in the same millisecond.

**Question 3 — "Is this rule about one thing's state, or about a relationship among many things?"** Rules about one thing's internal consistency (an order's lines and total) are natural aggregate invariants. Rules about relationships among many independent things (no two customers share an email; a warehouse's total stock across locations) are **set-based rules** — they don't fit in one aggregate without making it enormous, and need a different mechanism (Concepts 70–71).

**Question 4 — "Would the business accept a compensating action instead?"** Many rules that sound absolute turn out, on questioning, to be handled today by a person fixing things afterwards — an airline overbooks and compensates; a retailer occasionally oversells and apologizes with a voucher. If the business already compensates, the rule is a policy, not a transactional invariant, and it can live across aggregates with a process that detects and corrects violations.

What *isn't* an invariant, and shouldn't shape the boundary:

- **Navigation convenience** ("the order screen shows the customer's name") — that's a read concern (Concept 92).
- **Derived data** that can be recomputed or updated eventually (counts, totals across aggregates, summaries).
- **Lifecycle co-occurrence** ("they're usually created together") — being created in the same use case doesn't mean they must be consistent at every later commit.
- **Ownership in the UI sense** ("it's the customer's order") — ownership is a reference, not a containment.

The interview move: when asked to design aggregates, **list the invariants first, out loud, and classify each one** — "this one must hold at commit; this one can be eventual; this one is set-based." The aggregate boundaries fall out of that list. It's the same move as Module 9's "classify the invariants before choosing coordination."

---

## Concept 59 — Vernon's four rules

Vaughn Vernon's three-part essay *Effective Aggregate Design* (2011; free PDFs on dddcommunity.org) distilled the community consensus into four rules of thumb. They're the most-quoted guidance on aggregates, and knowing both the rules and the reasoning behind each is expected at senior level:

| Rule | What it says | Why |
|---|---|---|
| **1. Model true invariants in consistency boundaries** | The boundary encloses exactly what must be consistent at commit | The boundary exists *only* to protect invariants (Concept 56) |
| **2. Design small aggregates** | Prefer the root plus a minimal set of values and child entities | Loading cost, memory, contention and conflict rate all scale with size (Concept 60) |
| **3. Reference other aggregates by identity** | Hold `CustomerId`, not `Customer` | Keeps loads and transactions confined; makes distribution possible (Concept 61) |
| **4. Use eventual consistency outside the boundary** | Changes to other aggregates happen in separate transactions, driven by domain events | One aggregate per transaction is what makes rules 1–3 hold (Concept 62) |

And Vernon's explicit follow-up, which is what distinguishes a senior answer from a rule-follower's: **there are legitimate reasons to break them** (Concept 63). Rules of thumb exist to be understood well enough to know when they don't apply.

---

## Concept 60 — Small aggregates

The case for small aggregates is best made with Vernon's own example from *Effective Aggregate Design*. A Scrum project-management product initially modelled `Product` as one large aggregate containing its `BacklogItem`s, `Release`s and `Sprint`s — because "they all belong to the product."

What went wrong, in order:

1. **Concurrency conflicts.** Two users working on the same product — one planning a sprint, one adding a backlog item — both modified the `Product` aggregate. With optimistic concurrency on the aggregate, one of them failed, even though their changes had nothing to do with each other.
2. **Loading cost.** Every operation loaded the product with every backlog item, release and sprint. As products accumulated hundreds or thousands of items, a trivial operation loaded thousands of objects.
3. **Transaction scope.** Every change locked the whole product's data.

The fix was four aggregates — `Product`, `BacklogItem`, `Release`, `Sprint` — each referencing `ProductId` by identity. The only invariants that genuinely spanned them turned out to be few, and could be handled explicitly.

The general costs of large aggregates, stated so you can quote them:

- **Load cost** grows linearly with contained objects — every command pays for all of them.
- **Memory and GC pressure** grow with it (Module 14 — large graphs with mixed lifetimes promote to Gen 2).
- **Conflict probability** grows with the rate of commands *to the aggregate*, which grows with its size (Concept 65).
- **Lock scope** grows with it, raising deadlock risk when other transactions touch overlapping rows (Module 12).
- **Change coupling**: more concepts in one class, more reasons to change it, more merge conflicts.

The heuristic: **start with the root alone, and add objects inside the boundary only when a true invariant requires it.** Vernon's observation is that when teams do this honestly, most aggregates turn out to be a root entity plus a few value objects — child entities are the exception.

---

## Concept 61 — Reference other aggregates by identity

```csharp
// No: a navigation property to another aggregate
public sealed class Order
{
    public Customer Customer { get; private set; } = null!;   // loads, tracks, and invites modification of another aggregate
}

// Yes: a reference by strongly-typed identity
public sealed class Order
{
    public CustomerId CustomerId { get; private set; }
}
```

Why this matters more than it looks:

- **It makes the load boundary explicit.** Loading an order can never accidentally drag in a customer graph, or through it every other order the customer has.
- **It confines the transaction.** With a navigation property, `order.Customer.CreditLimit = …` inside an order use case silently modifies a second aggregate in the same transaction — breaking the one-aggregate-per-transaction rule without anyone deciding to. With an ID, it's impossible.
- **It prepares for distribution.** An ID reference works identically whether the customer is in the same database, a different schema, or a different service. A navigation property only works in one database, and becomes a rewrite at extraction time (Module 21, Concept 37).
- **It reduces memory and tracking overhead** — EF's change tracker isn't asked to watch objects the use case doesn't change.

In EF Core, that means **no navigation properties between aggregates**. Within one bounded context and one schema, you may still want a foreign-key *constraint* for referential integrity — map it with `HasOne<Customer>().WithMany().HasForeignKey(o => o.CustomerId)` and no navigation on either side. Across modules or contexts, no FK at all (Module 21, Concept 37); integrity is checked by the application at the point of use.

When a use case genuinely needs data from both — "place an order, checking the customer's credit" — the application service loads both, reads what it needs from one, and passes values into the other (Concept 51). What it does *not* do is modify both in one transaction, except under Concept 63's conditions.

---

## Concept 62 — Eventual consistency outside the boundary

If only one aggregate changes per transaction, how do the others find out? **Domain events**, handled in separate transactions (Part G):

```
Transaction 1:  Order.Place()  →  commits Order (Placed) + OrderPlaced event (outbox)
                                            │
                        (milliseconds to seconds later)
                                            ▼
Transaction 2:  OrderPlacedHandler →  Customer.RecordPurchase(total)   → commits Customer
Transaction 3:  OrderPlacedHandler →  Inventory.Reserve(lines)         → commits reservations
```

Three things make this work in practice:

1. **The event is committed atomically with the state change** (the outbox — Module 11, Module 19 Concept 35), so it can't be lost.
2. **Handlers are idempotent** (at-least-once delivery), so redelivery is harmless.
3. **The business agrees on the acceptable lag** — and on what the user sees in the meantime (Concept 84).

The objection you'll hear is "but then the data is inconsistent for a while." The answer is that the data is *consistent with the business's own rules*: the invariants that matter at commit are inside aggregates and are never violated; what's temporarily stale is derived or downstream information whose staleness the business already tolerates. Businesses ran on eventual consistency long before software — paper forms moved between departments in trays.

When the answer to "can this be eventual?" is genuinely no, that's a signal to revisit the boundary (maybe the two things are one aggregate after all) — not a reason to add a distributed transaction.

---

## Concept 63 — When to break the rules

Vernon lists four legitimate reasons to break the one-aggregate-per-transaction and small-aggregate rules, and a senior answer can name them and add the pragmatic fifth:

1. **User-interface convenience.** A batch operation in the UI — "create these twenty backlog items at once" — may reasonably create several aggregates in one transaction, when they're all new and nobody else can be contending for them.
2. **Lack of technical mechanisms.** If there's no reliable messaging or background processing available, eventual consistency is impractical and a multi-aggregate transaction may be the least-bad option.
3. **Global transactions imposed by policy.** Some organizations require a single transaction for regulatory or legacy-integration reasons.
4. **Query performance.** Occasionally, holding a direct reference to another aggregate is justified for read performance — though CQRS (Concept 92, Module 23) usually removes the need.

**5. Pragmatism inside one database and one context.** In a modular monolith where two aggregates live in the same database and the same bounded context, updating both in one local transaction is cheap, correct and common. Many experienced practitioners accept it when **contention is low** (the aggregates are rarely modified concurrently by different users), **both are owned by the same context**, and **the team understands what it's giving up**: independent scaling of the two, extraction without redesign, and isolation of their contention.

What you must *not* do is break the rules by accident — a navigation property that lets an order use case modify a customer, a domain event handler that updates another aggregate in the same transaction without anyone deciding that's acceptable. The difference between a pragmatic exception and a decaying design is whether the exception was **chosen, documented and bounded**. A one-line ADR ("Transfers debit and credit two accounts in one local transaction; accounts are locked in ID order to avoid deadlock; revisit if accounts move to separate partitions") is the difference.

---

## Concept 64 — Design aggregates from behaviour, not data

The most common aggregate-design mistake is starting from the **data model** — the entity-relationship diagram, the nouns in the requirements — and drawing boundaries around clusters of tables. That produces aggregates shaped like the schema, which are usually too large (because tables are connected) and anaemic (because tables don't have behaviour).

Start instead from the **commands and the decisions**:

1. **List the commands** the aggregate must handle (from EventStorming: the blue stickies next to one yellow constraint).
2. **For each command, list the rules** that decide whether it's accepted.
3. **List the data those rules need.** That's the aggregate's state — and usually much less than "everything we know about this thing."
4. **List the events it produces.**

An aggregate designed this way often looks surprisingly small. A `Claim` aggregate in the liability context might hold status, coverage decision, reserve amount and a list of evidence references — but not the claimant's contact details, the adjuster's notes or the correspondence history, because no liability decision depends on them. Those live elsewhere (another aggregate, another context, or a read model).

A consequence that surprises people: **the same business "thing" can be several aggregates** in different contexts, or even within one context, if different decisions need different data. "Order" in checkout (lines, prices, promotions) and "Order" in fulfilment (shipments, picks, carrier) are different aggregates with different invariants that happen to share an ID.

The sentence: *"I size an aggregate by the decisions it makes, not by the data we have about the thing. It holds exactly what its rules need to be checked at commit, and nothing else."*

---

## Concept 65 — Sizing with numbers: contention and the throughput ceiling

The quantitative part, and the one almost nobody brings to an interview.

**An aggregate is a serialization point.** Every command against one aggregate *instance* must be applied one at a time (Concept 56). With optimistic concurrency, a command reads the aggregate at version *v*, decides, and writes *v+1* only if nobody else wrote in between. So two parameters govern everything:

- **λ** — the rate of commands targeting the **same aggregate instance** (per second).
- **d** — the duration of one read–decide–write cycle (seconds). In a typical EF Core setup against a nearby database, two or three round trips plus processing puts *d* somewhere around 5–20 ms.

If commands arrive roughly independently (a Poisson process), the probability that some *other* command commits during a given command's window is approximately:

**P(conflict) ≈ 1 − e^(−λd)**

| λ·d | P(conflict) | What it feels like |
|---|---|---|
| 0.01 | ~1% | Invisible; a retry now and then |
| 0.1 | ~10% | Noticeable retries; latency tail grows |
| 0.5 | ~39% | Retries dominate; users see failures |
| 1 | ~63% | Most commands conflict at least once |
| 2+ | 86%+ | Livelock territory — retries collide with retries |

And the ceiling that follows from serialization: **one aggregate instance can commit at most about 1/d commands per second**, regardless of how many servers you add. At *d* = 10 ms, that's roughly **100 commits per second per aggregate instance**, and retries eat into it.

How to use this:

- **A user's shopping cart:** λ ≈ one command every few seconds at most → λ·d ≈ 0.001. Size is irrelevant; make it whatever the invariants need.
- **A shared team backlog:** a dozen people editing during planning, λ ≈ 1/s → λ·d ≈ 0.01. Fine, *if* the aggregate is the backlog item and not the whole product (Concept 60).
- **A flash-sale product's stock:** λ ≈ 2,000/s → λ·d ≈ 20. **No single-aggregate design can work** — you need Concept 66.

The **Aggregate Design Canvas** (DDD Crew) asks for exactly these inputs — command handling rate (average and peak), number of concurrent clients, and an estimate of concurrency-conflict chance — alongside the size dimensions: how many events or child objects accumulate, how long an instance lives, and how much is loaded per command. **Size** determines *d* (bigger loads, longer cycles); **throughput** determines λ.

The interview sentence: *"Before I fix the aggregate boundary I want the per-instance command rate. An aggregate is a serialization point, so its ceiling is roughly one over the read–decide–write time — around a hundred commits a second at ten milliseconds. If the hot instance needs more than that, no amount of hardware helps; the model has to change."*

---

## Concept 66 — Hot aggregates

When one aggregate instance receives more commands than its ceiling allows, you have a **hot aggregate**. The symptoms are distinctive: `DbUpdateConcurrencyException` rates that rise with load, a latency tail driven by retries, and throughput that plateaus while CPU and database are underused. The remedies, roughly from least to most invasive:

**1. Shrink the aggregate.** If the hot instance is hot because it contains unrelated things (Vernon's `Product`), split it. Often the whole fix.

**2. Split along the invariant.** A concert with 20,000 seats modelled as one `Show` aggregate is hopeless at on-sale time. Seat-level (or row/section-level) aggregates contend only when two buyers want the *same* seats. The invariant "a seat is sold at most once" is per seat, so the boundary can be per seat.

**3. Partition the quantity (escrow).** When the invariant is a *count* — "don't sell more than 10,000 units" — split the quantity into *k* buckets (ten buckets of 1,000), each its own aggregate; commands pick a bucket at random or by hash; a background process rebalances buckets that run low. Conflict rate drops by roughly *k*. This is Module 9's escrow technique applied as an aggregate design.

**4. Reserve now, confirm later.** Replace one contended "buy" command with a cheap "reserve with expiry" and a later "confirm." Reservations can be spread across buckets; confirmations are per reservation. Most ticketing and inventory systems work this way.

**5. Serialize in memory with a single writer.** Route all commands for an aggregate instance to one actor or one partitioned consumer (an Orleans grain, a Kafka partition keyed by aggregate ID). Commands queue instead of conflicting, and the aggregate's state lives in memory, so *d* shrinks from a database round trip to microseconds of processing plus an asynchronous persist. Throughput per instance can rise by one or two orders of magnitude (Concept 75).

**6. Use a conditional atomic write for a numeric bound.** For a pure counter invariant, a single conditional statement enforces it without a read–decide–write cycle:

```csharp
// "Decrement stock only if enough remains" — one statement, serialized by a row lock, no read beforehand
var updated = await db.StockLevels
    .Where(s => s.Sku == sku && s.Available >= quantity)
    .ExecuteUpdateAsync(s => s.SetProperty(x => x.Available, x => x.Available - quantity), ct);

if (updated == 0) return ReservationOutcome.InsufficientStock;
```

This bypasses the domain model — the rule is in SQL — and it also bypasses the change tracker, the interceptors and therefore your domain events and outbox (Module 19, Concept 34); if something must react to the change, write the outbox row explicitly in the same transaction. Reserve it for genuinely hot, genuinely simple invariants and document the exception. But it's the right tool when a counter is the bottleneck: the database serializes the updates in well under a millisecond each.

**7. Relax the invariant with the business.** Airlines overbook deliberately and compensate. A retailer may accept rare overselling and apologize. If the business prefers availability over the invariant (Module 7's trade-off in business terms), the rule becomes a policy enforced *eventually* with compensation (Concept 69).

The architect's version: *"A hot aggregate is a model problem, not an infrastructure problem. I'd first check whether the boundary contains more than the invariant needs, then split along the invariant, and only then reach for reservation, escrow or a single-writer actor."*

---

## Concept 67 — Growing collections are aggregates in disguise

A specific and very common sizing failure: an aggregate root with a child collection that **grows without bound** — `Customer.Orders`, `Account.Transactions`, `Forum.Posts`, `Device.Readings`.

Every command then loads (or at least tracks) an ever-growing collection; *d* grows over the aggregate's lifetime; eventually a routine operation times out on an old, busy customer. The collection also usually isn't protecting any invariant that needs *all* of its members.

The fix is to ask what invariant the collection actually serves:

- **None** → each member is its own aggregate, referencing the parent by ID. `Order` references `CustomerId`; `Customer` doesn't contain orders.
- **An invariant over a summary** → keep the **summary** in the parent and the members outside it. "An account's balance never falls below its overdraft limit" needs the *balance*, not every transaction. The `Account` aggregate holds `Balance` and enforces the limit; each posting is appended as a separate ledger record (or event) in the same transaction. The account stays small forever.
- **An invariant over a bounded window** → keep only the window. "At most 5 password-reset requests in 24 hours" needs the last 5 timestamps, not the history.

This is sometimes called a **slim aggregate**: it keeps only the state its decisions require. It's also the natural bridge to event sourcing (Module 24), where the full history lives in the event stream and the aggregate rebuilds only the few fields its decisions use.

The review heuristic: **any child collection on an aggregate root without a documented upper bound is a defect.** Write the bound down ("an order has at most 50 lines") and enforce it — or make the children separate aggregates.

---

## Concept 68 — Optimistic concurrency on aggregates in EF Core

Rule 7 (Concept 57) says the aggregate's invariants must hold at commit, which means **the whole aggregate is the unit of concurrency control**. EF Core doesn't know what an aggregate is, and the default mapping gets this subtly wrong.

**The trap.** Map `Order` with a SQL Server `rowversion` and `OrderLine` in its own table. Two requests arrive concurrently for an order with 49 lines and an invariant of "at most 50":

```
Request A: load order (49 lines, rowversion 0x01) → AddLine() passes → INSERT OrderLines …
Request B: load order (49 lines, rowversion 0x01) → AddLine() passes → INSERT OrderLines …
```

Both commit. The order now has 51 lines. Why didn't concurrency control catch it? Because **neither request modified the `Orders` row** — they only inserted child rows — so no `UPDATE Orders … WHERE RowVersion = @original` was ever issued. EF checks concurrency tokens only on the rows it writes. The aggregate's invariant was broken without an exception. (The same thing happens with a child *update* — changing a line's quantity touches only `OrderLines`.)

**The fix: make every change to the aggregate write the root.** Give the root an explicit version, configured as a concurrency token, and increment it whenever the aggregate changes:

```csharp
public abstract class AggregateRoot<TId> where TId : struct
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public TId Id { get; protected init; }
    public int Version { get; private set; }                          // concurrency token for the whole aggregate
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents;

    // Convention: every state change raises a domain event, so raising one also bumps the version.
    protected void Raise(IDomainEvent @event)
    {
        _domainEvents.Add(@event);
        Version++;
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Mapping
b.Property(o => o.Version).IsConcurrencyToken();
b.Ignore(o => o.DomainEvents);
```

Now request A's `SaveChanges` issues `UPDATE Orders SET Version = 50 WHERE Id = @id AND Version = 49` alongside its insert, in one transaction. Request B's identical update affects zero rows, EF throws `DbUpdateConcurrencyException`, and B's transaction — including its child insert — rolls back. The invariant holds.

Notes on the design:

- **Tying the version bump to `Raise`** makes it impossible to forget *as long as every state change raises an event* — a convention worth enforcing anyway (Part G). If some changes don't raise events, bump explicitly in a `protected void Touch()` called from those methods, and add a test per command asserting that `Version` changed.
- **Incrementing more than once per transaction is harmless** — the `WHERE` clause uses the *original* value.
- **A `rowversion` on the root doesn't help** by itself for child-only changes, because nothing writes the root row. You can keep a `rowversion` for other reasons, but the aggregate-level guard is the explicit version.
- **Alternative: store the children in the root's row.** With EF Core 10's `ComplexCollection(…).ToJson()` (Concept 47), an order's lines can be a JSON column on the `Orders` row — valid when lines are genuinely *values* (replaced rather than individually tracked). Then *every* change writes the root row, and ordinary row-level concurrency covers the whole aggregate. The trade-off is queryability of lines and row size.
- **The version doubles as an HTTP ETag.** Return it as `ETag` on GET, require `If-Match` on updates made by humans, and reject stale edits with 412 Precondition Failed (Concept 69).

This is one of the highest-signal details in the module. *"EF only checks concurrency on rows it writes, so if a command only touches child rows, two concurrent commands can both pass the aggregate's invariant. I version the aggregate root explicitly and bump it on every change, so every command writes the root row."*

---

## Concept 69 — Resolving conflicts, and why race conditions don't exist

A concurrency exception tells you two commands collided. What happens next is a design decision.

**Strategy 1 — Retry the command (for machine-issued commands).** Reload the aggregate at its new version and **re-run the whole decision** — not "re-apply the same changes," which would skip the invariant checks. This is safe precisely because the decision is a pure function of (command, current state): if the new state still permits the command, it succeeds; if not, it's rejected for a real reason.

```csharp
public async Task<PlaceOrderResult> Handle(PlaceOrder cmd, CancellationToken ct)
{
    for (var attempt = 1; ; attempt++)
    {
        var order = await _orders.GetAsync(cmd.OrderId, ct);
        if (order is null) return PlaceOrderResult.NotFound;

        var outcome = order.Place(cmd.CreditAvailable, _clock.GetUtcNow());   // the full decision, on fresh state
        if (outcome is not PlaceOutcome.Placed) return PlaceOrderResult.From(outcome);

        try
        {
            await _unitOfWork.SaveChangesAsync(ct);
            return PlaceOrderResult.Placed;
        }
        catch (DbUpdateConcurrencyException) when (attempt < 3)
        {
            _unitOfWork.Clear();   // detach everything; the next attempt reloads at the new version
        }
    }
}
```

Keep the retry count small, add jitter under load, and put the loop in one place (a pipeline behaviour or a small wrapper), not in every handler. The retry is only safe because nothing irreversible happened before `SaveChanges` — no emails, no external calls — which is one more reason those belong behind the outbox (Concept 82).

**Strategy 2 — Surface the conflict (for human edits).** When a person edited a form based on what they saw, silently retrying their change on top of someone else's may produce something neither intended. Use the aggregate version as an ETag: the client sends `If-Match`, a mismatch returns **412 Precondition Failed** (or 409 with problem details), and the UI shows "this was changed by someone else — review and resubmit."

**Strategy 3 — Merge (for commutative changes).** If two changes commute — adding two different tags, incrementing a counter — they can be merged automatically without a conflict. This is a modelling choice (Module 9's CRDT thinking applied to one aggregate) and usually means restructuring so the commuting parts don't share a version.

**Udi Dahan's *Race Conditions Don't Exist* (2010)** is the essay to cite for the deeper point: a race condition in business software is usually a **missing business rule**. "What if a customer cancels an order at the same moment the warehouse ships it?" isn't a question for the database — it's a question for the business, and the business already has an answer: once shipped, a cancellation becomes a return. If a millisecond difference in timing would change the business outcome, the model is missing a concept (a cancellation *request*, a return, a cutoff time). Model the business's actual policy, and the "race" becomes two well-defined outcomes.

---

## Concept 70 — Set-based rules and uniqueness

The classic hard case: **"no two customers may share an email address."** It's an invariant over the *set* of all customers, so it can't live in one `Customer` aggregate, and one `AllCustomers` aggregate would be a hot aggregate by definition. The options, and what each gives up (Khorikov's trilemma from Concept 51 — completeness, purity, performance):

**1. A unique constraint in the database, translated into a domain outcome.** The most common and usually the right answer.

```csharp
try
{
    await _unitOfWork.SaveChangesAsync(ct);
}
catch (DbUpdateException ex) when (ex.IsUniqueViolation("UX_Customers_Email"))
{
    return RegisterCustomerResult.EmailAlreadyInUse;
}
```

Keeps purity and performance; gives up some completeness (the rule's enforcement lives in the database schema, not the model). Write a test that proves the constraint exists and maps correctly, and name the constraint so the translation is explicit.

**2. Check first in the application service, then rely on the constraint.** Gives a better error message in the common case; the constraint still guards the race window between check and insert. Never rely on the check alone.

**3. A domain service with a repository-backed checker passed into the aggregate** (`IEmailUniquenessChecker`). Keeps completeness; sacrifices purity (the domain now does I/O), and still has a race unless backed by the constraint.

**4. Make the unique value the identity of a small aggregate.** Model an `EmailRegistration` aggregate whose **ID is the normalized email address itself**. Registering a customer creates an `EmailRegistration(email → customerId)`; a second registration with the same email is an insert of a duplicate primary key — the uniqueness rule becomes a single-aggregate existence check, which is exactly what aggregates are good at. This generalizes to any uniqueness rule in any store that enforces key uniqueness (including document databases and event stores where the stream ID is the email), and it's the pattern to name for distributed or NoSQL settings. Changing the email becomes a small process: claim the new registration, update the customer, release the old one.

**5. Dynamic Consistency Boundary** in an event-sourced store: the append condition "no `EmailClaimed` event with this email tag exists" enforces uniqueness directly (Concept 74).

**6. Accept duplicates and resolve them.** For low-stakes uniqueness (display names, nicknames), detect duplicates asynchronously and ask one party to change. Cheap, and often what the business actually wants.

The senior framing: *"Uniqueness is a set-based rule, so it doesn't fit inside one aggregate. In a relational store I'd use a unique index and translate the violation into a domain result, with a pre-check for a friendly message. In a store without secondary unique indexes, I'd make the email the identity of a small registration aggregate, which turns the set rule into a key-uniqueness rule."*

---

## Concept 71 — Cross-aggregate invariants

Beyond uniqueness, some rules genuinely span aggregates: "a team has at most 10 members and a person belongs to at most 3 teams"; "transfer money between two accounts atomically"; "a course has at most 30 students and a student takes at most 5 courses." The options, with their costs:

**1. Re-examine the boundaries.** Sometimes the rule reveals a missing concept. A `Membership` aggregate (person–team pair) with the counts maintained separately, or a `Roster` aggregate per team, may express the rule better than either `Team` or `Person` could.

**2. Co-transact in one database (Concept 63, reason 5).** Load both aggregates, check the rule, modify both, save in one local transaction, with both versions checked. Correct and simple *inside one database and one context* when contention is low. For transfers, **lock/update in a consistent order (e.g. by account ID)** to avoid deadlocks (Module 12).

**3. Reservation.** Reserve capacity on each side first (each a single-aggregate operation that can fail cleanly), then confirm, with expiry for abandoned reservations. Works across databases and services.

**4. Process manager with compensation.** Perform the steps as separate transactions — withdraw from A, deposit to B — with a process manager that tracks progress and compensates on failure (Concept 85; Module 12's sagas). For money, model the in-between state honestly (funds "in transit") rather than pretending it doesn't exist; that's what real banking does.

**5. Dynamic Consistency Boundary** for event-sourced systems: one append condition covering both entities' events (Concept 74).

**6. Detect and correct.** Let the rule be violated briefly, detect violations with a policy, and correct them — when the business already works that way.

The decision heuristic: **inside one database and one context with low contention → co-transact deliberately; across databases or services → reservation or process manager; event-sourced → consider DCB; when the business already compensates → detect and correct.**

---

## Concept 72 — Aggregates as state machines

Most interesting aggregates have a **lifecycle** — a claim is submitted, assessed, accepted or rejected, settled, closed, possibly reopened — and the most important rules are about which transitions are allowed from which state. Make the state machine explicit rather than scattering status checks:

```csharp
public enum ClaimStatus { Submitted, UnderAssessment, LiabilityAccepted, Rejected, Settled, Closed, Reopened }

public sealed class Claim : AggregateRoot<ClaimId>
{
    private static readonly FrozenDictionary<ClaimStatus, ClaimStatus[]> Transitions =
        new Dictionary<ClaimStatus, ClaimStatus[]>
        {
            [ClaimStatus.Submitted]         = [ClaimStatus.UnderAssessment, ClaimStatus.Rejected],
            [ClaimStatus.UnderAssessment]   = [ClaimStatus.LiabilityAccepted, ClaimStatus.Rejected],
            [ClaimStatus.LiabilityAccepted] = [ClaimStatus.Settled],
            [ClaimStatus.Rejected]          = [ClaimStatus.Reopened, ClaimStatus.Closed],
            [ClaimStatus.Settled]           = [ClaimStatus.Closed, ClaimStatus.Reopened],
            [ClaimStatus.Reopened]          = [ClaimStatus.UnderAssessment],
            [ClaimStatus.Closed]            = [ClaimStatus.Reopened],
        }.ToFrozenDictionary();

    public ClaimStatus Status { get; private set; }

    public void AcceptLiability(Money reserve, AdjusterId by, Money adjusterAuthority)
    {
        EnsureCanMoveTo(ClaimStatus.LiabilityAccepted);                 // check every rule first…
        if (reserve.Amount > adjusterAuthority.Amount)
            throw new DomainException("Reserve exceeds adjuster authority; refer to a supervisor.");

        Status = ClaimStatus.LiabilityAccepted;                          // …then change state, all or nothing
        Reserve = reserve;
        Raise(new LiabilityAccepted(Id, reserve, by));
    }

    private void EnsureCanMoveTo(ClaimStatus next)
    {
        if (!Transitions[Status].Contains(next))
            throw new DomainException($"A claim cannot move from {Status} to {next}.");
    }
    // … Reserve and other members omitted
}
```

The transition table is a single, reviewable statement of the lifecycle — the kind of thing a domain expert can check line by line. It also makes "what can happen next?" queryable for the UI.

**States as types** (Concept 49) go further: each state carries only its own data, and C# 15's `closed` hierarchies make handling exhaustive. The practical difficulty is persistence: EF Core's complex types don't support inheritance, so a polymorphic state is usually stored either as a status column plus nullable per-state columns *encapsulated by the aggregate*, or as a JSON column via a `System.Text.Json` polymorphic converter (`[JsonPolymorphic]`/`[JsonDerivedType]`, a discriminator you control). A reasonable split: **use states-as-types where the per-state data is rich and the switches are many; use a status enum plus a transition table where the states mostly gate behaviour.** Either way, the invariant is the same — transitions happen only through named methods that check them.

---

## Concept 73 — The functional aggregate: the Decider

Jérémie Chassaing's *Functional Event Sourcing Decider* (2021) reduces an aggregate to three things:

- **`initialState`** — the state before anything happened.
- **`decide : Command × State → Events`** — given a command and the current state, return the events that should happen (or a rejection). **Pure.**
- **`evolve : State × Event → State`** — apply one event to a state. **Pure.**

Current state is a left fold: `state = events.Aggregate(initialState, evolve)`. Handling a command is: fold, decide, append.

In C#, with C# 15 closed hierarchies for the command, event and outcome sets (on .NET 10, use abstract records and a throwing discard arm):

```csharp
public closed record SectionCommand;
public sealed record OpenSection(IReadOnlySet<SeatNo> Seats) : SectionCommand;
public sealed record HoldSeats(HoldId Hold, CustomerId Customer, IReadOnlySet<SeatNo> Seats, DateTimeOffset At) : SectionCommand;
public sealed record ConfirmHold(HoldId Hold) : SectionCommand;
public sealed record ExpireHolds(DateTimeOffset Now) : SectionCommand;

public closed record SectionEvent;
public sealed record SectionOpened(IReadOnlySet<SeatNo> Seats) : SectionEvent;
public sealed record SeatsHeld(HoldId Hold, CustomerId Customer, IReadOnlySet<SeatNo> Seats, DateTimeOffset ExpiresAt) : SectionEvent;
public sealed record HoldConfirmed(HoldId Hold) : SectionEvent;
public sealed record HoldExpired(HoldId Hold, IReadOnlySet<SeatNo> Seats) : SectionEvent;

public closed record Decision;
public sealed record Accepted(IReadOnlyList<SectionEvent> Events) : Decision;
public sealed record Rejected(string Reason) : Decision;

public sealed record Hold(HoldId Id, IReadOnlySet<SeatNo> Seats, DateTimeOffset ExpiresAt);
public sealed record SectionState(ImmutableHashSet<SeatNo> Available, ImmutableDictionary<HoldId, Hold> Holds);

public static class SectionDecider
{
    public static readonly SectionState Initial =
        new(ImmutableHashSet<SeatNo>.Empty, ImmutableDictionary<HoldId, Hold>.Empty);

    public static Decision Decide(SectionCommand command, SectionState state) => command switch
    {
        OpenSection when !state.Available.IsEmpty || !state.Holds.IsEmpty
            => new Rejected("The section is already open."),
        OpenSection c
            => new Accepted([new SectionOpened(c.Seats)]),

        HoldSeats c when c.Seats.Count is 0 or > 8
            => new Rejected("A hold must contain between 1 and 8 seats."),
        HoldSeats c when !c.Seats.All(state.Available.Contains)
            => new Rejected("One or more seats are no longer available."),
        HoldSeats c
            => new Accepted([new SeatsHeld(c.Hold, c.Customer, c.Seats, c.At.AddMinutes(10))]),

        ConfirmHold c when !state.Holds.ContainsKey(c.Hold)
            => new Rejected("The hold has expired or does not exist."),
        ConfirmHold c
            => new Accepted([new HoldConfirmed(c.Hold)]),

        ExpireHolds c
            => new Accepted([.. state.Holds.Values
                                 .Where(h => h.ExpiresAt <= c.Now)
                                 .Select(h => new HoldExpired(h.Id, h.Seats))]),
    };

    public static SectionState Evolve(SectionState s, SectionEvent e) => e switch
    {
        SectionOpened o => s with { Available = s.Available.Union(o.Seats) },
        SeatsHeld h     => s with { Available = s.Available.Except(h.Seats),
                                    Holds = s.Holds.Add(h.Hold, new Hold(h.Hold, h.Seats, h.ExpiresAt)) },
        HoldConfirmed c => s with { Holds = s.Holds.Remove(c.Hold) },            // seats stay unavailable: sold
        HoldExpired x   => s with { Available = s.Available.Union(x.Seats),
                                    Holds = s.Holds.Remove(x.Hold) },
    };
}
```

Why this shape is worth knowing:

- **It's the most testable form an aggregate can take.** A test is: given these events, when this command, then these events (or this rejection). No mocks, no database, no clock (time comes in on the command) — Concept 93.
- **It separates deciding from persisting completely.** The same decider works with an **event store** (fold the stream, append the new events with the expected version) or a **state store** (load the state, decide, `newState = events.Aggregate(state, Evolve)`, save with a version check). Choosing event sourcing later (Module 24) doesn't change the domain code.
- **It makes the aggregate's contract explicit.** The command set, event set and state are closed, named types in the ubiquitous language — which is exactly what an EventStorming wall produced (Concept 36).
- **It composes.** Deciders can be combined — two deciders running side by side over a combined state — which is one way to express rules across what would otherwise be separate aggregates.

The OO aggregate of Concept 72 and the functional decider are the same pattern with different packaging: both are a consistency boundary with a decision function. Teams on EF Core with a state-stored model usually use the OO form; teams moving toward event sourcing, or wanting maximal testability, increasingly use the decider. A candidate who can show both — and say they're equivalent — is demonstrating understanding rather than style preference.

---

## Concept 74 — Dynamic Consistency Boundary

The most significant challenge to the aggregate in years, and a live topic in 2025–2026.

**The problem it addresses.** In event sourcing, the aggregate is also the **event stream**: each aggregate instance has a stream, and optimistic concurrency is "append to stream X only if its version is still *n*." That makes the consistency boundary *fixed at design time*. When a rule spans two entities — the canonical example is "a course has at most 30 students *and* a student can take at most 5 courses" — you must choose between a bigger aggregate (contention), a saga (complexity and eventual consistency), or a reservation dance.

**The idea.** Sara Pellegrini's blog post *Killing the Aggregate* proposed dropping per-aggregate streams: every event lives in one ordered log per bounded context and carries **tags** naming the domain concepts it concerns (`course:c1`, `student:s7`). A decision **reads the events matching a query** (by type and tags), decides, and appends new events with an **append condition**: *fail if any event matching this query has been appended after position p* (the position of the last event the decision read). The consistency boundary is therefore **defined per decision, by the query** — dynamically — rather than per object.

**The specification.** Formalized at **dcb.events** by Bastian Waidelich, Sara Pellegrini and Paul Grimshaw: events have a type, data and tags; a query is a set of query items (types OR'd, tags AND'd within an item; items OR'd); a store must support reading by query and appending atomically with an append condition (`failIfEventsMatch` plus an optional `after` position). Implementations exist in several languages; **Axon Framework 5 with Axon Server 2025.1** added DCB support (initially experimental), and **Marten 9.0** (May 2026) supports it in .NET.

In Marten, adapted from its DCB documentation (Marten 9.x — check the current docs for exact signatures):

```csharp
// Registration: tag types are strongly typed IDs; each can drive a boundary aggregate
opts.Events.RegisterTagType<CourseId>("course").ForAggregate<EnrollmentBoundary>();
opts.Events.RegisterTagType<StudentId>("student").ForAggregate<EnrollmentBoundary>();

// The decision's state: built from every event tagged with this course OR this student.
// A "pure" boundary aggregate with no single-stream identity needs Marten's [BoundaryAggregate] marker.
[BoundaryAggregate]
public sealed class EnrollmentBoundary
{
    public Dictionary<CourseId, int> StudentsPerCourse { get; } = new();
    public Dictionary<StudentId, int> CoursesPerStudent { get; } = new();

    public void Apply(StudentEnrolled e)
    {
        StudentsPerCourse[e.Course] = StudentsPerCourse.GetValueOrDefault(e.Course) + 1;
        CoursesPerStudent[e.Student] = CoursesPerStudent.GetValueOrDefault(e.Student) + 1;
    }
}

// Handling "enroll student in course"
var query = new EventTagQuery().Or<CourseId>(courseId).Or<StudentId>(studentId);
var boundary = await session.Events.FetchForWritingByTags<EnrollmentBoundary>(query);
var state = boundary.Aggregate ?? new EnrollmentBoundary();

if (state.StudentsPerCourse.GetValueOrDefault(courseId) >= 30) return EnrollOutcome.CourseFull;
if (state.CoursesPerStudent.GetValueOrDefault(studentId) >= 5)  return EnrollOutcome.StudentAtLimit;

var enrolled = session.Events.BuildEvent(new StudentEnrolled(studentId, courseId));
enrolled.WithTag(studentId, courseId);
boundary.AppendOne(enrolled);

await session.SaveChangesAsync(ct);   // DcbConcurrencyException if a matching event was appended since the read
```

Both rules are enforced atomically, with no `Course` or `Student` aggregate owning the other, and a conflict occurs **only** if another enrollment touched *this course or this student* in the meantime.

**What it buys:**

- Cross-entity invariants without oversized aggregates or sagas.
- Contention scoped to exactly what the decision read — often less than a fixed aggregate.
- Refactorable boundaries: because the event log is one sequence with tags, changing a decision's boundary means changing its query, not migrating streams.
- Some classic hard cases become simple: uniqueness ("no `UsernameClaimed` event with this tag exists"), gapless sequences (read the single latest `InvoiceNumbered` event), idempotency keys.

**What it costs:**

- **It requires event sourcing and a DCB-capable store.** It doesn't directly change a state-stored EF Core model — which is most .NET systems.
- **Tag discipline becomes correctness-critical.** An event missing a tag is silently excluded from every boundary that should include it — the one failure a consistency mechanism must not have. (Marten added declarative tag rules precisely to close this gap.)
- **Query-based concurrency has a cost** — tag indexes, per-tag version tracking — and popular tags are still hot: every enrollment in a popular course conflicts with every other.
- **Reads per decision** may pull more events than a snapshot-able stream, so projection and caching strategy changes.
- **Maturity.** The specification and most implementations are young; production experience is still accumulating.

**The first-principles view — and the senior sentence.** DCB is to event stores what serializable snapshot isolation's predicate-level conflict detection is to relational databases (Module 12): *the boundary is "whatever data the decision read."* It doesn't abolish consistency boundaries; it makes them **per decision instead of per object**. In a relational system you get a similar effect today with conditional writes whose `WHERE` clause expresses the invariant (Concept 66) or with `SERIALIZABLE` over the rows a decision reads. *"DCB doesn't kill the need to think about consistency boundaries — it moves the boundary from the class to the decision. I'd consider it for an event-sourced core with genuine cross-entity rules; for a state-stored EF model, aggregates are still the right tool."*

---

## Concept 75 — Aggregates at runtime scale

Because the aggregate is the unit of consistency, it's also the natural unit for **placement** — and the infrastructure patterns that scale systems line up with it neatly:

- **Partition key.** In Cosmos DB, a logical partition supports transactional batches and stored procedures within that partition only (Module 8, Module 27). Make the partition key the aggregate ID (or a key that co-locates everything one aggregate needs) and **cross-partition = cross-aggregate = eventually consistent** — the same rule, enforced by the platform. The same logic applies to shard keys in a sharded relational setup.
- **Message ordering key.** In Kafka or Service Bus sessions, keying messages by aggregate ID gives per-aggregate ordering, which is exactly the ordering domain events need (Module 11).
- **Actor / virtual actor.** A **Microsoft Orleans grain** per aggregate instance is a natural fit: a grain activation has a single-threaded, turn-based execution model and by default processes each request to completion before starting the next, so commands to one aggregate are serialized in memory rather than conflicting at the database. Keep optimistic concurrency (ETags) on the persisted state anyway: depending on the grain directory in use, single activation has historically been best-effort under failures, and a stale activation must never be able to overwrite newer state. (Reentrancy options relax the one-request-at-a-time model; for an aggregate host, leave them off.) Dapr actors offer the same shape across languages. This is Concept 66's "single writer" remedy as an architecture.
- **Cache key.** An aggregate is the natural unit of caching for write-side reads — with the caveat that caching aggregates for *writes* requires version checks on save anyway.

The unifying sentence: *"I'd make the aggregate ID the partition key, the message key and the actor key. Then the platform enforces the same boundary the model does — anything that crosses aggregates also crosses partitions, and it's eventual by construction."*

---

## Concept 76 — The aggregate design checklist

The DDD Crew's **Aggregate Design Canvas** is the best one-page tool for designing or reviewing an aggregate. Its sections, condensed into the checklist you'd walk in a design review:

1. **Name and description** — in the ubiquitous language; one sentence of purpose.
2. **State transitions** — the lifecycle (Concept 72).
3. **Enforced invariants** — each one written as a sentence the business would recognize. *If this list is empty, it shouldn't be an aggregate.*
4. **Corrective policies** — rules enforced *eventually* by reacting to events, which therefore live outside this boundary.
5. **Handled commands** — every command, with the invariants each one checks.
6. **Created events** — every event, named in the past tense.
7. **Throughput** — average and peak command rate per instance, number of concurrent clients, estimated conflict chance (Concept 65).
8. **Size** — growth rate of children or events, lifetime of an instance, amount loaded per command (Concept 67).

And the review questions that go with it:

- Does every object inside the boundary participate in at least one invariant?
- Is every reference to another aggregate by identity?
- Does every child collection have a documented upper bound?
- Is the root the only public entry point for changes? Are there any public setters?
- Does every state change raise an event and bump the version (Concept 68)?
- What happens when two commands arrive concurrently for the same instance — and has anyone tested it?
- Is the peak per-instance command rate comfortably below 1/*d*?
- Which of the invariants are set-based, and how are they enforced?

A reviewer who walks these eight questions will find most aggregate defects before they reach production.

---

# Part G — Domain events

If aggregates are small and change one per transaction, something has to carry the consequences of a change to everything else. Domain events are that mechanism — inside a context between aggregates, and, once translated, between contexts. Modules 11, 19 and 20 covered the delivery mechanics (outbox, dispatch placement, domain vs. integration events); this part covers the modelling: what an event *is*, how to design one, how it flows from an aggregate to the outside world, and what eventual consistency looks like to the people using the system.

---

## Concept 77 — Domain events, defined

**A domain event is a record of something that happened in the domain that domain experts care about**, expressed in the ubiquitous language, in the past tense, and immutable.

Evans' 2003 book didn't include domain events as a pattern; they emerged from community practice over the following decade (Udi Dahan, Greg Young and others) and Evans added them to his 2015 *DDD Reference* as one of three patterns new since the book. That history matters for one reason: domain events are now considered part of the *model*, not an infrastructure add-on. They're the tactical counterpart of EventStorming's orange stickies — the same facts, now in code.

Properties:

- **Past tense, factual.** `ClaimSubmitted`, `LiabilityAccepted`, `PaymentCaptured`. An event states that something *did* happen; it can't be rejected by a receiver. If a handler could "refuse" it, it's a command in disguise.
- **Named in the language.** The experts would recognize the name, and ideally used it on the EventStorming wall.
- **Immutable.** An event is history. Records are the natural C# shape.
- **Raised by the aggregate that made the decision**, as part of the state change — not by an application service observing the result.
- **Carries what happened**, with the identifiers and facts needed to understand it.

```csharp
public interface IDomainEvent
{
    DateTimeOffset OccurredAt { get; }
}

public sealed record LiabilityAccepted(
    ClaimId ClaimId,
    Money Reserve,
    AdjusterId AcceptedBy,
    DateTimeOffset OccurredAt) : IDomainEvent;
```

Why they're valuable beyond wiring: they make **side effects explicit and decoupled**. Without events, `Claim.AcceptLiability` either knows about everything that should follow (scheduling payment, notifying the claimant, updating reserves reporting) — coupling the aggregate to every downstream concern — or the application service does it all in one place, and the next use case that accepts liability forgets half of it. With events, the aggregate states the fact once, and each consequence is a separate, independently testable handler.

---

## Concept 78 — Kinds of events

"Event" means several different things, and interviewers probe whether you distinguish them. Two distinctions matter.

**Domain events vs. integration events** (Module 20, Concept 45 — the short version):

| | Domain event | Integration event |
|---|---|---|
| **Scope** | Inside one bounded context | Crosses a context (or service) boundary |
| **Shape** | Rich, uses domain types | Flat, primitives, versioned — the published language |
| **Stability** | Changes when the model changes | Changes slowly, additively |
| **Delivery** | In-process, often inside the transaction | Outbox → broker → at-least-once → idempotent consumers |

**Martin Fowler's four meanings of "event-driven"** (*What do you mean by "Event-Driven"?*, 2017) — worth having ready because they explain *what an event is for*:

1. **Event notification** — "something happened; ask me if you need details." Thin events, minimal coupling, but receivers often call back for data, reintroducing temporal coupling.
2. **Event-carried state transfer** — the event carries the state receivers need, so they can maintain their own copy without calling back. Decouples at the cost of larger events and replicated data (Module 21, Concept 38's option 2).
3. **Event sourcing** — events *are* the system of record; state is derived from them (Module 24).
4. **CQRS** — separate write and read models, frequently connected by events (Module 23).

A single domain event can serve several of these roles at once, but the *integration* event you publish should be deliberately designed for one: usually event-carried state transfer for data that consumers replicate, or notification for pure triggers.

---

## Concept 79 — Designing events

Three design rules that separate useful events from noise:

**1. Name the business intent, not the data change.** `CustomerUpdated` or `OrderStatusChanged` says a row changed; it doesn't say *why*, and consumers have to reverse-engineer the business meaning from a diff. Compare:

| CRUD-shaped | Intent-revealing |
|---|---|
| `CustomerAddressUpdated` | `CustomerRelocated` (moved house) vs. `CustomerAddressCorrected` (fixed a typo) |
| `OrderStatusChanged { Status = 4 }` | `OrderShipped`, `OrderCancelledByCustomer`, `OrderCancelledForFraud` |
| `PolicyModified` | `PolicyRenewed`, `CoverageExtended`, `PolicyLapsed` |

The distinction is often business-critical: a relocation might trigger a re-rating of an insurance premium and a new tax jurisdiction; a typo correction must not. If both arrive as `AddressUpdated`, every consumer has to guess.

**2. Carry what consumers need to decide, not everything you know.** For domain events inside a context, include the aggregate ID and the facts that changed. For integration events, the balance is between **thin** (IDs only — consumers call back, creating temporal coupling) and **fat** (the whole aggregate — every internal change becomes a contract change). The usual answer is the *facts of this change plus the data most consumers need to act without calling back*, and nothing that exposes internal structure.

**3. Include metadata for the mechanics.** Consumers need to deduplicate, order and trace events, so every integration event should carry — usually in an envelope rather than the payload — an **event ID** (idempotency), the **aggregate ID and version** (ordering and gap detection), **occurred-at**, and **correlation and causation IDs** (tracing a chain of events back to the command that started it — Module 28).

Granularity follows from the language: one event per business fact. A single command often produces several events (`OrderPlaced`, and `LoyaltyThresholdReached` if the order pushed the customer over a threshold) — that's fine and usually clearer than one event with optional fields.

---

## Concept 80 — Raising events

Three ways for an aggregate to raise events, with trade-offs:

**1. Collect on the aggregate root (the mainstream choice).** The root holds a list; methods add to it; infrastructure drains it at save time. This is the pattern in Concept 68's `AggregateRoot` base class, in Microsoft's eShop, and in Jimmy Bogard's influential 2014 post *A better domain events pattern*.

- ✅ The aggregate stays pure — no dependencies, no I/O.
- ✅ Events and state changes are saved together (with the outbox, atomically).
- ✅ Easy to test: call the method, inspect `DomainEvents`.
- ⚠️ Someone must remember to clear the list after dispatch, and events are only dispatched when the aggregate is saved through the unit of work.

**2. Return events from methods (the functional choice).** `IReadOnlyList<IDomainEvent> Place(...)` or the decider's `Decide` (Concept 73). The caller is responsible for persisting and dispatching them.

- ✅ Maximally explicit and testable; no hidden state on the aggregate.
- ✅ Natural fit for event sourcing.
- ⚠️ Every application service must handle the returned events correctly — more ceremony in a state-stored system.

**3. A static `DomainEvents.Raise(...)` (the historical choice).** Udi Dahan's 2009 post *Domain Events – Salvation* popularized a static raiser that dispatched immediately to registered handlers.

- ❌ Handlers run *before* the transaction commits, so a rollback leaves side effects behind (an email sent for an order that was never saved).
- ❌ A hidden static dependency — hard to test, hard to reason about under concurrency.
- It's worth knowing because it's in older codebases and older blog posts; Bogard's 2014 post was largely a response to its problems, and the community moved to option 1.

The recommendation to state: *"I collect events on the aggregate root and drain them during `SaveChanges`, so they're committed atomically with the state change through the outbox. If the team is moving toward event sourcing, I'd return them from a decider instead — same idea, more explicit."*

---

## Concept 81 — Dispatching events

Module 19 (Concept 36) gave you the three placements — **before `SaveChanges` in-process**, **after `SaveChanges` in-process**, and **the outbox** — and the rule: in-transaction handlers for effects that stay in your own database and must be consistent with the write; the outbox for anything that leaves the process. Here's the full mechanism, without a mediator library (MediatR 13+ is commercial — Module 21, Concept 49 — and the domain model shouldn't care either way):

```csharp
public interface IDomainEventHandler<in TEvent> where TEvent : IDomainEvent
{
    Task Handle(TEvent domainEvent, CancellationToken ct);
}

// A ~25-line dispatcher: resolves handlers from DI, caches one delegate per event type.
internal sealed class DomainEventDispatcher(IServiceProvider services) : IDomainEventDispatcher
{
    private delegate Task Invoker(IServiceProvider sp, IDomainEvent e, CancellationToken ct);
    private static readonly ConcurrentDictionary<Type, Invoker> Invokers = new();

    public Task DispatchAsync(IDomainEvent e, CancellationToken ct) =>
        Invokers.GetOrAdd(e.GetType(), Build)(services, e, ct);

    private static Invoker Build(Type eventType) =>
        typeof(DomainEventDispatcher)
            .GetMethod(nameof(InvokeAll), BindingFlags.NonPublic | BindingFlags.Static)!
            .MakeGenericMethod(eventType)
            .CreateDelegate<Invoker>();

    private static async Task InvokeAll<TEvent>(IServiceProvider sp, IDomainEvent e, CancellationToken ct)
        where TEvent : IDomainEvent
    {
        foreach (var handler in sp.GetServices<IDomainEventHandler<TEvent>>())
            await handler.Handle((TEvent)e, ct);
    }
}

// Drains events during SaveChanges, in the same transaction, until the cascade settles.
internal sealed class DomainEventsInterceptor(IDomainEventDispatcher dispatcher) : SaveChangesInterceptor
{
    private const int MaxRounds = 5;

    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default)
    {
        var context = eventData.Context!;
        for (var round = 1; ; round++)
        {
            var raisers = context.ChangeTracker.Entries<IAggregateRoot>()
                .Select(entry => entry.Entity)
                .Where(aggregate => aggregate.DomainEvents.Count > 0)
                .ToList();

            if (raisers.Count == 0) break;
            if (round > MaxRounds)
                throw new InvalidOperationException($"Domain event cascade did not settle after {MaxRounds} rounds.");

            var events = raisers.SelectMany(a => a.DomainEvents).ToList();
            raisers.ForEach(a => a.ClearDomainEvents());

            foreach (var domainEvent in events)
                await dispatcher.DispatchAsync(domainEvent, ct);   // handlers may raise more events → next round
        }

        return result;   // EF detects changes made by handlers (new outbox rows, modified entities) after this point
    }
}
```

Points a reviewer looks for:

- **The loop.** Handlers can modify aggregates that raise further events; a single pass would silently drop them. The round limit turns an accidental event cycle into a loud failure instead of a hang.
- **Clear before dispatch**, so a handler that re-enters `SaveChanges` (it shouldn't) or raises on the same aggregate doesn't re-dispatch old events.
- **In-transaction handlers write to the database only** — including writing integration events to the outbox (Concept 83). Anything with an external side effect happens after commit, from the outbox (Concept 82).
- **Lifetimes.** The interceptor and dispatcher are scoped (they resolve scoped handlers), which works with `AddDbContext` but **not with `AddDbContextPool`**, whose options are singleton-scoped (Module 19, Concept 39). With pooling, resolve handlers from the current scope some other way.
- **AOT.** `MakeGenericMethod` isn't trimming/AOT-friendly; for Native AOT, generate the dispatch table with a source generator or register a typed invoker per event.

---

## Concept 82 — What event handlers may do

The rules of engagement, because this is where designs quietly go wrong:

**Inside the transaction (in-process handlers during `SaveChanges`), a handler may:**

- **Write integration events to the outbox** — the most common and most important use (Concept 83).
- **Update derived data in the same context** that must be consistent with the write — a denormalized counter, a search-projection row in the same database.
- **Modify another aggregate in the same context** — *only* as a deliberate, documented application of Concept 63's pragmatic exception (low contention, same database, same context). Microsoft's eShop does this (an order-started event updates the buyer aggregate in the same transaction); know that it's a multi-aggregate transaction wearing an event's clothing, and decide it on purpose.

**Inside the transaction, a handler must not:**

- **Perform I/O with external effects** — send email, call an HTTP API, publish directly to a broker. The transaction may still roll back, and you can't un-send an email. That's the dual-write problem (Module 11), and the outbox exists to solve it.
- **Call `SaveChanges`** — it's already inside one.
- **Do slow work** — every millisecond extends the transaction and raises the conflict rate (Concept 65).

**After commit (outbox-driven handlers), a handler:**

- Runs in its **own transaction**, per event, possibly in another process.
- Must be **idempotent** — at-least-once delivery means it will see duplicates (store processed event IDs, or make the operation naturally idempotent).
- Must tolerate **reordering** unless the transport guarantees per-aggregate ordering (Concept 75's message key).
- May modify **one** aggregate (in its own context) and raise further events — which is how eventual consistency between aggregates actually happens (Concept 62).

The one-line rule: **same-context, same-database, must-be-consistent → in the transaction; everything else → the outbox.**

---

## Concept 83 — Translating at the boundary

The moment a domain event is translated into an integration event is the moment a bounded context **writes its published language** (Concept 28). It deserves to be an explicit, reviewed piece of code rather than a generic serializer:

```csharp
// Ordering context: translate the internal fact into the published contract, inside the same transaction
internal sealed class PublishOrderPlaced(IOutbox outbox) : IDomainEventHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced e, CancellationToken ct)
    {
        outbox.Enqueue(new Ordering.Contracts.V1.OrderPlacedV1(
            OrderId:     e.OrderId.Value,
            CustomerId:  e.CustomerId.Value,
            TotalAmount: e.Total.Amount,
            Currency:    e.Total.Currency,
            LineCount:   e.Lines.Count,
            PlacedAtUtc: e.PlacedAt));
        return Task.CompletedTask;          // IOutbox adds a row to the context's own outbox table
    }
}

// Loyalty context: the inbound handler is part of Loyalty's anticorruption layer
internal sealed class AwardPointsOnOrderPlaced(ILoyaltyAccounts accounts, IProcessedMessages processed)
    : IIntegrationEventHandler<Ordering.Contracts.V1.OrderPlacedV1>
{
    public async Task Handle(Ordering.Contracts.V1.OrderPlacedV1 message, Guid messageId, CancellationToken ct)
    {
        if (await processed.ContainsAsync(messageId, ct)) return;             // idempotency

        var member  = new MemberId(message.CustomerId);                        // Ordering's "customer" is Loyalty's "member"
        var spend   = Money.Of(message.TotalAmount, message.Currency);
        var account = await accounts.GetAsync(member, ct);
        if (account is null) return;                                           // not enrolled: not Loyalty's concern

        account.RecordQualifyingSpend(spend, message.PlacedAtUtc);             // Loyalty's own language and rules
        await processed.MarkAsync(messageId, ct);                              // saved in the same transaction as the account
        await accounts.SaveAsync(ct);
    }
}
```

What's happening, in DDD terms:

- **Upstream** keeps its internal model private and publishes a stable contract — an **open host service with a published language**.
- **Downstream** translates the published language into its own concepts — a "customer" becomes a "member," a "total" becomes "qualifying spend" — so Ordering's vocabulary never enters Loyalty's model. That translation is Loyalty's **anticorruption layer**.
- **The contract is versioned** (`V1` in the namespace), additive-only, and owned by the publisher. A breaking change is a new `V2` published alongside `V1` until consumers migrate (Module 21, Concept 70's expand/contract).

In an interview, this code is the concrete answer to "how do bounded contexts communicate?": *"The upstream translates its domain event into a versioned integration event in its outbox, in the same transaction. The downstream's handler translates that contract into its own language and applies it to its own aggregate, idempotently. Neither context's model ever appears in the other."*

---

## Concept 84 — Eventual consistency, as the user sees it

Eventual consistency is a technical property with a **user-visible** consequence, and the design is incomplete until the user experience is designed too. The patterns:

**Distinguish "received" from "done."** When a user places an order, what's immediately true is that the order was *received and accepted for processing*; payment capture, stock reservation and confirmation follow. Say exactly that: "Thanks — we've received your order and will confirm shortly." Businesses have always worked this way (a paper order form is "received" before it's "confirmed").

**Model pending states as real domain states.** `PaymentPending`, `AwaitingStockConfirmation`, `CancellationRequested` aren't technical artifacts; they're states the business can reason about, and they belong in the aggregate's lifecycle (Concept 72). Making them explicit is what turns "the data is temporarily inconsistent" into "the order is in a well-defined intermediate state."

**Give users read-your-writes.** Module 7's session guarantee: after a user's own change, their next read must reflect it, even if other views lag. Techniques: return the new state (or version) from the command and render it directly; route the user's reads to the primary for a short window; or have the client poll until the read model reaches the version the command returned.

**Push completion, don't make users refresh.** When the downstream step completes, notify the client (SignalR, a webhook, an email) rather than relying on them to check back.

**Make failure paths explicit.** If stock reservation fails after the order was accepted, what does the customer see? That's a compensation (Concept 85), and it's a product decision: cancel and refund, back-order, or offer a substitute. The business must choose — ideally during EventStorming, where it shows up as a red hotspot next to a policy.

**Agree the lag budget.** "Eventually" needs a number: the business should know that loyalty points appear within a minute, not "eventually." It's an SLO like any other (Module 28), and it's monitored.

The architect's line: *"Eventual consistency is a UX design problem as much as a data one. I'd model the in-between states explicitly, give users read-your-writes on their own changes, and get the business to put a number on 'eventually' so we can monitor it."*

---

## Concept 85 — Processes as aggregates

When a business process spans several aggregates or contexts over time — order fulfilment, claim settlement, customer onboarding — the process itself has **state** (which steps are done, what's pending, when things time out) and **rules** (what to do if payment fails after stock was reserved). That makes it a candidate aggregate in its own right: a **process manager** (orchestration) whose instance is a consistency boundary for the *workflow*, reacting to events and issuing commands.

Module 12 (sagas, compensation) and Module 21 (Concept 87, orchestration vs. choreography) covered the distributed mechanics. The DDD points:

- **A process manager is an aggregate.** It has an ID (usually correlated with the thing it coordinates), a lifecycle, invariants ("don't ship before payment is captured"), commands it handles (the events it reacts to arrive as inputs), and events it raises. It's loaded, decided on and saved exactly like any other aggregate — including optimistic concurrency, because two events for the same process can arrive concurrently.
- **Udi Dahan's observation** that in well-designed messaging systems the *sagas often become the most important aggregate roots* is worth knowing: the business logic of "how things proceed" lives in the process, while the entities it coordinates stay simple.
- **Timeouts are domain events too.** "If payment isn't captured within 30 minutes, release the reservation" is a rule, and the timeout is a scheduled message the process manager handles like any other.
- **Compensations are business decisions** (Module 21, Concept 23) — the process manager is where they're encoded, in the ubiquitous language: `ReleaseReservation`, `IssueRefund`, `NotifyCustomerOfBackorder`.

Implementation options in .NET, from lightest: a hand-rolled process aggregate persisted with EF Core plus a scheduled-message mechanism; **Wolverine** sagas (MIT); **Dapr Workflow** or **Azure Durable Functions** for durable orchestration; **MassTransit** state machines (commercial from v9). Choose by what's already in the stack; the modelling is the same.

The sentence: *"When a process spans aggregates, I model the process as its own aggregate — it has state, rules and a lifecycle — and it coordinates the others through events and commands. That keeps each entity small and puts the workflow logic somewhere it can be read and tested."*

---

# Part H — Repositories, the application layer and persistence in .NET

The model is only useful if it can be loaded, changed and saved without the database or the framework reshaping it. Module 19 covered EF Core's machinery and Module 20 the layering; this part is the DDD-specific assembly: one aggregate, end to end, from repository to mapping to test — and the places where reads, validation and errors should *not* go through the domain model at all.

---

## Concept 86 — Repositories

**A repository provides the illusion of an in-memory collection of aggregate roots**: you ask it for an aggregate by identity, you add new ones to it, and you don't think about SQL. Evans' rules follow directly from the aggregate rules (Concept 57):

- **One repository per aggregate root** — never per entity. There's an `IOrderRepository`; there is no `IOrderLineRepository`, because order lines are only reachable through their order.
- **It returns whole aggregates**, fully loaded, so invariants can be checked (Module 20, Concept 36).
- **Its interface belongs to the domain (or application) layer**, in the domain's language; its implementation belongs to infrastructure.

The perennial .NET debate — *"`DbContext` is already a unit of work and `DbSet<T>` is already a repository, so why wrap them?"* — has a DDD answer rather than a taste answer. A `DbSet<Order>` lets any caller load order lines directly, query any shape, attach partial graphs and skip `Include`s. A repository **restricts access to whole aggregates through their roots**, which is the rule that keeps invariants enforceable. Microsoft's own architecture guidance acknowledges both positions. The pragmatic synthesis:

- **In a core context with a rich model**, use thin repositories per aggregate root — they encode the aggregate boundary, which is worth encoding.
- **In supporting contexts with transaction scripts** (Concept 8), use `DbContext` directly. There's no aggregate boundary to protect.
- **For reads**, bypass repositories altogether (Concept 92).

---

## Concept 87 — Repository API design

A good repository interface is small and boring:

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(OrderId id, CancellationToken ct);
    void Add(Order order);
}

public interface IUnitOfWork
{
    Task SaveChangesAsync(CancellationToken ct);
    void Clear();                                   // used by the concurrency retry (Concept 69)
}

internal sealed class OrderRepository(OrderingDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(OrderId id, CancellationToken ct) =>
        db.Orders.SingleOrDefaultAsync(o => o.Id == id, ct);   // lines come with it via AutoInclude (Concept 89)

    public void Add(Order order) => db.Orders.Add(order);
}
```

What's deliberately absent, and why:

- **No `Update` method.** Aggregates loaded through the repository are tracked by the unit of work; changes are persisted by `SaveChanges`. An `Update(order)` method invites the "load, map a DTO onto it, call update" anti-pattern that bypasses the aggregate's methods.
- **No `Delete` unless the domain has a deletion concept.** Most business objects aren't deleted; they're cancelled, closed, archived or anonymized — domain operations with rules and events. A physical `Delete` is a deliberate exception.
- **No `IQueryable`, no `GetAll`, no generic `Find(Expression<…>)`.** Exposing `IQueryable` exports the storage's query semantics into every caller and lets them load partial aggregates (Module 19 Concept 66; Module 20 Concept 32). A generic repository (`IRepository<T>`) erases exactly the per-aggregate distinction the pattern exists to encode.
- **Query methods only when a *write* use case needs them.** `GetByOrderNumberAsync(OrderNumber)` is fine if a command arrives with an order number. `GetOrdersForDashboardAsync` is a read — it belongs on the query side (Concept 92).
- **`SaveChanges` is on the unit of work, not the repository**, because one application operation commits once, and the outbox rows and domain-event handlers (Concept 81) must be in the same commit.

A useful review test: *if you can list everything a repository does in one breath, it's the right size.*

---

## Concept 88 — Application services

The application service (or command handler) is the use case, and it has a characteristic shape — **load, decide, save** — with the domain event dispatch and outbox handled by infrastructure during save:

```csharp
public sealed record PlaceOrder(OrderId OrderId, IdempotencyKey Key);

public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    ICreditService credit,               // port to the Billing context (sync decision needed now)
    IIdempotencyStore idempotency,
    IUnitOfWork unitOfWork,
    TimeProvider clock)
{
    public async Task<PlaceOrderResult> Handle(PlaceOrder cmd, CancellationToken ct)
    {
        if (await idempotency.TryGetResultAsync<PlaceOrderResult>(cmd.Key, ct) is { } previous)
            return previous;                                                   // retried request: same answer

        var order = await orders.GetAsync(cmd.OrderId, ct);
        if (order is null) return PlaceOrderResult.NotFound;

        var available = await credit.GetAvailableCreditAsync(order.CustomerId, order.Total.Currency, ct);
        var outcome = order.Place(available, clock.GetUtcNow());               // the business decision

        var result = PlaceOrderResult.From(outcome);
        idempotency.Record(cmd.Key, result);                                  // same transaction as the order
        await unitOfWork.SaveChangesAsync(ct);                                 // events → handlers → outbox, one commit
        return result;
    }
}
```

The rules that keep application services healthy:

- **No business rules.** The handler orchestrates: it fetches what the decision needs (the credit figure), calls the aggregate, and saves. "Is there enough credit?" is decided inside `Order.Place`. If you could delete the handler and lose a business rule, the rule is in the wrong place.
- **One aggregate modified per command** (Concept 59), unless a Concept 63 exception has been chosen deliberately.
- **One transaction per command**, committed once.
- **Cross-cutting concerns live here or in a pipeline around it** — authorization, idempotency, logging, retries for concurrency conflicts (Concept 69), transaction management — not in the domain.
- **Inputs are domain types.** The command carries `OrderId`, not `Guid` — parsing happened at the edge (Concept 48).

Whether this is a class called `PlaceOrderHandler`, a Wolverine handler, a MediatR request handler or a minimal-API endpoint delegate is a packaging choice (Module 23 takes it up). The shape is what matters.

---

## Concept 89 — Mapping an aggregate with EF Core 10

Here's the `Order` aggregate from the earlier concepts, complete, followed by its mapping — every line is there for a DDD reason.

```csharp
public enum OrderStatus { Draft, Placed, Paid, Shipped, Cancelled }
public enum PlaceOutcome { Placed, NotDraft, Empty, InsufficientCredit }

public sealed class Order : AggregateRoot<OrderId>
{
    public const int MaxLines = 50;
    private readonly List<OrderLine> _lines = [];

    public CustomerId CustomerId { get; private set; }                 // reference by identity (Concept 61)
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; } = null!;
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset? PlacedAt { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();       // no public mutation of the collection

    private Order() { }                                                // for EF materialization only (Concept 90)

    private Order(OrderId id, CustomerId customer, string currency, DateTimeOffset now)
    {
        Id = id;
        CustomerId = customer;
        Status = OrderStatus.Draft;
        Total = Money.Zero(currency);
        CreatedAt = now;
    }

    public static Order Draft(OrderId id, CustomerId customer, string currency, DateTimeOffset now)
    {
        var order = new Order(id, customer, currency, now);
        order.Raise(new OrderDrafted(id, customer, now));
        return order;
    }

    public void AddLine(Sku sku, Quantity quantity, Money unitPrice, DateTimeOffset now)
    {
        EnsureDraft();
        if (unitPrice.Currency != Total.Currency)
            throw new DomainException($"This order is priced in {Total.Currency}.");

        var existing = _lines.Find(l => l.Sku == sku);
        if (existing is not null)
            existing.IncreaseBy(quantity);
        else if (_lines.Count >= MaxLines)
            throw new DomainException($"An order can have at most {MaxLines} lines.");
        else
            _lines.Add(new OrderLine(NextLineNumber(), sku, quantity, unitPrice));

        RecalculateTotal();
        Raise(new OrderLineAdded(Id, sku, quantity, unitPrice, now));   // also bumps Version (Concept 68)
    }

    public PlaceOutcome Place(Money availableCredit, DateTimeOffset now)
    {
        if (Status != OrderStatus.Draft) return PlaceOutcome.NotDraft;
        if (_lines.Count == 0)           return PlaceOutcome.Empty;
        if (Total.Amount > availableCredit.Amount) return PlaceOutcome.InsufficientCredit;

        Status = OrderStatus.Placed;
        PlacedAt = now;
        Raise(new OrderPlaced(Id, CustomerId, Total,
            [.. _lines.Select(l => new PlacedLine(l.Sku, l.Quantity, l.UnitPrice))], now));   // snapshot, never live entities
        return PlaceOutcome.Placed;
    }

    private void EnsureDraft()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException($"Lines can only change while the order is a draft (it is {Status}).");
    }

    private int NextLineNumber() => _lines.Count == 0 ? 1 : _lines.Max(l => l.LineNumber) + 1;

    private void RecalculateTotal() =>
        Total = _lines.Aggregate(Money.Zero(Total.Currency), (sum, line) => sum.Add(line.LineTotal));
}

public sealed class OrderLine
{
    public int LineNumber { get; private set; }                      // local identity (Concept 41)
    public Sku Sku { get; private set; }
    public Quantity Quantity { get; private set; }
    public Money UnitPrice { get; private set; } = null!;
    public Money LineTotal => UnitPrice.MultiplyBy(Quantity.Value);

    private OrderLine() { }

    internal OrderLine(int lineNumber, Sku sku, Quantity quantity, Money unitPrice)
    {
        LineNumber = lineNumber;
        Sku = sku;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }

    internal void IncreaseBy(Quantity more) => Quantity = Quantity.Add(more);   // only the root calls this
}
```

And the mapping, all in infrastructure (no attributes in the domain):

```csharp
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("Orders", "ordering");                               // schema per module (Module 21)
        b.HasKey(o => o.Id);
        b.Property(o => o.Id).ValueGeneratedNever();                   // the domain assigns identity
        b.Property(o => o.Version).IsConcurrencyToken();               // aggregate-level concurrency (Concept 68)
        b.Property(o => o.Status).HasConversion<string>().HasMaxLength(32);   // strings, never ints
        b.Ignore(o => o.DomainEvents);

        b.ComplexProperty(o => o.Total, m =>
        {
            m.Property(x => x.Amount).HasColumnName("TotalAmount").HasPrecision(19, 4);
            m.Property(x => x.Currency).HasColumnName("Currency").HasMaxLength(3);
        });

        // CustomerId is a plain column: no navigation to another aggregate. Add
        // HasOne<Customer>().WithMany().HasForeignKey(o => o.CustomerId) only if Customer
        // lives in the same context and schema — never across modules.

        b.HasMany(o => o.Lines).WithOne().HasForeignKey("OrderId").OnDelete(DeleteBehavior.Cascade);
        b.Navigation(o => o.Lines)
            .UsePropertyAccessMode(PropertyAccessMode.Field)           // EF writes to _lines, not the read-only view
            .AutoInclude();                                            // loading an Order always loads the whole aggregate
    }
}

internal sealed class OrderLineConfiguration : IEntityTypeConfiguration<OrderLine>
{
    public void Configure(EntityTypeBuilder<OrderLine> b)
    {
        b.ToTable("OrderLines", "ordering");
        b.Property<OrderId>("OrderId");
        b.HasKey("OrderId", nameof(OrderLine.LineNumber));             // local identity: unique within its order
        b.Property(l => l.LineNumber).ValueGeneratedNever();
        b.Ignore(l => l.LineTotal);
        b.ComplexProperty(l => l.UnitPrice, m =>
        {
            m.Property(x => x.Amount).HasColumnName("UnitPriceAmount").HasPrecision(19, 4);
            m.Property(x => x.Currency).HasColumnName("UnitPriceCurrency").HasMaxLength(3);
        });
    }
}

// In OrderingDbContext: strongly-typed IDs and domain primitives by convention (Module 16)
protected override void ConfigureConventions(ModelConfigurationBuilder c)
{
    c.Properties<OrderId>().HaveConversion<OrderIdConverter>();
    c.Properties<CustomerId>().HaveConversion<CustomerIdConverter>();
    c.Properties<Sku>().HaveConversion<SkuConverter>().HaveMaxLength(40);
    c.Properties<Quantity>().HaveConversion<QuantityConverter>();
}
```

The DDD reasons behind the less obvious lines:

- **`AutoInclude()`** makes "an `Order` is always loaded whole" a property of the model rather than something every query must remember — the aggregate boundary is encoded in the mapping. Reads that don't want lines use projections (which ignore includes) or `IgnoreAutoIncludes()` (Concept 92).
- **Composite key `(OrderId, LineNumber)`** is local identity made physical: a line is only unique within its order.
- **Removing a line from `_lines`** marks it as an orphan of a required relationship, and EF deletes it on save — the aggregate controls its children's lifecycle (Concept 57, rule 6).
- **`internal` members on `OrderLine`** mean code outside the domain assembly can't modify a line directly. (Code inside the assembly still could; an architecture test or the "only the root touches children" review rule covers that.)
- **No lazy loading.** Lazy-loading proxies let a traversal silently cross the aggregate boundary and turn one load into N+1 (Module 19). For aggregates, they're a defect.
- **Several child collections?** Use `AsSplitQuery()` in the repository to avoid a Cartesian explosion (Module 19) — or reconsider the aggregate's size (Concept 60).

---

## Concept 90 — Creation vs. reconstitution

When EF materializes an aggregate from the database, it's **reconstituting** an existing object, not **creating** a new one. The two must behave differently (Concept 52):

| | Creation (`Order.Draft(...)`) | Reconstitution (EF materialization) |
|---|---|---|
| **Validates invariants** | Yes | **No** — the data was valid when written; rules may have changed since |
| **Assigns identity** | Yes | No — identity comes from storage |
| **Raises events** | Yes (`OrderDrafted`) | **No** — nothing happened |
| **Path** | Public/static factory, private constructor | Private parameterless constructor (or constructor binding) + backing fields |

EF Core supports this cleanly:

- **A private parameterless constructor** is enough for EF; it's invisible to application code, so the only way to *create* an order is the factory.
- **Constructor binding**: EF can call a constructor whose parameters match mapped properties by name — useful for immutable value objects (complex types) with private constructors.
- **Backing fields**: by default EF prefers writing to a property's backing field when it can find one, so logic in a property setter doesn't run on materialization — which is what you want. If you use setter logic for invariants (for instance with C# 14's `field` keyword), add a test that loads a row which would *fail* today's rule and confirm it still materializes.

The reason "don't validate on load" matters in practice: rules tighten. If the maximum order size drops from 50 lines to 30, existing orders with 40 lines are historical facts, not invalid data. A model that re-validates on load turns every rule change into a production outage or a data migration. New rules apply to new *changes*.

---

## Concept 91 — Persistence ignorance, priced

Module 20 (Concept 29) priced the purist position: a domain model that knows nothing about persistence requires a **separate persistence model** and a mapping layer, and you lose EF's change tracking on your domain objects (you must diff or rewrite on save). Its three compromises — private parameterless constructors, private setters or backing fields, and IDs as properties — make a domain model EF-friendly at a small cost in purity.

The DDD-specific judgment:

- **The pragmatic default:** domain classes shaped to be EF-friendly (the compromises above), all mapping in infrastructure via fluent configuration, no EF attributes or types in the domain assembly. This keeps the model expressive and the persistence reasonably invisible.
- **A separate persistence model earns its cost** when the storage schema is dictated by something else — a legacy schema you can't change, a shared database during a migration, or a document store whose shape differs radically from the model. Then the translation *is* an anticorruption layer between your model and the storage (Concept 27), and it deserves that level of care.
- **The aggregate-as-document option** is underused: if the aggregate is small and always loaded whole, storing it as one document — a Cosmos DB item in a partition keyed by aggregate ID, a PostgreSQL `jsonb` column, or an EF Core entity whose children are JSON-mapped complex collections — gives you one read, one write, and one version per aggregate, so concurrency covers the whole boundary by construction (Concept 68). The cost is cross-aggregate querying, which should go to read models anyway (Concept 92).

The interview answer to "should the domain know about EF?": *"The domain shouldn't reference EF, but I'll accept three small compromises — a private constructor, private setters and IDs as properties — to keep EF's change tracking. I'd only introduce a separate persistence model if the schema is dictated by something I can't change, and then I'd treat the mapping as an anticorruption layer."*

---

## Concept 92 — Reads bypass the aggregate

The domain model exists to protect invariants during **changes**. Reads don't change anything, so they don't need it — and forcing them through it is one of the most common sources of DDD pain in .NET: aggregates loaded with every child to render a list, getters added to aggregates only for screens, repositories with dozens of query methods.

The rule — and the CQRS-lite position from Module 20 (Concepts 34–35), taken fully in Module 23:

- **Commands** go through the aggregate: repository → aggregate method → save.
- **Queries** go straight to storage and project to exactly the shape the screen needs:

```csharp
// IDs and enums serialize as plain strings via the converters from Module 16
public sealed record OrderSummaryDto(OrderId Id, OrderStatus Status, decimal Total, string Currency, int LineCount, DateTimeOffset? PlacedAt);

internal sealed class OrderQueries(OrderingDbContext db)
{
    public Task<List<OrderSummaryDto>> RecentForCustomerAsync(CustomerId customer, int take, CancellationToken ct) =>
        db.Orders
          .AsNoTracking()
          .Where(o => o.CustomerId == customer)
          .OrderByDescending(o => o.CreatedAt)
          .Take(take)
          .Select(o => new OrderSummaryDto(
              o.Id, o.Status, o.Total.Amount, o.Total.Currency, o.Lines.Count, o.PlacedAt))
          .ToListAsync(ct);
}
```

(Projections ignore `AutoInclude`, and EF translates `o.Lines.Count` into a subquery rather than loading lines.) For more demanding reads — cross-aggregate reports, dashboards, search — use Dapper or SQL directly, or a dedicated read model fed by events (Module 21 Concept 38; Module 23).

Two cautions. First, **business calculations stay in the domain even if they're only displayed** — if "total including tax" is a business rule, compute it at write time and store it, or call a domain function from the projection; don't let a second implementation of the rule appear in SQL (Module 20, Concept 35). Second, the query side can read *across* aggregates freely within a context — that's one of its main purposes — but across contexts it reads from its own replicated data or a read model, never another context's tables.

---

## Concept 93 — Testing the domain model

A well-designed domain model is the easiest code in the system to test: no I/O, no framework, no mocks. The patterns:

**Given / when / then on the aggregate**, named in the ubiquitous language (the green cards from Example Mapping, Concept 37, become these tests almost verbatim):

```csharp
public sealed class PlacingAnOrder
{
    private static readonly DateTimeOffset Now = new(2026, 9, 25, 10, 0, 0, TimeSpan.Zero);

    [Fact]
    public void Over_available_credit_is_rejected_and_changes_nothing()
    {
        var order = OrderBuilder.Draft("EUR").WithLine("SKU-1", quantity: 2, unitPrice: 60m).Build();  // total 120
        order.ClearDomainEvents();
        var versionBefore = order.Version;

        var outcome = order.Place(availableCredit: Money.Of(100m, "EUR"), Now);

        Assert.Equal(PlaceOutcome.InsufficientCredit, outcome);
        Assert.Equal(OrderStatus.Draft, order.Status);
        Assert.Empty(order.DomainEvents);
        Assert.Equal(versionBefore, order.Version);
    }

    [Fact]
    public void Within_credit_places_the_order_and_announces_it()
    {
        var order = OrderBuilder.Draft("EUR").WithLine("SKU-1", quantity: 1, unitPrice: 60m).Build();
        order.ClearDomainEvents();

        var outcome = order.Place(Money.Of(100m, "EUR"), Now);

        Assert.Equal(PlaceOutcome.Placed, outcome);
        var placed = Assert.IsType<OrderPlaced>(Assert.Single(order.DomainEvents));
        Assert.Equal(Money.Of(60m, "EUR"), placed.Total);
    }
}
```

**Deciders are even simpler** (Concept 73): `given` a list of events, fold them into state; `when` a command, call `Decide`; `then` assert the returned events or rejection. A single helper makes every test three lines.

**Property-based tests for invariants.** Where a rule must hold for *all* inputs, generate the inputs. With FsCheck (or CsCheck), the allocation invariant from Concept 46 becomes:

```csharp
[Property]
public bool Allocation_never_loses_or_invents_a_cent(PositiveInt cents, NonEmptyArray<PositiveInt> ratios)
{
    var amount = Money.FromMinorUnits(cents.Get, "EUR");
    var parts  = amount.Allocate([.. ratios.Get.Select(r => r.Get % 1_000 + 1)]);
    return parts.Aggregate(Money.Zero("EUR"), (sum, p) => sum.Add(p)) == amount;
}
```

**One real concurrency test per aggregate** that has concurrency-sensitive invariants — against a real database (Testcontainers, Module 21), because this is exactly the bug that in-memory tests can't see:

```csharp
[Fact]
public async Task Concurrent_line_additions_cannot_exceed_the_line_limit()
{
    var orderId = await SeedDraftOrderWithLines(Order.MaxLines - 1);

    await using var db1 = CreateContext();
    await using var db2 = CreateContext();
    var first  = await new OrderRepository(db1).GetAsync(orderId, Ct);
    var second = await new OrderRepository(db2).GetAsync(orderId, Ct);

    first!.AddLine(Sku.Parse("A-1"), Quantity.Of(1), Money.Of(10m, "EUR"), Now);
    second!.AddLine(Sku.Parse("B-1"), Quantity.Of(1), Money.Of(10m, "EUR"), Now);

    await db1.SaveChangesAsync(Ct);
    await Assert.ThrowsAsync<DbUpdateConcurrencyException>(() => db2.SaveChangesAsync(Ct));
}
```

Remove the version bump from Concept 68 and this test fails — which is the point of writing it.

What *not* to do: mock the aggregate's collaborators to verify interactions ("verify that `repository.Save` was called once"). Domain tests should assert **state and events**, not calls. Test builders (`OrderBuilder`) keep setup short and expressive, and they're the only test infrastructure the domain needs.

---

## Concept 94 — Validation layering

A frequent interview probe: "where does validation go?" The DDD answer is that there are several *kinds* of validation, and each has exactly one home:

| Kind | Example | Home | Failure |
|---|---|---|---|
| **Request shape** | Required field missing, wrong type, string too long for the contract | API edge: model binding, endpoint filters, FluentValidation on request DTOs | 400 with field-level problem details |
| **Format of a domain primitive** | Not a valid email, IBAN or SKU | Value-object factory (`TryParse`), called at the edge | 400 — the edge translates the value object's rejection |
| **Aggregate invariant** | Order has no lines; lines can't change after placement; credit exceeded | Aggregate methods | 409 or 422 with a domain reason |
| **Set-based rule** | Email already in use | Database constraint (+ optional pre-check), registry aggregate, or DCB (Concept 70) | 409 |
| **Cross-aggregate policy** | Loyalty tier recalculated after spend | Event handler or process manager, eventually | Not a request failure; a later state change or compensation |
| **Authorization** | This user may not place orders for that customer | Application layer / pipeline | 403 |

Two rules of thumb that settle most arguments:

1. **The domain never trusts the edge, and the edge never duplicates the domain.** Format rules live in value objects; the edge calls them. Business rules live in aggregates; the edge never re-implements them to "give a nicer error" — it maps the domain's outcome to a response.
2. **FluentValidation belongs on request DTOs, not on entities.** Validating an entity from the outside ("the validator checks the order before save") is the anaemic pattern — it means the entity can exist in an invalid state, which is exactly what an always-valid model prevents.

---

## Concept 95 — Errors and outcomes

When an aggregate rejects a command, how does it say so? Two mechanisms, and the senior answer uses both deliberately:

**Outcome types for expected business results.** "Insufficient credit," "course is full," "seat no longer available" are normal outcomes that the caller must handle — each typically becomes a different response or user message. Return them. Before C# 15, an enum (`PlaceOutcome`) or an `abstract record` hierarchy works; with C# 15, a `union` over existing result types or a `closed` hierarchy makes handling exhaustive (Module 16, Concept 39):

```csharp
public sealed record Placed(OrderId Id);
public sealed record Rejected(string Reason);
public sealed record NotFound;
public union OrderPlacementResult(Placed, Rejected, NotFound);        // C# 15

// At the edge (an endpoint returning IResult): translate once, exhaustively
return result switch
{
    Placed p   => TypedResults.Ok(new { orderId = p.Id.Value }),
    Rejected r => TypedResults.Problem(detail: r.Reason, statusCode: StatusCodes.Status422UnprocessableEntity,
                                       title: "The order was rejected"),
    NotFound   => TypedResults.NotFound(),
};
```

**Exceptions for violated preconditions and broken programs.** Calling `AddLine` on a shipped order through a UI that hides the button, adding EUR to USD, or a null where a value object is required means a *caller* made a mistake — a bug or a race the caller could have avoided. A `DomainException` is appropriate; the edge maps it to 409 Conflict with problem details. Infrastructure failures (database down) stay ordinary exceptions and become 500s/503s.

The line between "expected outcome" and "precondition violation" is a judgment call, and the honest advice is to **pick a convention per context and apply it consistently**: many teams return outcomes for everything a user can legitimately cause and throw only for what indicates a bug. What matters in review is that the aggregate never returns `false` or `null` for a business rejection — callers ignore booleans, and the reason is lost.

HTTP mapping worth stating: **400** for malformed requests, **422** for well-formed requests that violate a business rule, **409** for conflicts with current state (wrong lifecycle state, concurrency — Concept 69), **412** for a failed `If-Match`, **404** for missing aggregates — all as RFC 9457 problem details.

---

# Part I — The architect's view

Everything above is technique. This part is judgment: when *not* to apply it, what reviewers look for, how to use DDD in a design round without sounding like a textbook, how to use it in an architect round as an organizational and investment tool, and what AI changes.

---

## Concept 96 — The anaemic domain model

Martin Fowler named it in 2003: a domain model whose objects have **data and no behaviour** — properties with public getters and setters — with all the business logic in service classes that manipulate them. Fowler called it an anti-pattern because it has **all the costs of a domain model** (mapping, layering, persistence plumbing) **and none of the benefits** (encapsulated rules, invariants enforced in one place). It's really a transaction script wearing a domain model's clothes.

The DDD nuance, and the part that distinguishes a senior answer: **an anaemic model is a problem in the core domain and often the correct choice elsewhere.** In a supporting subdomain whose logic is mostly CRUD with light validation (Concept 8), a set of plain entities manipulated by transaction scripts is honest, cheap and easy to change. What's harmful isn't the absence of behaviour — it's:

- **Anaemia where the rules are complex** — the core domain's rules scattered across services, duplicated between use cases, enforced inconsistently.
- **Pretending** — layering, repositories, "aggregates" and domain events wrapped around what is actually CRUD, so the team pays the ceremony tax without any of the protection.

Two forms are worth distinguishing in review: **"anaemic but honest"** (a supporting context built as transaction scripts, with no DDD vocabulary) is fine; **"rich but fake"** (entities with private setters and a `Domain` folder, but every rule still in a `*Service` class that reaches in via `internal` methods) is the one to fix.

The interview line: *"In the pricing context I'd refactor that toward a rich model, because the rules are where the business competes and they're duplicated across four services. In the admin context I'd leave it anaemic — it's CRUD, and a domain model there would slow the team down without making anything more correct."*

---

## Concept 97 — DDD-lite

The name for **adopting the tactical patterns without the strategic ones** (Concept 3). The symptoms are easy to spot:

- A `Domain/Entities`, `Domain/ValueObjects`, `Domain/Repositories` folder structure (Concept 55's anti-pattern).
- A repository per table, often generic (`IRepository<T>`).
- "Aggregates" that are simply entities with child collections, sized by the schema rather than by invariants.
- Domain events named after CRUD (`CustomerUpdated`) and consumed by nobody.
- No glossary, no context map, no subdomain classification — one model for the whole system.
- The same `Customer` class shared across every feature, growing indefinitely.

Why it's worse than doing nothing: the team pays for the patterns — extra layers, mapping, indirection, slower onboarding — but gets none of the benefits, because the benefits come from **a consistent model within an explicit boundary, expressed in the business's language**. And it poisons the well: teams that have lived through DDD-lite often conclude that "DDD is over-engineering," when what they experienced was ceremony without the method.

The fix is not more patterns; it's the strategic work that was skipped. Start with a glossary and a context map (Concepts 5, 22), classify the subdomains (Concept 11), and then decide *where* tactical patterns are worth it — which will usually mean removing them from some places.

---

## Concept 98 — The canonical enterprise model

Many large organizations have attempted, at some point, a **single enterprise-wide data model**: one authoritative definition of Customer, Product, Order and so on, shared by every system. It's a natural idea — "one version of the truth" — and DDD's position on it is unambiguous: **it doesn't work as a domain model**, for exactly the reasons bounded contexts exist (Concept 16).

The predictable trajectory: the canonical `Customer` must satisfy sales, billing, support, marketing, compliance and logistics; it accumulates every attribute any of them needs; every change requires cross-departmental agreement; teams work around it with local extensions and "extra" fields; and eventually each system has its own shadow model plus a translation to the canonical one — the worst of both worlds.

What *does* work, and is worth distinguishing:

- **Canonical models for integration messages** — a *published language* (Concept 28) used at context boundaries, often industry-standard (ISO 20022, FHIR, ACORD). That's a contract, not a domain model, and it can be stable precisely because no context's internal logic depends on it.
- **Master data management for identity and reference data** — agreeing on *which* customer this is (a shared identifier, golden record for contact data), not on *what* a customer is in every context.

The architect-round answer when someone proposes a canonical model: *"I'd agree on shared identifiers and a published language for integration, but I wouldn't try to share a single domain model. Each context needs its own model of 'customer' to stay precise — the canonical part should be the contract between them, not the thing inside them."*

---

## Concept 99 — Leaky aggregates

An aggregate is only a consistency boundary if nothing can change its internals without going through the root. The leaks reviewers look for, most common first:

| Leak | Why it breaks the boundary | Fix |
|---|---|---|
| **Public setters** (`public OrderStatus Status { get; set; }`) | Any code can put the aggregate in any state | Private setters; state changes only through named methods |
| **Exposed mutable collections** (`public List<OrderLine> Lines`) | Callers add/remove children without the root's checks | `IReadOnlyList<T>` view over a private field; mutation via root methods |
| **Navigation properties to other aggregates** | Loads, tracks and invites modification of a second aggregate in the same transaction | Reference by ID (Concept 61) |
| **Lazy loading** | Traversal silently crosses boundaries; N+1; partial aggregates | Disable; load aggregates whole (Concept 89) |
| **Children with public mutators** (`line.Quantity = 5`) | Bypasses the root's recalculation and invariants | `internal`/private mutators called only by the root |
| **Aggregates serialized as API responses** | The API contract is now the model; internals leak to clients; changes break consumers | Map to response DTOs or read models (Concept 92) |
| **Object initializers for construction** (`new Order { … }`) | Creation bypasses the factory and its invariants | Private constructors; named factories (Concept 52) |
| **Events carrying live entities** | Consumers hold references into the aggregate's internals | Immutable snapshots in events (Concept 79) |
| **A "helper" service that sets internal state via `internal` members** | Rules move out of the aggregate while it still looks encapsulated | Move the logic into the aggregate or a proper domain service |

Most of these are cheap to prevent mechanically: private setters by convention, architecture tests that fail on public setters in domain assemblies or on navigation properties between aggregate roots, and a banned-API analyzer rule for lazy-loading proxies (Module 21, Concept 48).

---

## Concept 100 — Over-modelling

The opposite failure, and increasingly common among teams that learned DDD enthusiastically:

- **A value object for every string.** `FirstName`, `LastName`, `Comment`, `Description` — types that validate nothing and have no behaviour. The test from Concept 45 applies: wrap a primitive when mixing it up is a plausible bug or when it has a format, a range or behaviour.
- **Domain events nobody consumes**, raised "for completeness." Every event is a contract to maintain. Raise the ones that drive a reaction or record a business fact someone needs.
- **Aggregates, repositories and factories in a supporting CRUD context** (Concept 96).
- **Specifications for single-use predicates** — an inline condition with a good name is clearer than a class used once.
- **Factories for trivial construction** — a constructor or static method is enough until creation is genuinely complex.
- **Deep generic base-class hierarchies** (`Entity<TId>` → `AuditableEntity<TId>` → `AggregateRoot<TId>` → `SoftDeletableAggregateRoot<TId>`) that encode infrastructure concerns into the domain.
- **Mapping layers stacked three deep** — request DTO → command → domain → persistence model → response DTO — where two would do.

The principle that resolves both Concept 96 and this one: **modelling effort should be proportional to domain complexity and business value** — which is precisely what the subdomain classification (Concept 14) is for. DDD done well is *uneven*: rich where it matters, deliberately plain elsewhere.

---

## Concept 101 — The review catalogue

What a reviewer looks for in a domain model, in the order that finds the most important problems first:

1. **Is there a ubiquitous language, and does the code speak it?** Class, method and event names an expert would recognize; a glossary; no `Manager`, `Processor`, `Helper` in the domain.
2. **Is the context boundary explicit?** One model per context; no types shared across contexts except a tiny shared kernel.
3. **Is this subdomain worth a rich model?** If it's supporting CRUD, is the team paying for ceremony it doesn't need?
4. **Does each aggregate protect named invariants?** Can someone list them? Does every object inside the boundary participate in one?
5. **Are aggregates small, with bounded collections?** Any unbounded child collection is a defect (Concept 67).
6. **Are references between aggregates by ID?** No navigation properties across roots.
7. **Is state changed only through intention-revealing methods?** No public setters, no exposed mutable collections.
8. **Is the model always valid?** Factories and value objects validate at creation; nothing validates on load.
9. **Does concurrency control cover the whole aggregate?** Root version bumped on every change (Concept 68), with a real-database test.
10. **Is one aggregate modified per transaction** — or is each exception deliberate and documented?
11. **Are domain events intent-revealing, immutable and dispatched reliably** (outbox for anything leaving the context)?
12. **Is translation explicit at every context boundary** — ACL downstream, published language upstream?
13. **Do reads bypass the aggregate?** No aggregate getters that exist only for screens.
14. **Are set-based rules enforced somewhere that survives concurrency** — a constraint, a registry aggregate, or DCB?
15. **Are application services thin?** No business rule would be lost if they were rewritten.

---

## Concept 102 — DDD in the system-design round

Most system-design interviews never say "DDD," but the concepts decide whether your design survives the deep dive. The trick is to **use the ideas without the jargon** — interviewers who aren't DDD practitioners hear "aggregate root" as a buzzword, but they hear "the booking is our unit of consistency" as insight.

Where it shows up in the 7-step framework (Module 3):

| Step | The DDD move, in plain words |
|---|---|
| **1. Requirements** | Ask about the rules, not just the features: *"What must never happen — double-booking, overselling, overdraft? And what's allowed to lag?"* That's invariant discovery (Concept 58). |
| **2. Estimation** | Estimate the **per-entity write rate** for the hottest entity, not just total QPS: *"A popular show gets 5,000 booking attempts a second at on-sale time — all against one show."* That's the contention input (Concept 65). |
| **3. API design** | Commands named after business intent (`POST /holds`, `POST /holds/{id}/confirm`) rather than CRUD on fields; resources correspond to consistency units. |
| **4. Data model** | Group data by what must change atomically; reference other groups by ID; name the partition key after the consistency unit (Concepts 56, 75). |
| **5. High-level design** | Components along business capabilities with their own data (bounded contexts), talking via events for state changes (Concept 33). |
| **6. Deep dive** | Concurrency on the hot entity (Concepts 65–66), uniqueness (Concept 70), cross-entity rules (Concept 71), what the user sees while things converge (Concept 84). |
| **7. Wrap-up** | Which rules are enforced immediately vs. eventually, and what compensates when eventual steps fail. |

The sentence that does the most work in a design round: *"Before I draw the data model, let me list the rules that must hold at commit and the ones that can lag — that tells me what needs to be updated together, which is what should be stored together and partitioned together."*

---

## Concept 103 — DDD in the architect round

In an architect round, DDD's strategic half is the tool, and it's used for three conversations that senior ICs rarely get asked:

**1. Organizational design (the context map is an org chart proposal).** Conway's law (Module 21, Concept 9) means bounded contexts and team boundaries must align, or the architecture will drift toward the org chart. Proposing contexts *is* proposing team responsibilities. The context map shows who depends on whom, where the power sits (Concept 31) and where coordination cost will concentrate. The inverse Conway manoeuvre — shaping teams to produce the architecture you want — is a DDD strategic move.

**2. The investment case (the core domain is a business argument).** Executives don't respond to "we need a rich domain model," but they respond immediately to *"we're spending most of our engineering effort on things we could buy, and too little on the pricing capability that's why customers choose us."* The subdomain classification (Concepts 11–14) is a portfolio analysis: where to build, buy, or simplify. This is the most direct link between DDD and Module 33's build-vs-buy language.

**3. The brownfield strategy (where to start).** In a legacy estate, the strategic questions are which capability to carve out first and how to protect it — a bubble context behind an ACL (Concept 30), chosen because it's core, painful and separable. That's a more credible plan than "we'll migrate to microservices," and it produces value in the first quarter.

The deliverables that make it concrete: a **subdomain map with classifications**, a **current-state context map with relationships annotated honestly**, a **target context map**, **Bounded Context Canvases** for the contexts that matter, and an **ADR** for each strategic choice (Module 31).

---

## Concept 104 — DDD and AI

A question you can now expect: *"Does DDD still matter when AI writes the code?"* The primary-source position (Concept 34) is Eric Evans' own working hypothesis, stated in his DDD Europe 2026 keynote: **domain models, bounded contexts and language still matter — but the models will look different.** A balanced answer covers what changes and what doesn't.

**What doesn't change:**

- **Invariants and consistency boundaries are business decisions.** Whether a seat can be double-sold, what happens when a cancellation races a shipment, which rules may lag — none of that is in the training data; it's in the business, and it needs people who can ask and decide.
- **Deterministic core logic stays deterministic.** A probabilistic component can *propose*; the domain model *decides* against its invariants (Concept 34).

**What changes:**

- **The ubiquitous language becomes more valuable, not less.** It's the contract that keeps an AI component — and an AI coding assistant — using terms precisely. A glossary and a well-named model are exactly the context an assistant needs to generate code that fits; a codebase where `Status = 3` means "lapsed" gives it nothing to go on.
- **New bounded contexts appear.** LLM components are contexts with their own language and a probabilistic consistency model, and they need anticorruption layers (Concept 34).
- **Some mechanical DDD work gets cheaper**: drafting glossaries from interview transcripts, summarizing an EventStorming wall, checking code for vocabulary drift, first drafts of ACL translators. Evans himself has experimented with LLMs extracting domain vocabulary from code to compare contexts. The DDD Crew maintains a small repository of DDD-oriented prompts and rules for coding assistants.
- **Some work doesn't get cheaper**: facilitating experts, deciding aggregate boundaries, and recognizing when a rule is really a policy — the activities where a wrong answer is most expensive.

The interview answer: *"I'd say it raises the value of the strategic work. The code is getting cheaper to produce; knowing which rules must hold together, and keeping the language precise enough that both people and tools use it correctly, is the part that isn't."*

---

## Concept 105 — The 60-second answers

Three questions arrive in some form in almost every loop that touches DDD. The shapes:

**"What is DDD?"**

> *"It's an approach to complex business software built around three decisions. Where must consistency be immediate — that's aggregates, sized by the invariants the business can't tolerate breaking. Where does the meaning of words change — that's bounded contexts, each with its own model and language, integrated through explicit translation. And where does the business compete — that's the core domain, where you invest your best people and your richest model, while you buy or simplify the rest. The tactical patterns — entities, value objects, repositories, events — are how you implement those decisions in code, and they're only worth it where the rules are genuinely complex."*

**"How do you design an aggregate?"**

> *"I start from the commands and the rules, not the data. I list the invariants and ask of each one whether it must hold the instant a change commits or can become true shortly after — the business answers that, not me. The ones that must hold at commit define the boundary, which I keep as small as they allow, with other aggregates referenced by ID and updated eventually through events. Then I check the numbers: the per-instance command rate against roughly one over the read–decide–write time, because an aggregate is a serialization point. And I make sure concurrency control covers the whole aggregate — in EF that means versioning the root, because a change to a child row alone won't trip the check."*

**"How do you find bounded contexts?"**

> *"From language first: where the same word means different things to different experts, or different words mean the same thing at a handoff. I'd run a Big Picture EventStorming session to find pivotal events, swimlanes and hotspots, check the candidate boundaries against who owns which decisions and how often each area changes, and then validate them by walking the ten most important scenarios across them and counting the messages. If a common scenario needs a chatty conversation between two contexts, the boundary is in the wrong place. Each context gets one team, one language and its own data — and whether it's a module or a service is a separate decision."*

---

## Concept 106 — The close

The paragraph to leave in the interviewer's head:

> *"The way I use DDD is as three decisions rather than a set of patterns. Consistency: I find the rules that must hold at the instant of commit and draw the smallest boundaries that protect them — everything else is referenced by ID and updated through events, and I check the per-instance write rate, because every aggregate is a serialization point. Language: I draw bounded contexts where meaning changes, give each one team and its own model, and translate explicitly at every boundary — an anticorruption layer on the way in, a published language on the way out — so no context's model leaks into another's. Investment: I put the rich model and the best people on the core domain, keep supporting areas deliberately simple, and buy what's generic. The result is uneven on purpose — rich where the business competes, plain everywhere else — and that unevenness is the point."*

What's on display: a framework, a derivation, a number, a mechanism for each boundary, and a judgment about where *not* to apply it. That's the full set of things this topic is scored on.

---

# Putting it together

Seven worked examples in the shapes these questions actually arrive.

---

## Worked example 1 — "Design the booking model for a concert ticketing system."

**Don't draw tables yet. Ask for the rules and the peak** (thirty seconds):

> *"Three things decide the model: what must never happen, how hot the hottest show gets, and what the customer sees while a purchase is in progress. I'd assume a seat must never be sold twice, there's a per-customer limit per show, and at on-sale time a popular show gets thousands of attempts per second."*

Say the interviewer confirms: 20,000-seat venue, per-customer limit of 8 tickets per show, ~5,000 booking attempts/s in the first minute, payment happens after seats are held.

**Classify the invariants** (Concept 58):

| Rule | Must hold at commit? | Scope |
|---|---|---|
| A seat is sold at most once | **Yes** — double-selling is the worst outcome | Per seat |
| A customer holds at most 8 tickets per show | Business says a brief overshoot is acceptable if corrected before payment | Per (customer, show) |
| Held seats are released if not paid within 10 minutes | Eventually (a timeout policy) | Per hold |
| Show-level counters (sold, remaining) | No — derived, for display | Per show |

**Do the arithmetic before choosing the boundary** (Concept 65). A single `Show` aggregate containing all seats: λ ≈ 5,000/s, *d* ≈ 10 ms → λ·*d* ≈ 50. Conflict probability ≈ 1, ceiling ≈ 100 commits/s for the whole show. **That design cannot work.**

**The model:**

> *"The invariant that must hold at commit is per seat, so the consistency unit should be as close to the seat as possible. I'd make each **section** — a few hundred seats — an aggregate that holds seats, because customers usually pick seats within one section, and a hold needs all its seats to succeed together. Now 5,000 attempts/s are spread across ~60 sections, and contention is only between people competing for the same section. For the very front sections, which get disproportionate demand, I'd go further — row-level aggregates, or a single-writer actor per section so holds queue in memory instead of conflicting at the database.*
>
> *The hold is a reservation with a 10-minute expiry: `HoldSeats` either takes all requested seats or rejects, `ConfirmHold` after payment turns them into tickets, and an expiry policy releases stale holds. That's the decider shape — commands, events, pure decisions — so the rules are easy to test.*
>
> *The per-customer limit spans sections, so it's not a section invariant. I'd enforce it on a small `CustomerShowAllowance` aggregate keyed by (customer, show): a hold first reserves quantity there, then seats in the section. It's low-contention — one customer — so it's cheap. If the section hold fails, the allowance is released.*
>
> *Show-level counts for the seat map are a read model, updated from events — they can lag by a second. And before any of this, a virtual waiting room in front of the on-sale smooths 5,000/s into a rate the sections can absorb, which is a business-friendly way to turn a contention problem into a queue."*

**Why this scores:** invariants classified before modelling, arithmetic that rules out the naive design, a boundary chosen from the invariant's scope, a separate small aggregate for the cross-section rule, the hot-spot remedy named, and the read side kept out of the write model.

---

## Worked example 2 — "Review this code."

```csharp
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public Customer Customer { get; set; }
    public string Status { get; set; }
    public decimal Total { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
    public virtual ICollection<Shipment> Shipments { get; set; }
}

public class OrderService(AppDbContext db, IEmailSender email)
{
    public async Task PlaceOrder(Guid orderId)
    {
        var order = await db.Orders.Include(o => o.Customer).FirstAsync(o => o.Id == orderId);
        if (order.Lines.Count == 0) throw new Exception("Empty order");
        if (order.Customer.CreditLimit < order.Total) throw new Exception("No credit");
        order.Status = "Placed";
        order.Customer.OpenOrderCount++;
        await email.SendAsync(order.Customer.Email, "Order placed");
        await db.SaveChangesAsync();
    }
}
```

**The findings, most severe first:**

> *"First, correctness under concurrency: `Lines` isn't included, so `order.Lines.Count` is always zero on a freshly loaded order unless lazy loading is on — and if it is, the `virtual` shipments collection is lazy too, so this is a partial, lazily-traversed aggregate. Either way the emptiness check is unreliable.*
>
> *Second, the email is sent before `SaveChangesAsync`. If the save fails — including on a concurrency conflict — the customer gets a confirmation for an order that was never placed. That's the dual-write problem; the email should be driven from an `OrderPlaced` event through the outbox.*
>
> *Third, two aggregates change in one transaction: the order and the customer's `OpenOrderCount`, through a navigation property. Nobody decided that; the navigation made it possible. Every order placement now contends on the customer row. The count is derived data and should be updated eventually from the event — or dropped and computed by a query.*
>
> *Fourth, the model can't defend itself: public setters on `Status` and `Total`, a public mutable `List<OrderLine>`, and `Status` as a free-form string. Any code can put an order into any state, and `Total` can disagree with its lines.*
>
> *Fifth, the rules are in the service, not the order — so the next use case that places orders (bulk import, re-order) will reimplement them, slightly differently. And the credit check compares against a limit with no notion of what's already outstanding; that rule belongs to Billing, which should answer 'available credit' through a port.*
>
> *Smaller points: `FirstAsync` throws a generic exception for a missing order; `Exception` for business rejections loses the reason type; `Guid` IDs can be swapped with each other; no concurrency token at all.*
>
> *The refactoring: `Order` becomes an aggregate with private setters, a private lines collection, a `Place(availableCredit, now)` method returning an outcome, a status enum with a transition rule, `CustomerId` without a navigation, and a version bumped on every change. The service becomes a thin handler: load, ask Billing for available credit, call `Place`, save — with the email and the customer statistics moved to event handlers behind the outbox."*

(Concept 89 has the refactored `Order` in full.)

**Why this scores:** the concurrency and reliability bugs come before the style points, each finding names the principle it violates, and the fix is a coherent design rather than a list of patches.

---

## Worked example 3 — "An insurer wants to modernize. How would you identify the bounded contexts?"

> *"I'd start with the business, not the systems. A Big Picture EventStorming session with underwriters, claims handlers, finance, customer service and distribution, to lay out the end-to-end flow: quote requested, risk assessed, policy issued, premium billed, claim notified, liability decided, claim paid, policy renewed or lapsed. Then I'd look for three things on the wall."*

**Language divergence** (Concept 18):

| Word | Underwriting | Policy administration | Billing | Claims |
|---|---|---|---|---|
| **Policy** | A risk decision with terms and exclusions | A contract with versions, endorsements, a term | A premium schedule | A coverage definition to check a loss against |
| **Customer** | The insured risk (driver, vehicle, history) | The policyholder | The payer (may differ) | The claimant (may be a third party) |
| **Premium** | A price derived from risk factors | A contract term | An amount due on a date | Irrelevant |

**Pivotal events** — `PolicyIssued`, `ClaimNotified`, `LiabilityAccepted`, `ClaimSettled` — where responsibility and vocabulary change.

**Subdomain classification** (Concepts 11–12):

- **Core:** risk pricing/underwriting (the insurer wins on pricing accuracy) and claims handling (customer retention is decided at the claim).
- **Supporting:** broker management, policy document variations, customer-service case handling.
- **Generic:** payments, identity, the general ledger, document rendering, address validation — buy or adopt.

**Candidate contexts and context map:**

```
Pricing & Underwriting  [U, OHS/PL: RatingResult v1]  →  [D, conformist]  Policy Administration
Policy Administration   [U, OHS/PL: PolicyIssuedV1, CoverageQuery] → [D, ACL] Billing
Policy Administration   [U, OHS: CoverageQuery]       →  [D, ACL]         Claims
Claims                  [D, ACL]                      →  [U]              Payments provider (generic, bought)
Claims                  [U, PL: ClaimSettledV1]       →  [D]              Finance / General Ledger (bought)
Legacy PAS (big ball of mud)  ←  [ACL]  new Claims bubble context during migration
```

> *"Claims is where I'd start the modernization: it's core, it's where customers feel the service, and it can be built as a bubble context behind an ACL to the legacy policy system — reading coverage through a translator, not the legacy tables. Pricing is also core, but it's usually the riskiest to touch, so I'd protect it with a published contract first and modernize it second.*
>
> *Then I'd validate the boundaries by walking scenarios — a mid-term address change that re-rates the premium, a claim on a lapsed policy, a payer who isn't the policyholder — and counting the messages between contexts. The address change is interesting: in Policy Administration it's an endorsement, in Pricing it's a re-rating trigger, and in Billing it may change the tax jurisdiction. Three contexts, three meanings, one business event — which is exactly why it needs to be an intent-revealing event, `PolicyholderRelocated`, not `AddressUpdated`."*

**Why this scores:** the boundaries come from language, events and investment, not from existing systems; the core is justified by business reasons; the map carries integration patterns *and* a migration strategy; and the validation step shows the boundaries are hypotheses to test.

---

## Worked example 4 — "How do you guarantee that no two users register with the same email?"

> *"It's a set-based rule, so it can't live inside one `User` aggregate, and one aggregate containing all users would be the hottest possible aggregate. Which mechanism I'd use depends on the store.*
>
> *In a relational database, a unique index on the normalized email is the enforcement point. The domain normalizes it — an `EmailAddress` value object lower-cases and trims, so equality means what the business means — and the application service translates the unique-violation into a domain result, `EmailAlreadyInUse`, which the API returns as a 409. I'd add a pre-check for a friendlier message in the common case, but the index is what closes the race window. That trades a little completeness — part of the rule lives in the schema — for purity and performance, and I'd make it explicit with a named constraint and a test.*
>
> *In a store without secondary unique indexes — Cosmos DB, an event store — I'd make the email the identity of a small `EmailRegistration` aggregate. Registering claims it by inserting a document or stream whose ID is the normalized email, so a duplicate is a key conflict. Changing an email becomes a small process: claim the new one, update the user, release the old one.*
>
> *And if we were event-sourced on a DCB-capable store, the append condition 'no `EmailClaimed` event with this email tag exists' enforces it directly.*
>
> *The one thing I wouldn't do is check-then-insert without a constraint — that's a race that works in every test and fails in production."*

**Why this scores:** the rule is classified before a mechanism is chosen, there's an answer per storage model, the trade-off is named, and the race is called out explicitly.

---

## Worked example 5 — "Two teams share a `Customer` library and keep breaking each other. What do you do?"

> *"Before changing code I'd find out what 'customer' means to each team, because this is almost always two models forced into one. I'd put both teams in a room with their product owners and list the attributes and rules each one actually uses. Typically, Sales cares about pipeline stage, segment and account owner; Billing cares about legal entity, VAT number, payment terms and credit. The overlap is usually just an identifier, a name, and maybe a contact email.*
>
> *That's the evidence for two bounded contexts with their own `Customer` models. The shared part shrinks to what's genuinely common — a `CustomerId` and perhaps a tiny shared kernel for the identifier format — and each fact gets one owner: Sales owns contact and segment, Billing owns legal and financial details.*
>
> *Integration moves from a shared library to events in a published language: when Sales converts a prospect, it publishes `CustomerAcquiredV1` with the fields Billing needs; Billing's handler translates it into a `BillingAccount` in its own model — that handler is Billing's anticorruption layer. Each team can now change its model without the other noticing.*
>
> *The migration is incremental: freeze the shared library, introduce each team's own model behind their existing code, move each team off the shared types one use case at a time, and delete the library when nothing references it. Organizationally, I'd make sure each context has exactly one owning team; if the shared library had an owner, that team's job becomes the published contract.*
>
> *I'd also expect pushback that this 'duplicates the customer.' The answer is that they're different concepts sharing an ID; merging them is what's been causing the breakage, because every change to either team's customer requires agreement from both."*

**Why this scores:** diagnosis through language before code, a clear ownership rule per fact, the specific context-map patterns (published language upstream, ACL downstream), an incremental migration, and a prepared answer to the predictable objection.

---

## Worked example 6 — "We're building a subscription SaaS. Should we use DDD?"

> *"In parts. Let me classify first. A subscription SaaS typically has user and tenant management, a product catalogue and plan definitions, subscriptions and billing, usage metering and rating, invoicing and payments, an admin portal, and notifications.*
>
> *Identity, payments and email are generic — I'd use Entra External ID or a similar provider, a payment provider, and an email service, each behind an adapter so they can be swapped. The admin portal, plan maintenance and notification preferences are supporting and mostly CRUD — plain transaction scripts or EF directly, no aggregates, no domain events beyond what integration needs.*
>
> *The core is usually usage rating and subscription lifecycle — proration on mid-cycle plan changes, usage tiers, trials converting, grace periods, dunning, entitlements that must match what was paid for. That's where the rules are complex, change often and are directly tied to revenue, and it's where I'd invest in a proper domain model: a `Subscription` aggregate with an explicit lifecycle, value objects for money, billing periods and usage quantities with the proration and rounding logic inside them, and events like `PlanChanged` and `GracePeriodEntered` that drive invoicing and entitlements. If the business needs to explain how an invoice was computed months later, I'd consider event-sourcing the rating context specifically.*
>
> *So the answer is: DDD's strategic analysis everywhere, because it tells us where not to spend effort; the tactical patterns in one or two contexts. Probably 15–20% of the codebase ends up as a rich model, and that's the part that makes money."*

**Why this scores:** the classification drives the answer, the core is justified by revenue and rule complexity, generic parts are bought, supporting parts are deliberately simple, and there's a concrete estimate of how much of the system gets the full treatment.

---

## Worked example 7 — "Our `Order` aggregate throws `DbUpdateConcurrencyException` 30% of the time at peak. Fix it."

> *"A 30% conflict rate means λ·d is around 0.35 on the affected instances, so first I'd find out which instances. If it's spread evenly across orders, that's surprising — orders are usually edited by one customer — so I'd suspect something else is writing to them: a background job updating status, an event handler touching the order, or a navigation property dragging it into other transactions. If it's concentrated on a few instances, those are hot aggregates.*
>
> *Assume the investigation shows that every payment-provider webhook, every warehouse scan and every customer edit loads and saves the whole order, and that big B2B orders with hundreds of lines get dozens of scans a minute during picking. Then the aggregate is doing too much: payment status, fulfilment progress and the customer's line edits don't share an invariant once the order is placed. I'd split it — `Order` owns lines and pricing until it's placed; a `Fulfilment` aggregate owns picks and shipments; payment status lives with a `Payment` aggregate — linked by `OrderId` and coordinated by events. Each now has its own writers and its own version, and warehouse scans stop conflicting with customer edits.*
>
> *Two quick wins meanwhile: shorten d — make sure each command loads only what it needs, drop lazy loading, and move any I/O out of the transaction — and make the retry correct, re-running the decision on fresh state with a small, jittered retry budget.*
>
> *And I'd check the opposite failure: if some commands only touch child rows and the root isn't versioned, we may be seeing fewer conflicts than we should, with invariants silently broken. A real-database concurrency test per aggregate would tell us."*

**Why this scores:** it quantifies the symptom, looks for the cause before prescribing, splits by invariant rather than by table, offers short-term mitigations, and raises the subtle opposite bug.

---

## Common questions and what a strong answer contains

**"What's the difference between an entity and a value object?"** Identity vs attributes: if two instances with the same attributes are the same thing, it's a value (immutable, value equality, self-validating); if not, it's an entity (identity, lifecycle, equality by ID). Default to value objects (Concepts 41, 43).

**"What is an aggregate?"** A consistency boundary: the smallest cluster that must change atomically to keep a non-confluent invariant true. One root, references to other aggregates by ID, one per transaction, invariants guaranteed at commit (Concepts 56–57).

**"How big should an aggregate be?"** As small as its invariants allow. Check the per-instance command rate against ~1/d; bound every child collection (Concepts 60, 65, 67).

**"Why reference other aggregates by ID?"** Confines loads and transactions, prevents accidental multi-aggregate writes, and works unchanged across databases and services (Concept 61).

**"What if a use case needs to change two aggregates?"** Usually eventual consistency via domain events; within one database, one context and low contention, a deliberate single transaction; across services, reservation or a process manager (Concepts 62–63, 71).

**"What is a bounded context?"** The boundary within which one model and its language are consistent. One team, one glossary, its own data; a model boundary, not necessarily a service (Concepts 16, 21).

**"Subdomain vs bounded context?"** Problem space vs solution space; aim for 1:1, accept n:m with a reason (Concept 17).

**"Core, supporting, generic?"** Differentiating and complex; specific but simple; complex but solved. Build rich, build simple, buy (Concepts 11, 14).

**"What's an anticorruption layer?"** A downstream-owned translation layer that converts another model into yours at the boundary, so its concepts never enter your domain — essential for a core context consuming legacy or third-party models (Concept 27).

**"Name the context-mapping patterns."** Partnership, shared kernel, customer–supplier, conformist, ACL, open host service, published language, separate ways, big ball of mud — and explain each as a coupling choice (Concepts 22–32).

**"Domain events vs integration events?"** Internal, rich, in-process, can change freely vs cross-boundary, flat, versioned, outbox-delivered, the published language (Concepts 78, 83).

**"Where are domain events dispatched?"** Collected on the root, drained during `SaveChanges`; same-context database effects in the transaction, everything else through the outbox; loop until settled (Concepts 80–82).

**"How do you handle concurrency on an aggregate in EF Core?"** A version on the root as a concurrency token, bumped on every change — because child-only changes don't touch the root row. Retry machine commands by re-running the decision; surface human conflicts via ETag/412 (Concepts 68–69).

**"How do you enforce uniqueness?"** Unique index + translated violation in relational stores; the unique value as the identity of a small registration aggregate elsewhere; DCB if event-sourced; never check-then-insert alone (Concept 70).

**"What's the anaemic domain model and is it always bad?"** Data without behaviour, rules in services. Bad in the core, often correct in supporting CRUD — the harm is pretending (Concept 96).

**"What's DDD-lite?"** Tactical patterns without language, contexts or subdomain analysis — the costs without the benefits (Concept 97).

**"Should the domain model know about EF Core?"** No references to EF, three small compromises for EF-friendliness, mapping in infrastructure; a separate persistence model only when the schema is dictated elsewhere (Concept 91).

**"Do queries go through aggregates?"** No — project straight from storage; the model exists to protect writes (Concept 92).

**"How do you map value objects in EF Core 10?"** Complex types (value semantics), `ComplexCollection` to JSON for collections, converters for single-value types and IDs; owned types are legacy for this (Concept 47).

**"What's the Decider pattern?"** An aggregate as `decide(command, state) → events` and `evolve(state, event) → state`, both pure; works with event or state storage; maximally testable (Concept 73).

**"What's a Dynamic Consistency Boundary?"** Query-scoped optimistic locking in an event store: the boundary is whatever events the decision read, via tags. Solves cross-entity invariants without big aggregates; requires event sourcing, tag discipline and a DCB-capable store (Marten 9, Axon 5) (Concept 74).

**"How would you introduce DDD to an organization?"** Strategic first: subdomain map, current context map, core identified with the business; then a bubble context for one core capability behind an ACL; the DDD Starter Modelling Process as the method (Concepts 30, 38, 103).

**"Does DDD matter with AI?"** Evans' 2026 position: models, contexts and language still matter; LLM components are bounded contexts behind ACLs; the language becomes more valuable as a contract for people and tools (Concepts 34, 104).

**"How do you know your context boundaries are right?"** Walk the important scenarios across them and count the messages; chatty boundaries and ownerless decisions mean they're wrong (Concept 39).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Recites building blocks as a definition of DDD | Frames DDD as three decisions — consistency, language, investment — and derives the patterns |
| Starts with entities and repositories | Starts with subdomains, language and contexts; tactical design comes last |
| Designs aggregates from the ER diagram | Designs from commands and invariants; the aggregate holds only what decisions need |
| "The order contains the customer" | References other aggregates by ID; loads and transactions stay confined |
| Large aggregates "because they belong together" | Classifies invariants; keeps the boundary as small as they allow |
| No numbers for aggregate sizing | Per-instance command rate vs ~1/d; conflict probability 1 − e^(−λd) |
| Assumes a rowversion protects the whole aggregate | Knows child-only changes skip the root row; versions the root explicitly and tests it |
| Retries a conflict by re-applying the same changes | Re-runs the full decision on fresh state; surfaces human conflicts with ETags |
| Checks uniqueness in code, then inserts | Uses a constraint, a registry aggregate, or DCB — and names the trade-off |
| One `Customer` class for the whole company | One model per context; shared identity; each fact has one owner |
| Shares domain types across contexts | Published language upstream, ACL downstream; model coupling converted to contract coupling |
| Treats bounded context = microservice | Context is a model boundary; deployment is a separate decision |
| Uses the full tactical toolkit everywhere | Rich model in the core, transaction scripts in supporting areas, bought components for generic |
| Calls every anaemic model an anti-pattern | Distinguishes "anaemic but honest" in supporting areas from anaemia in the core |
| Events named after data changes (`OrderUpdated`) | Intent-revealing events (`OrderCancelledByCustomer`), immutable, with snapshots not entities |
| Sends email inside the transaction | Outbox for every external effect; in-transaction handlers write only to the database |
| Validates entities from the outside before saving | Always-valid model; parse at the edge; invariants in the aggregate |
| Loads aggregates to render lists | Projects reads straight from storage; the model is for writes |
| Mocks collaborators to test the domain | Given/when/then on state and events; property tests for invariants; one real concurrency test |
| Unaware of current developments | Can discuss EF 10 complex types, C# 15 closed hierarchies, DCB, and Evans' position on AI |
| Talks about DDD only as code | Uses context maps for org design, the core domain for investment, bubble contexts for brownfield |

---

## Practice exercises

**Exercise 1 — Classify a real domain (1 hr).** Pick a business you know well (your own product, a former employer's, or a public one). Write its subdomain map, classify each subdomain as core, supporting or generic with a one-sentence business justification, and plot them on a Core Domain Chart (differentiation × complexity). Then write down what you'd buy, what you'd build simply, and where you'd put your strongest engineers.

**Exercise 2 — The language audit (1 hr).** Take a codebase you know. List the ten most important domain terms and, for each, find every class, property and method that represents it. Where does the code use a different word from the business? Where does one class represent two meanings? Write a one-page glossary for one context, including "terms we deliberately don't use."

**Exercise 3 — A Big Picture EventStorming, solo or with a friend (2 hrs).** Use a free whiteboard tool and the DDD Crew's glossary and cheat sheet. Model an end-to-end process (online grocery delivery works well): events first, then pivotal events, hotspots and swimlanes. Draw candidate context boundaries and fill in a Bounded Context Canvas for the one you'd consider core.

**Exercise 4 — Draw a context map (1 hr).** For the system from Exercise 1 or 3, draw the current (or plausible) context map with upstream/downstream annotations and patterns at each end. Then draw the target map. Annotate each changed relationship with its Balanced Coupling reasoning — strength, distance, volatility.

**Exercise 5 — Value objects with behaviour (2 hrs).** Implement `Money` with `Allocate` (Concept 46), `DateRange` with `Overlaps`, `Intersect` and `Days`, and `EmailAddress` with normalization. Write property-based tests (FsCheck or CsCheck) proving that allocation never loses a cent and that intersection is commutative. Then deliberately write the positional-record `Percentage` from Concept 44 and demonstrate the `with` bypass before fixing it with `field`.

**Exercise 6 — Build and map the `Order` aggregate (3 hrs).** Implement Concept 89's `Order` and `OrderLine`, the `AggregateRoot` base with the version bump, and the EF Core 10 mapping with complex types, strongly-typed ID conventions and `AutoInclude`. Generate the migration and read the SQL: confirm the composite key, the concurrency token and the complex-type columns.

**Exercise 7 — Prove the concurrency bug and the fix (1.5 hrs).** With Testcontainers, write Concept 93's concurrent line-addition test. Remove the version bump and watch the test fail (the order ends up with 51 lines). Restore it and watch it pass. Then try the alternative: map lines as a `ComplexCollection` JSON column and confirm ordinary row concurrency now covers the aggregate.

**Exercise 8 — Contention in numbers (1 hr).** Write a small load test (or a simulation) that fires commands at one aggregate instance at increasing rates. Measure conflict rate and throughput, and compare them with 1 − e^(−λd) and 1/*d*. Then split the aggregate in two and re-run. Carry the numbers into interviews.

**Exercise 9 — Uniqueness three ways (2 hrs).** Implement "unique email" with (a) a unique index and a translated exception, (b) an `EmailRegistration` aggregate keyed by the email, and (c) a check-then-insert without a constraint. Hammer each with concurrent registrations and count duplicates. (c) should fail; that's the point.

**Exercise 10 — Domain events end to end (2 hrs).** Implement Concept 81's dispatcher and interceptor, a same-transaction handler that writes an integration event to an outbox, a background publisher, and a consumer in another module that translates the event into its own model (Concept 83). Make the consumer idempotent and prove it by delivering the same message twice.

**Exercise 11 — A decider (2 hrs).** Implement Concept 73's `SectionDecider` (C# 15 if you're on the .NET 11 RC; abstract records otherwise). Write given/when/then tests for every rule. Then persist it two ways — as an event stream and as a state row with a version — without changing the decider.

**Exercise 12 — DCB in Marten (2 hrs, optional).** Using Marten 9 and PostgreSQL, implement the course-enrollment rules from Concept 74 with tags and `FetchForWritingByTags`. Write a concurrent test that tries to enroll the 30th and 31st students simultaneously and confirm one gets `DcbConcurrencyException`. Write down what you'd need to change to do the same with aggregates, and compare.

**Exercise 13 — Review a real model (1 hr).** Open `dotnet/eShop`'s Ordering domain or `kgrzybek/modular-monolith-with-ddd` and walk Concept 101's review catalogue. For each item, note what the sample does and whether you'd do the same. Pay attention to where domain events are dispatched and whether child-only changes are covered by concurrency control.

**Exercise 14 — The three 60-second answers, recorded (45 min).** Record yourself answering Concept 105's three questions. Play them back and check each contains a framework, a number or mechanism, and a judgment about where *not* to apply DDD. Re-record until they do.

---

## Free resources

Roughly ninety free resources, grouped. Current-state notes are included where something has moved, been superseded, or needs a caveat before you cite it.

### Primary sources — start here

| Resource | What it covers | Why read it |
|---|---|---|
| [Domain-Driven Design Reference (PDF)](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf) — Eric Evans, 2015, CC BY 4.0 | Every pattern from the 2003 book in summary form, reorganized, plus three added since: domain events, partnership, big ball of mud | The authoritative short form. Free, ~60 pages. Read Parts I and IV first |
| [DDD Reference page](https://www.domainlanguage.com/ddd/reference/) · [DDD resources](https://www.domainlanguage.com/ddd/) — Domain Language | The reference plus the "Manager's Guided Tour" and the legacy-systems paper | Evans' own curated starting points |
| [Model Exploration Whirlpool](https://www.domainlanguage.com/ddd/whirlpool/) — Evans | Scenario → model → code probe → challenge → harvest | **Concept 7**, in Evans' words |
| [Effective Aggregate Design, Part I (PDF)](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_1.pdf) · [Part II](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_2.pdf) · [Part III](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_3.pdf) — Vaughn Vernon, 2011 | The four rules, the Scrum `Product` example, "whose job is it?", and discovering true invariants | **Concepts 58–63.** The single most-cited free text on aggregates |
| [Life beyond Distributed Transactions: an Apostate's Opinion](https://queue.acm.org/detail.cfm?id=3025012) — Pat Helland, ACM Queue 2016 (orig. CIDR 2007) | Entities as the unit of atomicity on one machine; messaging between them | **Concept 56** — the aggregate derived from the infrastructure side |
| [Coordination Avoidance in Database Systems](https://arxiv.org/abs/1402.2237) — Bailis et al. | Invariant confluence: which invariants need coordination | **Concept 56** — the formal reason aggregates exist (Module 9, Concept 31) |
| [Race Conditions Don't Exist](https://udidahan.com/2010/08/31/race-conditions-dont-exist/) — Udi Dahan, 2010 | Concurrency questions as missing business rules | **Concept 69.** Short, and it changes how you model conflicts |
| [Specifications (PDF)](https://www.martinfowler.com/apsupp/spec.pdf) — Eric Evans & Martin Fowler | The specification pattern: validation, selection, building to order | **Concept 53**, from the source |
| [Big Ball of Mud](http://www.laputan.org/mud/) — Brian Foote & Joseph Yoder | Why systems degrade into unstructured tangles, and why that persists | The origin of **Concept 30's** context-map pattern |
| [Domain-Driven Design Quickly](https://www.infoq.com/minibooks/domain-driven-design-quickly/) — InfoQ minibook | A ~100-page summary of the Blue Book | Useful refresher; dated in places but accurate on the fundamentals |

### Martin Fowler's bliki — short, precise definitions

| Entry | Covers |
|---|---|
| [DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html) | What DDD is, in two paragraphs, with pointers |
| [BoundedContext](https://martinfowler.com/bliki/BoundedContext.html) | The shortest correct definition, and why it's about language (**Concept 16**) |
| [UbiquitousLanguage](https://martinfowler.com/bliki/UbiquitousLanguage.html) | **Concept 5** |
| [DDD_Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html) | Aggregates as units of consistency and loading |
| [ValueObject](https://martinfowler.com/bliki/ValueObject.html) | Value equality, immutability, and the aliasing bugs they prevent (**Concept 43**) |
| [EvansClassification](https://martinfowler.com/bliki/EvansClassification.html) | Entities, value objects and services, as Evans classified them |
| [AnemicDomainModel](https://martinfowler.com/bliki/AnemicDomainModel.html) | The 2003 critique (**Concept 96**) |
| [Domain Event](https://martinfowler.com/eaaDev/DomainEvent.html) | Early description of events as a modelling concept |
| [What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html) | Notification, state transfer, sourcing, CQRS (**Concept 78**) |

### Strategic design, context mapping and coupling

| Resource | What it covers |
|---|---|
| [DDD Crew — Context Mapping](https://github.com/ddd-crew/context-mapping) | Every pattern and team relationship, with a cheat sheet (**Concepts 22–31**) |
| [DDD Crew — Context Mapping Quiz](https://github.com/ddd-crew/context-mapping-quiz) | Test yourself on the patterns — good pre-interview drill |
| [DDD Crew — Bounded Context Canvas](https://github.com/ddd-crew/bounded-context-canvas) | The one-page context design template, updated May 2026 (**Concept 19**) |
| [DDD Crew — Core Domain Charts](https://github.com/ddd-crew/core-domain-charts) | Differentiation × complexity; finding the core collaboratively (**Concept 12**) |
| [DDD Crew — Welcome to DDD](https://github.com/ddd-crew/welcome-to-ddd) | Definitions of the fundamental terms, community-agreed |
| [Strategic Domain Driven Design with Context Mapping](https://www.infoq.com/articles/ddd-contextmapping/) — Alberto Brandolini, InfoQ | The classic article on context maps as organizational diagnosis (**Concept 31**) |
| [Balanced Coupling](https://coupling.dev/) — Vlad Khononov | Integration strength, distance, volatility (**Concept 32**) |
| [Dimensions of Coupling](https://coupling.dev/posts/dimensions-of-coupling/) · [Balancing Coupling: a formula](https://coupling.dev/posts/core-concepts/balance/) | The three dimensions and the balance heuristic in detail |
| [Context Mapper](https://contextmapper.org/) | A DSL and tooling for context maps as code — generate diagrams, analyse relationships |
| [Getting Started with DDD When Surrounded by Legacy Systems — implementation](https://github.com/DDD-Hamburg/ddd-in-legacy-systems) | Bubble context, autonomous bubble and friends in code (**Concept 30**); the paper is linked from Domain Language's resources page |
| [Team Topologies — key concepts](https://teamtopologies.com/key-concepts) | Team types and interaction modes — the organizational half of the context map (**Concept 103**) |
| [Wardley Maps (free book chapters)](https://medium.com/wardleymaps) — Simon Wardley | Evolution from genesis to commodity (**Concept 13**) |

### Discovery and collaborative modelling

| Resource | What it covers |
|---|---|
| [EventStorming](https://www.eventstorming.com/) — Alberto Brandolini | The method, formats and resources (**Concept 35**) |
| [DDD Crew — EventStorming Glossary & Cheat Sheet](https://github.com/ddd-crew/eventstorming-glossary-cheat-sheet) | Notation, colours, facilitation tips — and why "aggregate" is now "constraint" |
| [DDD Crew — DDD Starter Modelling Process](https://github.com/ddd-crew/ddd-starter-modelling-process) | The eight-step process (**Concept 38**), updated May 2026 |
| [DDD Crew — Domain Message Flow Modelling](https://github.com/ddd-crew/domain-message-flow-modelling) | Validating boundaries by drawing scenario message flows (**Concept 39**) |
| [DDD Crew — Aggregate Design Canvas](https://github.com/ddd-crew/aggregate-design-canvas) | The aggregate checklist, including throughput and size (**Concepts 65, 76**) |
| [DDD Crew — Virtual Modelling Templates](https://github.com/ddd-crew/virtual-modelling-templates) | Miro-ready templates for remote workshops |
| [SAP — DDD Kata](https://github.com/SAP/curated-resources-for-domain-driven-design/blob/main/ddd-kata.md) | An end-to-end practice problem through EventStorming, message flow, canvases |
| [Domain Storytelling](https://domainstorytelling.org/) · [Egon modeller](https://egon.io/) | The method and the free tool (**Concept 37**) |
| [What is Event Modeling?](https://eventmodeling.org/posts/what-is-event-modeling/) — Adam Dymitruk | Slices, commands, events, views (**Concept 37**) |
| [Introducing Example Mapping](https://cucumber.io/blog/bdd/example-mapping-introduction/) — Matt Wynne | Rules, examples and questions per story (**Concept 37**) |

### Tactical patterns — the best short articles

| Resource | What it covers |
|---|---|
| [What is Domain-Driven Design?](https://verraes.net/2021/09/what-is-domain-driven-design-ddd/) — Mathias Verraes, 2021 | A careful modern definition centred on language and models |
| [Design and Reality](https://verraes.net/2021/09/design-and-reality/) — Mathias Verraes & Rebecca Wirfs-Brock, 2021 | Models are invented, not discovered (**Concept 4**) |
| [Entity vs Value Object: the ultimate list of differences](https://enterprisecraftsmanship.com/posts/entity-vs-value-object-the-ultimate-list-of-differences/) — Vladimir Khorikov | **Concepts 41, 43** |
| [Always-Valid Domain Model](https://enterprisecraftsmanship.com/posts/always-valid-domain-model/) — Khorikov, 2021 | Invariants vs input validation (**Concept 48**) |
| [Domain model purity vs. completeness (the DDD trilemma)](https://enterprisecraftsmanship.com/posts/domain-model-purity-completeness/) — Khorikov, 2020 | Completeness, purity, performance — pick two (**Concepts 51, 70**) |
| [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — Alexis King | Types that can only hold valid values (**Concept 48**) |
| [Designing with types](https://fsharpforfunandprofit.com/series/designing-with-types/) — Scott Wlaschin | Making illegal states unrepresentable, step by step; F#, directly applicable to C# 15 (**Concept 49**) |
| [Domain Modeling Made Functional (talk & resources)](https://fsharpforfunandprofit.com/ddd/) — Scott Wlaschin | DDD with algebraic types |
| [Functional Event Sourcing Decider](https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider) — Jérémie Chassaing, 2021 | decide/evolve/initialState (**Concept 73**) |
| [A better domain events pattern](https://lostechies.com/jimmybogard/2014/05/13/a-better-domain-events-pattern/) — Jimmy Bogard, 2014 | Collect on entities, dispatch at save (**Concept 80**) |
| [Domain Events – Salvation](https://udidahan.com/2009/06/14/domain-events-salvation/) — Udi Dahan, 2009 | The static-raiser pattern — read for history and to recognize it in old code |
| [Strengthening your domain: a primer](https://lostechies.com/jimmybogard/2010/02/04/strengthening-your-domain-a-primer/) — Jimmy Bogard | A series on moving behaviour into the model |
| [An introduction to strongly-typed entity IDs](https://andrewlock.net/using-strongly-typed-entity-ids-to-avoid-primitive-obsession-part-1/) — Andrew Lock | The series that popularized them in .NET (**Concept 45**) |

### Kamil Grzybek — .NET-specific DDD implementation posts

| Post | Covers |
|---|---|
| [How to publish and handle Domain Events](https://www.kamilgrzybek.com/blog/posts/how-to-publish-handle-domain-events) | Dispatch mechanics in .NET (**Concept 81**) |
| [Handling Domain Events: Missing Part](https://www.kamilgrzybek.com/blog/posts/handling-domain-event-missing-part) | Reliability — the outbox side |
| [Domain Model Encapsulation and PI with Entity Framework](https://www.kamilgrzybek.com/blog/posts/domain-model-encapsulation-ef) | Encapsulated aggregates mapped with EF (**Concept 89**) — written for EF Core 2.2; principles hold, APIs moved on |
| [Domain Model Validation](https://www.kamilgrzybek.com/blog/posts/domain-model-validation) | Business-rule validation inside the model (**Concept 94**) |
| [Processing multiple aggregates — transactional vs eventual consistency](https://www.kamilgrzybek.com/blog/posts/processing-multiple-aggregates-transactional-vs-eventual-consistency) | **Concepts 62–63** |
| [Handling concurrency — Aggregate Pattern and EF Core](https://www.kamilgrzybek.com/blog/posts/handling-concurrency-aggregate-pattern-ef-core) | Aggregate-level concurrency in EF (**Concept 68**) |

### Dynamic Consistency Boundary

| Resource | What it covers |
|---|---|
| [dcb.events](https://dcb.events/) · [Specification](https://dcb.events/specification/) · [Examples](https://dcb.events/examples/) | The specification (Waidelich, Pellegrini, Grimshaw) and worked examples: course subscriptions, unique usernames, gapless invoice numbers, idempotency (**Concept 74**) |
| [Marten — Dynamic Consistency Boundary](https://martendb.io/events/dcb.html) | The .NET implementation: tags, `FetchForWritingByTags`, `DcbConcurrencyException`, storage modes |
| [Marten 9.0, Polecat 4.0 and Wolverine 6.0 are live](https://jeremydmiller.com/2026/05/24/marten-9-0-polecat-4-0-and-wolverine-9-0-are-live/) — Jeremy Miller, May 2026 | The Critter Stack 2026 release, including the DCB changes |
| [Dynamic Consistency Boundary in Axon Framework 5](https://www.axoniq.io/blog/dcb-in-af-5) | The JVM implementation and its positioning |
| [Dynamic consistency boundaries](https://javapro.io/2025/10/28/dynamic-consistency-boundaries/) — JAVAPRO | A balanced write-up of what DCB changes and its risks |

### Microsoft Learn — .NET and Azure guidance

| Resource | What it covers and its current state |
|---|---|
| [Tackle business complexity with DDD and CQRS patterns](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/) | The DDD chapter of the free .NET Microservices e-book. Still the most complete .NET treatment; references the archived eShopOnContainers and dispatches events before `SaveChanges` via MediatR |
| [Design a microservice domain model](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model) | Aggregates, entities, anaemic vs rich — with Vernon's PDFs linked |
| [Implement a microservice domain model with .NET](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/net-core-microservice-domain-model) | Encapsulated aggregates in C# |
| [Seedwork (reusable base classes)](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/seedwork-domain-model-base-classes-interfaces) | `Entity`, `ValueObject`, `IAggregateRoot` base types |
| [Implement value objects](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/implement-value-objects) | The equality-components pattern; note it predates EF Core 10 complex types |
| [Use enumeration classes instead of enum types](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/enumeration-classes-over-enum-types) | **Concept 54** |
| [Design validations in the domain model layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-model-layer-validations) | **Concept 94** |
| [Domain events: design and implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation) | **Concepts 80–82**, Microsoft's version |
| [Design the infrastructure persistence layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design) · [Implement it with EF Core](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core) | Repositories and unit of work (**Concepts 86–87**) |
| [Use domain analysis to model microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis) | Strategic DDD worked through a drone-delivery example; now recommends Khononov's book |
| [Use tactical DDD to design microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/tactical-domain-driven-design) | Updated Feb 2026 — "no smaller than an aggregate and no larger than a bounded context" (**Concept 20**) |
| [Identify microservice boundaries](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/microservice-boundaries) | From aggregates and contexts to service boundaries |
| [Anti-corruption Layer pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) | **Concept 27** in the Azure pattern catalogue |
| [API design for microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/api-design) | Aggregates as REST resources; coarse-grained APIs over aggregates |

### EF Core documentation for domain models

| Resource | What it covers |
|---|---|
| [Complex types](https://learn.microsoft.com/en-us/ef/core/modeling/complex-types) | Value semantics, optional complex types, `ComplexCollection` → JSON (**Concept 47**) |
| [What's new in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew) · [EF Core 11](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-11.0/whatsnew) | The complex-type changes and the owned-type migration advice |
| [Handling concurrency conflicts](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) | Concurrency tokens and resolution (**Concept 68**) |
| [Backing fields](https://learn.microsoft.com/en-us/ef/core/modeling/backing-field) · [Entity types with constructors](https://learn.microsoft.com/en-us/ef/core/modeling/constructors) | Encapsulation-friendly mapping; reconstitution (**Concepts 89–90**) |
| [Value conversions](https://learn.microsoft.com/en-us/ef/core/modeling/value-conversions) | Strongly-typed IDs and single-value VOs (**Concept 45**) |
| [Eager loading (including AutoInclude)](https://learn.microsoft.com/en-us/ef/core/querying/related-data/eager) | Loading aggregates whole by configuration (**Concept 89**) |
| [Interceptors](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors) | `SaveChangesInterceptor` for events and outbox (**Concept 81**) |

### .NET reference implementations and libraries

| Resource | What it covers |
|---|---|
| [dotnet/eShop — Ordering.Domain](https://github.com/dotnet/eShop/tree/main/src/Ordering.Domain) | Microsoft's current reference: aggregates, seedwork, domain events. Review it critically with **Concept 101** |
| [kgrzybek/modular-monolith-with-ddd](https://github.com/kgrzybek/modular-monolith-with-ddd) | The fullest open .NET DDD reference: modules, aggregates, business rules, domain events, outbox, ADRs |
| [kgrzybek/sample-dotnet-core-cqrs-api](https://github.com/kgrzybek/sample-dotnet-core-cqrs-api) | A smaller, single-context example of the same ideas |
| [oskardudycz/EventSourcing.NetCore](https://github.com/oskardudycz/EventSourcing.NetCore) | Aggregates, deciders and event sourcing in .NET, with exercises — the bridge to Module 24 |
| [event-driven.io](https://event-driven.io/en/) — Oskar Dudycz | Practical articles on aggregates, slim aggregates, deciders and events |
| [SteveDunn/Vogen](https://github.com/SteveDunn/Vogen) | Source-generated value objects with analyzers that ban invalid construction (**Concept 45**) |
| [andrewlock/StronglyTypedId](https://github.com/andrewlock/StronglyTypedId) | Lighter-weight, ID-focused generator |
| [ardalis/Specification](https://github.com/ardalis/Specification) | Query specifications for EF Core — the *selection* use of **Concept 53** |
| [Marten](https://martendb.io/) · [JasperFx/marten](https://github.com/JasperFx/marten) | PostgreSQL document store and event store with DCB (**Concept 74**) |
| [Microsoft Orleans — overview](https://learn.microsoft.com/en-us/dotnet/orleans/overview) · [Request scheduling](https://learn.microsoft.com/en-us/dotnet/orleans/grains/request-scheduling) | Virtual actors; the single-threaded, turn-based execution model that makes a grain a single-writer aggregate host (**Concept 75**) |
| [citerus/dddsample-core](https://github.com/citerus/dddsample-core) | The cargo-shipping sample from Evans' book, in Java — worth reading for the model, not the stack |

### Community, curated lists and collections

| Resource | What it is |
|---|---|
| [DDD Crew — Free DDD Learning Resources](https://github.com/ddd-crew/free-ddd-learning-resources) | A curated list of free material, all accessible without paying |
| [SAP — Curated Resources for DDD](https://github.com/SAP/curated-resources-for-domain-driven-design) | A structured learning path with commentary, from introduction to deep dives |
| [awesome-ddd](https://github.com/heynickc/awesome-ddd) | The long list: books, talks, samples, libraries, communities |
| [Domain-Driven Design: The First 15 Years](https://leanpub.com/ddd_first_15_years) — DDD Europe | Essays by Fowler, Wirfs-Brock, Coplien, Conway, Khononov and others; **minimum price free** on Leanpub |
| [DDD Crew — AI DDD prompts and rules](https://github.com/ddd-crew/ai-ddd-prompts-and-rules) | DDD-oriented prompts and rules for coding assistants (**Concept 104**) |
| [Virtual DDD](https://virtualddd.com/) | Online community with regular free sessions and recordings |
| [DDD Community](https://www.dddcommunity.org/) | The long-running community library (hosts Vernon's essays) |
| [DDD Europe](https://dddeurope.com/) | The main conference; years of talks free on its YouTube channel |

### Current-state reading on DDD and AI

| Resource | What it covers |
|---|---|
| [Context Mapping with an AI-based Component](https://www.domainlanguage.com/articles/context-mapping-an-ai-based-component/) — Eric Evans, Jan 2026 | An LLM as a bounded context; ACLs at the probabilistic seam (**Concept 34**) |
| [DDD Europe 2026 — Opening Keynote, Eric Evans](https://2026.dddeurope.com/program/opening-keynote-eric-evans/) | "Domain models still matter, bounded contexts still matter, language still matters — but the models will look different" |
| [Eric Evans Encourages DDD Practitioners to Experiment with LLMs](https://www.infoq.com/news/2024/03/Evans-ddd-experiment-llm/) — InfoQ, 2024 | The Explore DDD 2024 keynote: a trained language model as a bounded context |

### Talks worth an hour (search by title — free on YouTube)

| Talk | Why |
|---|---|
| **"DDD & Microservices: At Last, Some Boundaries!"** — Eric Evans, GOTO 2015 | Bounded contexts as the organizing idea behind service boundaries — and where teams go wrong |
| **"Tackling Complexity in the Heart of Software"** — Eric Evans, DDD Europe 2016 | Evans' own retrospective framing, strategic-first |
| **["Getting Started with DDD When Surrounded by Legacy Systems"](https://www.youtube.com/watch?v=ZbnF0Dn6dAA)** — Eric Evans | The bubble-context strategies (**Concept 30**) |
| **["What I've learned about DDD since the book"](https://www.infoq.com/presentations/ddd-eric-evans/)** — Eric Evans, InfoQ | Domain events, bounded contexts and what he'd emphasize differently |
| **"Domain Modeling Made Functional"** — Scott Wlaschin | Types as a design tool; the best talk on illegal-states-unrepresentable |
| **"50.000 Orange Stickies Later"** — Alberto Brandolini | EventStorming from its inventor, including what goes wrong |
| **["Functional Event Sourcing Decider"](https://www.youtube.com/watch?v=kgYGMVDHQHs)** — Jérémie Chassaing (Event-Driven Information Systems) | The decider in depth (**Concept 73**) |
| **["Balancing Coupling in Software Design"](https://www.infoq.com/presentations/video-podcast-vlad-khononov/)** — Vlad Khononov, InfoQ | Strength, distance, volatility (**Concept 32**) |
| **"Dissecting Bounded Contexts"** — Nick Tune, DDD Europe 2020 | Heuristics for splitting and sizing contexts |
| **"Critically Engaging with Models"** — Mathias Verraes & Rebecca Wirfs-Brock, DDD Europe 2022 | Models as tools; knowing when a model has stopped helping |
| **"Finding Service Boundaries"** — Udi Dahan | Boundaries by business capability and data ownership, illustrated |

### Books (not free, listed for completeness)

- **Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software*** (2003). The Blue Book. Read Part IV (strategic design) before Part II.
- **Vaughn Vernon — *Implementing Domain-Driven Design*** (2013). The Red Book; the practical companion, strategy first.
- **Vaughn Vernon — *Domain-Driven Design Distilled*** (2016). The shortest good book on the subject.
- **Vlad Khononov — *Learning Domain-Driven Design*** (O'Reilly, 2021). The best modern introduction; subdomain-to-pattern heuristics (**Concepts 8, 14**). Now recommended by Microsoft's architecture guidance.
- **Vlad Khononov — *Balancing Coupling in Software Design*** (Addison-Wesley, 2024). **Concept 32**'s theory.
- **Scott Millett & Nick Tune — *Patterns, Principles, and Practices of Domain-Driven Design*** (Wrox, 2015). Encyclopaedic, with C# examples.
- **Scott Wlaschin — *Domain Modeling Made Functional*** (2018). F#, but the best book on modelling with types.
- **Stefan Hofer & Henning Schwentner — *Domain Storytelling*** (2021).
- **Alberto Brandolini — *Introducing EventStorming*** (Leanpub).
- **Nick Tune with Jean-Georges Perrin — *Architecture Modernization*** (Manning, 2024). DDD, EventStorming, Wardley mapping and Team Topologies applied to modernization — the architect-track book.
- **Rebecca Wirfs-Brock & Mathias Verraes — *Design and Reality*** (Leanpub). Essays on modelling; includes material not published elsewhere.
- **Alexey Zimarev — *Hands-On Domain-Driven Design with .NET Core*** (Packt, 2019). Dated framework versions, sound modelling.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| DDD in one sentence | A method for three decisions — consistency (aggregates), language (contexts), investment (subdomains) |
| Why DDD at all | Essential complexity lives in business rules; if the rules are trivial, DDD is overhead |
| Strategic vs tactical | Strategy constrains tactics; tactical-only is DDD-lite |
| A model | A tool for a purpose, not a copy of reality — judged by what it makes easy and impossible |
| Ubiquitous language | One vocabulary per context, in speech, tests and code; renames are design changes |
| When not to use DDD | Supporting CRUD → transaction scripts; generic → buy; rich model only where rules are complex |
| Core / supporting / generic | Differentiating + complex; specific + simple; complex + solved → build rich, build simple, buy |
| Finding the core | "If we were twice as good at this, would we win?" — Core Domain Charts |
| Subdomain vs bounded context | Problem space vs solution space; aim 1:1, accept n:m with a reason |
| Bounded context | Boundary of one consistent model and language; one team; own data |
| Context size | No bigger than one team and one language; no smaller than an aggregate |
| Context vs service | Model boundary vs deployment boundary — separate decisions |
| Context-map patterns | Partnership, shared kernel, customer–supplier, conformist, ACL, OHS, PL, separate ways, big ball of mud |
| ACL | Downstream translation layer; keeps another model out of yours; essential for a core context |
| OHS + PL | Upstream's stable, versioned protocol for many consumers — not its internal model |
| Balanced coupling | Strength XOR distance, or low volatility; ACL/PL turn model coupling into contract coupling |
| Legacy strategy | Bubble context behind an ACL; autonomous bubble; expand |
| LLMs in DDD | A bounded context with a probabilistic model; wrap in an ACL; output is a proposal |
| EventStorming | Big Picture → Process → Design level; orange events, blue commands, yellow constraints |
| Validating boundaries | Walk real scenarios; count cross-boundary messages |
| Entity | Identity and lifecycle; equality by ID; assign identity at construction |
| Identity on SQL Server | `Guid.CreateVersion7()` isn't sequential in SQL Server's ordering — use SQL-Server-ordered GUIDs |
| Value object | Attributes only; immutable; self-validating; prefer them |
| VO traps in C# | Positional records skip validation; `with` skips it again; collections compare by reference |
| VOs in EF Core 10 | Complex types; `ComplexCollection` → JSON; converters for IDs; owned types are legacy |
| Always-valid | Construct valid or not at all; parse at the edge; never validate on load |
| Illegal states | States as types — C# 15 `closed` hierarchies; data per state |
| Domain service | Domain logic with no natural owner; stateless; pure; not an application service |
| DDD trilemma | Completeness, purity, performance — pick two; usually keep purity |
| Factories | Named creation; creation ≠ reconstitution |
| Deriving the aggregate | Non-confluent invariants need serialization; aggregate = smallest serialized unit |
| Aggregate rules | Root only; local identity inside; reference others by ID; invariants hold at commit |
| True invariant test | "Must it hold at commit?" and "whose job is it?" |
| Vernon's rules | True invariants; small; by ID; eventual outside — and when to break them |
| Aggregate sizing | P(conflict) ≈ 1 − e^(−λd); ceiling ≈ 1/d (~100/s at 10 ms) |
| Hot aggregate | Split along the invariant; escrow; reserve; single writer; conditional write; relax |
| Growing collection | Separate aggregate, or keep only the summary the invariant needs |
| EF concurrency trap | Child-only changes don't write the root — version the root, bump on every change |
| Conflict resolution | Re-run the decision for machines; ETag/412 for humans; race conditions are business rules |
| Uniqueness | Unique index + translated error; or the value as a registration aggregate's ID; or DCB |
| Cross-aggregate rules | Rethink boundary; co-transact deliberately; reserve; process manager; DCB |
| Decider | decide(cmd, state) → events; evolve(state, event) → state; pure |
| DCB | Query-scoped append condition; boundary per decision; needs event sourcing and tag discipline |
| Aggregate at scale | Aggregate ID = partition key = message key = actor key |
| Domain event | Past-tense business fact, immutable, raised by the deciding aggregate |
| Event naming | Intent (`CustomerRelocated`), never CRUD (`CustomerUpdated`) |
| Event dispatch | Collect on root; drain in `SaveChanges`; loop; DB-only in-transaction; outbox for the rest |
| Translation | Domain event → versioned integration event (PL) → downstream ACL → downstream model |
| Eventual consistency UX | Received vs done; pending states as domain states; read-your-writes; a lag budget |
| Process manager | A workflow's consistency boundary — an aggregate with timeouts and compensations |
| Repository | One per aggregate root; `Get`/`Add`; no `Update`, no `IQueryable` |
| Application service | Load → decide → save; no rules; one aggregate; one transaction |
| Reads | Bypass the aggregate; project from storage |
| Testing | Given/when/then on state and events; properties for invariants; one real concurrency test |
| Errors | Outcomes for expected rejections; exceptions for violated preconditions; 400/409/412/422 |
| Anaemic model | Bad in the core; honest and fine in supporting CRUD |
| The close | Three decisions; derived boundaries; numbers; explicit translation; uneven on purpose |

---

This module closes the loops it was created to close:

- **Module 12's promise** (Concepts 32 and 47) — that aggregates would turn the transaction boundary from a storage constraint into a design method — is kept: Concept 56 derives the aggregate from invariant confluence, Concepts 58–66 turn "where's the transaction boundary?" into questions the business answers and numbers you can compute.
- **Module 9's invariant confluence** (Concept 31) is now the first-principles reason aggregates exist, and escrow reappears as a hot-aggregate remedy (Concept 66).
- **Module 16's** records, strongly-typed IDs, "make illegal states unrepresentable" and C# 15 `closed`/`union` types became the implementation vocabulary for value objects, lifecycles, deciders and outcomes (Concepts 44–49, 72–73, 95).
- **Module 19's** complex types, concurrency tokens, interceptors and DDD mapping features are assembled into one aggregate end to end — including the child-row concurrency trap EF doesn't warn you about (Concepts 47, 68, 81, 89).
- **Module 20's** domain-centric layering, domain vs integration events and the CQRS-lite read side now have their DDD justification (Concepts 78, 83, 92).
- **Module 21's** bounded-contexts-are-the-source, duplication-is-a-feature and EventStorming are now a method: language heuristics, context maps, coupling analysis and scenario validation (Parts C and D).
- **Module 11's** outbox and delivery semantics are the transport for domain and integration events (Concepts 81–83).

Threads left open on purpose:

- **CQRS and MediatR** — whether the read side of Concept 92 deserves a separate model, store or framework, and what the MediatR licensing change means for command dispatch — is **Module 23**.
- **Event sourcing** — where the domain events of Part G become the system of record, the decider of Concept 73 becomes the natural aggregate shape, and DCB (Concept 74) competes with stream-per-aggregate — is **Module 24**.
- **Resilience with Polly** — including how the concurrency-retry loop of Concept 69 composes with transient-fault retries without doubling up — is **Module 25**.
- **Cosmos DB partitioning** — the aggregate-as-partition-key idea of Concept 75, with RU costs and transactional batches — is **Module 27**.
- **ADRs and the C4 model** — documenting context maps and aggregate decisions so they survive personnel changes — are **Module 31**.
- **Brownfield modernization** — the organizational side of Concept 30's bubble contexts — is **Module 32**.
- **Build-vs-buy and the executive conversation** — the language for Concept 103's investment case — is **Module 33**.

Next in the curriculum: **Module 23 — CQRS & MediatR**: when separating the read model from the write model is worth its complexity tax, how far to take it (from a separate query path in one database to separate stores fed by events), and what the 2025–2026 licensing changes mean for the libraries most .NET teams reach for first.
