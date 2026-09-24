# Module 24 — Event Sourcing
*Phase 5: .NET Architecture Patterns · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **event sourcing is one decision — the events are the system of record, and every piece of current state is a derived, disposable interpretation of them — and almost everything that makes it hard follows from one consequence of that decision: you can never again change the past, only add to it, so the shape of your events, the length of your streams and the personal data inside them become commitments you must design to last a decade, not a sprint.**

That sentence has two halves, and the module is built around both.

The first half is the *promise*. When the log of facts is the truth, state stops being precious. You can compute today's balance, last March's balance, a dashboard nobody thought of when the events were written, or the exact sequence of decisions that led to a customer complaint — all from the same source. The write model can be a pure function (Module 22's decider). Every read model becomes rebuildable by definition (Module 23's hardest operational problem disappears). Concurrency gets a clean, cheap mechanism: "append these facts only if the stream is still at version *n*." Audit is not a feature you add; it is the storage format.

The second half is the *price*. An event stored today will be read by code written years from now, by people who were not in the room, under regulations that did not exist when it was written. Events cannot be `UPDATE`d when you discover a typo in a property name, a wrong enum value, a bug that emitted the wrong fact for three weeks, or a customer's right to be forgotten. Streams that grow without bound slow down every command that touches them. There is no `SELECT … WHERE` over the write side — every question needs a projection, and every projection needs subscriptions, checkpoints, idempotency and rebuilds. The team must learn a different way of thinking, and the tooling ecosystem, while much better than it was, is smaller than the relational one.

A mid-level candidate can define event sourcing ("store events instead of state") and draw an event store with projections coming off it. A senior candidate can explain why it's a *persistence* decision scoped to one bounded context rather than an architecture; why command sourcing and event streaming are not event sourcing; how optimistic concurrency on a stream works and what the expected version really protects; why `evolve` must never throw; how to decide stream boundaries and why streams must be kept short; when a snapshot pays for itself, in milliseconds; how a relational event store can hand a subscriber events out of order and silently lose one; how to make the append condition of a Dynamic Consistency Boundary actually safe under concurrency; how to evolve an event schema over five years without rewriting history; why crypto-shredding is only as good as your key backups; how long a full replay will take, in hours; and — above all — when *not* to use any of it.

This module closes threads that earlier modules opened by name. **Module 11** separated event sourcing from event-driven architecture and warned against proposing event sourcing as a top-level architecture; here that warning gets its full argument. **Module 12** gave you append-only logs, LSM trees, isolation levels and the write-skew anomaly; they reappear as the physical event store and as the reason a naïve DCB append condition is unsafe (Concept 36). **Module 22** gave you aggregates as consistency boundaries (Concepts 56–66), domain events (Concepts 78–83), the Decider (Concept 73) and Dynamic Consistency Boundaries (Concept 74), and promised that "choosing event sourcing later doesn't change the domain code"; Part D shows that promise kept. **Module 23** described event sourcing as level 4 of the CQRS spectrum (Concept 8), built the entire projection toolkit — idempotency, ordering, checkpoints, poison events, rebuilds, lag, consistency tokens (Concepts 34–57) — and said explicitly that event sourcing, "where the write side *is* the event log, projections become the only way to query, rebuilds become replays, and events carry personal data that must be erasable," is this module. **Module 7** gave you per-object linearizability and session guarantees; a stream with an expected version is exactly the former. **Module 9** gave you logs as the core of replication and consensus; an event store is a replicated log with a domain on top.

It shows up in five places in an interview loop: the **design round** (a wallet, a ledger, a booking system, an order lifecycle with a compliance requirement — the interviewer listens for whether you scope event sourcing to the part that needs it, and whether you mention versioning and privacy unprompted), the **deep technical round** ("how do you handle a concurrency conflict? when do you snapshot? how do you change an event's schema? how do you delete a customer's data?"), the **code-review round** (an event-sourced aggregate whose `Apply` method validates and throws, or a projection that sends emails, is a favourite), the **architect round** ("should we event-source the whole platform?" is a question whose best answer is usually "no, and here is the one context where yes"), and the **behavioural round** ("tell me about a technical decision you'd make differently" — event sourcing adopted too broadly is a classic story, on either side of the table).

**Current platform state (verified September 2026).** .NET 10 (November 2025) is the production-current LTS; .NET 11 RC1 shipped 8 September 2026 ahead of GA on 10 November 2026. What changed in this module's ecosystem, and why each item matters:

- **EventStoreDB is now KurrentDB, under a source-available licence, and its licensing line moves again in 26.2.** The rebrand shipped as KurrentDB 25.0 (March 2025), which also moved the licence from the Event Store License v2 to the **Kurrent License v1 (KLv1)** — source-available, *not* OSI-approved open source, with enterprise features switched on by licence key — and introduced **archiving** of old chunks to object storage. **25.1** (October 2025) added **secondary indexes** (by category and event type, stored in an embedded DuckDB rather than as `$by_category`/`$by_event_type` link events) and **multi-stream appends** with an expected version per stream. **26.0** (January 2026) is the current LTS: user-defined secondary indexes over event content, Azure Blob and GCP as archive targets. **26.1** (May 2026) added SQL access through Apache Arrow Flight SQL, a webhook source connector, major persistent-subscription stability work and an experimental "Projections V2" engine. On **13 August 2026** Kurrent announced that from **26.2** (expected Q3 2026; previewed 20 August) clustering moves from the Paxos-derived protocol to **Raft**, and **multi-node clustering, read-only replicas, archiving and encryption at rest require an Enterprise licence**, while 21 previously licensed features (connectors, OAuth/LDAP/X.509 authentication, OpenTelemetry exporters, the Kubernetes Operator, SQL access and others) become free; **single-node production stays free**, and versions up to 26.1 keep their current terms for their full LTS period. The .NET client is **`KurrentDB.Client` 1.4.x**; the legacy TCP API is gone (its last server was EventStoreDB 23.10). **Why it matters here:** "we'll run a three-node KurrentDB cluster" is now a procurement conversation, and "EventStoreDB is open source" is no longer true (Concepts 41, 100).
- **The Critter Stack shipped its 2026 wave, and the .NET event-store choice now spans three databases with one API.** **Marten 9.0**, **Polecat 4.0** and **Wolverine 6.0** shipped together in May 2026 on a shared **JasperFx 2.0** foundation, targeting .NET 9 and 10, with faster cold start and AOT work (the 9.3x line runs event-sourced applications under Native AOT). Marten 9's default append mode is `QuickWithServerTimestamps`; it added an opt-in **HStore** storage mode for Dynamic Consistency Boundary tags; **9.4 fixed a correctness bug in which truly concurrent DCB appends could both commit** (Concept 36); and it is at 9.39.x in September 2026. **Polecat** — a port of most of Marten to **SQL Server 2025's native `json` type**, MIT-licensed, System.Text.Json only, source-generated — went 1.0 in March 2026 and is at 5.31 (23 September 2026); event upcasting was still an open work item in early September. **Fisher 1.0** (19 August 2026) brings the same API to **embedded SQLite**; all three stores enrol in a shared `JasperFx.Events.ComplianceTests` suite. Wolverine 6.x (6.27, August 2026) added store-agnostic attributes for its aggregate-handler workflow that run unchanged against all three. **Why it matters here:** "we're a SQL Server shop" is no longer an argument against a mature .NET event-sourcing library — and "which store?" now has a real decision matrix (Concepts 40, 98).
- **Dynamic Consistency Boundaries went from an idea to an ecosystem.** The specification at **dcb.events** (Bastian Waidelich, Sara Pellegrini, Paul Grimshaw) is implemented by Marten and Polecat in .NET, Axon Server/Framework 5 on the JVM, EventSourcingDB, UmaDB, the Python `eventsourcing` library and several .NET community projects (Sekiban.Dcb on Orleans, Opossum on the file system). **Why it matters here:** the interviewer who asks "how do you enforce a rule across two aggregates?" may now expect DCB as one of the options — and a senior answer includes its costs and its concurrency subtleties (Concepts 35–37).
- **Microsoft rewrote its guidance, and it now leads with a warning.** The Azure Architecture Center's *Event Sourcing pattern* was substantially rewritten on 27 March 2026. It opens by calling event sourcing a complex pattern with significant trade-offs that is costly to migrate to or from and constrains future design; it says most systems don't need it; it adds sections on intent-focused event design, versioning strategies (tolerant deserialization, versioned events, upcasting, in-place migration as a last resort), idempotent consumers, crypto-shredding and keeping personal data out of the store, and an explicit warning **not to confuse an event store with a message broker such as Kafka**. Its overview diagram still routes new events through a queue to a handler that writes them to the store — a design that, taken literally, loses the optimistic-concurrency check its own text relies on (Concept 12 explains why you should read it critically). **Why it matters here:** the platform vendor now says what this module says — scope it, version it, protect privacy, and don't use Kafka as the store.
- **The legal ground under crypto-shredding moved, without settling.** The CJEU's judgment in ***EDPS v SRB*** (C-413/23 P, 4 September 2025) held that whether pseudonymised data is personal data depends on whether the party holding it can reasonably re-identify the subject — a relative, context-dependent test — while the **EDPB's Guidelines 01/2025 on pseudonymisation** (adopted for consultation in January 2025) treat pseudonymised data as personal data for the controller that can reverse it. **Why it matters here:** "we crypto-shred, so we're compliant" is an engineering claim about a legal question; the senior answer designs the mechanism *and* routes the sufficiency question to the data protection officer (Concepts 79–81).
- **Other .NET options remain stable and specialised.** **Eventuous** (Apache-2.0; 0.16.4 stable, 0.17 documentation in preparation in September 2026) offers aggregates, command services, subscriptions and producers over KurrentDB, PostgreSQL, SQL Server and others. **Orleans 10** keeps `JournaledGrain` in `Microsoft.Orleans.EventSourcing` for event-sourced actors. **Cosmos DB** remains a viable do-it-yourself event store (Concept 42), with Microsoft's own design-pattern samples. **Why it matters here:** you can name the options and say what each is for.

This module has ten jobs:

1. **Define event sourcing precisely** — as a persistence decision where history is the source of truth for decisions — and separate it from its neighbours: event-driven architecture, event streaming, CQRS, audit logs and command sourcing.
2. **Design events that last** — named for intent, carrying outcomes, with a stable envelope, stable type names and deliberate serialization.
3. **Design streams and concurrency** — stream boundaries as consistency boundaries, short streams, optimistic concurrency, idempotent appends, cross-stream rules and Dynamic Consistency Boundaries done correctly.
4. **Understand event stores from the inside** — what one must provide, how to build one on a relational database, why global ordering has gaps, and what Marten, Polecat, KurrentDB, Cosmos DB and Kafka actually give you.
5. **Build the write side** — the command loop, OO aggregates and deciders over a store, snapshots, process managers and side effects.
6. **Treat projections as the only read path** — subscriptions, checkpoints, safe reading of the global log, rebuilds as replays, temporal queries, integration publishing and read-your-writes.
7. **Evolve events over years** — the change catalogue, tolerant reading, upcasting, mixed-version deployments, copy-and-transform, and fixing wrong events.
8. **Handle privacy, retention and security** — erasure in an immutable log, crypto-shredding and its traps, archiving, compaction, tamper evidence and multi-tenancy.
9. **Operate it** — metrics, arithmetic for rehydration, snapshots, rebuilds and storage, testing, debugging, and migrating to and away from event sourcing.
10. **Take the architect's view** — scope, store choice, build-vs-buy, licences, anti-patterns, review checklists, and how to talk about all of it in design and architect rounds.

Ten framings to carry through:

1. **History is the database; everything else is a cache.** Aggregate state, snapshots and read models are interpretations. If you can't delete and rebuild one, you've made it a second source of truth.
2. **Store decisions, not requests.** Events record what the system decided; replaying them must never re-decide. That is the line between event sourcing and command sourcing.
3. **An event is a contract with your future self.** It will be read by code that doesn't exist yet, for years. Name it for intent, give it a stable type name, and put in it what the future will need.
4. **A stream is a consistency boundary, and its length is a design parameter.** Choose boundaries from invariants (Module 22); keep streams short by modeling lifecycles.
5. **The expected version is the lock.** Optimistic concurrency on the stream is the entire write-side concurrency model; a conflict means "decide again on fresher facts."
6. **`evolve` records; `decide` judges.** Validation lives in the decision. Applying a stored fact must be total, pure and unable to fail.
7. **The past is fixed; the interpretation isn't.** You can't edit events, but you can change every projection, every aggregate and every upcaster — that freedom is the payoff.
8. **Version by conversion, or it's a new event.** A new version of an event must be derivable from the old one; if it isn't, you have a new fact, not a new schema.
9. **Personal data in an immutable log is a design decision made on day one.** Keep it out, encrypt it per subject, or accept rewriting — but decide before the first event is stored.
10. **Scope it to one context, and be able to say why.** Event sourcing is a persistence choice for a part of a system with a reason — audit, temporal questions, a ledger, a workflow — never a platform default.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | One decision, one consequence | Events are the record; state is derived; the past can only be appended to |
| 2 | From ledgers to software | Accountants never erase; CRUD throws away the "how" and the "why" |
| 3 | What an event is | An immutable, past-tense, business-meaningful record of a decision |
| 4 | State is a left fold | `state = events.Aggregate(initial, evolve)` — deterministic, total, pure |
| 5 | Events as state vs state from events | Event sourcing means history is the source of truth *for decisions* |
| 6 | What event sourcing is not | Not EDA, not event streaming, not CQRS, not an audit log, not CDC |
| 7 | Command sourcing: the trap next door | Store outcomes, not requests — replay must never re-decide |
| 8 | Other ways to keep history | Audit tables, temporal tables, ledger tables, CDC — and what each lacks |
| 9 | Why teams choose it | Audit, temporal questions, new read models from the past, intent, debugging |
| 10 | What it costs | Evolution, learning curve, tooling, rebuilds, privacy — and a different way of thinking |
| 11 | Where it pays, where it doesn't | Per context: ledgers, workflows, collaborative domains — not CRUD |
| 12 | Anatomy of an event-sourced system | Load → fold → decide → append-if-version → subscribe → project/react/publish |
| 13 | The stored event: envelope and payload | IDs, stream, version, position, type, schema version, times, correlation, tags |
| 14 | Naming events | Past tense, intent, ubiquitous language — never `CustomerUpdated` |
| 15 | Granularity | One business fact per event; a command may produce several |
| 16 | What goes in an event | The decision's outcome plus what consumers need — including computed results |
| 17 | What stays out | Derived state, whole-entity snapshots, blobs, unnecessary personal data |
| 18 | Time in events | Occurred vs recorded vs effective; inject the clock |
| 19 | Internal vs published events | The event store is private; publish translated integration events |
| 20 | Event type names are forever | Stable string names decoupled from CLR types; a registry with aliases |
| 21 | Serialization | System.Text.Json, source generation, tolerant reading, the enum/decimal/date traps |
| 22 | Metadata that pays for itself | Correlation, causation, actor, tenant, trace context — in the envelope |
| 23 | A contract with your future self | Events outlive code, teams and frameworks — design for a decade |
| 24 | Designing events with Event Modeling | Commands, events and views on a timeline, sliced |
| 25 | Streams | An ordered sequence per consistency boundary; `category-id` naming |
| 26 | What an event store must provide | Append-if-version, read stream, read all, subscribe, idempotent append |
| 27 | Positions: version vs global position | Gapless per stream, possibly gappy globally — and why subscribers care |
| 28 | Optimistic concurrency on streams | The expected version is the lock; a conflict means "decide again" |
| 29 | Idempotent appends | Event IDs, store-level de-duplication, command idempotency keys |
| 30 | Stream boundaries are consistency boundaries | Stream design is aggregate design (Module 22) |
| 31 | Keep streams short | Lifecycle and temporal modeling; closing the books |
| 32 | Hot streams | Conflict arithmetic, append-rate ceilings and remedies |
| 33 | Invariants across streams | Bigger stream, reservation, registry stream, process manager, relax |
| 34 | Multi-stream atomic appends | Marten transactions, KurrentDB 25.1 — capability vs smell |
| 35 | Dynamic Consistency Boundaries, in depth | Tags, query, append condition — a boundary per decision |
| 36 | Getting the append condition right | Check-then-act is write skew; convert predicates into row conflicts |
| 37 | DCB or aggregates? | Index cost, contention scope, tag discipline, tooling maturity |
| 38 | A relational event store, designed | Tables, keys, indexes, isolation and the append statement |
| 39 | The global-order gap problem | Sequences commit out of order; read only below a safe high-water mark |
| 40 | The .NET event-store landscape | Marten, Polecat, Fisher, KurrentDB, Eventuous, Orleans, Cosmos DIY |
| 41 | KurrentDB in depth | `$all`, subscriptions, indexes, projections, archiving, clustering, licence |
| 42 | Cosmos DB as an event store | Stream = logical partition, `id` = version, transactional batch, change feed |
| 43 | Kafka is not an event store | No per-key expected version or per-entity reads — a distribution layer |
| 44 | The command-handling loop | Load, fold, decide, append — and nothing else |
| 45 | The OO event-sourced aggregate | Uncommitted events, apply methods, version tracking |
| 46 | The decider over an event store | One generic handler for every decider |
| 47 | Evolve never decides | Total, pure, no validation, no exceptions, no I/O |
| 48 | Invariants, decisions and bad events | Validate before emitting; a wrong fact is fixed forward |
| 49 | Resolving concurrency conflicts | Re-decide, surface, or merge by event semantics |
| 50 | Snapshots | A cache of the fold — measured, versioned, disposable |
| 51 | Live, inline and async write models | Where the write model's state comes from, and how fresh it is |
| 52 | Process managers in an event-sourced system | Event-sourced workflow state, timeouts, compensations |
| 53 | Side effects and reactors | Never in evolve; after commit; idempotent; never on replay |
| 54 | Testing the write side | Given events, when command, then events — plus properties |
| 55 | Projections become mandatory | Level 4: every query is answered by a projection |
| 56 | Kinds of projection | Single-stream, multi-stream, per-event, flat table, live |
| 57 | Inline, async, live | Consistency, cost and failure coupling of each lifecycle |
| 58 | Subscriptions | Catch-up vs persistent vs change feed; checkpoints; ordering |
| 59 | Reading the global log safely | High-water marks, gap detection, skip-ahead |
| 60 | Idempotency by position | The event's position is the natural idempotency key |
| 61 | Rebuild = replay | Parallelism, blue/green projection versions, no side effects |
| 62 | Projections that need other data | Enrich from events, not from live lookups |
| 63 | Temporal queries | As-of state, time travel, bi-temporal reads |
| 64 | Projections, reactors, process managers | Three consumers, three replay rules |
| 65 | Publishing integration events from the store | Subscription as outbox; translation; delivery guarantees |
| 66 | Read-your-writes in event-sourced systems | Inline projections, returned versions, consistency tokens |
| 67 | Why versioning is the hard part | A new version must be convertible from the old — or it's a new event |
| 68 | The change catalogue | Add, rename, retype, split, merge, re-mean, remove |
| 69 | Weak schema and tolerant readers | Absorb additive change without new versions |
| 70 | Upcasting | Transform old shapes on read; chain; keep one current type |
| 71 | Mixed-version deployments | Old code reading new events; deploy order; downcasting |
| 72 | Copy-and-transform | Migrate streams into new streams or stores — the heavy tool |
| 73 | Rewriting history in place | The last resort — and what it breaks |
| 74 | Fixing wrong events | Compensate, correct, skip — never silently edit |
| 75 | Guarding compatibility | Golden event samples and approval tests in CI |
| 76 | Retiring event types | Upcast forward, keep the name, delete the class |
| 77 | Evolving everything else freely | Aggregates and projections change without touching the past |
| 78 | The five proven methods | Overeem et al.'s methods mapped to tools and decisions |
| 79 | Immutability meets the right to erasure | What GDPR asks, and why it isn't only a technical question |
| 80 | Keep personal data out of events | Reference by ID; a mutable, deletable personal-data store |
| 81 | Crypto-shredding | Per-subject keys, envelope encryption — and the backup trap |
| 82 | Masking, tombstones and deletion | Rewriting and removing, with eyes open |
| 83 | Erasure must propagate | Projections, caches, search, analytics, consumers, logs |
| 84 | Retention, archiving and compaction | Archive streams, archive tiers, compaction, closing the books |
| 85 | Backup and restore | The log is the database — restore, then replay |
| 86 | Securing the log | Append-only permissions, tamper evidence, encryption, least privilege |
| 87 | Multi-tenancy | Conjoined vs database-per-tenant, and the tenant-bleed failure |
| 88 | Observability | Append latency, conflicts, stream lengths, lag, dead letters |
| 89 | Rehydration and snapshot arithmetic | When a snapshot pays, in milliseconds |
| 90 | Rebuild and storage arithmetic | Hours to replay, gigabytes per year |
| 91 | The testing pyramid | Deciders, projections, upcasters, golden files, containers, concurrency |
| 92 | Debugging with history | Time travel, local replay, reproducing from streams |
| 93 | Migrating to event sourcing | Import events, migration events, one context at a time |
| 94 | Migrating away | Freeze the fold into tables; reversibility as a criterion |
| 95 | The human cost | Learning curve, onboarding, tooling, on-call |
| 96 | Scope: per context, never the whole system | A persistence choice inside a boundary |
| 97 | Across services and modules | Private streams, published contracts, no shared store |
| 98 | Choosing a store | Platform fit, operations, features, licence |
| 99 | Build, buy or DIY | When a hand-rolled store is fine, and when it isn't |
| 100 | Licences and vendor risk | KLv1, the 26.2 clustering licence, MIT plus commercial support |
| 101 | The anti-pattern catalogue | Fourteen ways event-sourced systems go wrong |
| 102 | The review checklist | What a reviewer checks, in order |
| 103 | Event sourcing in the design round | Where it appears in the 7 steps, and how to justify it |
| 104 | Event sourcing in the architect round | Cost, reversibility, team, operations, the ADR |
| 105 | The 60-second answers and the close | What it is; when to use it; one paragraph of judgment |

---

# Part A — What event sourcing actually is

Most explanations of event sourcing start with a picture: an event store in the middle, a command handler writing to it, projections fanning out to read databases, a bus carrying events to other services. That's a *system* that happens to use event sourcing, and starting there is why so many teams adopt the whole picture when they needed one piece of it. Part A starts from the decision itself, separates it from the five things it is routinely confused with, and ends with the question that matters most in an interview: where does it pay, and where doesn't it?

---

## Concept 1 — One decision, one consequence

The whole of event sourcing is one decision:

> **The system of record is the sequence of events that have happened. Current state is not stored as truth; it is computed from the events whenever it's needed.**

In a conventional ("state-stored") system, the database holds the current state of each entity — an `Orders` row with `Status = 'Shipped'`, `Total = 180.00`. Each change overwrites the previous value. History exists only if someone adds it separately.

In an event-sourced system, the database holds facts:

```
order-42:
  v1  OrderDrafted        { customerId: c-7, currency: EUR }
  v2  OrderLineAdded      { sku: KB-01, qty: 1, unitPrice: 120.00, newTotal: 120.00 }
  v3  OrderLineAdded      { sku: MS-04, qty: 2, unitPrice: 30.00,  newTotal: 180.00 }
  v4  OrderPlaced         { total: 180.00, placedAt: 2026-09-12T10:14Z }
  v5  OrderShipped        { carrier: DHL, trackingNo: …, shippedAt: 2026-09-13T08:02Z }
```

"The order is shipped and totals €180" is an *interpretation* of those five facts — computed by folding them (Concept 4). It can be computed again tomorrow, differently if the business asks a different question.

That decision has **one consequence** from which almost every difficulty follows:

> **You can never change the past; you can only add to it.**

Everything in this module is either a benefit of that consequence or a cost of it:

| Because the past is fixed… | …you get | …and you pay |
|---|---|---|
| Nothing is ever overwritten | A complete, trustworthy history — audit by construction | Storage that only grows; streams that get long (Concepts 31, 84) |
| State is always derived | Any view, including new ones, can be rebuilt from the beginning (Concept 61) | Every query needs a projection (Concept 55) |
| Events are the contract | Rich, intent-revealing facts (Concept 14) | Event schemas must be evolved without rewriting (Part F) |
| Facts can't be edited | Non-repudiation; reproducible bugs | Mistakes are fixed forward, never erased (Concept 74) |
| Data is kept forever by default | Temporal questions: "what did we know on 1 March?" (Concept 63) | Personal data conflicts with the right to erasure (Part G) |
| Writes are appends | A clean concurrency primitive: append-if-version (Concept 28) | Contention concentrates on hot streams (Concept 32) |

The interview framing: *"Event sourcing is a persistence decision: the events are the system of record and state is derived from them. Its one consequence — you can only append to the past — gives you audit, replay and rebuildable read models for free, and makes schema evolution, stream length and personal data permanent design problems. So I'd use it where the history itself has business value, and scope it to that part of the system."*

---

## Concept 2 — From ledgers to software

Event sourcing is not a software invention. It is how every serious record-keeping discipline has worked for centuries.

**Accounting.** A general ledger is an append-only list of journal entries. The balance of an account is the sum of its entries. An error is never erased — it is corrected by a **reversing entry** and a new correct entry, so the books show both the mistake and the fix. Periodically the books are **closed**: balances are carried forward as opening balances of the next period, and the old period is archived. Pat Helland's line — accountants don't use erasers — is the whole philosophy in five words, and Concept 31's "closing the books" pattern is borrowed directly from it.

**Law and medicine.** A patient record is appended to; a correction is a new, dated, signed entry referencing the old one. A land registry records transfers; ownership is derived.

**Databases themselves.** Every relational database already keeps an append-only log — the write-ahead log (Module 12) — and the tables are, in a real sense, a cache of the log's effect. Replication ships the log (Module 8); point-in-time restore replays it. Jay Kreps' essay *The Log* and Martin Kleppmann's *Turning the database inside out* make this observation the foundation of stream processing. Event sourcing applies the same idea one level up: the log records *business* facts instead of page writes.

What a CRUD system throws away, by comparison:

```
UPDATE Customers SET Street = 'Bulevar Kralja Aleksandra 73', City = 'Belgrade', Version = 18
WHERE Id = 'c-7' AND Version = 17;
```

After this statement, the database no longer knows:

- **What the address was before** (unless an audit table was added).
- **Why it changed** — did the customer move, or was a typo corrected? Those are different business events with different consequences (a move might change the tax region and the delivery zone; a typo correction changes nothing downstream).
- **When the change became true in the world** versus when it was recorded (Concept 18).
- **Who decided it**, and in response to what.

A CRUD model can be made to keep all of that — with audit tables, temporal tables, change data capture and careful discipline (Concept 8). Event sourcing makes it the *default*, because there is nowhere else for the information to go.

---

## Concept 3 — What an event is

In event sourcing, an **event** is a record of a decision the system has made, with a precise set of properties. Each property has a consequence:

| Property | Meaning | Consequence |
|---|---|---|
| **A fact** | It happened. It is not a request, a suggestion or an intention | It can't be rejected by anyone who reads it; readers adapt to it |
| **In the past tense** | `OrderPlaced`, `PaymentCaptured`, `SeatsReserved` | The name states what became true, not what someone asked for |
| **Immutable** | Once stored, never modified | Mistakes are corrected by *new* events (Concept 74) |
| **Business-meaningful** | Named in the ubiquitous language (Module 22); a domain expert would recognise it | Events serve as documentation, audit and the basis for new read models |
| **Intent-revealing** | Records *why*, not just *what changed* | `CustomerRelocated` and `CustomerAddressCorrected` are different events even though both change an address |
| **Self-sufficient** | Carries what the future needs to interpret it | No need to look up mutable data at replay time (Concept 16) |
| **Ordered within its stream** | Has a position relative to other events about the same thing | State can be computed deterministically by folding in order (Concept 4) |
| **Attributable** | Has an identity, a time, and metadata about who and what caused it | Enables idempotency, tracing and audit (Concepts 13, 22) |

A test for whether something is an event in the event-sourcing sense: **would a domain expert say "that happened" and care?** "The shopping cart was saved" fails — saving is a technical act. "An item was added to the cart" passes. "The customer's record was updated" fails. "The customer moved to a new address" passes.

A second test: **could the event be refused?** If a reader could say "no" to it, it's not an event, it's a command (Module 23, Concept 11). An event describes something the system already decided; its only possible readers are ones that adapt their own state in response.

Three kinds of "event" that get confused, and which ones live in an event store:

- **Domain events** (Module 22, Concept 78): facts raised by an aggregate about its own state change. In an event-sourced context, **these are what you store** — they are the state.
- **Integration events** (Module 22, Concept 83): facts published to other contexts in a stable, versioned, often flatter shape. In an event-sourced context, **these are derived from stored events** and published — they are not the storage format (Concept 19).
- **Notification or technical events** ("cache invalidated", "projection rebuilt", "heartbeat"): system mechanics. **These don't belong in the domain's streams** — mixing them in is one of the mistakes Nat Pryce's team described after their first event-sourced system (Concept 101).

---

## Concept 4 — State is a left fold

If events are the truth, current state is a function of them. The function has a specific shape — the **left fold** (`Aggregate` in LINQ, `reduce` elsewhere):

```
state = events.Aggregate(initialState, evolve)       // evolve : (State, Event) → State
```

Module 22 (Concept 73) introduced this as the `evolve` half of the Decider. Here's the same idea in C# for the order stream above:

```csharp
public abstract record OrderEvent;                                   // on .NET 11: public closed record OrderEvent;
public sealed record OrderDrafted(Guid CustomerId, string Currency) : OrderEvent;
public sealed record OrderLineAdded(string Sku, int Qty, decimal UnitPrice, decimal NewTotal) : OrderEvent;
public sealed record OrderLineRemoved(string Sku, decimal NewTotal) : OrderEvent;
public sealed record OrderPlaced(decimal Total, DateTimeOffset PlacedAt) : OrderEvent;
public sealed record OrderShipped(string Carrier, string TrackingNo, DateTimeOffset ShippedAt) : OrderEvent;
public sealed record OrderCancelled(string Reason, DateTimeOffset CancelledAt) : OrderEvent;

public enum OrderStatus { None, Draft, Placed, Shipped, Cancelled }

public sealed record OrderState(
    OrderStatus Status, Guid CustomerId, string Currency, decimal Total,
    ImmutableDictionary<string, int> Lines, DateTimeOffset? PlacedAt)
{
    public static readonly OrderState Initial =
        new(OrderStatus.None, Guid.Empty, "", 0m, ImmutableDictionary<string, int>.Empty, null);
}

public static class OrderEvolve
{
    public static OrderState Evolve(OrderState s, OrderEvent e) => e switch
    {
        OrderDrafted d     => s with { Status = OrderStatus.Draft, CustomerId = d.CustomerId, Currency = d.Currency },
        OrderLineAdded a   => s with { Lines = s.Lines.SetItem(a.Sku, s.Lines.GetValueOrDefault(a.Sku) + a.Qty), Total = a.NewTotal },
        OrderLineRemoved r => s with { Lines = s.Lines.Remove(r.Sku), Total = r.NewTotal },
        OrderPlaced p      => s with { Status = OrderStatus.Placed, PlacedAt = p.PlacedAt, Total = p.Total },
        OrderShipped       => s with { Status = OrderStatus.Shipped },
        OrderCancelled     => s with { Status = OrderStatus.Cancelled },
        _                  => s,                                        // unknown event types: ignore (Concept 69)
    };

    public static OrderState Fold(IEnumerable<OrderEvent> events) => events.Aggregate(OrderState.Initial, Evolve);
}
```

Four properties the fold **must** have, and why:

1. **Deterministic.** The same events must produce the same state, today and in five years. No `DateTime.UtcNow`, no `Guid.NewGuid()`, no random numbers, no configuration lookups, no calls to other services. If `evolve` read the current VAT rate from configuration, replaying last year's events would compute last year's orders with this year's VAT. Everything `evolve` needs must be *in the event* (Concept 16).
2. **Total.** It must accept every event that can appear in the stream, including events it doesn't care about, and never throw (Concept 47). A stored fact that makes `evolve` throw is a stream that can never be loaded again — a permanently broken entity.
3. **Pure.** No side effects. `evolve` runs every time the entity is loaded, during every rebuild and in every test. An email sent from `evolve` is sent on every load.
4. **Order-sensitive, but only within a stream.** Folding `OrderShipped` before `OrderPlaced` gives the wrong state, which is why a stream has a strict order (Concept 27). Across streams, order is irrelevant to this fold.

One more point that separates senior answers: **the fold is the *definition* of state, not necessarily how you compute it at runtime.** You may cache it (snapshots, Concept 50), maintain it incrementally (inline projections, Concept 57) or compute it lazily. All of those are optimizations that must produce the same result as the fold — and a good test suite checks exactly that (Concept 91).

---

## Concept 5 — Events as state vs state from events

The phrase "event sourcing" is used for two different things, and interviews go wrong when the candidate and the interviewer mean different ones. Mathias Verraes named the distinction (*Eventsourcing: State from Events or Events as State?*, 2019):

**State from Events.** You have a stream of events — from anywhere: clickstream, IoT telemetry, a CDC feed, another system's integration events — and you compute state from them. That's stream processing, analytics, materialized views, projections. It says nothing about how the events were *produced*.

**Events as State.** The events are the **single source of truth for the component that produced them**, and that component **uses its own history to make decisions**: to accept a new command, it loads its events, folds them into state, checks its invariants against that state, and appends new events only if the history hasn't changed in the meantime.

Verraes' proposal — and the definition this module uses — is to reserve **event sourcing** for the second:

> **Event sourcing: a component's events are its system of record, and it enforces its business rules against its own event history when deciding what new events to record.**

Why the distinction matters in practice:

| | State from events | Events as state (event sourcing) |
|---|---|---|
| Where events come from | Anywhere | The component itself, as the outcome of its decisions |
| Is the log the source of truth for decisions? | Not necessarily — the producer may have its own database | Yes — there is no other copy |
| Consistency requirement on appends | Usually none (append whatever arrives) | **Strong**: append-if-version, per stream (Concept 28) |
| Typical tools | Kafka, Event Hubs, stream processors, CDC | Event stores: KurrentDB, Marten, Polecat, custom |
| Example | A fraud model computing risk scores from payment events | A wallet that refuses a withdrawal if the stream's balance is insufficient |

Oskar Dudycz makes the same point from the other side (*Event Streaming is not Event Sourcing!*): if the write model's state is read from a materialized view that was built from events, the truth has been outsourced to that view — the component isn't event-sourced, even if events exist.

The interview line: *"I use 'event sourcing' in the narrow sense: the component's own events are its system of record and it decides against its history, with append-if-version concurrency. Computing state from someone else's event stream is stream processing — useful, but a different design with different guarantees."*

---

## Concept 6 — What event sourcing is not

Interviewers probe these because each confusion leads to an expensive design. Event sourcing is **not**:

**1. Event-driven architecture.** EDA (Module 11) is about components communicating through events: a service publishes `OrderPlaced`, others react. Nothing in EDA says how the publisher *stores* its state — most event-driven services are state-stored with an outbox. Event sourcing is about *persistence* inside one component. You can have either without the other, and Martin Fowler's *What do you mean by "Event-Driven"?* lists event sourcing as only one of four distinct meanings of the phrase (with event notification, event-carried state transfer and CQRS).

**2. Event streaming (Kafka, Event Hubs, Pulsar).** A streaming platform moves events between systems and retains them for replay. It does not provide per-entity optimistic concurrency or efficient reads of one entity's history — the two things an event store must do (Concept 26). Using Kafka as an event store is a well-documented failure mode (Concept 43).

**3. CQRS.** Module 23 stated the dependency precisely: event sourcing *needs* CQRS (you can't query an event store by attributes, so reads need projections), but CQRS doesn't need event sourcing (most CQRS systems are state-stored).

**4. An audit log.** An audit log is a *secondary* record written alongside the real state. It can drift from the state (written in a different transaction, or not written at all by a bulk update), and nobody notices because nothing reads it for decisions. In event sourcing, the "audit log" is the only record — if an event is missing, the state is wrong, visibly. That's why event sourcing's audit guarantee is stronger: the history is *load-bearing*.

**5. Change data capture.** CDC (Module 23, Concept 38) emits row-level diffs after the fact. It captures *what* changed in a table, not *why*, and the table remains the truth. CDC on an outbox table is a publishing mechanism, not event sourcing.

**6. A top-level architecture.** Greg Young, who popularized the term, has said repeatedly — most memorably in *A Decade of DDD, CQRS, Event Sourcing* (DDD Europe 2016) — that CQRS and event sourcing are not top-level architectures and should be applied to specific parts of a system. Dennis Doomen's experience report puts it bluntly: event sourcing is a tool, not an architecture style to use everywhere.

**7. Microservices.** Event sourcing is orthogonal to deployment. An event-sourced bounded context can live in a modular monolith (Module 21) as comfortably as in a service.

**8. The same as storing domain events in an outbox.** An outbox (Module 11) is a *transport* table of events to publish; rows are deleted or archived after publishing, and the entities' state is in its own tables. An event store *is* the state.

**9. Only for high-scale systems.** Event sourcing's benefits are about history, intent and replay, not throughput. Many excellent event-sourced systems handle a few writes per second; many very high-throughput systems are state-stored.

The interview line: *"Event sourcing is a persistence choice for one component: its events are its state. It's not the same as publishing events, not Kafka, not CQRS (though it needs CQRS), not an audit table, and not an architecture for the whole system."*

---

## Concept 7 — Command sourcing: the trap next door

**Command sourcing** stores the *requests* the system received — `PlaceOrder`, `WithdrawFunds` — and derives state by re-executing them. It looks like event sourcing with different names, and it's a trap:

```
Stored (command sourcing):            Stored (event sourcing):
  WithdrawFunds { amount: 100 }         FundsWithdrawn { amount: 100, balanceAfter: 400 }
  WithdrawFunds { amount: 500 }         WithdrawalRejected { amount: 500, reason: InsufficientFunds }   // if you choose to record rejections
```

Why re-executing requests is wrong:

- **Replay re-decides.** Re-running `WithdrawFunds(500)` today, with today's code and rules, may produce a different result than it did then — the overdraft limit changed, a bug was fixed. The past is no longer fixed; it's recomputed, and it can change.
- **Replay repeats side effects.** Re-executing `CapturePayment` either calls the payment provider again or needs every handler to know it's "just a replay" — the distinction event sourcing encodes structurally (Concept 53).
- **Replay depends on the world.** A command handler reads other data: prices, exchange rates, the current time. Re-executing it later sees different data.
- **Rejected commands pollute history.** A command that was rejected is still in the log, and every replay must reject it again, identically.

Event sourcing avoids all of this by storing **outcomes**: the decision has already been made, and replay merely applies it (`evolve`, Concept 4). A good test: **could a stored record be re-evaluated and come out differently?** If yes, you're storing commands.

Two nuances a senior answer includes:

- **Storing commands alongside events is fine** — as an audit of requests, for debugging, or as the causation link from event to command (Concept 22). What matters is that *state is derived from events, not from commands*.
- **Recording rejections as events is a modeling choice.** `WithdrawalRejected` is a legitimate fact if the business cares ("three rejected withdrawals in an hour triggers a fraud review"). If it doesn't, rejections are returned to the caller and not stored. Either way, the rejection is stored as an *outcome*, not re-derived.

---

## Concept 8 — Other ways to keep history

Before choosing event sourcing for "we need an audit trail", know the alternatives. Each gives some of what event sourcing gives, at lower cost:

| Mechanism | What it records | Intent? | Load-bearing? | Can rebuild new views from history? | Cost |
|---|---|---|---|---|---|
| **Audit table / audit log** written by the application | Whatever the code writes: who, when, often before/after values | Only if developers record it | No — can drift silently | Partially, if detailed enough | Low; discipline required |
| **EF Core interceptor audit** (`SaveChangesInterceptor` writing change entries) | Every tracked change: entity, property, old/new values, user | No (property diffs) | No; misses `ExecuteUpdate`/raw SQL | Poorly — diffs, not facts | Low |
| **SQL Server / Azure SQL system-versioned temporal tables** | Every previous row version with validity periods | No | No (state table is truth) | Row-level "as of" queries: `FOR SYSTEM_TIME AS OF` | Low; storage grows |
| **SQL Server ledger tables** (updatable or append-only, SQL Server 2022+/Azure SQL) | Row history with a cryptographic hash chain and database digests | No | No for updatable ledger; append-only ledger tables are insert-only | Row-level history; tamper evidence | Low–moderate |
| **CDC / change event streaming** (Module 23, Concept 37) | Row diffs, in commit order, for a retention window | No | No | Only within retention | Moderate; operational |
| **Outbox with archived messages** | Published events, if archived instead of deleted | Yes, for published events | No | Partially | Low |
| **Event sourcing** | The business facts, as the truth | **Yes** | **Yes** | **Yes, completely** | High; pervasive |

The decision this table supports:

- If the requirement is **"show who changed what and when"** — compliance audit on a CRUD entity — temporal tables or an interceptor-based audit log are usually enough and cost a fraction of event sourcing.
- If the requirement is **tamper evidence** — proving records weren't altered — ledger tables give cryptographic verification without changing the programming model.
- If the requirement is **"answer questions about the past we haven't thought of yet," "understand why things happened," or "the history *is* the business"** (ledgers, bookings, claims, workflows with regulatory scrutiny) — that's where event sourcing's load-bearing, intent-revealing history pays for itself.

The senior sentence: *"Before event sourcing, I'd ask what the history is for. If it's 'who changed this row', temporal tables or an audit interceptor are much cheaper. Event sourcing earns its cost when history drives decisions or when we need to answer new questions from old facts."*

---

## Concept 9 — Why teams choose it

Michiel Overeem, Marten Spoor, Slinger Jansen and Sjaak Brinkkemper interviewed 25 engineers about 19 event-sourced systems (*An Empirical Characterization of Event Sourced Systems and Their Schema Evolution — Lessons from Industry*, Journal of Systems and Software, 2021). Their four categories of rationale are a good checklist; here they are with the concrete benefits behind each:

**1. History and audit.**
- A complete, trustworthy, load-bearing history — "what happened, in what order, and why" — for regulators, auditors, dispute resolution and customer support.
- Non-repudiation: the record of a decision can't be quietly edited.

**2. Flexibility of the read side.**
- **New read models from old data.** A report invented in year three can be built from year one's events (Module 23, Concept 43 — the rebuild problem disappears).
- **Temporal queries.** "What was this policy's coverage on the date of the claim?" is a fold up to a point in time (Concept 63).
- **Analytics and ML** get the richest possible data: every decision, with intent, not periodic snapshots.

**3. Domain modeling and complexity reduction.**
- Events make the domain's behaviour explicit and give domain experts a shared vocabulary (Event Modeling, Concept 24).
- The write model can be a pure decision function (Concept 46), trivially testable with given-when-then (Concept 54).
- No object-relational mapping for the write side: one serialized event type per fact, no navigation properties, no change tracking, no N+1 (Module 19's problems largely vanish on the write side).

**4. Technical properties.**
- **A natural concurrency model**: append-if-version per stream (Concept 28).
- **Integration**: every fact is available to publish, reliably, from one place (Concept 65).
- **Debugging and support**: reproduce a bug by loading the exact stream that caused it (Concept 92).
- **Append-only writes** are cheap and contention-free across different streams.

Two benefits that are often claimed but deserve caution:

- **"It scales better."** Appends are cheap, but reads now require projections, rebuilds take hours, and hot streams serialize. Event sourcing can scale very well; it doesn't scale *automatically*.
- **"It makes microservices easier."** Only if you keep the store private and publish translated integration events (Concept 97). Sharing an event store across services recreates the shared database with worse tooling.

---

## Concept 10 — What it costs

The same study identified five challenges practitioners report; add the ones that follow from them and you have the honest cost list. This is what interviewers want to hear — unprompted.

| Cost | Why it exists | Where it's handled |
|---|---|---|
| **Event schema evolution** — the most cited and hardest challenge | Stored events can't be changed, and code must read every version ever written | Part F |
| **Steep learning curve** | Thinking in facts and folds, eventual consistency, projections and versioning are unfamiliar to most teams | Concept 95 |
| **Less mature, less familiar tooling** | Fewer tools, fewer people who know them, fewer answers on the internet | Concepts 40, 98 |
| **Rebuilding projections** | Replays of large logs take hours and must be operated | Concepts 61, 90 |
| **Data privacy** | Personal data in an immutable log conflicts with erasure | Part G |
| **Every read needs a projection** | The event store can only answer "give me stream X" efficiently | Part E |
| **Eventual consistency in reads** | Async projections lag (Module 23, Part E) | Concept 66 |
| **Stream design is hard to change** | Stream boundaries are consistency boundaries baked into stored data | Concepts 30–31 |
| **Storage only grows** | Nothing is ever overwritten | Concepts 84, 90 |
| **Operational surface** | Subscriptions, checkpoints, daemons, dead letters, rebuilds, archives | Concept 88 |
| **Hard to migrate to — and away from** | The data model is fundamentally different | Concepts 93–94 |
| **Bugs are permanent** | A bug that emits wrong events leaves wrong facts in history | Concept 74 |

Chris Kiehl's *Don't Let the Internet Dupe You, Event Sourcing is Hard* is the best-known practitioner account of these costs appearing one after another in a first system, and Microsoft's March 2026 rewrite of the Azure Architecture Center page opens with the same warning. A senior candidate doesn't need to be negative about event sourcing — but must be able to list these costs before the interviewer does.

---

## Concept 11 — Where it pays, where it doesn't

The decision is made **per bounded context** (Module 22) — and increasingly per *part* of a context. Use the benefits (Concept 9) and costs (Concept 10) as a scoring sheet.

**Strong fits** — the history is the business, or decisions depend on it:

- **Ledgers, wallets, accounts, loyalty points.** Balances are sums of entries; auditors want every entry; disputes require history. (Caveat: Oskar Dudycz's *Why a bank account is not the best example of Event Sourcing* — an account's stream grows forever unless you close the books, Concept 31.)
- **Bookings, reservations, inventory allocation.** Collaborative domains with contention and reversals: holds, confirmations, expiries, cancellations.
- **Claims, cases, applications, approvals.** Long-running workflows with regulatory scrutiny where "why was this decided?" matters.
- **Orders and fulfilment lifecycles** in a core domain where analytics on the process itself (time between steps, abandonment points) has value.
- **Anything where "what did we know, and when?"** is a real question: insurance underwriting, pricing, clinical records, compliance.

**Weak fits** — use a state-stored model:

- **CRUD and reference data**: product categories, country lists, notification templates, user profiles, settings.
- **Supporting and generic subdomains** (Module 22, Concept 8) where nobody will ever ask about history — or where temporal tables answer the history question.
- **Prototypes, MVPs and short-lived systems**: the upfront investment in event design and projection infrastructure doesn't pay back.
- **Systems that need strongly consistent, ad-hoc queries over current state everywhere** with no tolerance for projection lag — though inline projections (Concept 57) narrow this gap.
- **Teams without the experience or appetite** — adopting event sourcing without event-driven foundations raises the risk of costly anti-patterns, as Microsoft's guidance now says explicitly.

A useful set of questions for the decision — ask the business, not the developers:

1. *"If we lost the history of this entity and kept only its current state, what would we lose?"* ("Nothing" → don't event-source.)
2. *"Will anyone ever ask why this entity is in its current state?"*
3. *"Are there questions about the past we'd want to answer that we can't predict today?"*
4. *"Is the history itself regulated, disputed or monetized?"*
5. *"How long must we keep it, and does it contain personal data?"* (Part G.)

The senior sentence: *"I'd event-source the parts of the system where history drives value — the ledger, the booking lifecycle, the claims process — and keep reference data, profiles and settings state-stored. It's a per-context decision, and I'd want the business to tell me what the history is worth."*

---

## Concept 12 — The anatomy of an event-sourced system

Here is the complete picture, with every component this module covers placed on it:

```
                                         ┌──────────────────── write side (Part D) ─────────────────────┐
 command ─▶ [edge: validate shape,       │  1. read stream "order-42" (+ snapshot)        (Concepts 44, 50)│
            authorize, idempotency key] ─▶  2. fold events → state                         (Concept 4)     │
                                         │  3. decide(command, state) → new events | reject (Concept 46)   │
                                         │  4. append(stream, newEvents, expectedVersion)  (Concept 28)    │
                                         └──────────────────────────────┬──────────────────────────────────┘
                                                                        │ atomic, append-only
                                                         ┌──────────────▼───────────────┐
                                                         │          EVENT STORE          │  (Part C)
                                                         │  streams + global log ($all)  │
                                                         └──────────────┬───────────────┘
                                          subscriptions / daemon / change feed (Concepts 58–59)
                     ┌───────────────────────────┬──────────────────────┼──────────────────────┬──────────────────────┐
                     ▼                           ▼                      ▼                      ▼                      ▼
             [projections]              [process managers]        [reactors]          [integration publisher]   [archiver]
          read models, search,          workflow state,            emails, calls        translate → broker         cold storage
          analytics (Part E)            timeouts (Concept 52)      (Concept 53)         (Concept 65)               (Concept 84)
             replay: YES                 replay: careful            replay: NO           replay: NO (idempotent)
```

Five things to notice:

**1. The append is the commit point, and it's synchronous.** The command handler appends directly to the store with an expected version. Only when the append succeeds has anything happened. Everything to the right of the store happens *afterwards*, from subscriptions. This is where Microsoft's own overview diagram deserves a critical reading: it shows command handlers pushing events to a queue and a handler writing them to the store. If events are written to the store *asynchronously* after being queued, the command handler can no longer detect a concurrent change to the stream, can't return a reliable outcome, and can't read its own write — the optimistic-concurrency check the same page describes becomes impossible. Queues belong *after* the store, not before it.

**2. The store is also the outbox.** Because the events *are* the state, and the append is atomic, there's no dual-write problem (Module 11) between "save state" and "publish event." Subscriptions read the committed log and publish from there (Concept 65).

**3. There are four kinds of consumer, with different replay rules** (Concept 64). Projections must be replayable. Reactors (side effects) must never replay. Process managers are somewhere between. Mixing them is the most common architectural bug in event-sourced systems.

**4. Every query goes through a projection** (Concept 55) — except loading one entity's own state by folding its stream.

**5. The event store is private to its bounded context** (Concept 97). Other contexts see integration events, never the store.

The rest of the module follows this picture from left to right: events (Part B), streams and the store (Part C), the write side (Part D), projections and consumers (Part E), then the long-term concerns — evolution (Part F), privacy and retention (Part G), operations (Part H) and the architect's view (Part I).

---

# Part B — Designing events

In a state-stored system, a bad column name is a migration away from being fixed. In an event-sourced system, a bad event design is permanent: every event of that shape ever written will be read by every version of the code that ever loads it. Part B is about designing events the way you'd design a public API that can never be deprecated — because that's what they are.

---

## Concept 13 — The stored event: envelope and payload

A stored event has two parts. The **payload** is the domain fact (`OrderPlaced { Total, PlacedAt }`). The **envelope** is everything the infrastructure needs to store, order, route, trace and secure it. Keeping them separate lets the domain type stay clean while the store keeps what it needs.

| Envelope field | Purpose | Who sets it |
|---|---|---|
| **Event ID** (GUID/UUID, ideally time-ordered v7) | Identity; idempotent appends (Concept 29); de-duplication by consumers | Application, before append |
| **Stream ID** | Which stream (entity/process) the event belongs to | Application |
| **Stream version** (revision) | Position within the stream: 1, 2, 3… gapless | Store, checked against expected version |
| **Global position** (sequence, commit position) | Position in the store-wide log, for subscriptions (Concept 27) | Store |
| **Event type name** | Stable string identifying the payload's schema (Concept 20) | Application / type registry |
| **Schema version** | Which version of that type's schema the payload follows (Concept 70) | Application / serializer |
| **Occurred at** (business time) | When the fact became true in the domain (Concept 18) | Application, from an injected clock |
| **Recorded at** (system time) | When the store committed it | Store (server timestamp) |
| **Correlation ID** | The business transaction or conversation this belongs to | Propagated from the request |
| **Causation ID** | The message (command or event) that directly caused this one | The handler |
| **Actor / user** | Who (or what service) made the decision | From the authenticated principal |
| **Tenant** | Multi-tenant partitioning (Concept 87) | From the request context |
| **Tags** | DCB tags naming the concepts the event concerns (Concept 35) | Application, by rule |
| **Trace context** (`traceparent`) | Link to the distributed trace (Module 28) | OpenTelemetry propagation |
| **Content type / encoding** | JSON, protobuf, encrypted fields (Concept 81) | Serializer |

Most stores give you most of these natively. KurrentDB stores an event ID, type, data and a separate **metadata** byte array (conventionally JSON carrying `$correlationId`, `$causationId` and your own fields). Marten stores the event ID, stream ID/key, version, sequence, type name, timestamp, tenant ID and — when enabled — correlation ID, causation ID, user name and arbitrary headers in dedicated columns.

A C# shape for the envelope, independent of any store:

```csharp
public sealed record EventEnvelope(
    Guid EventId,
    string StreamId,
    long StreamVersion,          // 1-based, gapless within the stream
    long GlobalPosition,         // store-wide, monotonic, possibly with gaps (Concept 27)
    string EventType,            // stable name, e.g. "order-placed" (Concept 20)
    int SchemaVersion,           // e.g. 2 (Concept 70)
    DateTimeOffset OccurredAt,   // business time
    DateTimeOffset RecordedAt,   // store commit time
    string? CorrelationId,
    string? CausationId,
    string? Actor,
    string? TenantId,
    IReadOnlyDictionary<string, string> Headers,
    object Payload);             // the deserialized domain event
```

The design rule: **domain event types contain only domain data.** No `EventId`, no `Version`, no `TenantId` properties on `OrderPlaced`. Those are envelope concerns, provided by the store and the pipeline. Mixing them into the payload couples the domain to storage and duplicates data that can drift.

---

## Concept 14 — Naming events

Names are the most durable part of an event — they appear in storage, code, projections, dashboards and conversations with domain experts. The rules:

**1. Past tense.** `OrderPlaced`, not `PlaceOrder` (a command) or `OrderPlacement` (a noun). The tense communicates that it already happened.

**2. Intent, in the ubiquitous language.** Name the business fact a domain expert would recognise:

| CRUD-shaped (avoid) | Intent-revealing (prefer) | Why it matters |
|---|---|---|
| `CustomerUpdated` | `CustomerRelocated`, `CustomerAddressCorrected`, `CustomerRenamedAfterMarriage` | A relocation changes tax region and delivery zone; a correction changes nothing downstream |
| `OrderStatusChanged { Status }` | `OrderPlaced`, `OrderShipped`, `OrderCancelled` | Each transition carries different data and triggers different reactions |
| `PolicyModified` | `CoverageExtended`, `PremiumRecalculated`, `BeneficiaryChanged` | Underwriting cares which one happened |
| `AccountBalanceSet { Balance }` | `FundsDeposited`, `FundsWithdrawn`, `InterestCredited`, `FeeCharged` | The balance is derived; the reasons are the history |

**3. Avoid "property sourcing".** Oskar Dudycz's term for events that mirror property setters — `PriceChanged`, `NameChanged`, `StatusChanged` for every field. It's CRUD with extra steps: you pay event sourcing's costs and lose its main benefit (intent). Its close cousin is **state obsession** — events that carry the entity's entire new state (`OrderUpdated { …all fields… }`) because the team is still thinking in rows.

**4. One name, one meaning, forever.** If the meaning changes, it's a new event type (Concept 67), not a new version of the old one.

**5. Name for the domain, not the UI.** `ButtonClicked` and `FormSubmitted` describe the interface; the domain fact is what the click *caused*.

Microsoft's March 2026 guidance makes the same point with its seat-reservation example: an event recording *two seats were reserved* is more valuable than one recording *remaining seats changed to 42* — the first tells you what happened, the second only the resulting state.

A quick review heuristic: **read the event names in a stream aloud as a story.** "Order drafted. Line added. Line added. Order placed. Payment authorised. Order shipped." If it reads like a story a domain expert would tell, the names are right. If it reads like "Order updated. Order updated. Order updated," the model hasn't found its events yet.

---

## Concept 15 — Granularity

How big should an event be? Three guidelines, then the trade-offs.

**One event per business fact.** If the domain expert would describe two things as having happened, record two events. If they'd describe one thing, record one — even if it changes many fields.

**A command can produce several events.** `CheckOut` might produce `OrderPlaced`, `DiscountApplied` and `LoyaltyPointsEarned`. They're appended atomically to the same stream (one append call, one expected version), so no reader ever sees half of the outcome.

**Don't split one fact to fit a UI or a projection.** If `OrderPlaced` is split into `OrderPlacedHeader` and `OrderPlacedLines` because a projection finds it easier, the history no longer reads as the domain.

| | Fine-grained events (`LineAdded`, `LineRemoved`, `QuantityChanged`) | Coarse-grained events (`OrderSubmitted { all lines }`) |
|---|---|---|
| Intent | High — every step is visible | Lower — steps before submission are lost unless recorded |
| Stream length | Longer | Shorter |
| Projection complexity | Each projection handles more event types | Fewer, bigger events |
| Analytics value | "How many carts had items removed?" is answerable | Only final outcomes |
| Versioning surface | More types to evolve | Fewer types, bigger schemas |

A useful middle ground: **fine-grained within a lifecycle phase that the business cares about, coarse where it doesn't.** A cart's edits are interesting for conversion analytics — keep them fine-grained. A support ticket's internal draft saves are not — record the submission.

---

## Concept 16 — What goes in an event

An event must carry **everything the future needs to interpret it without looking anything up**. That rule has three parts.

**1. The outcome of the decision, including computed results.** If the aggregate computed a price, a discount, a new balance or a due date, put the *result* in the event:

```csharp
// Weak: projections and evolve must recompute the total — re-implementing pricing at read time
public sealed record OrderLineAdded(string Sku, int Qty);

// Strong: the decision's result is recorded; replay never re-prices
public sealed record OrderLineAdded(string Sku, int Qty, decimal UnitPrice, decimal LineTotal, decimal NewOrderTotal);
```

Why: prices change, rules change, bugs get fixed. If `evolve` or a projection recomputes the total from `Sku` and `Qty` using *today's* price list, replaying last year's events produces last year's orders at this year's prices. Module 23 (Concept 34) made the same point for projections; in event sourcing it's also true for the write model itself.

**2. The inputs that justify the decision, when they matter for audit.** For regulated decisions — a credit limit granted, a claim approved — record the facts the decision was based on (`CreditLimitGranted { Limit, ScoreUsed, PolicyVersion }`). Verraes calls this **decision tracking**. An auditor can then see not only what was decided but on what basis, without reconstructing the world as it was.

**3. The data consumers need.** Carry enough for projections and process managers to do their jobs without calling back — identifiers of related entities, the new state of the fields that changed (state-carrying events make projections robust to reordering, Module 23 Concept 39), and denormalized labels where a projection would otherwise need a live lookup (Concept 62).

A useful distinction:

| Style | Example | Pros | Cons |
|---|---|---|---|
| **Delta** (what changed) | `ItemQuantityIncreased { Sku, By: 2 }` | Small, precise intent | Consumers must apply strictly in order; a missing event corrupts state |
| **State-carrying** (what it is now) | `ItemQuantityChanged { Sku, NewQuantity: 5 }` | Idempotent and order-tolerant for "latest wins" projections | Less explicit intent unless the name carries it |
| **Both** | `ItemQuantityIncreased { Sku, By: 2, NewQuantity: 5 }` | Intent *and* robustness | Slightly larger; the two fields must agree |

The third style is usually the best default for events that change a quantity or amount.

---

## Concept 17 — What stays out

Equally important is what *not* to put in an event:

- **Derived state that can be computed from other events** — unless it's the result of a rule (Concept 16). Storing `LineCount` on every `OrderLineAdded` is redundant; storing `NewOrderTotal` captures a pricing decision.
- **Whole-entity snapshots.** `OrderUpdated { entire order }` is state obsession (Concept 14). It bloats streams, hides intent and turns every event into a schema-evolution liability for every field.
- **Large blobs** — documents, images, PDFs. Store them in blob storage (Azure Blob Storage, immutable if needed) and put a reference and a content hash in the event. Event stores have size limits (a Cosmos DB item is at most 2 MB; KurrentDB has a configurable maximum record size) and every replay would stream the bytes.
- **Personal data you don't need.** Every piece of personal data in an event is a future erasure problem (Part G). Ask whether the event needs the customer's name or only their ID.
- **Secrets and credentials.** Never. Events are read by many consumers, copied into projections, logged in diagnostics and kept forever.
- **References to mutable data as if they were facts.** `OrderPlaced { ShippingAddressId }` is a reference to something that can change; if the order must be shipped to the address *as it was when the order was placed*, record the address itself (or a snapshot of it) — or accept that it's a live reference and design consumers accordingly. This is the invoice-name question from Module 23 (Concept 45): snapshot of the past, or view of the present?
- **Infrastructure data** in the payload — IDs, versions, timestamps of recording, tenant (Concept 13).

---

## Concept 18 — Time in events

Time is subtle in event sourcing, and interviewers who've built these systems probe it. Three different times can matter:

| Time | Meaning | Example | Where it lives |
|---|---|---|---|
| **Recorded time** (transaction time) | When the store committed the event | 2026-09-12 10:14:03.221Z | Envelope, set by the store |
| **Occurred time** | When the fact happened in the domain, as the system knows it | The order was placed at 10:14:03 | Envelope or payload, from an injected clock |
| **Effective time** (valid time) | When the fact takes effect in the business | A price change effective from 1 October; an address change backdated to 1 September | Payload |

Verraes calls events that carry more than one of these **multi-temporal events**. They matter because:

- **Late-arriving facts.** A paper claim form received today about an accident last week is recorded today, occurred last week. A report of "claims by accident date" must use the occurred time, not the recorded time.
- **Backdated corrections.** "The customer's address was actually wrong since 1 September" is recorded now, effective from 1 September. An "as-of 15 September" query (Concept 63) must see the corrected address; an "as-we-knew-it on 15 September" query must not. That's **bi-temporality** — valid time and transaction time — and events are the most natural way to support it.
- **Scheduled facts.** `PriceChangeScheduled { EffectiveFrom }` is recorded now, effective later.

Two implementation rules:

1. **Never read the clock in `evolve` or a projection.** Time comes in on the command, from an injected `TimeProvider` (the `TimeProvider` abstraction, testable with `FakeTimeProvider`), and is recorded in the event. `evolve` uses the event's time.
2. **Don't rely on timestamps for ordering.** Clocks skew across machines (Module 9). Stream order comes from the stream version; global order from the store's position. Timestamps are data, not ordering.

---

## Concept 19 — Internal vs published events

In an event-sourced context, the events in the store are the **internal storage format** of that context's state. Publishing them directly to other contexts is like letting other teams read your database tables: every internal refactoring becomes a breaking change for someone else.

Module 22 (Concept 83) distinguished domain events from integration events. In event sourcing the distinction becomes structural:

| | Stored (internal) events | Published (integration) events |
|---|---|---|
| Audience | The context's own aggregates, projections, process managers | Other bounded contexts, other services, partners |
| Shape | Fine-grained, rich, may use domain types | Flat, primitive, stable, often coarser |
| Versioning | Evolved with upcasters inside the context (Part F) | Versioned as a public contract (Module 11) |
| Count | Every fact | A deliberate subset ("explicit public events") |
| Personal data | Minimized, possibly encrypted (Part G) | Minimized further; often references only |
| Delivery | Store subscriptions | Broker (Service Bus, Event Hubs, Kafka) |

Verraes' pattern names for this: **Explicit Public Events** (mark a small subset of events as public; keep the rest private) and **Segregated Event Layers** (separate internal and external event layers, each in its own language). Oskar Dudycz's *Internal and external events, or how to design event-driven API* makes the same argument.

The translation happens in a subscription (Concept 65):

```csharp
// Internal events → integration event; runs in a subscription after commit
public static IntegrationEvent? Translate(EventEnvelope env) => env.Payload switch
{
    OrderPlaced p => new OrderPlacedV1(               // public contract, versioned independently
        OrderId: env.StreamId, CustomerId: /* from state or enriched */ default,
        Total: p.Total, PlacedAt: p.PlacedAt, CorrelationId: env.CorrelationId),
    OrderShipped s => new OrderShippedV1(env.StreamId, s.Carrier, s.ShippedAt),
    _ => null,                                          // most internal events are never published
};
```

A consequence worth saying out loud: **other contexts can't rebuild from your event store**, because they don't have access to it. If they need replay, give them a replayable published stream (for example a Kafka or Event Hubs topic with long retention) of integration events — a deliberate product, not a side door into your store.

---

## Concept 20 — Event type names are forever

Every stored event carries a **type name** that tells the deserializer which schema the payload follows. That name must be stable for as long as the event exists — which is forever.

The classic mistake is to store the CLR type name:

```
"$type": "Contoso.Ordering.Domain.Events.OrderPlaced, Contoso.Ordering.Domain, Version=1.0.0.0"
```

Rename the namespace, move the class to another assembly, or split the project, and every historical event becomes undeserializable. Some serializers and frameworks default to this; it's a trap.

The rule: **map each event type to an explicit, stable, short string, owned by the domain, registered in one place.**

```csharp
public sealed class EventTypeRegistry
{
    private readonly Dictionary<string, Type> _byName = new(StringComparer.Ordinal);
    private readonly Dictionary<Type, string> _byType = new();

    public EventTypeRegistry Map<T>(string name, params string[] aliases)
    {
        _byName.Add(name, typeof(T));
        _byType.Add(typeof(T), name);
        foreach (var alias in aliases) _byName.Add(alias, typeof(T));   // old names still resolve
        return this;
    }

    public string NameOf(Type t) => _byType.TryGetValue(t, out var n) ? n
        : throw new InvalidOperationException($"{t.Name} is not a registered event type.");
    public Type TypeOf(string name) => _byName.TryGetValue(name, out var t) ? t
        : throw new UnknownEventTypeException(name);                     // Concept 69 decides whether to ignore
}

var registry = new EventTypeRegistry()
    .Map<OrderDrafted>("order-drafted")
    .Map<OrderLineAdded>("order-line-added")
    .Map<OrderPlaced>("order-placed", aliases: "OrderSubmitted")         // renamed in 2025; old name still readable
    .Map<OrderShipped>("order-shipped");
```

Store-specific equivalents:

- **Marten** maps each event type to a name — by default derived from the class name in snake_case — and lets you set it explicitly with `opts.Events.MapEventType<T>("name")`; its versioning documentation covers namespace and type-name migrations (Concept 76).
- **KurrentDB** stores whatever type string you pass in `EventData`; the mapping is entirely yours (Eventuous provides a type map with attributes and source generation).
- **Polecat** follows Marten's conventions.

Three practical rules:

1. **Test the registry.** A unit test that every concrete event type is registered, and that no two types share a name (Concept 75).
2. **Never reuse a retired name** for a different type.
3. **Prefer names that describe the fact, not the class.** `order-placed` survives a refactoring that renames `OrderPlaced` to `OrderSubmitted`; the alias keeps the old stored name readable.

---

## Concept 21 — Serialization

Events are serialized once and deserialized for years, by code that changes. Serializer choices that are harmless in an API can be permanent in an event store.

**Format.** JSON is the default for good reasons: human-readable in the store, queryable (PostgreSQL `jsonb`, SQL Server 2025 `json`), tolerant of additive change, supported by every tool. Binary formats (Protocol Buffers, MessagePack) are smaller and faster and have well-defined evolution rules, at the cost of readability and ad-hoc querying; they're worth it mainly at very high volume. Marten supports binary event serialization as an option; most .NET event stores default to JSON.

**System.Text.Json settings that matter for events:**

```csharp
var options = new JsonSerializerOptions(JsonSerializerDefaults.Web)
{
    // Tolerant reading (Concept 69): unknown properties are ignored — this is the default; don't set Disallow
    UnmappedMemberHandling = JsonUnmappedMemberHandling.Skip,

    // Enums as strings: numeric enums silently change meaning if members are reordered
    Converters = { new JsonStringEnumConverter() },

    // .NET 9+: strict switches. Useful for commands and API input; think twice for stored events,
    // because an old event that lacks a now-required constructor parameter would fail to load.
    RespectRequiredConstructorParameters = false,
    RespectNullableAnnotations = false,
};
```

**For AOT and performance, use source generation** — a `JsonSerializerContext` listing every event type — which also forces the list of event types to be explicit (a nice side effect for Concept 20):

```csharp
[JsonSourceGenerationOptions(JsonSerializerDefaults.Web, UseStringEnumConverter = true)]
[JsonSerializable(typeof(OrderDrafted))]
[JsonSerializable(typeof(OrderLineAdded))]
[JsonSerializable(typeof(OrderPlaced))]
[JsonSerializable(typeof(OrderShipped))]
[JsonSerializable(typeof(OrderCancelled))]
internal partial class OrderingEventsJsonContext : JsonSerializerContext;
```

**The traps, each of which has broken a production event store somewhere:**

| Trap | What goes wrong | Rule |
|---|---|---|
| Numeric enums | Inserting a member shifts values; old events now mean something else | Serialize enums as strings; never rename members |
| Renaming properties | Old events have the old name; the property deserializes as default | Use `[JsonPropertyName("oldName")]` or upcast (Concept 70) |
| `DateTime` without offset | Kind ambiguity; local-time bugs after server moves | Use `DateTimeOffset` (or `Instant` with NodaTime), always UTC |
| `double` for money | Rounding drift | `decimal`; STJ writes decimals exactly |
| Polymorphic payloads via `$type` | CLR names in data (Concept 20) | Discriminator names you control (`[JsonDerivedType(typeof(X), "x")]`) or avoid polymorphism inside events |
| Value objects serialized by shape | Changing a value object's internals changes stored events | Give value objects explicit converters or flatten to primitives in events |
| Culture-sensitive formatting | Decimal commas, localized dates | Invariant culture everywhere; let the serializer handle it |
| Different serializer settings per service | Two services read the same stream differently | One options object, owned by the context, reused everywhere |

A principle worth stating: **stored events are a serialization contract, not an object graph.** Many teams make event types plain records of primitives and simple collections for exactly this reason — the domain can use rich value objects, and the events stay boring. Boring is what you want in something that must deserialize in 2031.

---

## Concept 22 — Metadata that pays for itself

Metadata turns an event store from a pile of facts into a navigable history. Five fields are worth having from day one:

**1. Correlation ID** — ties together everything that happened as part of one business transaction or user interaction: the HTTP request, the command, the events it produced, the process manager steps they triggered, the integration events published, and the downstream commands in other contexts.

**2. Causation ID** — the ID of the *immediate* cause: the command that produced this event, or the event that caused a process manager to issue the command. Together, correlation and causation let you draw the full causal graph of a business process:

```
request  corr=C1
 └─ command PlaceOrder           id=M1  corr=C1  cause=—
     ├─ event OrderPlaced        id=E1  corr=C1  cause=M1
     │   └─ command ReserveStock id=M2  corr=C1  cause=E1          (process manager)
     │       └─ event StockReserved  id=E2  corr=C1  cause=M2
     └─ event LoyaltyPointsEarned id=E3 corr=C1  cause=M1
```

**3. Actor** — who made the decision: user ID, service identity, or "system:scheduler". Essential for audit, and cheap to record. (Record an identifier, not a name — Part G.)

**4. Trace context** — the W3C `traceparent` of the span that appended the event, so an operator can jump from an event in the store to the distributed trace (Module 28). Consumers can start their processing spans as *links* to the originating trace rather than as children, which is the OpenTelemetry-recommended shape for asynchronous messaging.

**5. Tenant** — if the store is multi-tenant (Concept 87), the tenant ID must be on every event, set by infrastructure, never by the domain.

Two rules:

- **Metadata is set by the pipeline, not by domain code.** A middleware or decorator around the command handler stamps correlation, causation, actor, tenant and trace onto every event appended during that command. Domain code that sets `CorrelationId` is domain code that can forget to.
- **Metadata is not a dumping ground.** Don't put business data in headers because it was inconvenient to add a property. If a projection needs it, it's part of the fact.

---

## Concept 23 — A contract with your future self

Every design choice in this part comes back to one fact: **stored events outlive the code that wrote them.** A five-year-old event store has been read by dozens of releases, possibly two frameworks, several serializer upgrades and teams that have turned over completely.

What that implies, as a checklist for reviewing a new event type:

1. **Would a new team member understand this event from its name and fields alone**, without reading the code that emits it?
2. **Could it be deserialized by a different language or tool?** (Analytics in Python, a migration script, a support tool.) If only one C# class can read it, you've coupled history to an implementation.
3. **Does it depend on anything mutable** — a price list, a configuration value, a lookup table — to be interpreted? If yes, record the value (Concept 16).
4. **Is every field necessary?** Each field is a permanent obligation, and each personal-data field is a permanent liability (Part G).
5. **Does the name describe a fact the business would still recognise in five years?** Names tied to a UI, a vendor or a project codename age badly.
6. **Is there a sample of this event's JSON committed to the repository?** (Concept 75.) That sample is the contract; the class is an implementation of it.

Greg Young's framing from *Versioning in an Event Sourced System* is useful here: the question is never "can we change this event?" — you can't — but "can every future version of the system still understand what this event said?"

---

## Concept 24 — Designing events with Event Modeling

How do you discover the right events in the first place? Module 22 (Concept 36) covered EventStorming for discovering the domain. **Event Modeling**, developed by Adam Dymitruk, is the design technique most closely matched to event-sourced systems, because its output is almost directly an implementation plan.

An event model is a timeline, left to right, with four kinds of element:

```
 UI / trigger      [Cart screen]      [Checkout screen]            [Order status page]
                        │                    │                            ▲
 Commands          AddItem ───────▶     PlaceOrder ──────▶                │
                        │                    │                            │
 Events (facts)    ItemAdded ─────▶    OrderPlaced ───▶ PaymentCaptured ──┤
                        │                    │                            │
 Views (read)      CartSummary ◀──────       └─────▶ OrderStatusView ─────┘
```

Each vertical **slice** — a command with the events it produces, or a view with the events it reads — is a unit of work that can be designed, built and tested independently. That maps directly onto the code:

| Event model element | Code |
|---|---|
| Command → events | A decider's `decide` clauses, tested with given-when-then (Concept 54) |
| Events → view | A projection, tested with given-events-then-rows (Concept 91) |
| Events → automation → command | A process manager or reactor (Concepts 52–53) |
| External events → translation → command | An anti-corruption layer (Module 22, Concept 27) |

Why it helps event sourcing specifically:

- **It forces names and fields to be decided with domain experts** — the events on the timeline are the stored events.
- **It shows which views need which events** — so events carry what projections need (Concept 16) and nothing more.
- **It reveals stream boundaries** — commands that must see each other's events to decide belong to the same consistency boundary (Concept 30), or to a DCB query (Concept 35).
- **Every piece of information on a screen must be traceable back to an event.** If a view needs a field that no event provides, the model is incomplete — found at design time rather than in production.

The interview framing: *"I'd design the events with the domain experts on a timeline — Event Modeling works well — so each command, event and view is agreed before code. The events on that board become the stored events, so it's the most important hour of the design."*

---

# Part C — Streams, event stores and concurrency

Part B designed the facts. Part C designs where they live: the stream that groups them, the store that keeps them, and the concurrency mechanism that makes an append a decision rather than a race. This is where event sourcing's guarantees are actually made — and where most of its subtle bugs live, from out-of-order global positions to append conditions that silently let two conflicting decisions both commit.

---

## Concept 25 — Streams

A **stream** is an ordered sequence of events about one thing — one aggregate instance, one process instance, one period of one ledger. The stream is the unit of:

- **Ordering**: events in a stream have a strict, gapless order (version 1, 2, 3…).
- **Consistency**: appends are checked against the stream's current version (Concept 28).
- **Loading**: the write model reads one stream to decide.
- **Identity**: the stream ID *is* the entity's identity in storage.

Stream naming conventions matter because stores use them:

- **`category-id`** — `order-7f3c2a…`, `account-RS35…`, `booking-2026-10-04-hall-A`. KurrentDB derives a stream's **category** from the part before the first `-` (configurable) and indexes by category, so "all order streams" is a query it can answer efficiently. Stick to one separator convention across a context.
- **One stream per aggregate instance** is the default, from Module 22's aggregate design.
- **Stream IDs should be stable, opaque and assigned by the application** — a GUID (preferably time-ordered v7), or a natural key if it's truly immutable. Client-generated IDs make creation idempotent (Module 23, Concept 19).
- **Streams can also represent processes and periods** — a checkout process, a cashier's shift, an accounting month (Concept 31). Not everything that has a stream is a noun.

Marten supports stream identity as `Guid` or `string` (`StreamIdentity.AsGuid` / `AsString`), with the aggregate type recorded on the stream. KurrentDB streams are strings. Cosmos DB (Concept 42) would use the stream ID as the partition key.

---

## Concept 26 — What an event store must provide

An event store is a database with a specific, small contract. Anything that provides these operations with these guarantees can be one — a purpose-built database, a relational table, a document container:

| Operation | Guarantee | Why it's required |
|---|---|---|
| **Append(stream, events, expectedVersion)** | **Atomic** (all events or none), **ordered** within the stream, and **conditional**: fails if the stream's current version ≠ expected | This *is* the write-side concurrency model (Concept 28) |
| **Read(stream, fromVersion)** | Returns that stream's events in order, **efficiently** (proportional to the stream, not the store) | Loading an aggregate to decide (Concept 44) |
| **Read all(fromPosition)** | Returns *every* event in the store in a **global order** consistent with commit order | Projections, subscriptions, rebuilds (Part E) |
| **Subscribe(fromPosition)** | Delivers new events after they commit, catch-up then live | Keeping read models and reactors up to date |
| **Idempotent append** (by event ID) | Retrying the same append doesn't duplicate events | Safe retries after timeouts (Concept 29) |

Nice-to-have features that distinguish products: per-stream metadata (max age, max count, ACLs), soft/hard delete, category and event-type indexes, server-side projections, persistent (competing-consumer) subscriptions, snapshots, multi-stream atomic appends, tag queries with append conditions (DCB), archiving to cold storage, multi-tenancy, and encryption.

Two guarantees that are easy to overlook:

- **Read-your-own-append.** After a successful append, a read of the same stream from the same client must include the appended events. Relational stores give this naturally; in replicated stores it depends on read routing (reading from a lagging follower breaks it — Module 8).
- **Global order must be stable and gap-safe for readers.** A subscriber that has seen position 1,000 must never later have an event appear at position 999. That sounds obvious and is violated by naïve relational implementations (Concept 39).

The mental model Oskar Dudycz recommends — *event stores are key-value databases* — is a good one: logically, the key is the stream ID and the value is the ordered list of events. Everything about performance follows: a stream is as cheap as its length, and a very long stream is as slow as a very large document.

---

## Concept 27 — Positions: version vs global position

Event stores have two kinds of position, and conflating them causes bugs:

| | **Stream version** (revision, sequence-in-stream) | **Global position** (sequence, commit position, `$all` position) |
|---|---|---|
| Scope | One stream | The whole store |
| Values | 1, 2, 3… (or 0, 1, 2… — stores differ) | Monotonic 64-bit numbers |
| Gaps | **Never** | **Possible** (relational sequences skip on rollback; some stores' positions are byte offsets) |
| Used for | Optimistic concurrency; aggregate version; idempotency of per-stream projections | Subscriptions; checkpoints; rebuilds; consistency tokens across streams |
| Who assigns | Store, validated against expected version | Store, at commit |

Concrete examples:

- **Marten**: `mt_events.version` is the stream version; `mt_events.seq_id` is a PostgreSQL sequence value — monotonic but with gaps when transactions roll back or fail.
- **KurrentDB**: every event has a stream revision and a position in the `$all` log (a commit/prepare position pair derived from the transaction log). Positions are monotonic but not consecutive integers.
- **Cosmos DB DIY** (Concept 42): the stream version is whatever you write; there's no global position at all — the change feed is ordered only per logical partition.

Why subscribers care:

- A checkpoint is a **global position**, so a projection must only ever advance past positions it has processed.
- **Gaps are normal and must not be waited on forever** — a missing sequence number may never appear (a rolled-back transaction). But a gap may also be a transaction that *hasn't committed yet* and will appear later with a lower number than events already visible. Telling those two apart is the subject of Concept 39.

---

## Concept 28 — Optimistic concurrency on streams

The write side's entire concurrency model is one condition on the append:

> **Append these events to stream S only if S's current version is still V** (the version I read when I made my decision).

```
Handler A: read order-42 (version 4) ─ decide ─ append [OrderShipped] expecting 4 ─▶ OK, now version 5
Handler B: read order-42 (version 4) ─ decide ─ append [OrderCancelled] expecting 4 ─▶ CONFLICT (current is 5)
```

This is optimistic concurrency (Module 12) applied to a log. It gives each stream **linearizable** writes (Module 7): every successful append is based on the complete, current history of the stream. There are no locks held while deciding, and unrelated streams never contend.

Stores express the expectation slightly differently:

| Expectation | Meaning | KurrentDB (`KurrentDB.Client`) | Marten |
|---|---|---|---|
| **No stream** | The stream must not exist — creation | `StreamState.NoStream` | `StartStream` (fails if it exists) |
| **Exact version** | The stream must be at version V | a specific stream revision | `Append(id, expectedVersion, …)` or `FetchForWriting` |
| **Stream exists** | Any version, but it must exist | `StreamState.StreamExists` | — |
| **Any** | No check — last writer appends | `StreamState.Any` | `Append(id, …)` without a version |

With KurrentDB, a conflict surfaces as a `WrongExpectedVersionException`; with Marten, as a concurrency exception at `SaveChangesAsync` (`EventStreamUnexpectedMaxEventIdException` or `ConcurrencyException`, depending on the path).

```csharp
// Marten: FetchForWriting reads the current version and state, and enforces it on SaveChangesAsync
public async Task<ShipOutcome> Handle(ShipOrder cmd, IDocumentSession session, CancellationToken ct)
{
    var stream = await session.Events.FetchForWriting<Order>(cmd.OrderId, ct);   // state + version
    if (stream.Aggregate is null) return ShipOutcome.NotFound;

    var events = OrderDecider.Decide(cmd, stream.Aggregate);                      // pure decision
    if (events is ShipDecision.Rejected r) return ShipOutcome.Rejected(r.Reason);

    stream.AppendMany(((ShipDecision.Accepted)events).Events);
    await session.SaveChangesAsync(ct);                                          // throws if someone appended meanwhile
    return ShipOutcome.Shipped(stream.CurrentVersion ?? 0);
}
```

Three things a senior answer adds:

**1. `Any` is almost always wrong for commands.** Appending without a version check means the decision was made on possibly stale state — exactly the lost-update anomaly optimistic concurrency exists to prevent. `Any` is legitimate only for events whose validity doesn't depend on the stream's state (appending an independent observation, for example).

**2. The expected version can come from two places** (Module 23, Concept 20):
- From **the handler's own read** — protects against concurrent handlers; on conflict, the handler can simply re-run (Concept 49).
- From **the client** (the version the user saw, via `If-Match`) — protects against the user acting on a stale screen; on conflict, the user must review.

**3. A conflict is information, not an error.** It means "your decision was based on an outdated history." The correct response is to re-read and re-decide, not to force the write.

---

## Concept 29 — Idempotent appends

Networks time out after the server commits. Without protection, a retried append duplicates events — and duplicated events are permanent.

Three layers, from the store up:

**1. Expected version (natural protection).** If the retry carries the same expected version, and the first attempt succeeded, the retry conflicts — the stream is no longer at that version. The handler can then check whether the events it wanted are already there. This protects *exact-version* appends automatically; it doesn't help appends with `Any`.

**2. Event IDs (store-level de-duplication).** Assign event IDs *before* the first attempt and reuse them on retry. KurrentDB uses event IDs for idempotent writes: its documentation notes that idempotency is only guaranteed when you specify the expected stream revision; with `Any` it is best-effort. In a relational store, a unique constraint on `EventId` turns a duplicate into a constraint violation you can recognize.

```csharp
// Assign IDs once, outside the retry loop
var eventIds = decision.Events.Select(_ => Guid.CreateVersion7()).ToArray();

await retryPipeline.ExecuteAsync(async ct =>
{
    var data = decision.Events.Select((e, i) => serializer.ToEventData(e, eventIds[i]));
    await store.AppendAsync(streamId, expectedVersion, data, ct);   // same IDs on every attempt
}, ct);
```

**3. Command idempotency keys (application level).** Module 23 (Concept 19) covered storing command outcomes by idempotency key. In an event-sourced system there's an elegant variant: **record the command's idempotency key in the metadata of the events it produced**, and have the handler check the stream (or a small index projection) for it before deciding. The stream *is* the processed-commands table.

The rule: **generate event IDs deterministically per decision, before the first attempt**, and never regenerate them inside a retry.

---

## Concept 30 — Stream boundaries are consistency boundaries

Choosing what a stream contains is choosing what can be decided consistently — exactly the aggregate-design question of Module 22 (Concepts 56–66), now baked into stored data.

The rules carry over directly:

- **A stream contains the facts needed to protect one set of invariants.** If a rule must be checked against the facts of X and Y together, atomically, X and Y's facts must be in one stream — or you need one of the cross-stream techniques of Concept 33, or DCB (Concept 35).
- **Streams should be small in scope** — the smallest boundary that protects the true invariants. A "customer" stream holding every order the customer ever placed is the event-sourced version of the giant aggregate: every order operation contends on one stream, and it grows forever.
- **Reference other streams by ID.** An order event records `CustomerId`, not customer facts.

What's *different* from state-stored aggregates:

- **Boundaries are harder to change later.** In a state-stored system, splitting an aggregate is a table refactoring. In an event-sourced system, the stored streams embody the old boundary; changing it means copying and transforming history into new streams (Concept 72). Get the boundary conversation right early — Event Modeling helps (Concept 24).
- **The stream also defines the loading cost.** An aggregate that's too large in a relational model loads too many rows; a stream that's too long replays too many events (Concept 31).
- **Streams can model processes as well as things.** A checkout stream, an onboarding stream, a claim-handling stream — a process with its own lifecycle and invariants is a natural stream even if it doesn't correspond to an entity.

The senior sentence: *"The stream is the consistency boundary, so I design streams exactly as I'd design aggregates — around the true invariants, as small as possible — knowing that changing them later means migrating history, which makes the upfront modeling session worth more than in a CRUD design."*

---

## Concept 31 — Keep streams short

Oskar Dudycz calls keeping streams short the most important modeling practice in event sourcing. The reason is arithmetic: every command loads the stream; every load costs time proportional to its length (Concept 89). A stream that grows forever — an account, a customer, a product's price history, a device's telemetry — gets slower forever, and every snapshot you add to compensate is a cache you must now version and invalidate (Concept 50).

The fix is **modeling the lifecycle**, not optimizing the store:

**1. Find the natural lifecycle.** Most long-lived things are sequences of shorter processes with their own beginning and end:

| Long-lived thing (long stream) | Natural lifecycle (short streams) |
|---|---|
| Bank account | Accounting period (month) — opening balance, entries, closing balance |
| Cash register | Cashier shift — opened, transactions, closed with a count |
| Hotel | Business day — check-ins, check-outs, night audit |
| Product price history | Price period — effective from/to |
| Customer | Each order, each subscription term, each support case |
| IoT device | Daily or hourly telemetry window, summarized |

**2. Close the books.** The pattern comes straight from accounting (Concept 2). At the end of a period, the old stream records a closing event with a summary; a new stream for the next period starts with an opening event carrying the carried-forward state:

```
account-RS35-2026-08:  AccountingPeriodOpened { openingBalance: 1,200.00 }
                       FundsDeposited …   FundsWithdrawn …   (hundreds of entries)
                       AccountingPeriodClosed { closingBalance: 845.20, entryCount: 312 }

account-RS35-2026-09:  AccountingPeriodOpened { openingBalance: 845.20, previousPeriod: "2026-08" }
                       …
```

Deciding a September withdrawal loads only September's stream. August's stream is closed — immutable in practice as well as in principle — and can be archived (Concept 84).

**3. Make the transition part of the domain.** Who closes the period? Some lifecycles are closed by a person (the cashier ends the shift); some automatically (a scheduler issues `ClosePeriod` at midnight, and the *next* period is opened by a process manager reacting to `PeriodClosed`). Either way, the business owns the rule — and the conversation often reveals requirements nobody had mentioned ("can a transaction be booked into a closed period?").

**4. Predictable stream IDs** (`account-{id}-{yyyy-MM}`) let the write side find the current stream without a lookup. Where that's impossible — overlapping shifts on one register, say — keep a tiny "current period" read model or registry stream.

When long streams are acceptable: low-frequency entities (a company's registered address changes a few times a decade) never get long enough to matter. Dudycz's follow-up — *Should you always keep streams short?* — is a useful reminder that the goal is bounded load cost, not a dogma.

If you inherit long streams, **compaction** (Marten's `CompactStreamAsync`, Concept 84) and snapshots are the remedies — but they treat the symptom; the lifecycle model treats the cause.

---

## Concept 32 — Hot streams

A **hot stream** is one that receives appends faster than decisions on it can complete — every append invalidates every concurrent decision, and conflicts spiral. Module 22 (Concept 65) gave the arithmetic for hot aggregates; it applies unchanged:

- If appends to a stream arrive at rate **λ** per second and a decision takes **d** seconds (read, fold, decide, append), then the probability that another append lands during a given decision is roughly **P(conflict) ≈ 1 − e^(−λd)**.
- The throughput ceiling of one stream is about **1/d** successful appends per second, *regardless of hardware*, because decisions on one stream are serialized by the version check.

| Decision time *d* | Ceiling ≈ 1/d | P(conflict) at λ = 10/s | P(conflict) at λ = 50/s |
|---|---|---|---|
| 5 ms | ~200/s | ~5% | ~22% |
| 20 ms | ~50/s | ~18% | ~63% |
| 100 ms (a remote call inside the decision) | ~10/s | ~63% | ~99% |

The table makes the two levers obvious: **make decisions faster** (never call remote services while holding a version — fetch inputs first, or have them arrive on the command) and **reduce the append rate per stream** (redesign the boundary).

Remedies, most preferred first:

1. **Split the stream along the invariant.** A concert's seat inventory as one stream is hot; one stream per section, with a small aggregate for section-crossing rules, is not.
2. **Shorten decisions**: no I/O in `decide`; snapshots if load time dominates (Concept 50).
3. **Accept commutative events without re-deciding.** If two concurrent `TicketScanned` events can't violate any invariant, the handler can append with a retry that simply re-reads and re-appends — conflicts become cheap (Concept 49).
4. **Single writer.** Route all commands for a stream through one processor — a session-enabled Service Bus queue keyed by stream ID, a Kafka partition, or an **Orleans grain** (whose single-threaded activation serializes commands without conflicts) — so there's never more than one decision in flight.
5. **Escrow / reservation**: pre-allocate capacity into sub-streams (Module 9's escrow idea) — each shard of inventory decides independently until it runs out.
6. **Relax the invariant**: allow overbooking with compensation, if the business agrees.

---

## Concept 33 — Invariants across streams

Stream-per-aggregate enforces invariants *within* a stream. Rules that span streams — "a username must be unique," "a course has at most 30 students and a student takes at most 5 courses," "don't ship if the customer is blocked" — need something else. The options, in roughly increasing cost:

**1. Rethink the boundary.** Is it truly an invariant (must never be violated, even momentarily) or a policy that can be checked and corrected (Module 22, Concepts 56–58 and 70–71)? Many "cross-aggregate invariants" are policies.

**2. Put the rule's facts in one stream.** A `course-{id}-enrolments` stream owns the capacity rule. Fine if it doesn't become hot (Concept 32).

**3. Registry / reservation streams for uniqueness.** Make the unique value the identity of its own stream: `username-ivan.d` is created with `StreamState.NoStream`; if the append succeeds, the name is reserved. Two concurrent registrations for the same name conflict at the store level. The user stream then records `UsernameClaimed`. If the second step fails, a process manager releases the reservation. (A relational store can use a unique index on a side table instead — in the same transaction as the append, for Marten or Polecat.)

**4. Process managers with compensation** (Concept 52). Reserve in one stream, confirm in another, compensate on failure. Eventually consistent, with a window where the system shows an intermediate state — which must be modeled as a real state ("pending confirmation").

**5. Multi-stream atomic appends** (Concept 34), if the store supports them and all the relevant streams are read and version-checked together.

**6. Dynamic Consistency Boundaries** (Concepts 35–37): read the events relevant to *this* decision across streams by tags, decide, and append with a condition that no relevant event has been added since.

**7. Detect and correct.** Allow the violation, detect it asynchronously (a projection that counts), and compensate — for rules where occasional, short-lived violations are acceptable and cheap to fix.

The interview line: *"For cross-stream rules I first ask whether it's a true invariant or a policy. For true invariants: a registry stream for uniqueness, a dedicated stream if the rule has a natural owner, a DCB-capable store if the rule is really per-decision — and a process manager with compensation when eventual consistency is acceptable."*

---

## Concept 34 — Multi-stream atomic appends

Some stores can append to several streams in one atomic operation, each with its own expected version:

- **Marten and Polecat**: a session's `SaveChangesAsync` is one database transaction. Appending to several streams, each fetched with `FetchForWriting`, commits all or nothing, and each stream's version is checked. The same transaction can also update documents, inline projections and the Wolverine outbox.
- **KurrentDB 25.1+**: **multi-stream appends** write to several streams in a single operation with per-stream optimistic concurrency, through the client libraries that support the new API.
- **Cosmos DB**: transactional batches are limited to **one logical partition** — so multi-stream atomicity is available only for streams that share a partition key (Concept 42).
- **Relational DIY**: anything within one transaction.

When it's legitimate:

- **Registry plus entity**: claim `username-x` and create `user-7` atomically (instead of a process manager).
- **Transfers within one bounded context** where both sides must change together and neither stream is hot — `FundsReserved` on one account stream and `TransferInitiated` on a transfer stream.
- **Recording one fact that concerns two streams** — though DCB models this more naturally (one event, two tags).

When it's a smell:

- **It becomes the default.** If most commands touch several streams, the stream boundaries are wrong — the design has drifted back into large, implicit consistency boundaries with extra steps.
- **It crosses bounded contexts.** Two contexts' streams in one transaction means they share a database and a transaction — the coupling Module 21 warned against.
- **It hides contention.** Every stream in the transaction is a potential conflict; the probability of conflict grows with the number of streams involved.

Module 22 (Concept 63) listed the legitimate exceptions to "one aggregate per transaction"; multi-stream appends are the event-sourced mechanism for those exceptions — use them as deliberately as you'd break that rule.

---

## Concept 35 — Dynamic Consistency Boundaries, in depth

Module 22 (Concept 74) introduced **Dynamic Consistency Boundaries** as a challenge to the aggregate. Here's the mechanism in full, because interviewers who know it will probe the details — and because its safety depends on details (Concept 36).

**The problem.** With stream-per-aggregate, the consistency boundary is fixed at design time: an append is checked against *one* stream's version. Sara Pellegrini's example in *Killing the Aggregate* is the canonical one: a course accepts at most *N* students, and a student may subscribe to at most 10 courses. `Course` and `Student` are natural aggregates, but the subscription decision depends on both. The traditional answers — one giant aggregate, a saga with compensation, or two events recording the same fact in two streams — are all unsatisfying.

**The idea.** Drop fixed streams as the unit of consistency. Every event lives in one ordered log per bounded context and carries **tags** naming the domain concepts it concerns. A decision **reads the events matching a query**, builds its decision state from them, and appends new events with an **append condition**: *fail if any event matching the same query has been appended after the position I read up to.* The boundary is defined by the query — per decision, dynamically.

**The specification** (dcb.events):

| Element | Definition |
|---|---|
| **Event** | A type, data and a set of **tags** (strings such as `course:c1`, `student:s7`) |
| **Sequence position** | The event's position in the store's global order |
| **Query** | A set of **query items**; an event matches an item if its type is one of the item's types (if any are given) **and** it has **all** of the item's tags; it matches the query if it matches **any** item |
| **Read(query, after?)** | Returns matching events in position order |
| **Append(events, condition?)** | Atomically appends; the **append condition** is `failIfEventsMatch: query` plus `after: position` — the store must reject the append if any event matching the query exists after that position |

For the course example, the decision's query is:

```
[ { types: [CourseDefined, CourseCapacityChanged],             tags: [course:c1] },
  { types: [StudentSubscribedToCourse, StudentUnsubscribed],   tags: [course:c1] },
  { types: [StudentRegistered],                                tags: [student:s7] },
  { types: [StudentSubscribedToCourse, StudentUnsubscribed],   tags: [student:s7] } ]
```

and one event records the fact, tagged with both concepts:

```
StudentSubscribedToCourse { studentId: s7, courseId: c1 }   tags: [student:s7, course:c1]
```

**One fact, one event, two concepts.** That's the elegant part: no duplicated events, no saga for the happy path, and concurrent subscriptions of *different* students to *different* courses never conflict, because their queries don't overlap.

In Marten 9 (adapted from its DCB documentation — Marten 9.x; check the current docs for exact signatures):

```csharp
public sealed record StudentId(Guid Value);
public sealed record CourseId(Guid Value);

// Registration: each tag type gets storage and can drive a boundary aggregate
opts.Events.RegisterTagType<StudentId>("student").ForAggregate<SubscriptionBoundary>();
opts.Events.RegisterTagType<CourseId>("course").ForAggregate<SubscriptionBoundary>();
// Optional (9.x): declare tagging rules once instead of at every call site
opts.Events.TagWith<StudentSubscribedToCourse>(e => new StudentId(e.StudentId));

// The decision state: built from every event tagged with this course OR this student.
// The query returns all of course c1's subscriptions and all of student s7's — so the
// per-course count for c1 and the per-student count for s7 are complete; other keys aren't, and aren't read.
[BoundaryAggregate]
public sealed class SubscriptionBoundary
{
    public Dictionary<Guid, int> Capacity { get; } = [];
    public Dictionary<Guid, int> PerCourse { get; } = [];
    public Dictionary<Guid, int> PerStudent { get; } = [];
    public HashSet<(Guid Student, Guid Course)> Pairs { get; } = [];

    public void Apply(CourseDefined e) => Capacity[e.CourseId] = e.Capacity;
    public void Apply(StudentSubscribedToCourse e)
    {
        PerCourse[e.CourseId] = PerCourse.GetValueOrDefault(e.CourseId) + 1;
        PerStudent[e.StudentId] = PerStudent.GetValueOrDefault(e.StudentId) + 1;
        Pairs.Add((e.StudentId, e.CourseId));
    }
    public void Apply(StudentUnsubscribedFromCourse e)
    {
        PerCourse[e.CourseId]--; PerStudent[e.StudentId]--; Pairs.Remove((e.StudentId, e.CourseId));
    }
}

public async Task<SubscribeOutcome> Handle(SubscribeStudent cmd, IDocumentSession session, CancellationToken ct)
{
    var query = new EventTagQuery()
        .Or<CourseId>(new CourseId(cmd.CourseId))
        .Or<StudentId>(new StudentId(cmd.StudentId));

    var boundary = await session.Events.FetchForWritingByTags<SubscriptionBoundary>(query, ct);
    var s = boundary.Aggregate;

    if (s is null || !s.Capacity.TryGetValue(cmd.CourseId, out var capacity)) return SubscribeOutcome.CourseNotFound;
    if (s.Pairs.Contains((cmd.StudentId, cmd.CourseId)))                        return SubscribeOutcome.AlreadySubscribed;
    if (s.PerCourse.GetValueOrDefault(cmd.CourseId) >= capacity)                 return SubscribeOutcome.CourseFull;
    if (s.PerStudent.GetValueOrDefault(cmd.StudentId) >= 10)                     return SubscribeOutcome.StudentAtLimit;

    var evt = session.Events.BuildEvent(new StudentSubscribedToCourse(cmd.StudentId, cmd.CourseId));
    evt.WithTag(new StudentId(cmd.StudentId), new CourseId(cmd.CourseId));
    boundary.AppendOne(evt);

    await session.SaveChangesAsync(ct);                  // DcbConcurrencyException if the boundary moved
    return SubscribeOutcome.Subscribed;
}
```

**Streams are a special case.** A stream-per-aggregate store is a DCB store where every event has exactly one tag (`order:42`) and every query is "all events with that tag." DCB generalizes the boundary; it doesn't abolish aggregates. The dcb.events site has an example ("Event-Sourced Aggregate") making exactly this point — and its FAQ is explicit that DCB is an *alternative* way to achieve consistency, not an attack on the aggregate pattern itself.

---

## Concept 36 — Getting the append condition right

The append condition sounds simple: *"reject the append if a matching event exists after position P."* Implementing it naïvely on a relational database produces a textbook concurrency bug — and it isn't hypothetical.

**The naïve implementation:**

```sql
-- inside the append transaction, at READ COMMITTED
IF EXISTS (SELECT 1 FROM events e
           WHERE e.position > @after AND <e matches the query>)
    THROW 50001, 'DCB conflict', 1;
INSERT INTO events (...) VALUES (...);         -- the new, tagged event
COMMIT;
```

**The race.** Two sessions decide concurrently on overlapping queries (two students subscribing to the last seat in course c1):

```
Session A: EXISTS? → no matching events after P   (B hasn't committed)
Session B: EXISTS? → no matching events after P   (A hasn't committed)
Session A: INSERT StudentSubscribedToCourse(s7, c1); COMMIT
Session B: INSERT StudentSubscribedToCourse(s9, c1); COMMIT      → the course is over capacity
```

This is **write skew** — the anomaly Module 12 catalogued: two transactions read overlapping data, each writes something that would have changed the other's read, and neither sees the other's write. At `READ COMMITTED` (and even at snapshot isolation), nothing stops it, because the transactions don't write the *same row* — they each insert a *new* row that matches the other's predicate. It's a **phantom** problem.

This is exactly the bug Marten fixed in **9.4** (issue #4591): before 9.4, the DCB check was a separate, non-locking `SELECT EXISTS` before the inserts at `READ COMMITTED`, so truly concurrent boundary appends could both commit. The fix converted the predicate read into a row-level write conflict.

**The ways to make the condition safe**, with their costs:

| Technique | How | Cost |
|---|---|---|
| **Serialize all appends** | One global lock (a single row updated by every append, or an advisory lock) | Simple and correct; caps write throughput for the whole store |
| **`SERIALIZABLE` isolation** | PostgreSQL SSI detects the read-write dependency and aborts one transaction; SQL Server uses key-range locks | Correct if the predicate is indexable; aborts or blocking under contention; must retry serialization failures |
| **Lock the tag values** | Take locks (advisory locks, or `UPDLOCK` on tag rows) for every tag value in the query, in sorted order, before checking | Correct if *every* tagged append takes the same locks; deadlock-free only with a global lock order |
| **Convert the predicate into row conflicts** | A version row per tag value; the decision records the versions it read; the append does `UPDATE … SET version = version + 1 WHERE tag = @t AND version = @seen` for each tag and fails if any update matches zero rows; *every* tagged append bumps the rows for its tags | Correct at `READ COMMITTED`; conflicts are at tag-value granularity (coarser than event-type filters in the query — safe but occasionally over-conservative) |
| **A single-writer log** | The store serializes appends in one writer thread and evaluates the condition against the in-memory head of the log | Correct and fast; the store must be built this way (purpose-built DCB stores often are) |

Marten 9.4's mechanism is the fourth: a side table, `mt_dcb_tag_version`, with one row per `(tag type, tag value, tenant)`; `FetchForWritingByTags` records the versions it saw; `SaveChangesAsync` emits a conditional upsert per row in a deterministic order, and a zero-row result surfaces `DcbConcurrencyException`. Crucially, *every* save that appends a tagged event — even a plain `Append` outside any boundary — bumps the same rows, so an ordinary append can't slip past an in-flight boundary decision. Two more details from its documentation are worth knowing: a boundary fetch that ends with **nothing to append** asserts nothing (two sessions can both conclude "no action" and both commit, which is correct), and the side table grows with the number of **distinct tag values**, so ephemeral one-shot values make poor tags.

The general lesson — and a good interview answer when DCB comes up: **every DCB implementation must answer "what serializes two appenders whose queries overlap?"** If the answer is "a check before the insert," it's unsafe. Before adopting any DCB store, find its answer in the documentation or the source, and write a concurrency test (Concept 91) that runs two overlapping decisions in parallel thousands of times and asserts the invariant.

---

## Concept 37 — DCB or aggregates?

DCB is powerful and new; aggregates are well understood and supported everywhere. A balanced comparison:

| | Stream-per-aggregate | Dynamic Consistency Boundaries |
|---|---|---|
| Boundary | Fixed at design time, one stream | Per decision, defined by a query |
| Cross-entity rules | Registry streams, process managers, multi-stream appends | Natural: one event, several tags, one condition |
| Duplicate facts | Sometimes (the same fact recorded in two streams) | No — one event, many tags |
| Loading cost | Read one stream by key — very cheap | Query by tags and types across the log — needs good indexes (GIN/hstore, tag tables, purpose-built indexes) |
| Contention | Per stream | Per overlapping query — often lower, but a broad query is a broad boundary |
| Concurrency implementation | Simple: version check on one row/stream | Subtle (Concept 36) |
| Store support in .NET | Every store | Marten, Polecat (and Fisher); a few community stores |
| Tooling and community experience | Mature | Growing; patterns still settling |
| Modeling discipline required | Stream boundaries | **Tag discipline**: a forgotten tag isn't an error, it's a silently missing event in a decision (Marten added tag *rules* in 9.30 precisely because per-call-site tagging is easy to forget) |
| Evolving the boundary | Hard — history is in streams (Concept 30) | Easier — change the query; but adding a new tag to *old* events is a migration |
| Mental model for the team | "Load the object, call a method" | "Build a decision model from a query" |

Jeremy Miller's own assessment after shipping it in Marten 9 is candid: DCB is popular in the wider event-sourcing community, but he saw little demand from existing Marten users — partly because Marten can already make cross-stream decisions transactionally within one PostgreSQL transaction (Concept 34). That's a useful calibration: DCB matters most where a store *can't* offer multi-stream transactions.

When DCB is the better choice:

- Rules that genuinely span entities and would otherwise need sagas: capacity plus per-person limits, uniqueness (the "unique username" example on dcb.events), sequential numbering ("invoice number" example), opt-in tokens.
- Domains whose boundaries are still being discovered — the query can change without migrating streams.
- Vertical-slice codebases where each command slice builds exactly the decision model it needs (the Python `eventsourcing` library even offers "slice" abstractions for this).

When stream-per-aggregate is the better choice:

- Rules naturally scoped to one entity (most of them).
- Hot paths where a primary-key read of one stream matters for latency.
- Teams new to event sourcing — one new idea at a time.
- Stores without DCB support, or where the concurrency implementation is unclear.

The senior sentence: *"I'd default to stream-per-aggregate and reach for DCB for the specific decisions that span entities — capacity plus per-user limits, uniqueness — where it replaces a saga with one event and one condition. Before relying on it I'd want to know how the store serializes overlapping appends, and I'd enforce tagging with rules rather than trusting every call site."*

---

## Concept 38 — A relational event store, designed

You'll rarely build your own event store in production — Marten and Polecat exist — but designing one is a favourite deep-dive question, and understanding it is how you evaluate the products. Here's a SQL Server design (Azure SQL or SQL Server 2025, using the native `json` type; `nvarchar(max)` on older versions):

```sql
CREATE SCHEMA es;

CREATE TABLE es.Streams (
    StreamId     varchar(200)  NOT NULL CONSTRAINT PK_Streams PRIMARY KEY,
    StreamType   varchar(100)  NOT NULL,
    Version      bigint        NOT NULL,             -- current head version
    CreatedAt    datetime2(7)  NOT NULL CONSTRAINT DF_Streams_CreatedAt DEFAULT SYSUTCDATETIME(),
    IsArchived   bit           NOT NULL CONSTRAINT DF_Streams_IsArchived DEFAULT 0
);

CREATE TABLE es.Events (
    GlobalPosition bigint IDENTITY(1,1) NOT NULL CONSTRAINT PK_Events PRIMARY KEY CLUSTERED,
    StreamId       varchar(200)     NOT NULL,
    Version        bigint           NOT NULL,        -- 1..n, gapless per stream
    EventId        uniqueidentifier NOT NULL,
    EventType      varchar(200)     NOT NULL,        -- stable name (Concept 20)
    SchemaVersion  int              NOT NULL,
    Data           json             NOT NULL,
    Metadata       json             NULL,
    OccurredAt     datetimeoffset(7) NOT NULL,
    RecordedAt     datetime2(7)     NOT NULL CONSTRAINT DF_Events_RecordedAt DEFAULT SYSUTCDATETIME(),
    CommitVersion  rowversion       NOT NULL,        -- for gap-safe global reads (Concept 39)
    CONSTRAINT UQ_Events_Stream_Version UNIQUE (StreamId, Version),   -- the concurrency backstop
    CONSTRAINT UQ_Events_EventId        UNIQUE (EventId)              -- idempotent appends (Concept 29)
);
CREATE INDEX IX_Events_CommitVersion ON es.Events (CommitVersion);    -- subscriptions read by this
```

The append, as one transaction:

```csharp
public sealed class SqlEventStore(string connectionString, EventSerializer serializer)
{
    public async Task<long> AppendAsync(
        string streamId, string streamType, long expectedVersion,     // 0 = the stream must not exist
        IReadOnlyList<(Guid EventId, object Event, EventMetadata Meta)> events, CancellationToken ct)
    {
        await using var conn = new SqlConnection(connectionString);
        await conn.OpenAsync(ct);
        await using var tx = (SqlTransaction)await conn.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);

        try
        {
            var newVersion = expectedVersion + events.Count;

            if (expectedVersion == 0)
            {
                // Creation: a concurrent creator hits the primary key → conflict
                await conn.ExecuteAsync(
                    "INSERT es.Streams (StreamId, StreamType, Version) VALUES (@streamId, @streamType, @newVersion)",
                    new { streamId, streamType, newVersion }, tx);
            }
            else
            {
                // The version check: the row lock serializes concurrent appenders to this stream;
                // the loser waits, then sees a changed Version and updates zero rows
                var rows = await conn.ExecuteAsync(
                    "UPDATE es.Streams SET Version = @newVersion WHERE StreamId = @streamId AND Version = @expectedVersion",
                    new { streamId, newVersion, expectedVersion }, tx);
                if (rows == 0) throw new WrongExpectedVersionException(streamId, expectedVersion);
            }

            var v = expectedVersion;
            foreach (var (eventId, evt, meta) in events)
            {
                var (type, schemaVersion, json) = serializer.Serialize(evt);
                await conn.ExecuteAsync("""
                    INSERT es.Events (StreamId, Version, EventId, EventType, SchemaVersion, Data, Metadata, OccurredAt)
                    VALUES (@streamId, @version, @eventId, @type, @schemaVersion, @json, @metaJson, @occurredAt)
                    """,
                    new { streamId, version = ++v, eventId, type, schemaVersion, json,
                          metaJson = serializer.SerializeMetadata(meta), occurredAt = meta.OccurredAt }, tx);
            }

            await tx.CommitAsync(ct);
            return newVersion;
        }
        catch (SqlException ex) when (ex.Number is 2627 or 2601)          // PK or unique violation
        {
            await tx.RollbackAsync(ct);
            // Either a concurrent creator, a concurrent append that slipped past (the unique
            // (StreamId, Version) backstop), or a retried append whose EventIds already exist
            throw new WrongExpectedVersionException(streamId, expectedVersion, ex);
        }
    }

    public async Task<IReadOnlyList<StoredEvent>> ReadStreamAsync(string streamId, long fromVersion, CancellationToken ct)
    {
        await using var conn = new SqlConnection(connectionString);
        var rows = await conn.QueryAsync<StoredEventRow>(new CommandDefinition("""
            SELECT GlobalPosition, StreamId, Version, EventId, EventType, SchemaVersion, Data, Metadata, OccurredAt, RecordedAt
            FROM es.Events WHERE StreamId = @streamId AND Version > @fromVersion ORDER BY Version
            """, new { streamId, fromVersion }, cancellationToken: ct));
        return rows.Select(serializer.Deserialize).ToList();              // upcasting happens here (Concept 70)
    }
}
```

(Dapper is used for brevity. Production code would batch the event inserts — a table-valued parameter or a single `INSERT … SELECT FROM OPENJSON(@batch)` — and distinguish "retry of an append that already succeeded" by checking whether the conflicting `EventId`s are already in the stream.)

Design decisions worth narrating in an interview:

- **Two tables.** The `Streams` row is the concurrency point: an `UPDATE … WHERE Version = @expected` both checks and locks, so concurrent appenders to one stream serialize on one row while other streams proceed in parallel. The `UNIQUE (StreamId, Version)` constraint is a backstop that makes duplicate versions impossible even if the application logic is wrong.
- **`READ COMMITTED` is enough** for per-stream appends because the conflict is on one row. (It is *not* enough for DCB-style predicates — Concept 36.)
- **The clustered key is the global position**, so appends go to the end of the B-tree (no page splits in the middle) and "read all from position" is a range scan. The unique `(StreamId, Version)` index serves stream reads.
- **Events are never updated or deleted** by the application — enforce it with permissions (Concept 86).
- **Snapshots, subscriptions' checkpoints and projections live in other tables** — possibly in the same database, so inline projections and checkpoints can commit in the same transaction as events (Module 23, Concept 41).

PostgreSQL designs are the same shape (Marten's `mt_streams` and `mt_events` tables are a mature example worth reading), with `jsonb`, a `bigserial`/sequence for position, and — for gap-safe reads — a transaction-ID column (Concept 39).

---

## Concept 39 — The global-order gap problem

The most subtle bug in relational event stores — and a question that separates people who've operated one from people who've read about one.

**The setup.** Global positions come from an `IDENTITY` column or a sequence. Values are allocated **when the row is inserted**, but become visible **when the transaction commits** — and transactions commit in a different order than they started:

```
t1  Tx A inserts event → GlobalPosition 1000 (not committed; slow — maybe inline projections)
t2  Tx B inserts event → GlobalPosition 1001, commits
t3  Subscriber reads "WHERE GlobalPosition > 999" → sees 1001; checkpoint := 1001
t4  Tx A commits → event 1000 becomes visible
t5  Subscriber reads "WHERE GlobalPosition > 1001" → event 1000 is never processed. Silently.
```

A projection has now permanently missed an event, and nothing reported an error. Rolled-back transactions add a second phenomenon: gaps that will *never* fill (the value was consumed and discarded). A subscriber can't tell, by looking at the numbers, whether a gap is permanent or merely "not committed yet."

**Solutions**, from simplest to most scalable:

**1. Serialize all appends.** Take a global lock for every append (a single-row update, `sp_getapplock`, a PostgreSQL advisory lock) so transactions commit in position order. Correct and simple; caps the store's write throughput at one append transaction at a time. Acceptable for low-volume stores.

**2. Read only below a safe watermark, using the database's own knowledge of in-flight transactions.**

- **SQL Server**: add a `rowversion` column and read only rows whose value is below `MIN_ACTIVE_ROWVERSION()` — the lowest `rowversion` value still used by an active (uncommitted) transaction. Anything below it is committed and no lower value can appear later:

```sql
SELECT TOP (@batchSize) GlobalPosition, StreamId, Version, EventType, SchemaVersion, Data, Metadata, CommitVersion
FROM es.Events
WHERE CommitVersion > @lastCommitVersion              -- checkpoint is the rowversion, not the identity
  AND CommitVersion < MIN_ACTIVE_ROWVERSION()         -- nothing lower can still appear
ORDER BY CommitVersion;
```

- **PostgreSQL**: record the writing transaction's ID in each row (`transaction_id xid8 DEFAULT pg_current_xact_id()`), and read only events from transactions older than the oldest transaction still in progress — `WHERE transaction_id < pg_snapshot_xmin(pg_current_snapshot())` — ordered by `(transaction_id, position)`. Several open-source PostgreSQL event stores use this technique.

In both cases the checkpoint becomes the transaction-ordering value, and the global order is defined by commit, not by insertion.

**3. High-water-mark detection with a staleness threshold.** Read contiguous positions; when a gap appears, wait; if the gap persists longer than a threshold (seconds), assume it's a rollback and skip it. This is roughly how Marten's async daemon determines its "high water mark", with configurable staleness settings. It's pragmatic, and it has a failure mode: a transaction that stays open longer than the threshold is skipped. Keep append transactions short (no remote calls inside them), and treat daemon health warnings seriously.

**4. A store with a single-writer log.** KurrentDB appends every event through a single log writer, so positions are assigned in commit order and readers never see this problem. That's one of the genuine advantages of a purpose-built store.

**5. Per-stream subscriptions.** If a projection only needs one stream (or one partition), per-stream versions are gapless and the problem doesn't arise — but most projections subscribe to all events.

The interview line: *"In a relational event store, identity values are assigned at insert and become visible at commit, so a subscriber reading by position can skip an event that commits late. You fix it by serializing appends, by reading only below a watermark the database guarantees is fully committed — `MIN_ACTIVE_ROWVERSION` in SQL Server, the snapshot xmin in Postgres — or by gap detection with a timeout, which is what most libraries do. A purpose-built store with a single log writer doesn't have the problem."*

---

## Concept 40 — The .NET event-store landscape

As of September 2026, the realistic options for a .NET team:

| Option | Storage | Licence | Strengths | Watch out for |
|---|---|---|---|---|
| **Marten 9** | PostgreSQL (Azure Database for PostgreSQL flexible server) | MIT; commercial support from JasperFx | Most complete .NET library: streams, `FetchForWriting`, inline/async/live projections, async daemon, DCB, archiving, compaction, masking, multi-tenancy, EF Core projections, blue/green projections; documents in the same transaction; Wolverine integration | Requires PostgreSQL; many features, fast-moving (9.39 in September 2026 — read release notes); pre-generated code for production |
| **Polecat 5** | SQL Server 2025 / Azure SQL (native `json`) | MIT; JasperFx support | Marten's model for SQL Server shops; source-generated; DCB | Young (1.0 March 2026); requires SQL Server 2025 JSON features; some Marten features (e.g. upcasting) still arriving |
| **Fisher 1.0** | SQLite (embedded) | MIT | Zero-infrastructure: embedded apps, edge, tests; same API as Marten/Polecat | One writer per file; not a server database |
| **KurrentDB 26** + `KurrentDB.Client` | Purpose-built database (self-hosted or Kurrent Cloud) | KLv1 source-available; multi-node needs Enterprise licence from 26.2 | Built for event sourcing: single-writer log, `$all`, catch-up and persistent subscriptions, secondary indexes, connectors, SQL access, archiving | A separate database to operate; licence changes (Concepts 41, 100); projections in the server are JavaScript |
| **Eventuous** | KurrentDB, PostgreSQL, SQL Server, and more | Apache-2.0 | Opinionated, lightweight library: aggregates/functional services, subscriptions with checkpoints, producers (Kafka, RabbitMQ, Service Bus…), diagnostics | 0.x — "somewhat volatile" by its own description; smaller community |
| **Orleans 10 `JournaledGrain`** | Pluggable log-consistency providers | MIT | Event-sourced virtual actors: single-threaded activation serializes commands without conflicts (Concept 32) | Built-in providers are basic; production use typically needs a custom-storage provider to a real event store; projections are yours |
| **Cosmos DB, do-it-yourself** | Cosmos DB for NoSQL | Azure service | Global distribution, elastic scale, change feed, Azure-native | No global order; 20 GB per logical partition; you build versioning, subscriptions, snapshots (Concept 42) |
| **Hand-rolled on SQL** | SQL Server / PostgreSQL | — | Full control; tiny dependency surface | You own gap-safe subscriptions, upcasting, projections, tooling (Concepts 38–39, 99) |
| **Wolverine 6** (not a store) | Uses Marten, Polecat or Fisher | MIT | Aggregate-handler workflow over deciders, transactional outbox, durable messaging | A framework commitment (Module 23, Concept 78) |

Outside .NET, and relevant when a polyglot organization asks: **Axon Server/Framework** (JVM; DCB since Axon Server 2025.1), **EventSourcingDB** (a purpose-built store with EventQL and DCB support), **UmaDB** (a DCB-native store), and the **Python `eventsourcing`** library.

The decision criteria come in Concept 98. The short version: *use the database you already operate* — PostgreSQL → Marten; SQL Server 2025 → Polecat (with its youth in mind); a dedicated event store if its subscriptions, indexing and operational model justify running one more database.

---

## Concept 41 — KurrentDB in depth

KurrentDB (formerly EventStoreDB, created by Greg Young's company Event Store) is the best-known purpose-built event store. What to know about it:

**Storage model.** Every event is appended to one transaction log, stored in fixed-size **chunk files**. Streams are logical: an index maps stream IDs to log positions. `$all` is the log itself, read in commit order — so global order has no gaps problem (Concept 39). Every event has a stream **revision** (0-based) and an `$all` **position**.

**Writing.**

```csharp
// KurrentDB.Client 1.x — a sketch; check the client docs for your version's exact overloads
await using var client = new KurrentDBClient(KurrentDBClientSettings.Create("kurrentdb://localhost:2113?tls=false"));

var data = new EventData(
    Uuid.FromGuid(eventId),                               // same ID on retry → idempotent (Concept 29)
    "order-shipped",                                      // stable type name (Concept 20)
    JsonSerializer.SerializeToUtf8Bytes(evt, OrderingEventsJsonContext.Default.OrderShipped),
    JsonSerializer.SerializeToUtf8Bytes(metadata));       // correlation, causation, actor…

try
{
    // Expected revision: a ulong revision, or StreamState.NoStream / StreamExists / Any
    await client.AppendToStreamAsync("order-42", expectedRevision, [data], cancellationToken: ct);
}
catch (WrongExpectedVersionException)
{
    // Someone appended since we read: re-read, re-decide (Concept 49)
}
```

**Reading and subscribing.** `ReadStreamAsync` reads one stream; `ReadAllAsync` reads the log. Two subscription models:

- **Catch-up subscriptions** — the client subscribes from a position it stores itself (its checkpoint), reads history, then switches to live. Ordered; one consumer per subscription; the natural fit for projections that must be ordered (Concept 58).
- **Persistent subscriptions** — the server tracks the position for a named consumer group and distributes events across competing consumers with acknowledgements, retries and **parked** (dead-lettered) messages. Good for reactors that need load balancing; ordering is not guaranteed across consumers.

Both can filter server-side (by event type or stream prefix), and `$all` subscriptions with server-side filtering are generally recommended over thousands of per-stream subscriptions.

**Indexes and queries.** Historically, categories and event types were indexed by the system projections `$by_category` and `$by_event_type`, which wrote **link events** into the log — more writes, read amplification, and dangling links when originals were deleted. **25.1** replaced them with **secondary indexes** stored in an embedded DuckDB; Kurrent reported large reductions in storage and index rebuild time, and much higher subscription read throughput, from the change. **26.0** added **user-defined secondary indexes** over event content (for example orders by country), and 26.2 (previewed) extends them to multiple fields. **26.1** added SQL access via Arrow Flight SQL for analytical queries. These are welcome — and they don't change the architectural rule that operational read models are projections you own (Concept 55); ad-hoc queries are for exploration and analytics.

**Server-side projections.** KurrentDB can run JavaScript projections inside the server to build derived streams or state; 26.1 introduced an experimental "Projections V2" engine. Most .NET teams build projections in application code instead (subscriptions → their own read stores), which keeps projection logic in C#, testable and deployable with the application.

**Stream management.** Per-stream metadata supports `$maxAge`, `$maxCount` (truncation), ACLs and `$tb` (truncate-before). **Soft delete** hides a stream (it can be recreated, continuing the revision sequence); **hard delete** writes a tombstone and forbids recreation. Deleted and truncated events are physically removed only by **scavenging** — a background compaction process (auto-scavenge becomes free in 26.2).

**Archiving.** Since 25.0, completed chunks can be uploaded to object storage (S3, and Azure Blob Storage and GCP since 26.0) and removed from nodes' local disks; reads transparently fall back to the archive.

**Clustering and licensing.** High availability means a three-node cluster (or more), with quorum writes. From **26.2**, clustering runs on **Raft**, and **multi-node clusters, read-only replicas, archiving and encryption at rest require an Enterprise licence**; single-node, including production use with backup and restore, is free, and many previously licensed features (connectors, OAuth/LDAP/X.509 auth, OpenTelemetry export, the Kubernetes Operator, SQL access) become free. Versions up to 26.1 keep their existing licensing for their LTS term. **Kurrent Cloud** offers the managed alternative. LTS releases come once a year, around January, with at least two years of support.

The architect's summary: *KurrentDB gives you the cleanest event-store semantics — a single ordered log, first-class subscriptions, no gap problem — at the price of one more stateful system to run, and, for high availability on 26.2+, a commercial licence.*

---

## Concept 42 — Cosmos DB as an event store

Cosmos DB is a common choice for Azure-native teams, and Microsoft publishes event-sourcing samples for it. It works well if you design around its model.

**The design.**

- **One stream = one logical partition.** Partition key = stream ID. Everything about a stream — reads, appends, ordering — stays in one partition.
- **One event = one item**, with `id = "{version}"` (or `"{streamId}:{version}"`). Item IDs are unique within a logical partition, so **two writers creating the same version collide with a 409 Conflict** — that uniqueness *is* the optimistic-concurrency check.
- **Append several events atomically** with a **transactional batch** — all operations in one logical partition, all or nothing (up to 100 operations and 2 MB per batch).
- **Read a stream** with a single-partition query ordered by version.
- **Subscribe** with the **change feed processor**: because events are only ever created (never replaced), latest-version mode delivers every event, **in order per logical partition**, with leases as checkpoints (Module 23, Concept 48).

```csharp
public sealed record EventDocument(
    string id, string streamId, long version, string eventType, int schemaVersion,
    JsonElement data, JsonElement? metadata, DateTimeOffset occurredAt);

public async Task AppendAsync(string streamId, long expectedVersion, IReadOnlyList<(string Type, int Schema, JsonElement Data)> events, CancellationToken ct)
{
    var batch = container.CreateTransactionalBatch(new PartitionKey(streamId));
    var v = expectedVersion;
    foreach (var (type, schema, data) in events)
    {
        v++;
        batch.CreateItem(new EventDocument(v.ToString(CultureInfo.InvariantCulture), streamId, v, type, schema,
                                           data, null, clock.GetUtcNow()));
    }

    using var response = await batch.ExecuteAsync(ct);
    if (response.StatusCode == HttpStatusCode.Conflict)            // some version already exists
        throw new WrongExpectedVersionException(streamId, expectedVersion);
    if (!response.IsSuccessStatusCode)
        throw new EventStoreException($"Append failed: {response.StatusCode} {response.ErrorMessage}");
}
```

(If you also want an explicit stream "head" document — for stream type, archived flag or a snapshot pointer — include a `ReplaceItem` with an `IfMatchEtag` in the same batch. Either the version-as-id collision or the ETag gives you exact-version semantics.)

**What Cosmos DB doesn't give you, and what to do about it:**

| Missing | Consequence | Mitigation |
|---|---|---|
| **A global order across partitions** | "Read all events in order" doesn't exist; the change feed is ordered only per partition | Design projections to be per-stream-ordered and idempotent (Module 23, Concepts 39–40); don't promise cross-stream ordering |
| **Unbounded streams** | A logical partition is limited to **20 GB**; an item to **2 MB** | Keep streams short (Concept 31) — here it's a hard limit, not just a performance concern |
| **Cheap long reads** | Every stream load is a query costing RUs proportional to events read | Snapshots (Concept 50) for longer streams; short streams first |
| **Cross-stream transactions** | Batches are single-partition | Process managers, registry streams (Concept 33); no DCB |
| **Built-in projections, upcasting, snapshots** | You build them | Budget for it (Concept 99) |

The all-versions-and-deletes change-feed mode (GA in June 2026, Module 23) isn't needed for an append-only container — every item is a creation — but its retention depends on continuous backup, so latest-version mode is the natural fit here.

The senior sentence: *"On Cosmos I'd put each stream in its own logical partition, use the version as the item id so the 409 on a duplicate id is my concurrency check, append with a transactional batch, and subscribe with the change feed processor — accepting that there's no global order, that streams must stay well under 20 GB, and that projections, snapshots and upcasting are mine to build."*

---

## Concept 43 — Kafka is not an event store

Proposing Kafka (or Event Hubs, or Service Bus) as the event store for an event-sourced write model is one of the most common mistakes in design interviews, and Microsoft's 2026 guidance now warns against it explicitly. The reasons, from first principles (Concept 26):

| Event-store requirement | Kafka | Consequence |
|---|---|---|
| **Append with expected version** | No per-key optimistic concurrency: a producer can't say "append to key `order-42` only if its last offset is N" (a proposal for expected-offset produces, KAFKA-2260, has sat at minor priority for years) | Two command handlers can append conflicting events for the same entity; nothing detects it |
| **Read one entity's history efficiently** | Topics are partitioned logs; reading one key's events means scanning its whole partition (or maintaining your own index) | Loading an aggregate is O(partition), not O(stream) |
| **Keep history forever** | Retention is configurable (including infinite, and tiered storage in Kafka 4.x), and compaction keeps only the *latest* record per key | Compaction destroys event history; infinite retention is possible but not the design centre |
| **Strongly consistent read-your-append** | Consumers read asynchronously; there's no "read this key's events including my write" | The handler can't reliably load current state before deciding |

What goes wrong in practice: a service consumes its own topic to build aggregate state in memory or a database, decides from that state, and produces new events. The state it decides from is always slightly stale (consumer lag), concurrent handlers race, and duplicates or out-of-order events corrupt the model — the failures Oskar Dudycz catalogues in *Event Streaming is not Event Sourcing!*.

**The nuance a senior answer adds:** you *can* build event-sourced processing on Kafka if you guarantee a **single writer per key** — for example a stream processor where each partition is owned by exactly one instance, holding state in a local store backed by a changelog topic. Then there's no concurrent writer to detect. But you've replaced the store's concurrency check with a single-writer guarantee that must survive rebalances and zombie instances — which Kafka provides through producer epochs and fencing (Module 9's fencing tokens). It's a different architecture with different operational risks, and it doesn't give you cheap per-entity reads for request/response commands.

**Where Kafka and Event Hubs belong** in an event-sourced system: **after** the store, as the distribution layer — publishing integration events to other contexts, feeding analytics, retaining a replayable public stream (Concepts 19, 65).

The interview line: *"Kafka is a great distribution log but not an event store: it can't append conditionally on a key's current version, and it can't read one entity's history efficiently. I'd keep the source of truth in an event store with append-if-version, and publish from it to Kafka for everyone else."*

---

# Part D — The write side: aggregates and deciders over streams

Module 22 designed aggregates and deciders; Module 23 designed command handlers. Part D puts them on top of an event store — and the result is simpler than a state-stored write side. There's no ORM, no change tracking, no mapping of an object graph to tables: load facts, fold them, decide, append. What remains is a handful of rules that aren't optional — about purity, about what `evolve` may never do, about conflicts, snapshots and side effects.

---

## Concept 44 — The command-handling loop

Every command handler in an event-sourced system has the same four steps:

```
1. LOAD     events = store.Read(streamId)                     (+ snapshot, Concept 50)
2. FOLD     state  = events.Aggregate(initial, evolve)
3. DECIDE   result = decide(command, state)                   → new events | rejection
4. APPEND   store.Append(streamId, newEvents, expectedVersion: events.LastVersion)
```

Compared with the state-stored handler of Module 23 (Concept 17) — load, decide, save — two things disappear and one appears:

- **Gone: the mapping layer.** No `DbContext`, no navigations, no `Include`, no change tracking, no "did EF notice that the child changed?" (Module 22, Concept 68's trap). The write model is just state computed in memory.
- **Gone: dual writes of state and events.** The events *are* the state; there's no outbox to fill in the same transaction (Concept 65).
- **New: the fold.** State is computed on every load — cheap for short streams, a performance concern for long ones (Concepts 31, 50, 89).

The rules that make this loop correct:

1. **One stream per command, by default.** A command that needs several streams is either a multi-stream decision (Concept 34), a DCB decision (Concept 35) or a workflow (Concept 52).
2. **No I/O inside `decide`.** Everything the decision needs is either in the stream or on the command. If it needs external data (a price, a credit score), fetch it *before* loading the stream, pass it in, and record it in the resulting event (Concept 16) — both for determinism in tests and to keep the decision window short (Concept 32).
3. **The expected version is the version of the last event loaded** — not "any," and not something computed after deciding.
4. **Rejections append nothing** (unless the business wants rejection facts recorded, Concept 7).
5. **The handler returns an outcome and the new version** (Module 23, Concepts 12 and 53) — the version becomes the client's next expected version and its consistency token.

---

## Concept 45 — The OO event-sourced aggregate

The object-oriented style keeps the aggregate as a class with behaviour methods, like Module 22's aggregates, with two changes: **methods raise events instead of mutating fields**, and **fields are mutated only by applying events**.

```csharp
public abstract class EventSourcedAggregate
{
    private readonly List<object> _uncommitted = [];

    public string Id { get; protected set; } = "";
    public long Version { get; private set; }                      // version of the last *stored* event (0 = new)
    public IReadOnlyList<object> UncommittedEvents => _uncommitted;

    protected void Raise(object @event)
    {
        Apply(@event);                                               // state changes only through Apply
        _uncommitted.Add(@event);
    }

    public void LoadFromHistory(IEnumerable<object> history)
    {
        foreach (var e in history) { Apply(e); Version++; }
    }

    public void MarkCommitted(long newVersion) { _uncommitted.Clear(); Version = newVersion; }

    protected abstract void Apply(object @event);                   // must be total and pure (Concept 47)
}

public sealed class Order : EventSourcedAggregate
{
    private OrderStatus _status;
    private decimal _total;
    private readonly Dictionary<string, int> _lines = [];

    public static Order Draft(string orderId, Guid customerId, string currency)
    {
        var order = new Order { Id = orderId };                    // the stream ID travels in the envelope
        order.Raise(new OrderDrafted(customerId, currency));
        return order;
    }

    public void AddLine(string sku, int qty, decimal unitPrice)
    {
        if (_status != OrderStatus.Draft) throw new DomainException("Only draft orders can change.");  // or return an outcome
        if (qty <= 0) throw new DomainException("Quantity must be positive.");
        var lineTotal = qty * unitPrice;
        Raise(new OrderLineAdded(sku, qty, unitPrice, _total + lineTotal));
    }

    public void Place(DateTimeOffset now)
    {
        if (_status != OrderStatus.Draft) throw new DomainException("The order is not a draft.");
        if (_lines.Count == 0) throw new DomainException("An order needs at least one line.");
        Raise(new OrderPlaced(_total, now));
    }

    protected override void Apply(object @event)
    {
        switch (@event)
        {
            case OrderDrafted:      _status = OrderStatus.Draft; break;
            case OrderLineAdded a:  _lines[a.Sku] = _lines.GetValueOrDefault(a.Sku) + a.Qty; _total = a.NewTotal; break;
            case OrderPlaced p:     _status = OrderStatus.Placed; _total = p.Total; break;
            case OrderShipped:      _status = OrderStatus.Shipped; break;
            case OrderCancelled:    _status = OrderStatus.Cancelled; break;
            // unknown events: ignore — never throw (Concept 47)
        }
    }
}

// The repository: load by folding, save by appending the uncommitted events with the loaded version
public sealed class EventSourcedRepository<T>(IEventStore store, Func<T> factory) where T : EventSourcedAggregate
{
    public async Task<T?> GetAsync(string id, CancellationToken ct)
    {
        var events = await store.ReadStreamAsync(id, fromVersion: 0, ct);
        if (events.Count == 0) return null;
        var aggregate = factory();
        aggregate.LoadFromHistory(events.Select(e => e.Payload));
        return aggregate;
    }

    public async Task SaveAsync(T aggregate, CancellationToken ct)
    {
        if (aggregate.UncommittedEvents.Count == 0) return;
        var expected = aggregate.Version;                           // the stored version this command decided on
        var newVersion = await store.AppendAsync(aggregate.Id, expected, aggregate.UncommittedEvents, ct);
        aggregate.MarkCommitted(newVersion);
    }
}
```

(Note the bookkeeping: `Raise` applies each new event immediately so later checks in the same command see it, but `Version` only counts *stored* events — `LoadFromHistory` increments it, `MarkCommitted` sets it after the append — so it's exactly the expected version for the append. Off-by-one errors here are a classic bug, and one reason teams prefer the functional decider, Concept 46, or a library that does the bookkeeping.)

Marten supports this style directly — "self-aggregating" types with `Apply` or `Create` methods, loaded with `FetchForWriting<T>` or `AggregateStreamAsync<T>` — as does Eventuous, with a state class separated from the aggregate.

---

## Concept 46 — The decider over an event store

Module 22 (Concept 73) introduced Jérémie Chassaing's **Decider**: `initialState`, `decide(command, state) → events`, `evolve(state, event) → state`, all pure. Module 22 claimed the same decider works with an event store or a state store. Here's the event-store half of that claim — **one generic handler for every decider in the system**:

```csharp
public sealed record Decider<TCommand, TEvent, TState>(
    TState Initial,
    Func<TCommand, TState, Decision<TEvent>> Decide,
    Func<TState, TEvent, TState> Evolve);

public abstract record Decision<TEvent>;                                    // on .NET 11: closed record
public sealed record Accepted<TEvent>(IReadOnlyList<TEvent> Events) : Decision<TEvent>;
public sealed record Rejected<TEvent>(string Reason) : Decision<TEvent>;

public sealed class DeciderHandler<TCommand, TEvent, TState>(
    IEventStore store, Decider<TCommand, TEvent, TState> decider, ResiliencePipeline conflictRetry)
{
    public ValueTask<HandleResult> HandleAsync(string streamId, TCommand command, CancellationToken ct) =>
        conflictRetry.ExecuteAsync(async token =>                           // retry = re-run the whole loop (Concept 49)
        {
            var history = await store.ReadStreamAsync(streamId, fromVersion: 0, token);
            var state = history.Select(e => (TEvent)e.Payload).Aggregate(decider.Initial, decider.Evolve);
            var expectedVersion = history.Count == 0 ? 0 : history[^1].StreamVersion;

            switch (decider.Decide(command, state))
            {
                case Rejected<TEvent> r:
                    return HandleResult.Rejected(r.Reason, expectedVersion);
                case Accepted<TEvent> { Events.Count: 0 }:
                    return HandleResult.NoChange(expectedVersion);          // idempotent no-op (Concept 29)
                case Accepted<TEvent> a:
                    var newVersion = await store.AppendAsync(streamId, expectedVersion, a.Events.Cast<object>().ToList(), token);
                    return HandleResult.Accepted(newVersion);
                default:
                    throw new UnreachableException();
            }
        }, ct);
}
```

The payoff is structural:

- **All the infrastructure — loading, folding, versioning, appending, conflict retries, metadata — lives in one place.** Each context supplies only pure functions.
- **The domain is testable without any infrastructure** — given events, when command, then events (Concept 54).
- **Switching storage is trivial.** The same decider could be run against a state store (`state = load(); events = decide(); state = events.Aggregate(state, evolve); save(state)`), which is how teams can adopt deciders first and event sourcing later — or back out of event sourcing without rewriting the domain (Concept 94).

Wolverine's **aggregate handler workflow** over Marten, Polecat or Fisher is essentially this shape as a framework: a handler receives the command and the current aggregate (fetched with `FetchForWriting`), returns events, and the framework appends them, enforces the version and commits the outbox in the same transaction. Its documentation credits the Decider pattern as the inspiration.

---

## Concept 47 — Evolve never decides

The single most important rule for event-sourced write models, and the single most common bug in code reviews:

> **`evolve` (or `Apply`) records facts; it never judges them. It must be total, pure, deterministic and unable to fail.**

The bug:

```csharp
// WRONG: validation inside Apply
protected override void Apply(object @event)
{
    switch (@event)
    {
        case OrderLineAdded a:
            if (_status != OrderStatus.Draft)
                throw new DomainException("Cannot add lines to a placed order.");   // ← in Apply
            _lines[a.Sku] = _lines.GetValueOrDefault(a.Sku) + a.Qty;
            break;
    }
}
```

Why it's so dangerous: the check *looks* like it protects an invariant. But `Apply` runs every time the stream is loaded. If an `OrderLineAdded` event ever gets stored after `OrderPlaced` — a bug in an older version, a race in a migration, a business rule that changed ("placed orders can now have lines added for 5 minutes") — then **every future load of that stream throws**. The order can never be loaded again: not to fix it, not to cancel it, not to display it. A stored fact has become a permanent outage for that entity.

The rules:

| Rule | Why |
|---|---|
| **Validation belongs in the decision** (command methods, `decide`) | That's when you can still say no |
| **`evolve` accepts every event that exists in storage** | The events are facts; history is what it is |
| **`evolve` ignores events it doesn't know** | New event types may be added by newer code (Concept 71); old code must not crash on them |
| **`evolve` never does I/O, never reads the clock, never generates IDs** | Determinism (Concept 4) |
| **`evolve` never throws** — not even for "impossible" states | An impossible state in history needs a *fix forward* (Concept 74), not a crash |
| **`evolve` is cheap** | It runs on every load and every replay |

Oskar Dudycz's article *Should you throw an exception when rebuilding the state from events?* reaches the same conclusion, and it's worth one more framing: **`decide` is about the future (may this happen?); `evolve` is about the past (this happened — now what's true?).** You can't reject the past.

A code-review heuristic: *any `throw`, `if (… ) return error`, database call or `DateTime.Now` inside an `Apply`/`Evolve`/`When` method is a finding.*

---

## Concept 48 — Invariants, decisions and bad events

If `evolve` can't enforce invariants, where are they enforced, and what happens when the history already violates one?

**Invariants are enforced at decision time, against the current history.** The decision is the only moment the system can refuse. Once an event is appended, it's a fact. That's why:

- Decisions must be made on **complete and current** history — hence optimistic concurrency (Concept 28).
- The event must record **enough to show the decision was valid** at the time (Concept 16) — e.g. `CreditLimitApproved { Amount, ScoreAtDecision, PolicyVersion }`.

**When history contains a violation anyway** — a bug emitted wrong events, a race was mishandled, rules changed retroactively — the event-sourced response is to **fix forward**:

1. **Detect it**: a projection or a scheduled check that finds states the rules say are impossible (an order with lines added after placement, a negative balance).
2. **Decide what the business wants**: honour it, reverse it, or escalate to a person.
3. **Record the correction as new events**: `OrderLineRemovedByCorrection { Sku, Reason: "Added after placement due to bug #1234" }`, `BalanceCorrected { Amount, Reason }`. The history now shows the mistake *and* the fix — exactly as a ledger would (Concept 2).
4. **Make `evolve` handle the corrected history** — it already will, because it never rejected anything.

Concept 74 covers the full toolkit for wrong events (compensating events, correction events, skipping, and — rarely — rewriting).

A related design point: **what does a decision return when the command is a no-op?** `Place()` on an already-placed order could throw, return a rejection, or return *success with no events* ("already placed" is the desired end state). For commands retried by machines, the last is often best: the command is naturally idempotent (Module 23, Concept 19) and the handler reports success without appending.

---

## Concept 49 — Resolving concurrency conflicts

When an append fails with a version conflict, the command was decided on stale history. Module 22 (Concept 69) gave the three strategies; in event sourcing each has a clean implementation.

**1. Re-decide (automatic retry).** For machine-issued commands and for commands whose validity doesn't depend on what the user saw: re-read the stream, fold, decide again, append. The retry re-runs the *whole* loop — never just the append. Most conflicts resolve on the first retry. Bound the retries (three to five with jitter) and treat exhaustion as a hot-stream signal (Concept 32).

**2. Surface to the user.** When the user acted on what they saw — an edit screen, a price confirmation — the client sends the version it saw as the expected version (`If-Match`). A conflict returns **412 Precondition Failed** with the current version; the UI shows what changed. Retrying automatically would silently apply the user's decision to a state they never saw.

**3. Merge by event semantics.** Sometimes the conflicting events don't actually conflict *in meaning*. If A appended `NoteAdded` and B wants to append `TagAdded`, B's decision doesn't depend on A's event at all. The classic event-sourcing technique (described in Greg Young's early writing and implemented in several frameworks): on conflict, read the events that were appended since the expected version and check whether any of them **conflicts with the events being appended**, using a conflict table:

```csharp
// "Does any event appended since I read conflict with what I want to append?"
static bool ConflictsWith(object mine, object theirs) => (mine, theirs) switch
{
    (NoteAdded, _)                    => false,          // notes never conflict
    (_, NoteAdded)                    => false,
    (TagAdded, TagAdded)              => false,          // independent tags commute
    (OrderShipped, OrderCancelled)    => true,           // can't ship a cancelled order
    (OrderCancelled, OrderShipped)    => true,
    _                                 => true,           // default: conservative
};

// On WrongExpectedVersion: read the new events; if none conflicts, re-append with the new expected version.
// If any conflicts, re-decide (strategy 1) or surface (strategy 2).
```

This is more efficient than re-deciding when decisions are expensive, and it expresses concurrency rules in business terms. But it's subtler to get right than simply re-running the decision — re-deciding on fresh state is always correct, if the decision is cheap. Most systems should start with strategy 1 and reach for semantic merging only for proven hot paths.

The rule of thumb, from Module 22: **race conditions are business questions.** "Two agents approved the same claim at the same time" is a conversation with the business, not a retry policy.

---

## Concept 50 — Snapshots

A **snapshot** is the folded state of a stream at some version, stored so that loading can start from it instead of from the first event:

```
load = snapshot(v=900) + fold(events 901..927)       instead of fold(events 1..927)
```

The rules, which the Marten documentation, Microsoft's guidance and nearly every practitioner agree on:

**1. Snapshots are a performance optimization, never a design element.** The stream remains the source of truth; a snapshot is a cache of the fold. You must be able to delete every snapshot and lose nothing but speed.

**2. Don't snapshot until you've measured a need.** Loading a few dozen or a few hundred small events from a nearby database is typically a few milliseconds (Concept 89). If streams are short (Concept 31), you may never need snapshots. Oskar Dudycz's *Snapshots in Event Sourcing* and Marten's own documentation both say: keep it simple until performance forces it.

**3. Prefer short streams over snapshots.** A snapshot makes a long stream load faster; it doesn't fix its conflict rate, its versioning surface, its storage growth or its 20 GB partition limit on Cosmos DB. If you need snapshots everywhere, the streams are probably modeled wrong.

**4. Snapshots must be versioned and invalidated.** A snapshot is a serialization of your *state type*, which — unlike events — you're free to change (Concept 77). When the state shape or `evolve` logic changes, old snapshots are wrong. Store a **snapshot schema version** (or a hash of the evolve logic's version) with each snapshot, and ignore snapshots whose version doesn't match — falling back to a full fold and writing a new snapshot. Never upcast snapshots; just rebuild them.

**5. Choose when to take them.**

| Strategy | How | Trade-off |
|---|---|---|
| **Every N events** | After an append crosses a multiple of N (e.g. 100) | Simple, predictable load bound; snapshot writes add to some commands |
| **Asynchronously** | A subscription writes snapshots in the background | No write-path cost; loads may find an older snapshot and fold more events |
| **Inline, every append** | Persist folded state in the same transaction (Marten's inline snapshot/self-aggregate) | Load is one read; every append also writes state — effectively "state stored plus event log" |
| **On closing** | At the end of a lifecycle (Concept 31) | The closing event *is* the snapshot for the next period — the cleanest option |

**6. Where to store them.** A separate table/container keyed by stream ID (the common choice), a separate "snapshot stream" per stream in KurrentDB (read backwards for the latest), or the projected document itself (Marten stores inline/async single-stream projections as documents that `FetchForWriting` can start from, reading only events after the document's version).

```csharp
public async Task<(OrderState State, long Version)> LoadAsync(string streamId, CancellationToken ct)
{
    var snap = await snapshots.GetAsync<OrderState>(streamId, ct);                 // may be null
    var (state, fromVersion) = snap is { SchemaVersion: OrderState.SnapshotSchemaVersion }
        ? (snap.State, snap.Version)
        : (OrderState.Initial, 0L);                                                 // stale or missing: full fold

    var events = await store.ReadStreamAsync(streamId, fromVersion, ct);
    var folded = events.Select(e => (OrderEvent)e.Payload).Aggregate(state, OrderEvolve.Evolve);
    var version = events.Count > 0 ? events[^1].StreamVersion : fromVersion;

    if (version - fromVersion >= SnapshotEvery)                                     // opportunistic refresh
        await snapshots.SaveAsync(streamId, folded, version, OrderState.SnapshotSchemaVersion, ct);
    return (folded, version);
}
```

Marten 9 also offers an opt-in, node-local **cache of aggregates for writing** (`CacheAggregatesForWriting<T>`) for hot aggregates: the cached snapshot is only a baseline, and every call still reads the stream version and newer events and keeps the optimistic-concurrency assertion on append — its documentation notes a "trusted" variant that skipped the version read was measured and deliberately rejected, because it saved about 1.4% of the round trip in exchange for the concurrency guarantee. That's a nice illustration of the principle: **caches may speed up the fold, never replace the version check.**

---

## Concept 51 — Live, inline and async write models

Where does the write model's current state come from when a command arrives? Three options, which Marten names explicitly and other stores implement by hand:

| Lifecycle | How the write model's state is obtained | Consistency | Cost |
|---|---|---|---|
| **Live** | Fold the stream (from a snapshot if any) on every command | Always current | Load cost grows with stream length |
| **Inline** | The folded state is persisted as a document in the *same transaction* as each append; commands load the document | Always current (same transaction) | Every append also writes the state; state schema changes need a rebuild |
| **Async** | A background projection maintains the state document; commands load it **plus any newer events** and fold them | Current *if* the loader folds the delta; stale if it doesn't | Cheap appends; loads read a document plus a few events |

The trap is the async case. A write model read from an asynchronously maintained projection *without* folding the newer events is stale by the projection's lag — and deciding on stale state is exactly what optimistic concurrency exists to prevent. Marten's `FetchForWriting` handles it correctly: it loads the persisted state and then applies any events newer than the document's version, so the decision is made on current state regardless of the projection lifecycle — which also enables zero-downtime changes to the write-model projection (the projection can be rebuilt in the background while `FetchForWriting` effectively falls back to live aggregation for streams not yet caught up). Jeremy Miller's advice when the API was introduced was to keep write models inline or live unless you have a reason; `FetchForWriting`'s delta-folding is what makes async write models safe.

The rule to state in an interview: **a write model may be cached anywhere, but a decision must always be made on the stream's latest version — and the append must still check it.**

---

## Concept 52 — Process managers in an event-sourced system

Many business processes span several streams over time: a checkout reserves stock, captures payment and confirms the order; a claim moves from submitted to assessed to approved to paid, with deadlines. Module 22 (Concept 85) described **process managers** as the consistency boundary of a workflow; in an event-sourced system they're typically **event-sourced themselves**:

```
checkout-7c1 stream:
  CheckoutStarted          { orderId, customerId, amount }
  StockReservationRequested
  StockReserved            ← recorded when the Inventory context's integration event arrives
  PaymentRequested
  PaymentCaptured          ← recorded when the Payments context confirms
  CheckoutCompleted
```

The process manager is a decider whose commands are *incoming events and timeouts*, and whose output events include "what I've asked for next":

- It **subscribes** to the events it cares about (from its own context's store and from other contexts' integration events).
- For each input, it **loads its own stream, decides** what should happen next, **appends** that decision, and then **sends commands** to other aggregates or contexts — *after* the append commits (Concept 53).
- **Timeouts are inputs too.** A scheduled message ("PaymentTimeout for checkout-7c1 at 10:30") arrives as a command; the process manager decides whether it still matters (the payment may already have arrived) and compensates if needed. Verraes' **Passage of Time Event** pattern generalizes this: publish "day ended" or "hour passed" events and let processes react, instead of scattering cron jobs.
- **Compensation is recorded**: `StockReleased`, `PaymentRefunded`, `CheckoutAbandoned` — the history of the process shows everything that happened, including the undo.

Why event-source the process manager:

- Its state is the history of the conversation, which is exactly what support staff ask about ("where is this checkout stuck?").
- Replaying a process manager's *own* stream is safe (it just rebuilds state); what must not replay are the commands it sent (Concept 64).
- The same concurrency rules apply: a process manager receiving two events at once conflicts on its own stream and re-decides.

Wolverine's sagas and Eventuous's gateways/reactors are .NET building blocks for this; many teams hand-roll it with a decider plus a subscription plus a durable scheduler.

---

## Concept 53 — Side effects and reactors

Sending emails, calling payment providers, notifying partners, publishing to a broker — none of these belong in the command handler's transaction, and none may ever happen inside `evolve`. In an event-sourced system they're done by **reactors** (also called *policies*, *event handlers* or *automations*): consumers that subscribe to committed events and perform side effects.

The rules:

1. **Side effects happen after the append commits.** A reactor reads the committed log. This removes the dual-write problem (Module 11): the event exists, so the reaction will eventually happen; there's no window in which the email was sent for an order that was never stored.
2. **Reactors are at-least-once, so they must be idempotent.** Use the event ID (or global position) as the idempotency key when calling external systems that support it (payment providers usually accept an idempotency key), and record "reaction done" in the reactor's own store.
3. **Reactors must never run during replays or projection rebuilds.** A rebuild replays years of history; a reactor that re-sends every confirmation email is an incident. Keep reactors in **separate subscriptions** from projections, with their own checkpoints that start at "now" when first deployed — never at the beginning of the log (Concept 64).
4. **Reactions that matter to the domain are recorded as events.** If the business cares whether the confirmation email was sent, the reactor issues a command (`RecordConfirmationSent`) that appends an event. The event store then shows the full story.
5. **Failure handling is explicit**: retries with backoff, a dead-letter store for poison events, alerts (Module 23, Concept 42).

A diagram of where side effects may live:

```
command handler:   decide + append                     ✗ no emails, no HTTP calls, no broker publishes
evolve / Apply:    fold                                ✗ nothing, ever
projection:        update read model                   ✗ no side effects (it will be replayed)
reactor:           after commit, idempotent            ✓ side effects live here
process manager:   decide next step, append, then send ✓ commands to others, after its own append
```

---

## Concept 54 — Testing the write side

Event sourcing's write side is the most testable business logic you'll ever write, because deciders are pure functions over events. The canonical test shape — **given events, when command, then events** — reads like a specification a domain expert can review:

```csharp
public class PlacingAnOrder
{
    private static readonly DateTimeOffset Now = new(2026, 9, 26, 10, 0, 0, TimeSpan.Zero);
    private static DeciderSpec<OrderCommand, OrderEvent, OrderState> OrderSpec => new(OrderDecider.Decider);

    [Fact]
    public void A_draft_order_with_lines_can_be_placed() =>
        OrderSpec
            .Given(new OrderDrafted(CustomerId: Guid.Empty, Currency: "EUR"),
                   new OrderLineAdded("KB-01", 1, 120m, 120m))
            .When(new PlaceOrder(OrderId: "order-42", At: Now))
            .Then(new OrderPlaced(Total: 120m, PlacedAt: Now));

    [Fact]
    public void An_empty_order_cannot_be_placed() =>
        OrderSpec
            .Given(new OrderDrafted(Guid.Empty, "EUR"))
            .When(new PlaceOrder("order-42", Now))
            .ThenRejected("An order needs at least one line.");

    [Fact]
    public void Placing_twice_is_a_no_op() =>
        OrderSpec
            .Given(new OrderDrafted(Guid.Empty, "EUR"),
                   new OrderLineAdded("KB-01", 1, 120m, 120m),
                   new OrderPlaced(120m, Now))
            .When(new PlaceOrder("order-42", Now.AddMinutes(1)))
            .ThenNothing();
}

// A tiny generic helper over the decider — no mocks, no database
public sealed class DeciderSpec<TCommand, TEvent, TState>(Decider<TCommand, TEvent, TState> decider)
{
    private IReadOnlyList<TEvent> _given = [];
    private TCommand? _when;

    public DeciderSpec<TCommand, TEvent, TState> Given(params TEvent[] events) { _given = events; return this; }
    public DeciderSpec<TCommand, TEvent, TState> When(TCommand command) { _when = command; return this; }

    private Decision<TEvent> Run() =>
        decider.Decide(_when!, _given.Aggregate(decider.Initial, decider.Evolve));

    public void Then(params TEvent[] expected) =>
        Run().ShouldBeOfType<Accepted<TEvent>>().Events.ShouldBe(expected);       // record equality
    public void ThenRejected(string reason) =>
        Run().ShouldBeOfType<Rejected<TEvent>>().Reason.ShouldBe(reason);
    public void ThenNothing() =>
        Run().ShouldBeOfType<Accepted<TEvent>>().Events.ShouldBeEmpty();
}
```

Three more kinds of write-side test are worth having:

- **Property-based tests on `evolve`**: for any sequence of events the decider could have produced, `evolve` never throws, and invariants hold on the folded state (FsCheck or CsCheck can generate command sequences, run them through `decide`, and check). This is how you find the command sequence nobody thought of.
- **Snapshot equivalence**: for any stream, folding from a snapshot plus the tail equals folding from the beginning (Concept 50).
- **Concurrency tests** against a real store: two handlers deciding on the same stream in parallel, asserting that exactly one succeeds and the invariant holds (Concept 91) — and, for DCB, the overlapping-query test of Concept 36.

Marten's and Polecat's documentation also cover unit-testing handlers with a stub event stream (`StubEventStream<T>`) rather than mocking the store's interfaces — the same idea: test the decision, not the plumbing.

---

# Part E — Projections as the only read path

Module 23 built the projection toolkit — idempotency, ordering, checkpoints, poison events, rebuilds, lag measurement, read-your-writes. Everything there applies. Part E covers what changes when the write side *is* an event store: projections become mandatory, the source is always replayable, the global log has its own reading hazards, and a new distinction appears between consumers that may be replayed and consumers that must never be.

---

## Concept 55 — Projections become mandatory

In a state-stored system, a query can always fall back to the write tables (Module 23's level 1). In an event-sourced system there's nothing to fall back to: the event store answers "give me the events of stream X" and "give me all events after position P" efficiently — and essentially nothing else. "Orders over €500 placed this week by customers in Serbia" requires either a projection or a replay of everything.

So event sourcing is, by necessity, **CQRS at level 4** (Module 23, Concept 8):

| Question | Answered by |
|---|---|
| "What's the current state of order 42, to decide a command?" | Folding stream `order-42` (write model — Concept 44) |
| "Show order 42 on screen" | A projection (or, for simple cases, a live fold of the stream — Concept 56) |
| "List this customer's orders" | A projection |
| "Search, filter, sort, page, report" | Projections into the right stores (Module 23, Concept 47) |
| "What was order 42's state on 1 March?" | A fold up to a point in time (Concept 63) |

What's **better** than Module 23's level 3:

- **The source is always replayable.** Every projection can be rebuilt from the beginning, without the bootstrap-and-catch-up dance of Module 23 (Concept 43). A projection invented years later has the full history available.
- **The source is intent-revealing.** Projections consume business facts, not row diffs (Module 23, Concept 38).
- **There's no dual write** between state and events — the store is the outbox (Concept 65).

What's **harder**:

- **There's no escape hatch.** Every new question needs a projection (or a replay), and every projection needs operating.
- **Rebuilds are long** (Concept 90), because the source is *all* of history.
- **The global log has its own ordering hazards** (Concept 59).

---

## Concept 56 — Kinds of projection

Marten's taxonomy is the most complete in .NET, and it maps onto every store:

| Kind | What it builds | Example | Marten construct |
|---|---|---|---|
| **Single-stream projection** (aggregation) | One document per stream, folded from that stream's events | `OrderDetails` view of each order | `SingleStreamProjection<TDoc, TId>` or a self-aggregating type |
| **Multi-stream projection** | One document per *some other key*, folded from events of many streams | `CustomerOrderStats` per customer, from all order streams | `MultiStreamProjection<TDoc, TId>` with `Identity<TEvent>(e => e.CustomerId)` |
| **Event projection** | Arbitrary writes per event — rows, documents, deletes | An `OrderTimeline` row per event; a search-index document | `EventProjection` with per-event methods |
| **Flat-table projection** | Rows in a plain relational table, column-mapped | A reporting table queried by BI tools | Flat table projections |
| **Live aggregation** | Nothing persisted — fold on demand | "Show me order 42" computed per request from a short stream | `AggregateStreamAsync<T>` / live lifecycle |
| **Composite (chained) projections** | Projections consuming the output of other projections | Build a customer view from per-order views | Composite projections (Marten 8+) |
| **EF Core projections** | Rows written through an EF Core `DbContext` | Read models owned by an EF-based reporting module | EF Core projection support |

A multi-stream projection in Marten:

```csharp
public sealed class CustomerOrderStats
{
    public Guid Id { get; set; }                       // the customer ID
    public int OrdersPlaced { get; set; }
    public decimal LifetimeValue { get; set; }
    public DateTimeOffset? LastOrderAt { get; set; }
}

public sealed class CustomerOrderStatsProjection : MultiStreamProjection<CustomerOrderStats, Guid>
{
    public CustomerOrderStatsProjection()
    {
        Identity<OrderPlaced>(e => e.CustomerId);        // route each event to a customer's document
        Identity<OrderCancelled>(e => e.CustomerId);
    }

    public void Apply(CustomerOrderStats view, OrderPlaced e)
    {
        view.OrdersPlaced++;
        view.LifetimeValue += e.Total;
        view.LastOrderAt = e.PlacedAt;
    }

    public void Apply(CustomerOrderStats view, OrderCancelled e)
    {
        view.OrdersPlaced--;
        view.LifetimeValue -= e.RefundedAmount;
    }
}

// Registration
opts.Projections.Add<CustomerOrderStatsProjection>(ProjectionLifecycle.Async);
```

Note two design consequences for events (Concept 16): `OrderPlaced` must carry `CustomerId` for the projection to route it, and `OrderCancelled` must carry `CustomerId` and the refunded amount — the projection can't look them up. Event design and projection design are one activity.

(Why can the multi-stream projection above increment counters safely? Because the projection engine applies events in global order with checkpoints committed atomically with the documents — Module 23's Case 1 in Concept 41. The same code in a hand-rolled, at-least-once projector would double-count on replay; Concept 60 shows the fix.)

---

## Concept 57 — Inline, async, live

When a projection runs is the level-2-vs-level-3 decision of Module 23, with event-sourcing specifics:

| Lifecycle | When it runs | Consistency with the events | Costs and risks | Use for |
|---|---|---|---|---|
| **Inline** | In the same transaction as the append | Strong | Every append also writes the projection; projection failure fails the command; contention on shared rows; only in the same database | Write models (Concept 51); a few read models that must never lag, with low write rates |
| **Async** | After commit, from a subscription/daemon with checkpoints | Eventual (lag) | Must be idempotent, ordered, monitored, rebuildable | Most read models |
| **Live** | On request, by folding events | Strong | Read cost proportional to events folded; no persistence | Single-entity views of short streams; temporal queries (Concept 63); tests |

In Marten, the lifecycle is a registration choice (`ProjectionLifecycle.Inline` / `Async`, or `Live`), and the **async daemon** runs async projections — typically in `HotCold` mode, where one node in the cluster runs each projection shard and another takes over on failure (it uses database advisory locks for leadership). KurrentDB leaves projection hosting to you (a `BackgroundService` with a catch-up subscription, or a library such as Eventuous's subscriptions).

Two event-sourcing-specific points:

- **Inline projections can make eventual consistency disappear where it hurts most.** The "I saved it and it vanished" bug (Module 23, Concept 51) can be removed for the user's own views by projecting those inline — at the price of write latency. It's often the right call for the one or two screens a user sees immediately after a command.
- **Moving a projection from inline to async (or back) is a deployment, not a migration.** Because the events are the source, the new lifecycle's projection can be rebuilt from the log.

---

## Concept 58 — Subscriptions

A **subscription** delivers committed events to a consumer. Three shapes, with different guarantees:

| | **Catch-up subscription** | **Persistent (competing-consumer) subscription** | **Change feed** |
|---|---|---|---|
| Examples | KurrentDB catch-up; Marten async daemon; Eventuous subscriptions; polling a relational store by position | KurrentDB persistent subscriptions; a broker queue fed from the store | Cosmos DB change feed processor |
| Who stores the position | The consumer (its checkpoint) | The server (per consumer group) | Leases in a lease container |
| Ordering | Global order (or per stream, if subscribed to one) | Not guaranteed across consumers | Per logical partition |
| Scaling out | Partition the consumer by stream or key yourself; or one active instance per projection | Built-in load balancing across consumers | Built-in, by partition-key range |
| Retries and poison messages | Yours to build | Built-in retry counts, parking (dead-lettering), replay of parked messages | Retries the batch until it succeeds (Module 23, Concept 42) |
| Best for | Projections (order matters, rebuild from zero) | Reactors that need throughput and don't need order (Concept 53) | Cosmos-based stores |

The operational lifecycle of a catch-up subscription:

```
start at checkpoint (or 0 for a new projection)
  → catch-up phase: read history in large batches, fast
  → live phase: receive new events as they commit (push or short polling)
  → on failure: restart from last committed checkpoint (at-least-once)
```

Three rules:

1. **Projections use ordered, checkpointed subscriptions** — catch-up style — because they must apply events per entity in order (Module 23, Concept 39).
2. **Reactors may use competing consumers** — they need throughput and idempotency, not global order.
3. **Filter server-side** where the store supports it (event-type or stream-prefix filters on `$all` in KurrentDB; Marten projections declare the event types they consume), so subscriptions don't ship every event to every consumer.

---

## Concept 59 — Reading the global log safely

Concept 39 described the gap problem from the store's side. Here's what it means for the consumer — and why projection infrastructure is harder to hand-roll than it looks.

A catch-up subscriber over a relational store must answer, on every poll: **up to which position is it safe to read?** That position — the **high-water mark** — is the highest position below which every event is either committed and visible, or will never exist.

Strategies, from the consumer's perspective:

| Strategy | High-water mark | Failure mode |
|---|---|---|
| **Naïve**: read everything > checkpoint | The highest visible position | Silently skips events from transactions that commit late |
| **Contiguous only**: stop at the first gap | Last position before a gap | Stalls forever on permanent gaps (rolled-back transactions) |
| **Contiguous + staleness timeout** (the common library approach) | Last position before a gap, unless the gap has persisted longer than *T*, then skip it | Skips a long-running transaction that commits after *T* |
| **Database-assisted watermark** (`MIN_ACTIVE_ROWVERSION()` / snapshot `xmin`) | Positions from transactions guaranteed committed | Correct; ordering is by commit, so the checkpoint must use the commit-ordering column |
| **Single-writer store** (KurrentDB) | Whatever the store says is committed | Correct by construction |

Marten's async daemon uses high-water-mark detection with configurable staleness settings, and its documentation and issue tracker show how much care this takes in practice — the kind of edge case (a slow append transaction, a bulk import, a crashed session holding a sequence value) that only appears under production load. Two practical consequences for anyone running projections on a relational store:

- **Keep append transactions short.** Inline projections, remote calls or large batches inside the append transaction widen the window in which a position is allocated but invisible.
- **Monitor the high-water mark.** A high-water mark that stops advancing while events are being appended is an incident; so is a daemon that skipped a gap — both should be visible in metrics and logs (Concept 88).

The interview framing: *"The consumer's hard question is 'up to which position is it safe to read?' Naïve polling by position can skip late-committing events; waiting on every gap can stall forever. Libraries use gap detection with a timeout, databases can give you a true committed watermark, and single-writer stores avoid the problem. It's the main reason I'd rather use a mature projection engine than write my own."*

---

## Concept 60 — Idempotency by position

Module 23 (Concept 40) listed four techniques for making projections idempotent. In an event-sourced system, one of them becomes the natural default: **the event's position is its idempotency key.**

- **Per-stream read models** store the **stream version** of the last event applied. An event with a version ≤ the stored one is a duplicate; an event with version > stored + 1 indicates a gap (still in flight). This is Module 23's version guard, and the stream version is gapless (Concept 27), so the check is exact.
- **Cross-stream read models** (counters, aggregates by customer) store the **global position** of the last event applied — per projection — *in the same transaction* as the read-model change. On restart, everything at or below the checkpoint is skipped. That's why a projection engine that commits checkpoint and read model atomically can increment counters safely (Concept 56).
- **When the read store can't share a transaction with the checkpoint** (a search index, Redis, Cosmos DB), store the last applied position *on each document* and skip events at or below it; the document write and its position update are atomic even if the global checkpoint isn't.

```csharp
// Idempotent upsert into a SQL read model, guarded by stream version
const string sql = """
    UPDATE rm.OrderSummary
    SET Status = @Status, Total = @Total, LastVersion = @Version
    WHERE OrderId = @OrderId AND LastVersion = @Version - 1;          -- exactly the next event
    """;
// 0 rows: duplicate (LastVersion >= @Version) → skip; or gap (LastVersion < @Version - 1) → retry later
```

Two cautions:

- **Global positions in relational stores can have gaps** (Concept 27), so "next global position" checks don't work — use "greater than checkpoint," with the gap-safe reading of Concept 59.
- **Positions change if you copy events into a new store** (Concept 72). Projections that key idempotency by position must be rebuilt after such a migration — which they can be.

---

## Concept 61 — Rebuild = replay

In an event-sourced system a projection rebuild is the simplest possible operation in principle — reset the checkpoint to zero, clear the read model, replay — and one of the most expensive in practice (Concept 90). Module 23 (Concept 43) gave the blue/green procedure; here's what's specific to event stores.

**1. Rebuild alongside, then swap.** Never truncate the live read model and replay in place while it serves traffic. Build the new version into new tables/documents/index, let it catch up, verify, switch reads, keep the old one for rollback.

Marten supports this directly with **projection versioning**: bumping a projection's version (for example with a `[ProjectionVersion]` attribute or the `ProjectionVersion` property) makes Marten store the new version's documents in separately suffixed tables and track it with its own progress, so the old and new versions can run side by side while the new one rebuilds — enabling blue/green deployments of projection changes without downtime.

**2. Parallelize by partition.** Single-stream and many multi-stream projections are partitionable: events for different streams (or different target documents) can be processed in parallel. A replay of 500 million events at 20,000 events per second takes about 7 hours; split across 8 independent shards, well under an hour — if the read store can absorb the write rate (Concept 90).

**3. Batch aggressively during catch-up.** Replays are throughput-bound: large read batches, bulk writes to the read store, fewer checkpoint commits. Many engines switch to a "rebuild mode" with different batching than live processing.

**4. Replays must not trigger side effects.** Only projections are rebuilt. Reactors and process managers' outgoing commands are not replayed (Concept 64). If a projection *does* have side effects, that's a design bug — the rebuild will expose it expensively.

**5. Upcasting runs on every replayed event** (Concept 70). A slow upcaster that calls a service turns a 7-hour rebuild into a 70-hour one. Upcasters must be pure and cheap.

**6. Archived events must be reachable.** If old streams have been archived (Concept 84), a projection that needs full history must read the archive too — or accept that it's built from "active" history only. Decide per projection, before archiving.

**7. Measure rebuild time before you need it.** "How long does a full rebuild of our largest projection take?" is a number the team should know, rehearse and alert on regressions of — like a restore drill.

---

## Concept 62 — Projections that need other data

A projection building an "order list" wants the customer's name and the product names, but `OrderPlaced` carries only IDs. The tempting fix — look them up in another database from inside the projection — breaks three properties at once:

- **Determinism**: replaying the projection next year looks up next year's names.
- **Rebuild speed**: every replayed event now makes a remote call.
- **Availability**: the projection stalls whenever the other system is down.

Better options, in order of preference:

**1. Carry the data in the event** — at write time, when it's a fact about the decision (Concept 16). `OrderPlaced { CustomerId, CustomerDisplayName }` is right if the order should show the name *as it was when ordered* (an invoice).

**2. Build a local lookup from events.** The projection (or a sibling projection) subscribes to the events that define the lookup — `CustomerRegistered`, `CustomerRenamed` from the Customer context's integration events — and maintains its own table. The order projection joins to it at query time or copies from it at projection time (Module 23, Concept 45: denormalize vs join, and the "snapshot of the past or view of the present?" question).

**3. Enrich before applying, in a batch.** Marten supports **event enrichment** in projections: before applying a batch of events, look up reference data for the whole batch in one query (avoiding N+1) and attach it. It's still a lookup — so it's appropriate for data that is stable, or where "current value at projection time" is the desired semantics.

**4. Compose projections.** Build an intermediate projection (customers) and a dependent one (orders with customer names) that consumes both event streams and the intermediate state — Marten's composite projections support chaining.

The rule: **a projection's output should be a deterministic function of the events it consumed** — or, where it deliberately isn't (current reference data), the team should know that a rebuild can produce different results from the original run.

---

## Concept 63 — Temporal queries

The capability that most clearly distinguishes event sourcing from a state-stored model: **answering questions about the past, as of any point in time.**

**As-of state** — "what did policy P look like on 1 March?" — is a fold of the stream up to that point:

```csharp
// Fold only events recorded up to a moment (transaction time)
public async Task<PolicyState> AsOfAsync(string policyId, DateTimeOffset asOf, CancellationToken ct)
{
    var events = await store.ReadStreamAsync(policyId, fromVersion: 0, ct);
    return events.TakeWhile(e => e.RecordedAt <= asOf)
                 .Select(e => (PolicyEvent)e.Payload)
                 .Aggregate(PolicyState.Initial, PolicyEvolve.Evolve);
}

// Marten has this built in: fold up to a version or a timestamp
var asOfMarch = await session.Events.AggregateStreamAsync<Policy>(policyId, timestamp: marchFirst, token: ct);
var atV12     = await session.Events.AggregateStreamAsync<Policy>(policyId, version: 12, token: ct);
```

**Bi-temporal questions** need both times from Concept 18:

| Question | Filter by | Example |
|---|---|---|
| "What did we *know* on date T?" | Recorded (transaction) time ≤ T | "What coverage did our system show the customer on 1 March?" — for a complaint |
| "What was *true* on date T, as we know it now?" | Effective (valid) time ≤ T, using all events recorded up to now | "What was the coverage on the accident date, including the backdated correction recorded in April?" — for claim settlement |

A state-stored model can answer the first question only with temporal tables, and the second only with deliberate bi-temporal modeling; an event-sourced model can answer both if the events carry both times.

**Temporal read models**: for repeated temporal questions at scale ("monthly balances for every account for the last 7 years"), build projections that record state per period rather than folding on demand — for example, a projection that writes a row per account per month-end.

**Time travel for debugging and support** (Concept 92): load a stream as of a version, show the state a user saw when they reported a problem.

The interview framing: *"Because state is a fold, 'as of' is just a fold that stops early — by version or by time. If the events carry both when they happened and when we recorded them, we can answer both 'what did we know then' and 'what was true then', which is very hard to retrofit into a CRUD model."*

---

## Concept 64 — Projections, reactors, process managers

Three kinds of consumer read the same committed events. Treating them the same way is the most common architectural bug in event-sourced systems (Concept 101). The distinguishing question is **what happens when the consumer re-reads old events** — on a rebuild, after a checkpoint reset, or when first deployed.

| | **Projection** | **Reactor** (policy, automation) | **Process manager** (saga) |
|---|---|---|---|
| Purpose | Build a queryable view | Perform a side effect: email, HTTP call, broker publish | Coordinate a multi-step workflow |
| Output | Read-model writes | External effects (and possibly commands) | Its own events, then commands to others |
| Replay from the beginning? | **Yes — that's how it's rebuilt** | **Never** | Its *state* can be rebuilt from its own stream; its *outgoing commands* must not be re-sent |
| On first deployment, start from | Position 0 (build from all history) | "Now" (the current head) — or a deliberate backfill | Usually "now"; sometimes a deliberate start point |
| Idempotency | Version/position guards (Concept 60) | Idempotency keys for external calls | Own-stream concurrency + idempotent commands |
| Ordering needs | Per entity | Usually none | Per process instance |
| Subscription type | Catch-up, checkpointed | Persistent / competing consumers OK | Catch-up or persistent, keyed by process |

Concrete failure modes when they're mixed:

- A projection that sends a welcome email **re-sends every welcome email** on rebuild.
- A reactor registered as a projection **starts at position 0** when deployed, and processes years of history — charging, emailing, calling.
- A process manager implemented as a projection **re-issues every command** on replay; if commands aren't idempotent downstream, business actions repeat.

The structural fix: **separate registrations, separate subscriptions, separate checkpoints**, and code review that asks of every consumer "what happens if this runs over all of history?"

Marten makes the distinction explicit: projections (rebuildable) versus **event subscriptions** (for side effects and publishing), and, inside async projections, a `RaiseSideEffects()` hook that its documentation says is called only during *continuous* asynchronous execution — not during rebuilds, and not for inline projections unless you explicitly enable it. Use the mechanism your tool provides; don't simulate reactors with projections.

---

## Concept 65 — Publishing integration events from the store

Module 11 spent a whole part on the dual-write problem and the outbox. In an event-sourced system the problem mostly disappears — **the event store is the outbox**:

```
command handler ── append (atomic) ──▶ event store ── subscription ──▶ translate ──▶ broker (Service Bus / Event Hubs / Kafka)
```

- The append commits the facts; there's no second write that could fail independently.
- A publisher subscription reads committed events, translates internal events into integration events (Concept 19), publishes, and advances its checkpoint **after** the broker acknowledges.
- Delivery is **at-least-once** (a crash between publish and checkpoint re-publishes), so the integration event carries a stable **message ID derived from the source event** — for example the source event ID, or a hash of (stream, version) — so consumers de-duplicate (Module 11).
- **Ordering** per entity is preserved by publishing with the stream ID as the session ID (Service Bus) or partition key (Event Hubs, Kafka).

With Marten plus Wolverine, the alternative is to publish from the command handler through Wolverine's **transactional outbox**, which is committed in the same PostgreSQL transaction as the events — same guarantee, less machinery to write.

Two design decisions to make explicitly:

1. **Which events are public?** A deliberate, small subset (Concept 19). Publishing every internal event makes your storage format everyone's contract.
2. **Is the published stream replayable?** If downstream contexts need to rebuild from history, publish to a log with long retention (Event Hubs with capture, Kafka with long or tiered retention) — or provide a documented backfill. A Service Bus topic is not replayable after consumption.

---

## Concept 66 — Read-your-writes in event-sourced systems

Module 23 (Part E) gave six techniques for making eventual consistency invisible to the user who just acted. Event sourcing makes some of them unusually easy:

**1. The command's result is authoritative.** The handler knows the new version and, in the decider style, the resulting state (it just folded it). Returning the new version — and, where useful, the relevant resulting values — lets the client render immediately (Module 23, technique 1).

**2. Live aggregation for the entity the user just changed.** For a detail view of one short stream, fold it on request — strongly consistent, no projection lag. Marten's `FetchLatest<T>` returns the latest state of a single-stream projection regardless of whether it's inline, async or live, folding any events the async projection hasn't processed yet — designed for exactly this "show me what I just changed" case.

**3. Inline projections for the user's own views** (Concept 57).

**4. The consistency token.** The command returns the stream version (per-entity views) or the global position of its last event (cross-entity views); the query waits briefly until the projection has reached it (Module 23, Concept 53). In an event-sourced system the tokens come for free — every event has both.

**5. Push when the projection catches up** — the projector emits "applied position P" notifications that the client waits for (Module 23, technique 6).

The design most event-sourced systems converge on: **write models folded live; the user's immediate views via `FetchLatest`-style live-or-projected reads; everything else async, with a lag SLO and consistency tokens for the few cross-entity lists a user sees right after acting.**

---

# Part F — Evolving events over years

Overeem and colleagues found schema evolution to be the most complex challenge practitioners face, and Greg Young wrote a whole book on it. The problem is simple to state: stored events can't change, but the code that reads them must. Part F gives you the catalogue of changes, the tools for each — from "do nothing" to "copy the whole store" — and the discipline that keeps a five-year-old event store readable.

---

## Concept 67 — Why versioning is the hard part

In a state-stored system, a schema change is a migration: alter the table, transform the rows, deploy the new code. The old shape ceases to exist. In an event-sourced system, **the old shape exists forever**: every event ever stored in it will be read by every future version of every consumer — the write model, every projection, every rebuild.

Greg Young's framing (*Versioning in an Event Sourced System*) sets the rule for everything that follows:

> **A new version of an event must be convertible from the old version. If it isn't, it's not a new version of the event — it's a new event.**

That rule has a practical consequence: most "event changes" are really one of two things.

- **The shape changed, the meaning didn't.** A property was renamed, split, given a better type, or a new optional field was added. This is *versioning*, and the tools are tolerant reading (Concept 69), upcasting (Concept 70) and, rarely, copy-and-transform (Concept 72).
- **The meaning changed.** The business now distinguishes two things it used to treat as one, or a fact now carries a different implication. That's a **new event type**. The old events keep their old meaning — they describe what happened under the old rules — and new events describe what happens under the new ones. `evolve` and projections handle both.

Why it's hard in practice:

- **Every consumer reads every version.** Not just the aggregate — every projection, including ones written years later.
- **Deployments overlap.** During a rolling or blue/green deployment, old and new code run side by side and read each other's events (Concept 71).
- **Bugs write wrong events** that must be read forever (Concept 74).
- **Other contexts** may consume translated versions (Concept 19) — a second versioning surface with its own rules (Module 11).

And one reassurance: most changes are small, and the tools for them are simple. The discipline is to classify every change before making it (Concept 68).

---

## Concept 68 — The change catalogue

Every event change falls into one of these categories. Classify first; the tool follows.

| Change | Example | Compatible for readers of old events? | Tool |
|---|---|---|---|
| **Add an optional field** | `OrderPlaced` gains `Channel` (web/app/phone) | Yes — old events lack it; readers default it | Tolerant reader (Concept 69) |
| **Add a required field** | `OrderPlaced` gains `Currency`, which must never be null | Only with a sensible default for old events | Upcaster that fills a default (Concept 70) — or it's a new fact |
| **Rename a field** | `CustomerId` → `BuyerId` | No (naïvely) | Serializer alias, or upcaster |
| **Change a field's type** | `Amount: decimal` → `Amount: Money { Value, Currency }` | No | Upcaster |
| **Split a field** | `Name` → `FirstName`, `LastName` | No; the split may be lossy | Upcaster (best effort) — or keep both |
| **Rename the event type** | `OrderSubmitted` → `OrderPlaced` | No (naïvely) | Type-name alias (Concept 20) |
| **Split an event** | `OrderShipped` → `OrderPacked` + `OrderShipped` | No | Upcaster producing several events, or accept old events as-is |
| **Merge events** | `LineAdded` + `LineQuantityChanged` → `LineAdded { NewQuantity }` | No | Usually: keep reading both; new code emits the new one |
| **Remove a field** | `OrderPlaced.LegacyRef` no longer used | Yes — readers ignore it | Stop reading it; keep it in history |
| **Remove an event type** | `CartSaved` is no longer emitted | Old ones still exist | Keep a reader (or an upcaster to a newer type), or ignore in `evolve` |
| **Change the meaning** | "Cancelled" now includes "returned" | — | **New event type**; old events keep old meaning |

Two questions that decide how painful a change is:

1. **Can the new shape be computed from the old one alone?** If yes, it's an upcast. If it needs outside data (a currency the old event never recorded), you're either inventing data — acceptable only with a documented, business-approved default — or you have a new event.
2. **Who reads it?** Changes to internal events affect only your context (upcast freely). Changes to published integration events affect other teams (Module 11's compatibility rules and dual-publish windows apply).

---

## Concept 69 — Weak schema and tolerant readers

The cheapest versioning is none: design readers so that **additive changes don't require a new version**. Greg Young calls this the **weak schema** approach; Microsoft's 2026 guidance calls it **tolerant deserialization**.

The rules for a tolerant reader:

- **Unknown fields are ignored** (System.Text.Json's default — don't set `UnmappedMemberHandling.Disallow` for events).
- **Missing fields get defaults** — nullable types, or explicit defaults in the record.
- **Unknown event types are skipped** by `evolve` and projections (Concept 47) — never an exception.
- **Enums tolerate unknown values** — deserialize to a string or an `Unknown` member, not an exception.

```csharp
// v1 stored: { "total": 180.00, "placedAt": "2026-09-12T10:14:00Z" }
// v2 code adds Channel as optional; v1 events deserialize with Channel = null
public sealed record OrderPlaced(decimal Total, DateTimeOffset PlacedAt, SalesChannel? Channel = null);

// Consumers handle the absence explicitly — the absence is itself information: "recorded before we tracked channels"
var channel = e.Channel ?? SalesChannel.Unknown;
```

When weak schema is enough:

- Adding optional fields.
- Removing fields nobody needs any more.
- New event types that old code can safely ignore.

When it isn't:

- A field whose absence would be interpreted *wrongly* (a missing `Currency` silently treated as EUR). Then make the default explicit in an upcaster (Concept 70), where it's visible and tested — not scattered through consumers.
- Renames, type changes, splits — the reader can't guess.

A subtle trap: **.NET 9's `RespectRequiredConstructorParameters` and `RespectNullableAnnotations`** make System.Text.Json stricter — useful for API inputs, dangerous for stored events if enabled globally, because an old event lacking a newly required parameter will fail to deserialize. Keep the serializer options for events separate from API options (Concept 21).

---

## Concept 70 — Upcasting

An **upcaster** transforms an event from an older schema into the current one **as it's read**, before any consumer sees it. Consumers — `evolve`, projections, reactors — then only ever deal with the latest version. The stored events are untouched.

```
stored:   order-line-added v1  { sku, qty, price }
stored:   order-line-added v2  { sku, qty, unitPrice, currency }
current:  order-line-added v3  { sku, qty, unitPrice: Money, lineTotal: Money }

read path:  v1 ──upcast──▶ v2 ──upcast──▶ v3 ──▶ consumers
            v2 ──────────────upcast──────▶ v3 ──▶ consumers
            v3 ─────────────────────────────────▶ consumers
```

A minimal, store-agnostic upcasting pipeline working on raw JSON:

```csharp
public interface IUpcaster
{
    string EventType { get; }
    int FromVersion { get; }                         // transforms FromVersion → FromVersion + 1
    JsonObject Upcast(JsonObject payload, EventEnvelopeHeader header);
}

public sealed class OrderLineAddedV1ToV2 : IUpcaster
{
    public string EventType => "order-line-added";
    public int FromVersion => 1;

    public JsonObject Upcast(JsonObject p, EventEnvelopeHeader header)
    {
        // v1 had "price"; v2 calls it "unitPrice" and adds "currency".
        // Before v2, every order was in EUR — confirmed with Finance, documented in ADR-031.
        return new JsonObject
        {
            ["sku"] = p["sku"]?.DeepClone(),
            ["qty"] = p["qty"]?.DeepClone(),
            ["unitPrice"] = p["price"]?.DeepClone(),
            ["currency"] = "EUR",
        };
    }
}

public sealed class UpcastingDeserializer(IEnumerable<IUpcaster> upcasters, EventTypeRegistry types, JsonSerializerOptions options)
{
    private readonly ILookup<string, IUpcaster> _byType = upcasters.ToLookup(u => u.EventType);

    public object Deserialize(string eventType, int schemaVersion, string json, EventEnvelopeHeader header)
    {
        var node = JsonNode.Parse(json)!.AsObject();
        var chain = _byType[eventType].OrderBy(u => u.FromVersion).SkipWhile(u => u.FromVersion < schemaVersion);
        foreach (var up in chain) node = up.Upcast(node, header);        // v1→v2→v3…
        return node.Deserialize(types.TypeOf(eventType), options)!;       // always the current type
    }
}
```

In Marten, upcasting is built in — register transformations from old CLR types or raw JSON to the new type, keyed by event type name and schema version:

```csharp
// Adapted from Marten's versioning docs and Oskar Dudycz's "Event Versioning with Marten"
opts.Events
    .Upcast((V1.OrderLineAdded old) => new OrderLineAdded(old.Sku, old.Qty, Money.Eur(old.Price), Money.Eur(old.Price * old.Qty)))
    .Upcast(2, (V2.OrderLineAdded old) => new OrderLineAdded(old.Sku, old.Qty, new Money(old.UnitPrice, old.Currency),
                                                             new Money(old.UnitPrice * old.Qty, old.Currency)))
    .MapEventTypeWithSchemaVersion<OrderLineAdded>(3);
```

The rules for upcasters:

1. **Pure and cheap.** They run on every read of every old event — in every command that loads an old stream, and in every rebuild. Marten's documentation warns explicitly: an upcaster that calls external resources runs per event and invites N+1 problems and slow rebuilds. If an upcast needs outside data, it's probably not an upcast (Concept 67).
2. **Chained, one step at a time.** v1→v2, v2→v3 — each small and tested — rather than every old version jumping straight to current. Adding v4 then means one new upcaster, not N.
3. **Defaults are business decisions, documented.** "Old orders were EUR" must be *true*, confirmed by someone who knows — and recorded (an ADR, a comment, a test name).
4. **Tested with real stored samples** (Concept 75): a golden v1 JSON in the repository, upcast, deserialized, compared.
5. **Upcasters can emit several events** (splitting a coarse old event into finer new ones) — useful, but it changes stream versions as seen by readers, so do it only when readers don't depend on counting events.

Polecat, Marten's SQL Server sibling, was still implementing the shared JasperFx upcasting contract in September 2026 — a reminder to check upcasting support *before* choosing a store, not after the first breaking change.

---

## Concept 71 — Mixed-version deployments

Upcasting solves "new code reads old events." Rolling and blue/green deployments add the reverse: **old code reads new events**. During a rollout, version N and version N+1 run side by side on the same store. If N+1 appends a v3 event, N must survive reading it.

The strategies:

**1. Make old code tolerant of the future.** Old code that ignores unknown event types and unknown fields (Concept 69) survives most additive changes: it doesn't understand `OrderChannelRecorded`, so it skips it; it doesn't know `Channel`, so it ignores it. This is the most important reason `evolve` must never throw on unknown events (Concept 47).

**2. Deploy readers before writers** — the expand/contract (parallel change) pattern:

```
Release 1 ("expand"):   code can READ v1, v2 and v3, but still WRITES v2
Release 2 ("migrate"):  code WRITES v3 (all running instances can already read it)
Release 3 ("contract"): optional — stop special-casing v2 in writers; readers keep upcasting forever
```

Each release is backward- and forward-compatible with its neighbours, so a rolling deployment or a rollback never puts an event in front of code that can't read it.

**3. Downcasting** (rare). Transform new events into the old shape for old readers — for example, a subscription consumer pinned to an older contract. It's usually a sign that consumers should be tolerant instead, and it's lossy by nature.

**4. Version the projections, not just the events.** A new projection version that depends on v3 fields runs blue/green (Concept 61), alongside the old one, until everything has moved.

The rule for release planning: **never ship a change that writes an event shape some running code can't read.** With tolerant readers, that rule is easy to follow for additive changes; for anything else, it takes the three-release sequence above.

---

## Concept 72 — Copy-and-transform

Sometimes on-read transformation isn't enough: the stream boundaries were wrong (one giant stream should have been many), event granularity needs to change wholesale, a store migration is under way, or upcasting chains have become unmanageable. The heavy tool is **copy-and-transform** — read the old events, transform them, and write them into *new* streams or a *new* store.

Two scopes:

- **Stream transformation**: for each affected stream, write a new stream with transformed events, mark the old one as archived/superseded (for example with a `StreamMigrated { NewStreamId }` event), and redirect reads. Marten documents a "copy and transform stream" scenario.
- **Whole-store migration**: build a new event store from the old one, transforming events in flight — then cut over, blue/green, once the new store has caught up (Greg Young's "copy and replace"). This is also how you move between stores (EventStoreDB → Marten, or on-premises → cloud).

A whole-store migration, as a runbook:

```
1. Build the transformer: old event → zero or more new events (tested against golden samples).
2. Start a subscription on the OLD store from position 0, writing transformed events to the NEW store.
3. Let it catch up; keep it running so the new store follows live writes.
4. Rebuild all projections against the NEW store (positions differ — Concept 60).
5. Verify: stream counts, per-stream folded state equal (old fold + upcast vs new fold), sampled comparisons.
6. Cut over: stop writes briefly (or route writes to the new store), drain, switch readers and writers.
7. Keep the old store read-only for an agreed period, then archive it.
```

What copy-and-transform costs, and why it's the heavy option:

- **Event IDs, positions and possibly stream IDs change.** Anything that referenced them — idempotency records, external systems holding event IDs, integration events keyed by source position — must be mapped or accepted as broken.
- **The audit story changes.** The new store is a *derived* record; for regulated domains, keep the original, immutable store (archived, read-only) as the legal record, and document the transformation.
- **It takes as long as a full replay**, plus verification (Concept 90).

It's a legitimate, sometimes necessary tool — used deliberately, a few times in a system's life.

---

## Concept 73 — Rewriting history in place

The option every event-sourcing text lists last: `UPDATE` the stored events directly. Microsoft's guidance calls in-place migration a last resort because it breaks immutability and undermines the audit trail. The reasons:

- **The audit guarantee is gone.** If events can be edited, the history is no longer evidence of what happened — the property you adopted event sourcing for.
- **Consumers have already seen the old version.** Projections, published integration events, caches, analytics copies and backups all reflect the original. Rewriting the store makes them disagree with it — silently.
- **It's easy to get catastrophically wrong.** A mis-scoped `UPDATE` over an append-only log has no undo except a backup. Marten's documentation recounts exactly this kind of failure: under tenant-partitioned event storage, its masking and stream-compaction operations keyed updates and deletes on a sequence ID that wasn't unique across tenants, so masking or compacting one tenant's stream **silently overwrote or deleted other tenants' events** — damage that, once written, could only be repaired from backups. The bug was fixed, but it's the clearest possible illustration: **operations that rewrite history are the most dangerous code in an event-sourced system.**

When in-place rewriting is nonetheless justified:

- **Legal obligation**: erasure of personal data that was stored in events and can't be handled by the patterns of Part G (Concept 82 — masking).
- **Corrupted data**: bytes that can't be deserialized at all (a serializer bug wrote invalid JSON), where there's no meaningful "fact" to preserve.
- **Operational emergencies**, with a backup taken first, a script reviewed by two people, and an audit record of the rewrite itself.

If you do it: back up first; scope by primary key, never by a non-unique column; rebuild every affected projection afterwards; record *that* a rewrite happened (an event in an operations stream, a ticket, an ADR); and never do it as part of routine versioning.

---

## Concept 74 — Fixing wrong events

A bug shipped on Monday and emitted wrong events until Wednesday: `OrderLineAdded` events with the wrong `NewTotal`, say, or `DiscountApplied` events that should never have been emitted. Those events are now permanent facts about what the system *did*. How do you fix them?

**1. Compensate with new events** (the accounting answer, and the default). Append events that correct the state, with the reason recorded:

```
order-42:  … OrderLineAdded { newTotal: 1200.00 }   ← wrong: should have been 120.00 (bug #4711)
           OrderTotalCorrected { correctTotal: 120.00, reason: "Pricing bug #4711", correctedBy: "ops:script-2026-09-24" }
```

The history now shows the mistake *and* the correction — which is what an auditor wants. Downstream consumers see a new fact and react (a refund, a revised invoice). This is Microsoft's guidance too: undo by appending a compensating event, never by editing.

**2. Correction events designed for the purpose.** Some domains benefit from a generic, explicit correction mechanism — for example `EventRetracted { retractedEventId, reason }` — that `evolve` and projections honour by ignoring the retracted event. It's powerful and must be used sparingly, because every consumer has to implement it correctly.

**3. Upcast the bug away** (only when the wrong events are unambiguously wrong *and* nothing acted on them). An upcaster that fixes `NewTotal` on events in the affected window (identified by schema version, recorded time or a header set by the buggy release) makes all readers see correct data. It's invisible in history — so reserve it for data that was never externally visible (it didn't reach invoices, integration events or customers).

**4. Skip the events.** Marten supports **marking events as skipped**, so projections and aggregations ignore them — useful for events that should never have been appended (a duplicated batch from a faulty import). The events remain in storage for audit.

**5. Rewrite** — Concept 73, last resort.

How to choose:

| Did anything outside the system act on the wrong events? | Is the fix knowable from the event alone? | Use |
|---|---|---|
| Yes (invoices sent, integration events published, customers saw it) | — | **Compensating events** — the outside world needs to be told |
| No | Yes | Upcaster for the affected window, or compensate |
| No | No (needs investigation per case) | Compensate per stream, driven by a script that records its decisions |
| The events should never have existed (duplicates) | — | Skip, with an audit record |

The principle: **fix forward, visibly.** The system's history should explain itself, including its mistakes.

---

## Concept 75 — Guarding compatibility

The failure mode of event versioning is rarely a hard decision made badly; it's an innocent refactoring — renaming a property, changing a type, moving a class — that silently changes the stored format. The defence is to make the stored format a **tested contract**:

**1. Golden event samples.** For every event type and schema version, commit a sample of the stored JSON to the repository:

```
tests/EventSamples/order-line-added.v1.json
tests/EventSamples/order-line-added.v2.json
tests/EventSamples/order-placed.v1.json
```

**2. A test that every sample deserializes** — through the real serializer and upcaster pipeline — into the current type, with expected values:

```csharp
[Theory]
[MemberData(nameof(AllSamples))]                                 // one row per file in EventSamples/
public void Every_stored_version_still_deserializes(string file)
{
    var (type, version) = ParseName(file);                         // "order-line-added", 1
    var json = File.ReadAllText(file);

    var evt = deserializer.Deserialize(type, version, json, header: TestHeader);

    evt.ShouldNotBeNull();
    Verify(evt);                                                   // approval test of the upcast result (Verify, ApprovalTests)
}
```

**3. An approval test of what the current code writes.** Serialize a representative instance of each current event type and approve the JSON; a diff in a pull request means "you changed the stored format" — a reviewer must then confirm it's additive or that an upcaster was added.

**4. A registry test** (Concept 20): every event type registered, every name unique, no CLR type names in stored data.

**5. For published integration events**, contract tests or a schema registry with compatibility rules (Module 11).

These tests are cheap, fast and catch the whole class of "we broke deserialization of 2024's events" incidents in a pull request instead of in production.

---

## Concept 76 — Retiring event types

Event types accumulate. Code for a type emitted only in 2023 still has to exist in 2031 — or does it?

Options, from least to most effort:

1. **Keep the class, forever.** Simple, and the cost is some dead-looking code. Put retired event types in a clearly named folder (`Events/Retired/`) so readers know they're history, not behaviour.
2. **Upcast to a current type.** If `CartSaved` (retired) can be expressed as something current — or as nothing — register an upcaster that maps it (or drops it for consumers that don't care), and delete the class. The type *name* stays in the registry as a mapping; the CLR type goes.
3. **Map old type names to new classes.** When a class is renamed or moved, keep the stored name and register an alias (Concept 20); Marten's versioning documentation covers namespace and type-name migrations for exactly this.
4. **Copy-and-transform** old streams to eliminate the type from storage (Concept 72) — rarely worth it just to delete code.

Rules that never change:

- **Never delete the ability to read a type that still exists in storage** — unless it's been upcast away or migrated out, and a test proves every stored sample still loads (Concept 75).
- **Never reuse a retired type name for a new meaning.**
- **Archived streams count** (Concept 84): if old streams are archived rather than deleted, a rebuild or an audit may still need to read them.

---

## Concept 77 — Evolving everything else freely

The flip side of immutable events is the payoff this whole part has been building towards: **everything except the events can change freely.**

- **Aggregate state types and `evolve` logic** can be refactored at will — state is recomputed on every load. Renaming a field in `OrderState` is a code change, not a migration. (Invalidate snapshots when you do — Concept 50.)
- **Decision logic** changes with the business. New rules apply to new commands; old events remain facts about decisions made under old rules — exactly what an auditor wants.
- **Projections** can be rewritten, restructured, moved to different stores, split or merged — and rebuilt from history (Concept 61). A new report can include data from the first day of the system.
- **Stream boundaries for new data** can change (new streams for new lifecycles), though old streams stay as they are unless migrated.
- **Integration events** can be re-derived in new shapes from the same internal history.

This is the argument to make when someone objects that event sourcing is "rigid": *the facts are rigid; every interpretation of them is more flexible than in a state-stored system, because nothing but the facts is precious.* The discipline of Part F is the price of that flexibility.

---

## Concept 78 — The five proven methods

Overeem and colleagues' industry study identified five methods practitioners use for schema evolution — a useful vocabulary for interviews, mapped here to this part's concepts and to tools:

| Method (Overeem et al.) | What it is | This module | .NET support |
|---|---|---|---|
| **Versioned events** | New event types or explicit schema versions; consumers handle each | Concepts 67–68, 70 | Type registry + schema version in envelope; Marten `MapEventTypeWithSchemaVersion` |
| **Weak schema** | Tolerant mapping: ignore unknown, default missing | Concept 69 | System.Text.Json defaults |
| **Upcasting** | Transform old to new on read | Concept 70 | Marten upcasters; hand-rolled pipeline; Eventuous serializers |
| **In-place transformation** | Rewrite stored events | Concept 73 | Marten masking (for personal data); SQL scripts (last resort) |
| **Copy-and-transform** | Write transformed events to a new stream or store | Concept 72 | Subscriptions + a transformer; Marten "copy and transform stream" |

A defensible default policy for a team — the kind of thing worth putting in an ADR:

1. **Additive changes**: weak schema, no version bump.
2. **Shape changes with the same meaning**: bump the schema version and add one upcaster step, with a golden sample.
3. **Meaning changes**: a new event type; the old type stays readable.
4. **Boundary or granularity changes across history**: copy-and-transform, planned as a project.
5. **In-place rewriting**: only for legal erasure or corrupted data, with a documented procedure.

---

# Part G — Privacy, retention and security

An append-only log that keeps everything forever is an auditor's dream and a data protection officer's nightmare. Part G deals with the consequence of Concept 1 that bites hardest in regulated environments: personal data, retention periods and legal erasure in a store designed never to forget. The patterns are well established; the critical point is that they must be chosen **before the first event is stored** — retrofitting privacy into an event store means rewriting history.

---

## Concept 79 — Immutability meets the right to erasure

The GDPR (and laws modelled on it) gives people rights that collide with an immutable log:

| Obligation | GDPR | Collision with event sourcing |
|---|---|---|
| **Right to erasure** | Article 17 | Events containing a person's data can't be deleted without breaking streams |
| **Right to rectification** | Article 16 | Events can't be edited; a correction is a new event, but the wrong data remains in history |
| **Data minimisation** | Article 5(1)(c) | Events tend to carry "everything that might be useful later" |
| **Storage limitation** | Article 5(1)(e) | Event stores keep everything forever by default |
| **Data protection by design and by default** | Article 25 | The storage design must anticipate the above |

Three nuances a senior answer includes:

1. **Erasure is not absolute.** Article 17(3) lists exceptions — compliance with a legal obligation (financial records that must be retained for years), the establishment or defence of legal claims, and others. A bank's ledger events may have to be *kept* for regulatory periods. The question is per data category, and it's a legal question.
2. **"Deleted" has a legal meaning that engineers don't get to define.** Is encrypted data whose key has been destroyed "erased"? Is pseudonymised data "personal data"? The CJEU's *EDPS v SRB* judgment (4 September 2025) held that whether pseudonymised data is personal data depends on whether the party holding it can reasonably re-identify the person — a relative test — while the EDPB's draft Guidelines 01/2025 treat pseudonymised data as personal data for a controller that holds the means to reverse it. That's movement, not settlement. **Design the mechanism; let the data protection officer decide whether it satisfies the obligation.**
3. **The event store is rarely the only copy.** Projections, caches, search indexes, analytics exports, integration consumers, logs and backups all hold derived copies (Concept 83). Solving erasure in the store alone solves a fraction of the problem.

The design questions to ask on day one — ideally in the Event Modeling session (Concept 24):

- Which events carry personal data, and which fields?
- For each, what's the legal basis and the retention period?
- Does the business logic *need* the personal data in the event, or only a reference?
- What must happen when a person asks to be forgotten — and what must be *kept* (Article 17(3))?

---

## Concept 80 — Keep personal data out of events

The strongest pattern is the simplest: **don't put personal data in events at all.** Store it in a separate, mutable store keyed by a subject ID, and put only the ID (or a pseudonymous reference) in events. Mathias Verraes calls this **Forgettable Payloads**; Microsoft's 2026 guidance lists it first.

```
Events (immutable):        CustomerRegistered   { customerId: c-7, tier: Gold }
                           CustomerRelocated    { customerId: c-7, region: RS-Belgrade }     ← region, not address
Personal-data store:       c-7 → { name: "Ana Petrović", email: "…", street: "…" }          ← mutable, deletable
```

On an erasure request: delete (or anonymise) the personal-data record. The event history remains complete and structurally intact; it just refers to a subject whose details no longer exist.

Design rules:

- **Keep the business-relevant, non-identifying attributes in the event** — tier, region, age band, country — so projections and analytics keep working without the personal data. Choose them carefully: a combination of quasi-identifiers can re-identify a person.
- **Consumers fetch personal data on demand** (by ID, with authorization) rather than copying it; Verraes' advice is that consumers should never store the sensitive payload, only query it.
- **Publish a `PersonalDataErased { subjectId }` event** so consumers that did copy data can remove it (Concept 83).
- **Accept the limits.** Verraes points out that associations can identify a person even without their details — "Jane Doe exchanged messages with Bob and Charlie" — and forgettable payloads can't solve that. Neither can any other pattern; it's a modeling question.

The trade-off: every read that needs personal data now does a lookup — acceptable for display and for most processing, awkward for high-volume projections and for historical accuracy ("the name *as it was* on the invoice" must be kept somewhere, which may itself be a retained record under Article 17(3)).

---

## Concept 81 — Crypto-shredding

When personal data must be in events — because the event is legally the record, or because consumers need it inside the event — **crypto-shredding** is the standard alternative: encrypt each subject's personal fields with a key specific to that subject, and on erasure, **destroy the key**. The ciphertext stays in the store (and in every copy and backup) but can never be decrypted again.

```csharp
// Envelope encryption: per-subject data-encryption keys (DEKs), wrapped by a key-encryption key (KEK) in Key Vault
public sealed class SubjectKeyStore(KeysDbContext db, CryptographyClient kek, IMemoryCache cache)
{
    public async Task<byte[]> GetOrCreateAsync(string subjectId, CancellationToken ct)
    {
        if (cache.TryGetValue(subjectId, out byte[]? key)) return key!;

        var row = await db.SubjectKeys.FindAsync([subjectId], ct);
        if (row is null)
        {
            var dek = RandomNumberGenerator.GetBytes(32);                                   // AES-256
            var wrapped = await kek.WrapKeyAsync(KeyWrapAlgorithm.RsaOaep256, dek, ct);     // KEK never leaves Key Vault / HSM
            db.SubjectKeys.Add(new SubjectKey(subjectId, wrapped.EncryptedKey, kek.KeyId));
            await db.SaveChangesAsync(ct);
            key = dek;
        }
        else
        {
            if (row.ShreddedAt is not null) throw new SubjectShreddedException(subjectId);
            key = (await kek.UnwrapKeyAsync(KeyWrapAlgorithm.RsaOaep256, row.WrappedKey, ct)).Key;
        }
        cache.Set(subjectId, key, TimeSpan.FromMinutes(5));                                 // short TTL: shredding must take effect
        return key;
    }

    public async Task ShredAsync(string subjectId, CancellationToken ct)
    {
        await db.SubjectKeys.Where(k => k.SubjectId == subjectId)
            .ExecuteUpdateAsync(s => s.SetProperty(k => k.WrappedKey, Array.Empty<byte>())
                                      .SetProperty(k => k.ShreddedAt, DateTimeOffset.UtcNow), ct);
        cache.Remove(subjectId);                                                            // and on every node (Concept 83)
    }
}

public static class FieldCrypto
{
    public static EncryptedString Encrypt(byte[] key, string subjectId, string plaintext)
    {
        var nonce = RandomNumberGenerator.GetBytes(AesGcm.NonceByteSizes.MaxSize);        // 12 bytes
        var plain = Encoding.UTF8.GetBytes(plaintext);
        var cipher = new byte[plain.Length];
        var tag = new byte[16];
        using var aes = new AesGcm(key, tagSizeInBytes: 16);
        aes.Encrypt(nonce, plain, cipher, tag, associatedData: Encoding.UTF8.GetBytes(subjectId)); // binds ciphertext to subject
        return new EncryptedString(subjectId, Convert.ToBase64String(nonce), Convert.ToBase64String(cipher), Convert.ToBase64String(tag));
    }
}

// In the event, personal fields are encrypted values; everything else stays in clear text
public sealed record CustomerRegistered(Guid CustomerId, string Tier, EncryptedString Name, EncryptedString Email);
```

On read, the deserializer (or an upcaster-like decryption step) decrypts fields whose key exists and substitutes a placeholder — `"[erased]"` — for fields whose key has been shredded. `evolve` and projections must treat the placeholder as a normal value, never as an error (Concept 47).

**The traps**, which are where interview follow-ups go:

| Trap | What goes wrong | Mitigation |
|---|---|---|
| **The key is in the backups** | Wrapped DEKs stored in a database are in that database's backups; restore a backup and — as long as the KEK exists — the "shredded" key is back | Keep the key store separate, with short backup retention; treat erasure as complete when those backups age out; document it with the DPO. (Regulator guidance such as the UK ICO's generally accepts deletion from backups on the normal cycle if the data is put beyond use meanwhile — which a shredded key achieves only if the key isn't in those backups.) |
| **Plaintext copies downstream** | Projections, search indexes, caches and logs hold decrypted values | Projections store encrypted values or references, or subscribe to the erasure event and delete (Concept 83) |
| **Keys cached too long** | A shredded key remains usable in memory on some node | Short TTLs; broadcast an eviction on shred |
| **Per-subject keys in Key Vault itself** | Millions of keys; per-operation costs and throughput limits | Envelope encryption: few KEKs in Key Vault or Managed HSM, per-subject DEKs wrapped in your own store |
| **Performance** | Every read decrypts; every write encrypts | AES-GCM is fast; the cost is key lookups — cache (briefly) and batch |
| **Encryption ages** | Today's strong encryption may not be tomorrow's | Use standard AEAD (AES-256-GCM); keep crypto agility (algorithm identifier in the envelope) |
| **Indexes and joins on encrypted fields** | You can't query by an encrypted email | Keep searchable, non-identifying attributes in clear text; use keyed hashes (HMAC with a per-subject or rotating key) for lookups if necessary |
| **Legal sufficiency** | Encrypted data is still personal data while the key exists; whether destroying the key amounts to erasure is a legal question (Concept 79) | Get a documented decision from the DPO |

Verraes' own summary of the pattern is the right frame: crypto-shredding is only as good as your encryption and your key management.

---

## Concept 82 — Masking, tombstones and deletion

The in-place options, for when neither of the previous patterns was designed in — or when the law requires the data itself to be removed:

**1. Masking (rewriting specific fields).** Marten supports this directly: register **masking rules** per event type at configuration time, then apply them on demand to a stream, a filtered set of events or a tenant:

```csharp
// Configuration: how to mask each event type's personal fields
opts.Events.AddMaskingRuleForProtectedInformation<CustomerRegistered>(e => { e.Name = "****"; e.Email = "****"; });
opts.Events.AddMaskingRuleForProtectedInformation<IContainsPersonalData>(e => e.MaskPersonalData());

// On an erasure request: rewrite the events of this customer's stream, and record that it happened
await store.Advanced.ApplyEventDataMasking(x =>
{
    x.IncludeStream(customerStreamId);
    x.AddHeader("masked", DateTimeOffset.UtcNow);          // requires header tracking to be enabled
}, ct);
```

Marten's documentation notes that masking does **not** touch archived events or projected data — you must rebuild projections afterwards — and its own history (Concept 73) shows why rewriting operations deserve extra testing.

**2. Deleting whole streams.** When all of a subject's events live in their own streams (a customer profile context), deleting those streams is a clean erasure — if nothing else depends on them. KurrentDB offers **soft delete** (hidden, recreatable, removed physically on scavenge) and **hard delete** (a tombstone; the stream can never be recreated); Marten can delete or archive streams. The event IDs referenced elsewhere will then dangle — design consumers to tolerate that.

**3. Truncation by retention.** KurrentDB's `$maxAge`/`$maxCount` stream metadata plus scavenging removes old events physically — useful for data with a legal retention *maximum* (logs, telemetry-like streams).

The trade-offs: masking and deletion **break the audit property** for the affected data (which may be exactly what the law requires), **must be propagated** to every copy (Concept 83), and are **the riskiest operations** in the system (Concept 73). If you'll need them, test them as seriously as the write path.

---

## Concept 83 — Erasure must propagate

Whatever mechanism erases data in the store, personal data has almost certainly been copied elsewhere. An erasure design is a **propagation** design:

| Copy | How erasure reaches it |
|---|---|
| Projections / read models | Subscribe to `PersonalDataErased { subjectId }` and delete/anonymise rows; or rebuild after masking |
| Caches (HybridCache, Redis, output cache) | Evict by tag (`subject:{id}`); short TTLs for personal data |
| Search indexes | Delete or update documents on the erasure event |
| Integration consumers (other contexts, partners) | Publish an erasure integration event; contractually require handling |
| Analytics / lakehouse exports | Pseudonymise at export; a scheduled erasure job driven by erasure events; or crypto-shred exported fields with the same keys |
| Logs and traces | Don't log personal data in the first place; short retention; Wolverine's documentation, for example, warns about audited message members ending up in logs and telemetry |
| Backups | Age out on the normal cycle; data put "beyond use" meanwhile (Concept 81) |
| In-flight messages, dead-letter queues | Short retention; include DLQs in the erasure runbook |

The canonical flow:

```
ErasureRequested (command, after identity and legal checks)
  └─▶ personal-data store: delete record   /  key store: shred key   /  event store: mask (if required)
  └─▶ append PersonalDataErased { subjectId, requestedAt, basis } to the privacy context's stream   ← the audit record
        └─▶ projections, caches, search, analytics, integration publisher react
              └─▶ each confirms completion (a small erasure-tracking projection shows what's done)
```

Note the irony worth pointing out in an interview: **the record that a person's data was erased is itself an event you keep** — containing only the subject ID and the legal basis — because demonstrating compliance is also an obligation.

---

## Concept 84 — Retention, archiving and compaction

Even without personal data, an ever-growing event store has costs: storage, backup size, index size, rebuild time. Four tools, often combined:

**1. Model lifecycles so streams close** (Concept 31). A closed stream is a natural archiving unit — nothing will append to it again.

**2. Archive closed streams.**
- **Marten** can **archive streams** (`session.Events.ArchiveStream(id)`): archived events are excluded from normal queries and projections by default, and with archived-stream partitioning enabled they live in a separate PostgreSQL partition, keeping the hot table small.
- **KurrentDB** (25.0+, licensed) uploads completed chunk files to object storage (S3, Azure Blob Storage, GCP) and removes them from nodes' local volumes according to a retention policy; reads fall through to the archive transparently.
- **Cosmos DB**: move closed streams' items to cheaper storage (the analytical store, Azure Blob Storage) and delete them from the transactional container — keeping in mind the 20 GB logical-partition limit makes this more urgent (Concept 42).
- **Hand-rolled**: move closed streams to an archive table or database, with a pointer event in the hot store.

**3. Compact long streams.** Marten 8+ offers **stream compacting**: `CompactStreamAsync<T>` replaces events up to a version or timestamp with a single `Compacted<T>` event holding a snapshot, optionally handing the removed events to an archiver you implement (cold storage, another table). Marten's documentation is frank that this is what you do when you "failed to be omniscient in your event stream modeling" — a remedy for streams that should have been closed by a lifecycle. After compaction, single-stream projections restart from the snapshot, and you can no longer replay that stream from before the compaction point.

```csharp
await session.Events.CompactStreamAsync<Equipment>(equipmentId, x =>
{
    x.Timestamp = DateTimeOffset.UtcNow.AddDays(-90);   // keep the last 90 days of detail
    x.Archiver = coldStorageArchiver;                    // your IEventsArchiver<IDocumentOperations> implementation
});
```

**4. Decide per projection whether it needs archived history.** A projection rebuilt from active streams only will differ from one rebuilt from everything. That's often fine — "orders of the last two years" — but it must be a deliberate choice (Concept 61).

The retention policy — how long hot, how long archived, when deleted — is a business and legal decision per stream category. Write it down (an ADR), and implement it as a scheduled, monitored process, not a one-off script.

---

## Concept 85 — Backup and restore

In an event-sourced system, **the event store is the only thing that must be backed up for correctness.** Read models, snapshots and caches can be rebuilt; backing them up shortens recovery time but isn't needed to recover the truth. That changes the recovery plan:

```
1. Restore the event store to the chosen point (PITR: Azure Database for PostgreSQL flexible server or Azure SQL,
   KurrentDB backups/restore, Cosmos DB continuous backup).
2. Reset projection checkpoints to the restored position (or rebuild) — read models may be AHEAD of the restored store.
3. Restart subscriptions.
```

Step 2 is the subtle one. A read model backed up (or still running) *after* the restore point reflects events that no longer exist in the restored store. It must be rewound or rebuilt, or it will show facts the truth has forgotten.

And the subtler problem, which is a strong senior point: **the outside world may have seen events you no longer have.** If the store is restored to 10:00 but the integration publisher had already published events up to 10:15, other contexts, partners and customers acted on facts that are now missing from your history. The recovery runbook must include **reconciliation**: compare the publisher's record (or the broker's retained messages) with the restored store, and re-append or compensate for the difference. Point-in-time restore of an event store is not a rollback of reality.

Two practical rules:

- **Back up the key store** (Concept 81) on a schedule deliberately *shorter-lived* than the event store's — and never lose it accidentally, or every encrypted field becomes unreadable.
- **Rehearse**: restore to a scratch environment, rebuild projections, and time it (Concept 90). The recovery time objective is dominated by rebuild time, not restore time.

---

## Concept 86 — Securing the log

An event store is a high-value target: complete history, often with personal and financial data, and the system's legal record. Five layers:

**1. Append-only, enforced by the database, not by convention.** The application's database principal gets `INSERT` and `SELECT` on event tables — no `UPDATE`, no `DELETE`. Operations that must rewrite (masking, compaction, erasure) run under a separate, audited principal, through a break-glass procedure.

```sql
-- SQL Server: the application can append and read, never rewrite
CREATE ROLE es_app;
GRANT SELECT, INSERT ON es.Events  TO es_app;
GRANT SELECT, INSERT, UPDATE ON es.Streams TO es_app;     -- the version row (Concept 38)
DENY UPDATE, DELETE ON es.Events TO es_app;
```

**2. Tamper evidence.** For regulated domains, prove the log hasn't been altered:
- **SQL Server / Azure SQL append-only ledger tables** (`WITH (LEDGER = ON (APPEND_ONLY = ON))`) reject updates and deletes at the engine level and maintain a cryptographic hash chain whose database digests can be stored externally (for example in immutable Azure Blob Storage) and verified later. Note the tension: an append-only ledger table makes in-place masking impossible — which is a *feature* if personal data is kept out or crypto-shredded (Concepts 80–81), and a blocker if it isn't.
- **Application-level hash chaining**: each event's metadata includes a hash of the previous event in the stream (or of the previous global position), and a periodic job anchors the latest hash externally.

**3. Encryption at rest and in transit.** TDE and storage encryption are defaults on Azure SQL and Azure Database for PostgreSQL; for KurrentDB, encryption at rest is an Enterprise-licensed feature. TLS everywhere (KurrentDB 26.2 adds an option to run authenticated without TLS for development — keep it out of production). Field-level encryption for personal data (Concept 81).

**4. Least privilege per consumer.** Projections and reactors read; only command handlers append; admin operations are separate. KurrentDB supports stream ACLs and (from 26.2 in the free edition) policy-based stream authorization; in relational stores, use roles and, if necessary, row-level security by tenant (Concept 87). Managed identities for Azure resources (Module 29).

**5. Audit the auditors.** Record administrative operations — masking, compaction, archiving, restores — as events in an operations stream, so the history of the history is itself kept.

---

## Concept 87 — Multi-tenancy

A multi-tenant event-sourced system must guarantee that one tenant's events are never read, projected, masked or deleted on behalf of another. The models:

| Model | How | Isolation | Operations |
|---|---|---|---|
| **Conjoined** (shared tables) | A tenant ID on every stream and event; every query filtered | Logical; depends on every code path filtering | One database; simplest to run; noisy neighbours |
| **Conjoined + partitioned** | As above, with the events table partitioned by tenant | Logical, physically separated partitions | Per-tenant archiving and maintenance become easier |
| **Database (or schema) per tenant** | Each tenant's store is separate | Strong | Many databases to migrate, monitor and back up |
| **Store instance per tenant** (KurrentDB per tenant, container per tenant in Cosmos) | Fully separate | Strongest | Highest cost and operational load |

Marten supports conjoined tenancy, database-per-tenant (including dynamically added tenant databases) and tenant-partitioned event tables. Cosmos DB can use **hierarchical partition keys** (tenant, then stream) to keep a tenant's streams together while allowing a large tenant to exceed a single logical partition's limits.

The rules, whichever model:

- **The tenant is part of the stream's identity** — `tenant-a/order-42` and `tenant-b/order-42` are different streams — and the store must enforce it on append (Marten rejects appends whose tenant doesn't match the existing stream's tenant).
- **The tenant is set by infrastructure** from the authenticated request, never by domain code or command payloads (Concept 22).
- **Administrative operations are the dangerous ones.** Marten's tenant-partitioning bug (Concept 73) — masking or compaction keyed on a sequence number that wasn't unique across tenants, silently overwriting other tenants' events — is the canonical example: reads were carefully tenant-scoped, a maintenance operation wasn't. Test every rewrite, archive and delete path with at least two tenants holding colliding identifiers.
- **Projections and caches carry the tenant too** — a read model keyed only by order ID leaks across tenants the day two tenants share an ID.

---

# Part H — Operating, testing and migrating

An event-sourced system has more moving parts than a CRUD system — subscriptions, daemons, checkpoints, archives, rebuilds — and its failure modes are quieter: a projection that silently skipped an event, a stream that grew until every command on it slowed down, an upcaster that turned a rebuild into a weekend. Part H is the arithmetic and the discipline that keep it running, and the two migrations every architect should be able to plan: into event sourcing, and out of it.

---

## Concept 88 — Observability for event-sourced systems

Module 23 (Concept 50) made projection lag the read side's primary health metric. An event-sourced system adds write-side and store-level signals. The dashboard:

| Signal | Why | Alert when |
|---|---|---|
| **Append latency** (p50/p99) | The write path's core cost | p99 rises — often a hot stream or a slow inline projection |
| **Concurrency conflicts** per stream category, and retries per command | Hot streams (Concept 32); bad boundaries | Conflict rate above a few percent, or retry exhaustion |
| **Stream length distribution** (p50, p99, max per category) | Load cost (Concept 89); lifecycle modeling failures (Concept 31) | p99 or max growing week over week in a category that should close |
| **Events loaded per command** | Direct measure of rehydration cost | Rising — snapshot or lifecycle needed |
| **Projection lag** (time and position, per projection) | Read-side freshness (Module 23) | Level above SLO, or monotonic growth (stuck) |
| **High-water mark progress** (relational stores) | Gap handling (Concept 59) | Not advancing while appends continue; skipped gaps logged |
| **Dead-lettered / parked events**, per consumer | Poison events | Any, for projections; above threshold for reactors |
| **Upcast counts by type/version** | Which old shapes are still being read, and how often | Informational — shows when an upcaster can be retired by migration |
| **Snapshot hit rate and staleness** | Whether snapshots help | Low hit rate (wasted writes) or high "events after snapshot" |
| **Store size and growth**, hot vs archived | Capacity (Concept 90) | Growth beyond plan; archive jobs failing |
| **Subscription/daemon leadership** | Async projections need an active owner | No leader, or frequent failovers |

Tracing: tag every append span with stream category, stream ID (if not personal data), event types and resulting version; start consumer spans as **links** to the originating trace (Concept 22). Marten, Wolverine, Eventuous and the KurrentDB client all emit OpenTelemetry signals; Marten's async daemon exposes health checks (including a "same lag for too long" detection) that belong on the dashboard.

---

## Concept 89 — Rehydration and snapshot arithmetic

"When do we need snapshots?" has a numerical answer. Model the cost of loading a stream of **n** events:

```
T_load(n) ≈ T_roundtrip + n × (T_transfer + T_deserialize + T_upcast + T_evolve)
```

Illustrative orders of magnitude for small JSON events (a few hundred bytes) from a nearby database — measure your own:

| Component | Rough cost |
|---|---|
| Database round trip (same region) | ~0.5–2 ms |
| Transfer + deserialize one small event (System.Text.Json, source-generated) | ~a few µs |
| Upcast (pure JSON transformation) | ~a few µs, only for old versions |
| `evolve` (in-memory record `with`) | well under a µs to a few µs |

So per-event cost is on the order of **5–20 µs**, and:

| Events in stream | Load time ≈ round trip + n × ~10 µs | Verdict |
|---|---|---|
| 20 | ~1–2 ms | Irrelevant |
| 200 | ~3–4 ms | Fine |
| 2,000 | ~20–25 ms | Noticeable; widens the decision window (conflicts rise, Concept 32) |
| 20,000 | ~200 ms+ | Needs a lifecycle redesign (or, as a stopgap, snapshots/compaction) |

A snapshot turns `n` into `k` (events since the snapshot) at the cost of reading and deserializing one snapshot and occasionally writing one. The **break-even**: snapshots pay when streams regularly exceed a few hundred to a few thousand events **and** load latency or conflict rates matter. Below that, they add a versioned cache to maintain for no benefit — which is why the standard advice is to measure first (Concept 50).

Two second-order effects worth mentioning:

- **Load time is part of the decision window `d`** in the conflict formula `1 − e^(−λd)`. A stream that takes 200 ms to load has a ceiling of about 5 decisions per second — whatever the hardware.
- **Cosmos DB charges per read.** Loading 2,000 events is a multi-page query costing RUs proportional to the data read; snapshots (or short streams) are a cost optimization there, not just a latency one.

---

## Concept 90 — Rebuild and storage arithmetic

Two numbers every event-sourced system's owners should know.

**Rebuild time.**

```
T_rebuild ≈ N_events / throughput_per_shard / shards      (+ verification + swap)
```

| Store size | Throughput | Shards | Time |
|---|---|---|---|
| 50 M events | 10,000 events/s | 1 | ~1.4 hours |
| 500 M events | 20,000 events/s | 1 | ~7 hours |
| 500 M events | 20,000 events/s per shard | 8 | ~52 minutes (if the read store sustains 160k writes/s) |
| 2 B events | 20,000 events/s per shard | 8 | ~3.5 hours |

Throughput depends mostly on the **read store's write rate** and on **batching** — not on reading events, which is fast. Upcasting cost multiplies by the number of old events; a single slow upcaster can dominate. The practical implications: know the number, rehearse it, parallelize projections that can be partitioned, and make "rebuild duration" a tracked metric that fails a release if it regresses badly.

**Storage growth.**

```
Storage/year ≈ commands/day × events/command × bytes/event × 365 × overhead
```

Worked example: a booking context handling 200,000 commands a day, 1.5 events per command, ~600 bytes per event (JSON payload plus metadata), and an overhead factor of ~2.5 for indexes, row headers and the streams table:

```
200,000 × 1.5 × 600 B × 365 × 2.5 ≈ 164 GB/year  (≈ 110 M events/year)
```

Useful comparisons for the conversation: that's modest for PostgreSQL or Azure SQL; it's about 8 logical partitions' worth of Cosmos DB's 20 GB limit *if* it were one stream (it isn't — which is why streams must be short); and the rebuild time for five years of it (~550 M events) is several hours per projection unless sharded — which is the real constraint, and the argument for archiving closed streams from projections that don't need them (Concept 84).

---

## Concept 91 — The testing pyramid for event sourcing

Event sourcing changes what's cheap to test (business logic, via deciders) and adds things that must be tested (upcasters, projections, concurrency). The layers:

| Layer | What | How | Concept |
|---|---|---|---|
| **Decider specs** | Given events, when command, then events / rejection | Pure unit tests, hundreds of them, milliseconds each | 54 |
| **Evolve properties** | `evolve` never throws; invariants hold for any sequence `decide` can produce | Property-based (FsCheck/CsCheck) | 47, 54 |
| **Projection specs** | Given events, then read-model rows | In-memory or against a real database; include duplicates and re-ordering | Module 23, Concept 89 |
| **Golden event samples** | Every stored type/version still deserializes and upcasts correctly | Approval tests over committed JSON files | 75 |
| **Registry tests** | Every event type registered, names unique | Reflection-based unit test | 20 |
| **Store integration** | Append, read, conflicts, idempotent retry, subscriptions from position | Testcontainers (PostgreSQL, SQL Server, KurrentDB images); Fisher (SQLite) for fast Critter Stack tests | 26–29 |
| **Concurrency** | Two deciders on one stream: exactly one wins, invariant holds; for DCB, overlapping queries in parallel thousands of times | Parallel integration tests against the real store | 28, 36 |
| **Rebuild** | Rebuild a projection from a sample store; result equals the live-built one | Periodic job, or a pre-release pipeline step | 61 |
| **End to end** | HTTP → command → events → projection → query | `WebApplicationFactory` / Alba, with lag handled explicitly | Module 23, Concept 71 |

A concurrency test worth having for every write path that matters:

```csharp
[Fact]
public async Task Two_concurrent_withdrawals_cannot_overdraw()
{
    var accountId = await Given(new AccountOpened(...), new FundsDeposited(Amount: 100m));

    // Both decide on balance = 100; only one may append
    var results = await Task.WhenAll(
        handler.HandleAsync(accountId, new Withdraw(80m), ct).AsTask(),
        handler.HandleAsync(accountId, new Withdraw(80m), ct).AsTask());

    results.Count(r => r.IsAccepted).ShouldBe(1);
    (await LoadBalance(accountId)).ShouldBe(20m);                     // never -60
}
```

Run it in a loop (hundreds of iterations) in CI — race conditions are probabilistic. For DCB stores, the equivalent test with overlapping tag queries is how you'd have caught the pre-9.4 Marten bug (Concept 36).

---

## Concept 92 — Debugging with history

One of event sourcing's most practical benefits, and a good story for behavioural rounds: **the exact history that produced a bug is in the store.**

Techniques:

- **Load the stream as of any version** (Concept 63) and inspect the state at each step. A small internal tool — "show stream X, event by event, with the folded state after each" — pays for itself the first week.
- **Replay the stream through the current code** in a test: copy the events of the failing stream (after anonymising personal data — Part G) into a golden test case, reproduce the bug, fix it, and keep the test.
- **Diff two folds**: fold with the old `evolve` and the new `evolve`, compare — the safest way to check that a refactoring of state logic changes nothing it shouldn't.
- **Follow causation** (Concept 22): from a wrong event, walk causation IDs back to the command, the process manager step, the integration event that triggered it.
- **Answer "how did this happen?" questions from support** without developer archaeology: the stream reads as a story (Concept 14).

Two cautions: **production event data is production data** — copying streams to developer machines is a data-protection decision, and anonymisation tooling belongs in the plan; and **replaying through code with side effects** must be impossible by construction (Concept 64).

---

## Concept 93 — Migrating to event sourcing

Most event-sourced contexts are introduced into systems that already exist. The playbook:

**1. Pick one bounded context** where history has value (Concept 11), behind a clean boundary — ideally already a module (Module 21) that owns its data.

**2. Decide what to do with existing state.** Three options:

| Option | How | Trade-off |
|---|---|---|
| **Import current state as a starting event** | For each entity, append one `OrderImported { …current state… }` (or `AccountOpened { openingBalance }`) event | Simple; history before the import is lost — say so explicitly in the event name and docs |
| **Reconstruct history from audit data** | Generate the best-possible past events from audit tables, temporal tables or logs | Richer history; fabricated precision — mark them as reconstructed (a header, distinct event types) |
| **Start fresh for new entities only** | Old entities stay in the old model until they close | No migration; two models to run for a while (fine for short lifecycles like orders) |

Verraes' **Migration Events in a Ghost Context** pattern names a useful refinement: produce the migration events from a separate, temporary "ghost" context that translates the legacy model into the new language, so the new context's history is clean and the translation is explicit.

**3. Strangle the old write path** (Module 32 will cover the strangler fig in depth): route commands for migrated entities to the new context; keep the legacy tables as a read model fed by a projection until consumers move off them — so reports and integrations that read the old tables keep working during the transition.

**4. Run in parallel before cutting over** where the risk is high: shadow-write events from the legacy system (or shadow-decide in the new one) and compare the folded state with the legacy state daily until they agree.

**5. Keep the migration reversible** until you're confident (Concept 94).

---

## Concept 94 — Migrating away

Microsoft's guidance now says it plainly: event sourcing is costly to migrate to *or from*. An architect should know what backing out looks like — both because it happens, and because reversibility is a decision criterion.

The mechanics are straightforward in principle, because **a state-stored model is just a persisted projection**:

1. Build a projection that writes the current state of each entity into ordinary tables — often it already exists as a read model.
2. Switch the write side to load and save those tables (the decider style makes this a storage swap, not a domain rewrite — Concept 46).
3. Keep the event store read-only as a historical archive — or migrate its history into temporal or audit tables if history is still needed.

What makes it expensive in practice:

- **Consumers that depend on the events** — projections, reactors, other contexts — need a new source (an outbox publishing the same integration events, Module 11).
- **Temporal queries and rebuilds** stop working for new data unless you keep temporal tables.
- **Code written in the event-sourced style** (OO aggregates with `Apply` methods, event-driven process managers) must be adapted.

The design lesson, which is the reason this concept exists: **deciders, a clean event store boundary and translated integration events keep the way out open.** A system whose domain logic is pure decide/evolve functions and whose consumers depend on integration events rather than the store can leave event sourcing in weeks. One whose `Apply` methods validate, whose projections send emails and whose partners read the store directly cannot.

---

## Concept 95 — The human cost

Overeem's study found the steep learning curve to be one of the five main challenges, and every experience report agrees. What it consists of:

- **Thinking in facts, not rows.** Designing events for intent, resisting property sourcing and state obsession (Concept 14).
- **Eventual consistency everywhere on the read side**, and designing UX around it (Module 23, Part E).
- **Versioning discipline**: every change to an event type classified and tested (Part F).
- **Operational literacy**: subscriptions, checkpoints, rebuilds, gap handling, archives.
- **Debugging differently**: from "look at the row" to "read the stream and the projections' positions."

How to pay it down:

- **Start with one context and one experienced person** — someone who has shipped an event-sourced system before, as reviewer and teacher.
- **Adopt a mature library** (Marten/Polecat, KurrentDB with Eventuous) rather than hand-rolling the store (Concept 99).
- **Make the conventions executable**: templates for deciders and projections, registry and golden-sample tests, architecture tests that forbid I/O in `evolve` and side effects in projections.
- **Use Event Modeling** so domain experts and developers share the event vocabulary from the start (Concept 24).
- **Write the runbooks** — rebuild, restore, erasure, poison event — before the first production incident needs them.
- **Budget for onboarding.** New team members need days, not hours, to become productive in an event-sourced context; hiring for it is harder than hiring for EF Core.

The honest summary for a stakeholder: *"The technical patterns are well known; the cost is in people. It's worth it where the history has real business value, and it's a bad trade where it doesn't."*

---

# Part I — The architect's view

Parts A–H covered the pattern and its machinery. Part I is about the decisions an architect owns: where event sourcing belongs in a larger system, how contexts interact when some are event-sourced, which store to choose and whether to build one, what the licences mean, what goes wrong, what a reviewer checks — and how to talk about all of it in the rounds where it comes up.

---

## Concept 96 — Scope: per context, never the whole system

The most repeated warning in the event-sourcing literature, from its originator onward: **event sourcing is not a top-level architecture.** Greg Young said it in *A Decade of DDD, CQRS, Event Sourcing* (2016); Dennis Doomen said it at DDD Europe 2020; Microsoft's 2026 guidance says to apply it selectively — a payment ledger or order pipeline — and keep CRUD for user profiles and configuration.

Why a whole system built on event sourcing goes wrong:

- **Most of any system is CRUD.** Settings, reference data, profiles, catalogues. Event-sourcing them pays all of Concept 10's costs for none of Concept 9's benefits.
- **The costs compound.** Every event-sourced context adds event types to version, projections to operate, streams to archive and personal data to protect.
- **It becomes a platform decision instead of a design decision.** Teams stop asking "does this context's history matter?" and start asking "how do we event-source this?"

The architect's tool is a **per-context persistence map** — one row per bounded context:

| Context | Subdomain type (Module 22) | Persistence | Why |
|---|---|---|---|
| Ledger | Core | **Event-sourced** (streams per account-period) | Regulatory audit; balances as sums; disputes |
| Booking | Core | **Event-sourced** (streams per booking; DCB for capacity + per-customer limits) | Collaborative, reversals, analytics on the funnel |
| Catalogue | Supporting | State-stored (EF Core), temporal tables for price history | CRUD with a history question temporal tables answer |
| Customer profile | Generic | State-stored; personal data stays here (Concept 80) | Erasure-friendly home for personal data |
| Notifications | Generic | State-stored; outbox | No value in its history |

That table is also an ADR (Module 31): it states the decision per context and the reason, so the next team doesn't event-source the notification service because "we're an event-sourced company."

---

## Concept 97 — Across services and modules

When some contexts are event-sourced and others aren't — the normal case — the rules for interaction are the ones Module 21 and Module 22 established, applied strictly:

1. **An event store is private to its bounded context.** No other context reads it, subscribes to it or appends to it. Sharing an event store across contexts is sharing a database — with the added problem that every internal event type becomes a public contract.
2. **Contexts communicate through published integration events** (Concept 19), translated from internal events by the owning context and delivered through a broker (Concept 65) — the same mechanism as for state-stored contexts using an outbox. A consumer shouldn't be able to tell whether the producer is event-sourced.
3. **Consumers keep their own copies** of what they need (Module 21, Concept 41), built from integration events — as ordinary tables or, if the consumer is event-sourced, as events it records itself ("we learned that…") in its own language.
4. **Replays across contexts are a product feature, not a side door.** If downstream contexts need history, publish a replayable integration stream (long-retention Event Hubs or Kafka) — deliberately designed and versioned.
5. **In a modular monolith**, event-sourced modules may share a physical database (Marten can host several stores or schemas in one PostgreSQL instance) — but not event tables, streams or event types. Module boundaries are enforced exactly as in Module 21.

A particular anti-pattern to name: **"the event store as the enterprise event bus"** — one store, every service writing its events into it, every service subscribing to everyone else's streams. It couples every context to every other's storage format, makes versioning a cross-team negotiation, and turns the store into a single point of failure and contention for the whole organization.

---

## Concept 98 — Choosing a store

A decision framework for a .NET/Azure organization, in the order the questions should be asked:

**1. What do we already operate well?** The biggest cost of a new database is operating it — backups, upgrades, monitoring, security reviews, on-call knowledge.

| If you already run… | The natural candidate | Because |
|---|---|---|
| PostgreSQL (Azure Database for PostgreSQL flexible server) | **Marten** | Mature, feature-complete, transactional with documents; DCB; Wolverine integration |
| SQL Server 2025 / Azure SQL | **Polecat** — or a hand-rolled SQL store for simple needs | Marten's model on SQL Server; young (2026), so validate features you need (upcasting, DCB behaviour) |
| Cosmos DB, globally distributed | **Cosmos DIY** (Concept 42) | Azure-native, elastic; you build the event-store features |
| Orleans already | **`JournaledGrain`** with a custom storage provider to a real store | Single-writer actors remove conflicts on hot streams |
| Nothing suitable, and event sourcing is central | **KurrentDB** (self-hosted or Kurrent Cloud), with Eventuous or your own layer | Purpose-built semantics, subscriptions, no gap problem; one more system; licence from 26.2 for clusters |

**2. What features do we need?** Score the candidates against the requirements that matter to *this* context: multi-stream transactions or DCB (Concepts 34–35); inline projections in the same transaction (Concept 57); archiving and compaction (Concept 84); masking for erasure (Concept 82); multi-tenancy (Concept 87); global distribution; upcasting support (Concept 70); throughput per stream and overall.

**3. What does it cost to run and to leave?** Licences (Concept 100), support contracts, managed-service pricing, and the migration path out (Concept 94) — events in JSON in a relational database are far easier to move than events in a proprietary store.

**4. How mature is it for our use?** Release cadence, breaking-change history, documentation quality, community size, commercial support. A fast-moving library (Marten ships very frequently) is a strength and an upgrade tax.

The senior sentence: *"I'd start from the database the organization already runs — Postgres means Marten, SQL Server 2025 means looking hard at Polecat — and only add a dedicated event store like KurrentDB if its subscription model, single-writer log and operational tooling justify running another stateful system and, for high availability, paying for a licence."*

---

## Concept 99 — Build, buy or DIY

Should you write your own event store? Concept 38 showed that the core is a few hundred lines. The rest is where the time goes:

| Capability | Effort to build well |
|---|---|
| Append with expected version, read stream | Small (Concept 38) |
| Idempotent appends, retries | Small |
| Gap-safe global reads, subscriptions with checkpoints | **Significant, subtle** (Concepts 39, 59) |
| Async projection host: leadership, restarts, batching, dead letters, health | **Significant** |
| Rebuilds with blue/green versions | Significant |
| Upcasting pipeline, type registry | Moderate |
| Snapshots with invalidation | Moderate |
| Archiving, compaction, masking, multi-tenancy | Significant, and **dangerous** (Concept 73) |
| Tooling: stream browser, projection status, replay tools | Ongoing |
| DCB with a correct append condition | **Hard** (Concept 36) |

When DIY is reasonable:

- One context, low volume, simple projections, a team that understands Concepts 38–39 — and a real reason not to use a library (a platform constraint, an unusual store).
- Learning and interviews: building a minimal store once is the best way to understand the products.

When it isn't:

- Anything that needs async projections at scale, DCB, multi-tenancy or erasure tooling. The subtle bugs in these areas — out-of-order positions, concurrent boundary appends, tenant-crossing rewrites — have bitten mature, well-tested libraries (Concepts 36, 73). A hand-rolled store will meet them too, with fewer users to find them first.

The build-vs-buy framing for Module 33: the licence cost of a library or store is usually small; the engineering cost of *correctly* reproducing its subscription and projection machinery is large and ongoing.

---

## Concept 100 — Licences and vendor risk

Module 23 (Concepts 73–75) showed how a widely used library's licence can change under a codebase. The event-store market has its own versions of that story, as of September 2026:

| Product | Licence | What to watch |
|---|---|---|
| **KurrentDB** | **KLv1** — source-available, not OSI-approved; prohibits offering it as a hosted service to third parties; enterprise features via licence key | From **26.2**: multi-node clusters, read-only replicas, archiving and encryption at rest need an **Enterprise licence**; single-node production is free and gains many previously licensed features; 26.1 and earlier keep current terms for their LTS period. HA on 26.2+ is a procurement item |
| **Marten, Polecat, Fisher, Wolverine** | MIT | Commercial support, AI "skills" and the CritterWatch monitoring tool are paid JasperFx offerings; the libraries themselves are free. Fast release cadence |
| **Eventuous** | Apache-2.0 | Pre-1.0 and self-described as somewhat volatile; small maintainer team |
| **Orleans** | MIT | Stable; event-sourcing providers are basic |
| **Cosmos DB, Azure SQL, PostgreSQL** | Managed services | Cost model (RUs, vCores); no licence risk, some lock-in |
| **Axon Server** (JVM) | Commercial editions for clustering | Relevant only for polyglot organizations |

The architect's questions, as for MediatR in Module 23:

1. **What happens if the terms change again?** Events in an open format (JSON) in a general-purpose database are portable; a proprietary store is portable only through a copy-and-transform (Concept 72).
2. **Which features are we using that sit behind a licence line** — today and in the next version?
3. **Is there a supported, affordable path for high availability** in production?
4. **Can we insulate our code** — deciders, an `IEventStore` abstraction at the application boundary, integration events that don't expose the store — so that a store change is a project, not a rewrite (Concept 94)?

---

## Concept 101 — The anti-pattern catalogue

Fourteen ways event-sourced systems go wrong — each one a common code-review or architecture-review finding:

| # | Anti-pattern | Symptom | Fix |
|---|---|---|---|
| 1 | **Event-sourcing everything** | CRUD contexts with event stores; projections for settings pages | Per-context decision (Concept 96) |
| 2 | **CRUD events / property sourcing** | `CustomerUpdated`, `PriceChanged` for every field | Intent-revealing events (Concept 14) |
| 3 | **State obsession** | Events carrying the whole entity | Facts with outcomes (Concepts 16–17) |
| 4 | **Command sourcing** | Replays re-decide; history changes with code | Store outcomes (Concept 7) |
| 5 | **Validation in `evolve`** | Streams that can never be loaded again | Evolve never decides (Concept 47) |
| 6 | **Kafka as the event store** | Lost updates, stale write models | Event store + publish to Kafka (Concept 43) |
| 7 | **The event store as a message bus** | Technical events in domain streams; other services subscribing to your store | Private store; integration events via a broker (Concepts 3, 97) |
| 8 | **Publishing internal events** | Every refactoring breaks another team | Translate to integration events (Concept 19) |
| 9 | **Endless streams** | Slowing commands, snapshots everywhere, Cosmos partitions filling | Lifecycle modeling, closing the books (Concept 31) |
| 10 | **Snapshot-first** | Versioned caches everywhere, stale-snapshot bugs | Short streams; measure; snapshot as a cache (Concept 50) |
| 11 | **Side effects in projections** | Emails re-sent on rebuild | Reactors separate from projections (Concept 64) |
| 12 | **Decisions from projections** | Invariants checked against lagging read models | Decide on the stream; `FetchForWriting` semantics (Concept 51) |
| 13 | **No versioning strategy** | "We'll deal with it later"; broken deserialization of old events | Policy, upcasters, golden samples from day one (Concepts 75, 78) |
| 14 | **Personal data everywhere, no erasure plan** | A legal request becomes a rewrite project | Forgettable payloads or crypto-shredding by design (Concepts 80–81) |

Nat Pryce's retrospective on his team's first event-sourced system (reported by InfoQ) is a useful real-world companion: keeping a relational current-state model alongside the events as a second source of truth, confusing event-driven with event-sourced designs, and using the event store as a message bus with technical events mixed into the history — several of the rows above, found the hard way.

---

## Concept 102 — The review checklist

What a reviewer checks in an event-sourced context, in order:

**Scope and design**
1. Is there a written reason this context is event-sourced (Concept 11)? Does the persistence map agree (Concept 96)?
2. Were events designed with domain experts (Event Modeling or equivalent)? Do the stream's event names read as a story (Concept 14)?

**Events**
3. Past-tense, intent-revealing names; no CRUD or property-sourced events.
4. Events carry decision outcomes (prices, totals, results), not just inputs (Concept 16).
5. No whole-entity snapshots, blobs, secrets or unnecessary personal data (Concept 17).
6. Stable type names in a registry; no CLR names in storage (Concept 20); serializer options owned and tested (Concept 21).
7. Golden samples and approval tests exist for every stored type and version (Concept 75).

**Streams and concurrency**
8. Stream boundaries follow invariants; streams have a lifecycle that closes (Concepts 30–31).
9. Every command append uses an expected version; `Any` only where justified (Concept 28).
10. Event IDs are assigned before retries (Concept 29); conflicts re-run the whole decision (Concept 49).
11. Cross-stream rules have an explicit mechanism — and, for DCB, the store's append-condition implementation has been checked and concurrency-tested (Concepts 33–36).

**Write side**
12. `evolve`/`Apply` contains no validation, exceptions, I/O, clock reads or ID generation (Concept 47).
13. No I/O inside `decide`; external inputs arrive on the command and are recorded in events (Concept 44).
14. Snapshots, if any, are justified by measurement and versioned (Concept 50).

**Consumers**
15. Projections are idempotent, ordered per entity, checkpointed, rebuildable — and side-effect free (Concepts 60–61, 64).
16. Reactors are separate subscriptions starting from "now," idempotent, with dead-letter handling (Concept 53).
17. The subscription mechanism reads the global log safely (Concept 59).
18. Integration events are translated, versioned and published from the store (Concepts 19, 65).

**Lifecycle**
19. An erasure design exists and propagates (Concepts 79–83); a retention and archiving policy is written down (Concept 84).
20. Rebuild duration, restore procedure and reconciliation are rehearsed (Concepts 85, 90).
21. Dashboards cover append latency, conflicts, stream lengths, lag and dead letters (Concept 88).

---

## Concept 103 — Event sourcing in the design round

Where event sourcing appears in the 7-step framework (Module 3), and how to raise it without derailing the round:

| Step | What to say |
|---|---|
| **1. Requirements** | Listen for its triggers: audit, disputes, "why did this happen?", history-dependent rules, temporal questions, regulatory retention. Ask: *"Do we need to answer questions about how an entity got to its current state, or about the past as of a date?"* |
| **2. Estimation** | Add events/day, bytes/event, stream lengths per entity lifecycle, storage per year, and rebuild time (Concepts 89–90) |
| **3. API** | Commands named for intent; responses carry the new version (a consistency token); `If-Match` for user edits |
| **4. Data model** | Streams (what each contains, when it closes), key event types with their fields, the projections each screen needs |
| **5. High-level design** | Command handler → event store (append-if-version) → subscriptions → projections / reactors / publisher (Concept 12); **scope ES to the contexts that need it** |
| **6. Deep dive** | Pick one: concurrency on a hot stream, event versioning, projection rebuilds, erasure — the interviewer will usually steer to one |
| **7. Wrap-up** | Name the costs: versioning, eventual consistency, rebuild time, privacy, operational load — and what you'd *not* event-source |

A worked example — a **digital wallet**:

- *Requirements*: balances, top-ups, payments, refunds, disputes; regulators require full history for years; support asks "why is my balance X?"
- *Scope*: event-source the **wallet ledger** only; KYC documents and customer profiles are state-stored (personal data lives there, Concept 80).
- *Streams*: `wallet-{id}-{yyyy-MM}` — one per wallet per month, closed with `PeriodClosed { closingBalance }` and opened with the carried-forward balance (Concept 31). Typical stream: a few dozen events; no snapshots needed.
- *Events*: `FundsToppedUp { amount, source, newBalance }`, `PaymentAuthorized { amount, merchantRef, newBalance }`, `PaymentRefunded { originalPaymentId, amount, newBalance }`, `DisputeOpened`, `DisputeResolved { outcome }` — each with the resulting balance (Concept 16).
- *Concurrency*: append-if-version per wallet-period stream; two concurrent payments conflict and re-decide; the insufficient-funds check is in `decide` (Concept 91's test).
- *Projections*: current balance (inline, so the user always sees their own balance — Concept 57), transaction history (async), regulatory reports (async, per period), fraud features (published integration events to a stream processor).
- *Wrap-up*: versioning policy, 7-year retention with archiving of closed periods, rebuild time estimate, and why the merchant catalogue isn't event-sourced.

---

## Concept 104 — Event sourcing in the architect round

In an architect round, event sourcing is rarely asked about directly. It arrives as a proposal to evaluate — *"the team wants to event-source the new platform; what do you think?"* — and the interviewer is scoring judgment, not enthusiasm.

A structure for the answer:

1. **Ask what problem it solves here.** Audit? Temporal questions? A ledger? Integration? If the answer is "it's modern" or "microservices need events," the proposal conflates event sourcing with event-driven architecture (Concept 6).
2. **Scope it.** Propose the per-context map (Concept 96): event-source the one or two contexts whose history has value; everything else stays state-stored with an outbox.
3. **Price it** in the terms stakeholders care about: learning curve and hiring (Concept 95), operational components (Concept 88), rebuild and restore times (Concepts 85, 90), storage growth, licences (Concept 100), and the cost of leaving (Concept 94).
4. **De-risk it**: one context first, an experienced reviewer, a mature library, deciders so that storage can change, integration events so consumers don't depend on the store, versioning and erasure policies written before the first event is stored.
5. **Write the ADR** (Module 31): context, decision, alternatives considered (temporal tables, audit logs, outbox with archived events — Concept 8), consequences, and the review date.

The sentence that tends to land: *"I'd support event sourcing for the ledger and the booking lifecycle, where the history is the product, and argue against it for the rest. It's a persistence choice with a permanent cost, so I want it where the benefit is permanent too."*

---

## Concept 105 — The 60-second answers and the close

**"What is event sourcing?"**
> *"A persistence approach where a component's events — past-tense business facts — are its system of record. Current state is computed by folding them, and the component decides new commands against that history, appending new events only if the stream hasn't changed since it read it. The consequence is that you can only add to the past: that gives audit, temporal queries and rebuildable read models for free, and makes event versioning, stream length and personal data permanent design concerns."*

**"When would you use it?"**
> *"In the parts of a system where history has business value — ledgers, bookings, claims, workflows under regulatory scrutiny — and not for CRUD, reference data or profiles. It's a per-bounded-context decision, and before choosing it I'd check whether temporal tables or an audit log would meet the actual requirement at a fraction of the cost."*

**"How does it relate to CQRS and event-driven architecture?"**
> *"Event sourcing needs CQRS — every query is a projection — but CQRS doesn't need event sourcing. And it's independent of event-driven architecture: an event-sourced context should publish translated integration events like any other context, and keep its store private."*

**"What are the hard parts?"**
> *"Event schema evolution over years; keeping streams short; eventual consistency on the read side; projection rebuild time; erasing personal data from an immutable log; and the team's learning curve. Each has known techniques — upcasting, closing the books, consistency tokens, blue/green projections, forgettable payloads and crypto-shredding — but they have to be designed in from the start."*

**The close** — one paragraph that demonstrates judgment:

> *"I think of event sourcing as the right answer to a specific question: 'does the history of this thing matter more than its current state?' Where it does — a ledger, a booking lifecycle, a claims process — I'd event-source it with short, lifecycle-bounded streams, deciders as pure functions, append-if-version concurrency, projections for every read, and versioning and erasure policies agreed before the first event is written. Everywhere else I'd keep state-stored models and publish integration events through an outbox, so no other part of the system knows or cares how that context stores its data. That combination gets the benefits where they pay and keeps the costs — which are real and permanent — contained."*

---

# Common interview questions — model answers

**"What is event sourcing, in one sentence?"** Storing a component's state as the sequence of business events that produced it, deciding new commands against that history, and deriving every view of current state by folding the events (Concepts 1, 4, 5).

**"How is it different from event-driven architecture?"** EDA is about how components communicate; event sourcing is about how one component persists. Most event-driven services are state-stored with an outbox; an event-sourced service should still publish translated integration events and keep its store private (Concepts 6, 19, 97).

**"Isn't Kafka an event store?"** No: it can't append conditionally on a key's current version and can't read one entity's history efficiently, so it can't protect invariants. Use it after the store, for distribution (Concept 43).

**"How does concurrency work?"** Optimistic concurrency per stream: read the stream, decide, append with the expected version; if someone appended in between, the append fails and you re-read and re-decide. For user edits, the version the user saw is the expected version and a conflict becomes a 412 (Concepts 28, 49).

**"What's the difference between event sourcing and command sourcing?"** Event sourcing stores outcomes; command sourcing stores requests and re-executes them, so replay re-decides with today's code and data and repeats side effects (Concept 7).

**"Why must `Apply`/`evolve` never throw?"** It runs on every load; if a stored event makes it throw, that entity can never be loaded again. Validation belongs in the decision; history is fixed forward with new events (Concepts 47–48).

**"When do you use snapshots?"** When measured load times or conflict rates require them — usually streams of thousands of events. First try shortening streams with lifecycle modeling. Snapshots are versioned caches of the fold: disposable, invalidated when state logic changes, never a source of truth (Concepts 31, 50, 89).

**"How long should a stream be?"** As long as one lifecycle of the thing it models — a shift, a period, an order — so every command loads a bounded number of events. "Closing the books" carries state into the next period (Concept 31).

**"How do you enforce a rule across two aggregates?"** First ask whether it's a true invariant. Then: a stream that owns the rule, a registry stream for uniqueness, a multi-stream transaction if the store supports it, a DCB decision if the store supports that, or a process manager with compensation (Concepts 33–35).

**"What's a Dynamic Consistency Boundary?"** A per-decision consistency boundary: events carry tags, a decision reads the events matching a query, and appends with the condition that no matching event was appended since. One event can concern several entities; the store must serialize overlapping appends correctly (Concepts 35–37).

**"How would you implement the DCB append condition on SQL?"** Not as `SELECT EXISTS` then `INSERT` — that's write skew at `READ COMMITTED`. Use serializable isolation, locks on the tag values, a single-writer log, or convert the predicate into row conflicts on per-tag version rows, as Marten 9.4 does (Concept 36).

**"How would you build an event store on SQL Server?"** A streams table holding the head version, an events table with a unique `(StreamId, Version)`, unique `EventId` and a clustered global position; append in one transaction with a conditional update of the stream row; read streams by version; read globally below a committed watermark such as `MIN_ACTIVE_ROWVERSION()` (Concepts 38–39).

**"What's the gap problem?"** Identity or sequence values are assigned at insert but become visible at commit, so a subscriber reading by position can skip an event that commits late. Serialize appends, read below a database-provided committed watermark, or use gap detection with a timeout (Concepts 39, 59).

**"How do you version events?"** Classify the change: additive changes use tolerant readers; shape changes get a schema version and an upcaster; meaning changes get a new event type; wholesale restructuring uses copy-and-transform; in-place rewrites only for legal or corruption cases. Golden samples in CI guard it all (Concepts 67–78).

**"A bug wrote wrong events for three days. What now?"** If anything outside the system acted on them, append compensating or correcting events with the reason; if nothing did and the fix is mechanical, an upcaster for the affected window; duplicates can be marked skipped. Never silently edit (Concept 74).

**"How do you handle GDPR erasure?"** Decide before storing the first event: keep personal data out of events and reference it by ID; or crypto-shred per-subject keys, minding backups and downstream copies; or mask in place with a tested procedure. Propagate erasure to projections, caches, search, analytics and consumers, and let the DPO decide sufficiency (Concepts 79–83).

**"What's wrong with crypto-shredding?"** Nothing in principle; in practice the keys end up in backups, decrypted copies end up in projections and logs, keys are cached too long, and whether key destruction is legally erasure is not an engineering decision (Concept 81).

**"How do you rebuild a projection?"** Blue/green: build a new version alongside, from position zero, in batches and in parallel shards; verify; swap; keep the old one for rollback. Know how long it takes before you need it, and make sure nothing with side effects is replayed (Concepts 61, 64, 90).

**"What's the difference between a projection and a reactor?"** Projections build views and must be replayable; reactors perform side effects and must never be replayed — separate subscriptions, separate checkpoints, reactors starting from "now" (Concepts 53, 64).

**"How do you publish events to other services reliably?"** The store is the outbox: a subscription reads committed events, translates the public subset into integration events, publishes with a message ID derived from the source event, and checkpoints after acknowledgement (Concept 65).

**"How does the user see their own write?"** Return the new version from the command; fold the single stream live or use an inline projection for the views users see immediately; use consistency tokens for cross-entity lists (Concept 66).

**"Which event store would you choose in .NET?"** The database you already run: PostgreSQL → Marten; SQL Server 2025 → consider Polecat; a dedicated store (KurrentDB) if its log semantics and subscriptions justify another system — noting that from 26.2 multi-node clusters require an Enterprise licence (Concepts 40–41, 98, 100).

**"Can you use Cosmos DB as an event store?"** Yes: one stream per logical partition, version as item id so a duplicate id is the conflict, transactional batches for atomic appends, change feed for subscriptions — with no global order, a 20 GB partition limit and everything else to build yourself (Concept 42).

**"Should we event-source the whole platform?"** No — scope it to the contexts whose history has value, with a per-context persistence map, and keep the rest state-stored. Event sourcing is a persistence choice, not an architecture (Concepts 96, 104).

**"How would you migrate an existing system to event sourcing?"** One context at a time; import current state as explicitly named starting events (or reconstruct history, marked as such); strangle the old write path; keep legacy tables as a projection during the transition; run in parallel and compare before cutting over (Concept 93).

**"What does it cost to back out?"** Project current state into tables and switch the write side to them; keep the event store as an archive. Cheap if the domain is written as deciders and consumers depend on integration events; expensive otherwise (Concept 94).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Defines event sourcing as "storing events" or "using Kafka" | Defines it as events being the system of record, with decisions made against that history and append-if-version concurrency |
| Conflates event sourcing with EDA, CQRS or microservices | States the relationships precisely: ES needs CQRS; EDA and ES are independent; ES is per context |
| Proposes event sourcing for the whole system | Proposes a per-context persistence map with a reason for each event-sourced context |
| Reaches for event sourcing to get an audit trail | Compares temporal tables, ledger tables and audit logs first, and names what only ES gives |
| Names events `CustomerUpdated`, `StatusChanged` | Names intent: `CustomerRelocated`, `OrderShipped`; avoids property sourcing and state obsession |
| Recomputes prices and totals from inputs at replay | Records decision outcomes in events so replay is deterministic |
| Stores CLR type names in events | Uses a registry of stable names with aliases, tested |
| Validates inside `Apply` | Keeps `evolve` total and pure; validates in the decision; fixes history forward |
| Appends with `Any` | Uses expected versions everywhere, and re-runs the whole decision on conflict |
| Ignores retries | Assigns event IDs before the first attempt; knows the store's de-duplication rules |
| Lets streams grow forever, then adds snapshots | Models lifecycles and closes the books; snapshots only when measured |
| Uses Kafka as the store | Uses an event store for decisions and Kafka for distribution |
| Hand-waves DCB | Explains tags, queries and append conditions — and why a check-then-insert is write skew |
| Reads the global log naïvely by position | Knows the late-commit gap problem and the watermark and gap-detection fixes |
| Puts side effects in projections | Separates projections, reactors and process managers by replay rules |
| Decides commands from a lagging read model | Decides on the stream; loads async write models plus the event delta |
| "We'll version events when we need to" | Has a written versioning policy, upcasters with golden samples, and expand/contract deployments |
| Edits wrong events in place | Compensates or corrects visibly; upcasts only invisible, mechanical fixes; rewrites only as a last resort |
| Stores personal data freely in events | Keeps it out or crypto-shreds by design, and propagates erasure everywhere |
| Treats crypto-shredding as automatically compliant | Knows the backup, cache and copy traps and defers legal sufficiency to the DPO |
| No numbers | Quotes load time per event, the conflict formula, rebuild hours and storage per year |
| Restores a backup and calls it done | Rewinds or rebuilds projections and reconciles with already-published events |
| Publishes internal events to other teams | Translates a deliberate subset into versioned integration events |
| Picks a store by popularity | Chooses by what the organization operates, required features, licence terms and exit cost |
| Unaware of 2025–2026 changes | Knows KurrentDB's KLv1 and 26.2 clustering licence, Marten 9 and Polecat, Microsoft's 2026 guidance, and the DCB ecosystem |

---

## Practice exercises

**Exercise 1 — Build a minimal event store and break it (half a day).**
Implement Concept 38's SQL Server (or PostgreSQL) event store in a console project with Testcontainers. Write the concurrency test from Concept 91 and run it 1,000 times. Then implement a naïve catch-up subscriber that polls `WHERE GlobalPosition > @checkpoint`, and write a test that makes it skip an event: start transaction A, insert, pause; run transaction B to completion; poll; commit A; poll again. Fix it with `MIN_ACTIVE_ROWVERSION()` (or the PostgreSQL snapshot-xmin technique) and show the test passes. *Deliverable: the failing and passing tests, and a paragraph explaining the gap problem in your own words.*

**Exercise 2 — Model a lifecycle (60 minutes, whiteboard).**
Take a coffee-shop loyalty system: customers earn points on purchases, redeem them for drinks, and points expire after 12 months. Design the streams (what closes, when), the events with fields (including outcomes), and one projection. Estimate stream lengths for a heavy user. Then redesign with a single stream per customer and compare load cost after five years. *Deliverable: an event model on one page and the arithmetic.*

**Exercise 3 — Deciders and specs (2 hours).**
Implement the wallet of Concept 103 as a decider (`decide`, `evolve`, initial state) with a generic handler (Concept 46) over your store from Exercise 1. Write at least 15 given-when-then specs, including insufficient funds, refunds exceeding the original payment, operations on a closed period and idempotent retries. Add one property-based test asserting the balance never goes negative for any generated command sequence. *Deliverable: the tests, all green, running in under a second.*

**Exercise 4 — Marten end to end (half a day).**
Rebuild Exercise 3 on Marten 9: `FetchForWriting`, an inline single-stream projection for the balance, an async multi-stream projection for monthly statistics, and the async daemon. Force a concurrency conflict and observe the exception. Bump the async projection's version and watch the blue/green rebuild. Try `AggregateStreamAsync` with a timestamp for an "as of" balance. *Deliverable: notes on what Marten did for you that Exercise 1 made you do by hand.*

**Exercise 5 — Version an event three times (2 hours).**
Start with `FundsToppedUp { amount }`. Evolve it to v2 `{ amount, currency }` (old events were EUR), then v3 `{ amount: Money, source }` (source unknown for old events). Write chained upcasters (Concept 70), golden JSON samples for v1–v3, and approval tests (Concept 75). Then simulate a mixed-version deployment: run v2 code against a store containing v3 events and make it survive (Concept 71). *Deliverable: the samples, upcasters and a written versioning policy for the team.*

**Exercise 6 — DCB and its race (2–3 hours).**
Implement the course-subscription example (Concept 35) twice on PostgreSQL: once with a naïve `SELECT EXISTS … INSERT` append condition, once with per-tag version rows (Concept 36) — or use Marten 9.4+ for the second. Run 200 concurrent subscription attempts for 30 seats in a loop and count over-subscriptions in each version. *Deliverable: the numbers, and a one-paragraph explanation linking the failure to write skew (Module 12).*

**Exercise 7 — Erasure design (90 minutes, written).**
For the wallet, list every place a customer's name and email could end up (events, projections, caches, search, analytics export, logs, integration events, backups). Design an erasure flow using forgettable payloads for profile data and crypto-shredding for one field that must remain in events. Identify the backup trap for the key store and propose retention settings. Write the three questions you'd ask the DPO. *Deliverable: a one-page design and the erasure-tracking projection's schema.*

**Exercise 8 — Rebuild and restore drill (half a day).**
Load a million events into your Marten or custom store (a generator script). Time a full rebuild of the monthly-statistics projection single-threaded, then sharded. Then restore the database to an earlier point, observe the projection now being "ahead," and write the runbook steps to rewind, rebuild and reconcile with published integration events. *Deliverable: timings, and a runbook you'd be comfortable handing to on-call.*

**Exercise 9 — The architect memo (60 minutes, written).**
Your CTO asks: "The platform team wants to standardize on event sourcing with KurrentDB for all twelve services. Yes or no?" Write a one-page ADR-style memo: which services (if any) should be event-sourced and why, the per-context persistence map, costs (people, operations, licence from 26.2, exit), alternatives considered (temporal tables, outbox), and a de-risking plan. *Deliverable: the memo — and practise delivering its conclusion in 60 seconds.*

**Exercise 10 — Mock deep-dive (45 minutes, with a partner).**
Have a partner play the interviewer on "design a hotel booking system," steering to event sourcing. Practise: scoping it (bookings yes, hotel content no), streams and lifecycles, the overbooking rule (DCB vs per-room streams vs reservation), versioning, the stale-read bug after booking, and erasure. Record it and score yourself against the review checklist (Concept 102).

---

# Free resources

Everything below is free to read or watch. Links were checked in September 2026; product pages move, so if one breaks, search the title.

### Foundations — start here

| Resource | What it covers |
|---|---|
| [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) — Martin Fowler, 2005 | The original description and vocabulary (**Concepts 1–4**) |
| [What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html) — Martin Fowler, 2017 | The four meanings: notification, state transfer, event sourcing, CQRS (**Concept 6**) |
| [CQRS Documents (PDF)](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) — Greg Young, 2010 | The practitioner who named both patterns; event storage, concurrency, the event store (**Concepts 5, 26, 28**) |
| [Versioning in an Event Sourced System](https://leanpub.com/esversioning/read) — Greg Young (free to read online) | The definitive text on event evolution: weak schema, upcasting, copy-and-replace (**Part F**) |
| [Eventsourcing: State from Events or Events as State?](https://verraes.net/2019/08/eventsourcing-state-from-events-vs-events-as-state/) — Mathias Verraes | The precise definition this module uses (**Concept 5**) |
| [DDD and Messaging Architectures (pattern index)](https://verraes.net/2019/05/ddd-msg-arch/) — Mathias Verraes | Summary Event, Decision Tracking, Multi-temporal Events, Forgettable Payloads, Crypto-Shredding, Segregated Event Layers and more (**Concepts 16, 18, 19, 80–81**) |
| [Functional Event Sourcing Decider](https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider) — Jérémie Chassaing | decide / evolve / initial state (**Concept 46**) |
| [Event sourcing pattern](https://microservices.io/patterns/data/event-sourcing.html) — Chris Richardson | A concise pattern description with forces and consequences |
| [Exploring CQRS and Event Sourcing (the "CQRS Journey")](https://www.microsoft.com/en-us/download/details.aspx?id=34774) — Microsoft patterns & practices (free download) | A team's real journey building an event-sourced system on Azure, including the mistakes |

### Research and theory

| Resource | What it covers |
|---|---|
| [An Empirical Characterization of Event Sourced Systems and Their Schema Evolution](https://arxiv.org/abs/2104.01146) — Overeem, Spoor, Jansen, Brinkkemper (JSS 2021, arXiv) | 19 systems, 25 engineers: rationale, five challenges, five evolution methods (**Concepts 9–10, 78**) |
| [The dark side of event sourcing: Managing data conversion](https://www.semanticscholar.org/paper/The-dark-side-of-event-sourcing:-Managing-data-Overeem-Spoor/556eb79fb2493914732d020d2240cda71696b47c) — Overeem, Spoor, Jansen (SANER 2017) | Data conversion techniques and their trade-offs (**Part F**) |
| [Immutability Changes Everything](https://www.cidrdb.org/cidr2015/Papers/CIDR15_Paper16.pdf) — Pat Helland (CIDR 2015; also in [ACM Queue](https://queue.acm.org/detail.cfm?id=2884038)) | Why append-only records are the foundation of distributed data (**Concept 2**) |
| [The Log: What every software engineer should know…](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) — Jay Kreps | Logs as the unifying abstraction of data systems (**Concept 2**) |
| [Event Sourcing and Stream Processing at Scale](https://martin.kleppmann.com/2016/01/29/event-sourcing-stream-processing-at-ddd-europe.html) — Martin Kleppmann (talk page with slides) | Bridging the DDD and stream-processing communities (**Concepts 5, 43**) |

### Talks

| Resource | What it covers |
|---|---|
| [A Decade of DDD, CQRS, Event Sourcing](https://www.youtube.com/watch?v=LDW0QWie21s) — Greg Young, DDD Europe 2016 | History, and why ES is not a top-level architecture (**Concept 96**) |
| [Event Sourcing and Stream Processing at Scale](https://www.youtube.com/watch?v=avi-TZI9t2I) — Martin Kleppmann, DDD Europe 2016 | Event sourcing vs stream processing (**Concept 43**) |
| [Event Sourcing: You are doing it wrong](https://www.youtube.com/watch?v=GzrZworHpIk) — David Schmitz, Devoxx | Production mistakes and their fixes (**Concept 101**) |
| [Keep your streams short! (talk page)](https://2022.dddeurope.com/program/keep-your-streams-short!-or-how-to-model-event-sourced-systems-efficiently/) — Oskar Dudycz, DDD Europe 2022 | Temporal modeling and stream lifecycles (**Concept 31**) |
| [Webinar #16 — Simple patterns for events schema versioning](https://www.architecture-weekly.com/p/webinar-16-simple-patterns-for-events) — Oskar Dudycz | Versioning patterns with code in C#, Java and TypeScript (**Part F**) |

### Honest critiques and experience reports

| Resource | What it covers |
|---|---|
| [Don't Let the Internet Dupe You, Event Sourcing is Hard](https://chriskiehl.com/article/event-sourcing-is-hard) — Chris Kiehl | The costs of a first production system, one by one (**Concept 10**) |
| [What they don't tell you about event sourcing](https://medium.com/@hugo.oliveira.rocha/what-they-dont-tell-you-about-event-sourcing-6afc23c69e9a) — Hugo Rocha | Eventual consistency, versioning and operational caveats (**Concept 10**) |
| [Mistakes and recovery in event sourcing (Nat Pryce)](https://www.infoq.com/news/2019/07/mistakes-recovery-event-sourcing/) — InfoQ | Two sources of truth, event store as message bus, technical events in history (**Concept 101**) |
| [Event Sourcing Done Right — Dennis Doomen at DDD Europe](https://www.infoq.com/news/2020/02/event-sourcing-doomen-ddd-europe/) — InfoQ | "A tool, not an architecture"; autonomous projections (**Concepts 61, 96**) |
| [Event Streaming is not Event Sourcing!](https://event-driven.io/en/event_streaming_is_not_event_sourcing/) — Oskar Dudycz | Why Kafka isn't an event store (**Concept 43**) |

### Event design, streams and modeling

| Resource | What it covers |
|---|---|
| [What is Event Modeling?](https://eventmodeling.org/posts/what-is-event-modeling/) — Adam Dymitruk | Commands, events, views and slices on a timeline (**Concept 24**) |
| [Event stores are key-value databases, and why that matters](https://event-driven.io/en/event_stores_are_key_value_stores/) — Oskar Dudycz | The mental model behind stream performance (**Concept 26**) |
| [Let's talk about positions in event stores](https://event-driven.io/en/lets_talk_about_positions_in_event_stores/) — Oskar Dudycz | Stream versions vs global positions, gaps (**Concepts 27, 39**) |
| [Keep your streams short! Temporal modeling for fast reads and optimal data retention](https://www.kurrent.io/blog/keep-your-streams-short-temporal-modelling-for-fast-reads-and-optimal-data-retention) — Oskar Dudycz, Kurrent blog | Lifecycles and closing the books (**Concept 31**) |
| [Implementing the Closing the Books pattern](https://event-driven.io/en/closing_the_books_in_practice/) — Oskar Dudycz | A worked implementation with Marten (**Concept 31**) |
| [Should you always keep streams short in Event Sourcing?](https://event-driven.io/en/should_you_always_keep_streams_short/) — Oskar Dudycz | When long streams are acceptable (**Concept 31**) |
| [How to model event-sourced systems efficiently](https://www.kurrent.io/blog/how-to-model-event-sourced-systems-efficiently/) — Kurrent blog | Stream design strategies (**Concepts 30–31**) |
| [Snapshots in Event Sourcing](https://www.kurrent.io/blog/snapshots-in-event-sourcing/) — Oskar Dudycz, Kurrent blog | When and how to snapshot (**Concept 50**) |

### Projections

| Resource | What it covers |
|---|---|
| [Guide to Projections and Read Models in Event-Driven Architecture](https://event-driven.io/en/projections_and_read_models_in_event_driven_architecture/) — Oskar Dudycz | Checkpoints, idempotency, rebuilds, "live" thresholds (**Part E**) |
| [How to test event-driven projections](https://event-driven.io/en/testing_event_driven_projections/) — Oskar Dudycz | Projection test specifications (**Concept 91**) |
| [Projections, Consistency Models, and Zero Downtime Deployments with the Critter Stack](https://jeremydmiller.com/2025/03/26/projections-consistency-models-and-zero-downtime-deployments-with-the-critter-stack/) — Jeremy Miller | `FetchForWriting`, `FetchLatest`, blue/green projections (**Concepts 51, 61, 66**) |

### Versioning

| Resource | What it covers |
|---|---|
| [Simple patterns for events schema versioning](https://event-driven.io/en/simple_events_versioning_patterns/) — Oskar Dudycz | Mapping, upcasting, downcasting, stream transformations (**Concepts 69–72**) |
| [Event Versioning with Marten](https://event-driven.io/en/event_versioning_with_marten/) — Oskar Dudycz | Upcasting with schema versions in Marten (**Concept 70**) |
| [EventSourcing.NetCore — EventsVersioning sample](https://github.com/oskardudycz/EventSourcing.NetCore/tree/main/Sample/EventsVersioning) | Runnable C# versioning examples (**Exercise 5**) |
| [Marten — Events Versioning](https://martendb.io/events/versioning) | Namespace/type migrations, upcasting and its performance caveats (**Concepts 70, 76**) |

### Dynamic Consistency Boundaries

| Resource | What it covers |
|---|---|
| [dcb.events](https://dcb.events/) · [Specification](https://dcb.events/specification/) · [FAQ](https://dcb.events/faq/) | The specification, reasoning and common questions (**Concepts 35–37**) |
| [Example: course subscriptions](https://dcb.events/examples/course-subscriptions/) · [Example: unique username](https://dcb.events/examples/unique-username/) | Worked DCB decisions (**Concepts 33, 35**) |
| [DCB implementations and tools](https://dcb.events/resources/libraries/) | Stores and libraries by language, including .NET (**Concept 40**) |
| [Killing the Aggregate](https://sara.event-thinking.io/2023/04/kill-aggregate-chapter-1-I-am-here-to-kill-the-aggregate.html) — Sara Pellegrini | The post that started it (**Concept 35**) |
| [Marten — Dynamic Consistency Boundary](https://martendb.io/events/dcb) | Tags, queries, `FetchForWritingByTags`, storage modes, and how the 9.4 consistency check serializes (**Concepts 35–36**) |
| [Higher Performance DCB Development with Marten 9.0](https://jeremydmiller.com/2026/05/25/higher-performance-dynamic-consistency-boundary-development-with-marten-9-0/) — Jeremy Miller | The HStore mode and a candid view of demand (**Concept 37**) |
| [eventsourcing (Python) — Dynamic consistency boundaries](https://eventsourcing.readthedocs.io/en/stable/topics/dcb.html) | Implementation and indexing challenges, "slices" and "enduring objects" (**Concept 37**) |
| [EventSourcingDB — Dynamic Consistency Boundaries](https://docs.eventsourcingdb.io/best-practices/dynamic-consistency-boundaries/) | A purpose-built store's view: when DCB, when aggregates (**Concept 37**) |

### Privacy, erasure and GDPR

| Resource | What it covers |
|---|---|
| [Eventsourcing Patterns: Forgettable Payloads](https://verraes.net/2019/05/eventsourcing-patterns-forgettable-payloads/) — Mathias Verraes | Personal data outside the store (**Concept 80**) |
| [Eventsourcing Patterns: Crypto-Shredding](https://verraes.net/2019/05/eventsourcing-patterns-throw-away-the-key/) — Mathias Verraes | Per-resource keys and their limits, including the legal question (**Concept 81**) |
| [How to deal with privacy and GDPR in Event-Driven systems](https://event-driven.io/en/gdpr_in_event_driven_architecture/) — Oskar Dudycz | Practical erasure flows and consumer obligations (**Concept 83**) |
| [Marten — Removing Protected Information](https://martendb.io/events/protection) | Masking rules and `ApplyEventDataMasking`, with caveats (**Concept 82**) |
| [CJEU press release: EDPS v SRB, C-413/23 P (PDF)](https://curia.europa.eu/site/upload/docs/application/pdf/2025-09/cp250107en.pdf) | The September 2025 judgment on pseudonymised data (**Concept 79**) |
| [EDPB Guidelines 01/2025 on Pseudonymisation (PDF)](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf) | The regulators' draft position (**Concept 79**) |

### Microsoft and Azure

| Resource | What it covers |
|---|---|
| [Event Sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) — Azure Architecture Center (rewritten March 2026) | Trade-offs, versioning strategies, crypto-shredding, "an event store is not a message broker" — read its overview diagram critically (**Concept 12**) |
| [CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) · [Materialized View pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view) · [Compensating Transaction pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction) | The companion patterns (**Parts E, F**) |
| [Change feed design patterns in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-design-patterns) | Cosmos DB as an append-only event store; change feed modes (**Concept 42**) |
| [Azure Cosmos DB design patterns — code samples](https://learn.microsoft.com/en-us/samples/azure-samples/cosmos-db-design-patterns/design-patterns/) · [event-sourcing sample on GitHub](https://github.com/Azure-Samples/cosmos-db-design-patterns/blob/main/event-sourcing/README.md) · [Design patterns part 6: Event Sourcing](https://devblogs.microsoft.com/cosmosdb/azure-cosmos-db-design-patterns-part-6-event-sourcing/) | Runnable Cosmos DB event-sourcing samples (**Concept 42**) |
| [Orleans event sourcing overview](https://learn.microsoft.com/dotnet/orleans/grains/event-sourcing/) | `JournaledGrain`, log-consistency providers (**Concepts 32, 40**) |
| [.NET Microservices: DDD and CQRS patterns](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/) | The state-stored alternative and its CQRS reads, for contrast |
| [Temporal tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables) · [Ledger overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-overview) · [MIN_ACTIVE_ROWVERSION](https://learn.microsoft.com/en-us/sql/t-sql/functions/min-active-rowversion-transact-sql) | History and tamper evidence without ES; the gap-safe watermark (**Concepts 8, 39, 86**) |

### Marten, Polecat, Fisher and Wolverine

| Resource | What it covers |
|---|---|
| [Marten as Event Store](https://martendb.io/events/) · [Understanding Event Sourcing with Marten](https://martendb.io/events/learning) | The complete .NET event-store library (**Concept 40**) |
| [Command Handler Workflow (`FetchForWriting`)](https://martendb.io/scenarios/command_handler_workflow.html) | Optimistic concurrency and write models across lifecycles (**Concepts 28, 51**) |
| [Projections overview](https://martendb.io/events/projections/) · [Async daemon](https://martendb.io/events/projections/async-daemon) · [Rebuilding](https://martendb.io/events/projections/rebuilding) · [Side effects](https://martendb.io/events/projections/side-effects) | Projection kinds, lifecycles, rebuilds, rebuild-safe side effects (**Part E**) |
| [Optimizing performance](https://martendb.io/events/optimizing) · [Stream compacting](https://martendb.io/events/compacting) · [Archiving streams](https://martendb.io/events/archiving) | Aggregate caching, compaction, archival (**Concepts 50, 84**) |
| [Tutorial: Building a Freight & Delivery System](https://martendb.io/tutorials/) | End-to-end walkthrough from documents to event sourcing (**Exercise 4**) |
| [Marten 9.0, Polecat 4.0, and Wolverine 6.0 are Live!](https://jeremydmiller.com/2026/05/24/marten-9-0-polecat-4-0-and-wolverine-9-0-are-live/) — Jeremy Miller | The Critter Stack 2026 release wave |
| [Polecat documentation](https://polecat.jasperfx.net) · [Announcing Polecat](https://jeremydmiller.com/2026/03/22/announcing-polecat-event-sourcing-with-sql-server/) | Marten's model on SQL Server 2025 (**Concept 40**) |
| [Fisher 1.0 release notes](https://github.com/JasperFx/fisher/releases/tag/v1.0.0) | The embedded SQLite member of the family (**Concept 40**) |
| [Wolverine — Aggregate Handlers and Event Sourcing](https://wolverinefx.net/guide/durability/marten/event-sourcing.html) · [CQRS with Marten tutorial](https://wolverinefx.net/tutorials/cqrs-with-marten.html) | The decider-style aggregate handler workflow with outbox (**Concepts 46, 65**) |
| [EventSourcingWithMarten](https://github.com/jeremydmiller/EventSourcingWithMarten) — Jeremy Miller | Presentation materials and samples |

### KurrentDB

| Resource | What it covers |
|---|---|
| [KurrentDB documentation (26.0 LTS)](https://docs.kurrent.io/server/v26.0/) · [Release schedule](https://docs.kurrent.io/server/v26.0/release-schedule/) | Server features, LTS cadence (**Concept 41**) |
| [.NET client — getting started](https://docs.kurrent.io/clients/dotnet/) · [Appending events](https://docs.kurrent.io/clients/dotnet/v1.0/appending-events) | Expected revisions, idempotency (**Concepts 28–29**) |
| [KurrentDB 25.1: Secondary Indexes and Multi-Stream Appends](https://www.kurrent.io/releases/kurrentdb/25-1/) · [Secondary indexes explained](https://www.kurrent.io/blog/secondary-indexes/) | Indexes without link events; atomic multi-stream appends (**Concepts 34, 41**) |
| [KurrentDB 26.0](https://kurrentdb.kurrent.io/releases/kurrentdb/26-0/) · [KurrentDB 26.1](https://www.kurrent.io/releases/kurrentdb/26-1/) | User-defined indexes, SQL access, Projections V2 (**Concept 41**) |
| [Licensing in KurrentDB v26.2 and beyond](https://kurrentdb.kurrent.io/blog/licensing-in-kurrentdb-v26-2-and-beyond-what-s-free-and-what-s-licensed/) · [Quick preview of KurrentDB v26.2](https://kurrentdb.kurrent.io/blog/quick-preview-of-kurrentdb-v26-2/) | Raft clustering and the new licence line (**Concept 100**) |
| [KurrentDB licence (KLv1)](https://github.com/kurrent-io/KurrentDB/blob/master/LICENSE.md) | The source-available terms (**Concept 100**) |

### Other .NET libraries and reference code

| Resource | What it covers |
|---|---|
| [Eventuous documentation](https://eventuous.dev) · [Eventuous on GitHub](https://github.com/eventuous/Eventuous/releases) | Lightweight ES library for KurrentDB, PostgreSQL, SQL Server (**Concept 40**) |
| [oskardudycz/EventSourcing.NetCore](https://github.com/oskardudycz/EventSourcing.NetCore) | The largest free collection of .NET event-sourcing samples and self-paced workshop material |
| [eugene-khyst/postgresql-event-sourcing](https://github.com/eugene-khyst/postgresql-event-sourcing) | A reference PostgreSQL event store (Java) with a clear write-up of reliable global ordering (**Concept 39**) |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What event sourcing is | Events are the system of record; state is a fold; decisions against history; append-if-version |
| Its one consequence | You can only append to the past — audit and replay for free; versioning, stream length, privacy forever |
| ES vs EDA vs CQRS | ES = persistence; EDA = communication; ES needs CQRS, not vice versa |
| Command sourcing | Store outcomes, not requests; replay must never re-decide |
| Alternatives for history | Audit logs, temporal tables, ledger tables, CDC — cheaper when history isn't load-bearing |
| When to use it | Per context, where history drives value: ledgers, bookings, claims, regulated workflows |
| Event naming | Past tense, intent (`CustomerRelocated`), never CRUD or property sourcing |
| Event content | Decision outcomes and what consumers need; no whole-entity state, blobs, secrets or needless PII |
| Envelope | Event ID, stream, version, global position, type name, schema version, times, correlation, causation, actor, tenant, tags |
| Type names | Stable strings in a registry with aliases; never CLR names |
| Time | Occurred vs recorded vs effective; inject the clock; never order by timestamps |
| Internal vs public events | Private store; translated, versioned integration events |
| Event store contract | Append-if-version, read stream, read all, subscribe, idempotent append |
| Positions | Stream version gapless; global position may have gaps |
| Concurrency | Expected version is the lock; conflict = re-read and re-decide; 412 for users |
| Idempotent appends | Event IDs fixed before retries; store de-duplication; idempotency keys in metadata |
| Stream boundaries | = consistency boundaries = aggregate design; hard to change later |
| Stream length | Keep short via lifecycles; close the books |
| Hot streams | P(conflict) ≈ 1 − e^(−λd); ceiling ≈ 1/d; split, shorten, single writer, escrow |
| Cross-stream rules | Policy vs invariant; owner stream; registry stream; multi-stream tx; DCB; process manager |
| DCB | Tags + query + append condition; one fact, many concepts; per-decision boundary |
| DCB safety | Check-then-insert is write skew; per-tag version rows (Marten 9.4), serializable, locks or single writer |
| Relational store | Streams row as version lock; unique (stream, version) and event ID; clustered global position |
| Gap problem | Late commits skipped; serialize, `MIN_ACTIVE_ROWVERSION()` / snapshot xmin, or gap timeout |
| Store choice | What you already run: Postgres → Marten; SQL Server 2025 → Polecat; KurrentDB if justified |
| KurrentDB | Single-writer log, catch-up vs persistent subscriptions, secondary indexes, KLv1, clusters licensed from 26.2 |
| Cosmos DB | Stream = partition; id = version; transactional batch; change feed; no global order; 20 GB |
| Kafka | Not an event store: no per-key expected version, no per-entity reads; use for distribution |
| Command loop | Load, fold, decide, append — no I/O in decide |
| Evolve | Total, pure, never throws, ignores unknown events |
| Bad history | Fix forward with correction events; never crash on the past |
| Snapshots | Measured need only; versioned cache of the fold; short streams first |
| Write model sources | Live, inline, async — async only with delta folding (`FetchForWriting`) |
| Consumers | Projections replay; reactors never; process managers rebuild state, never resend |
| Global log reading | Safe high-water mark; alert when it stalls |
| Projection idempotency | Stream version per entity; global checkpoint atomic with the read model |
| Rebuilds | Blue/green, sharded, batched, no side effects, rehearsed; ~7 h for 500 M events at 20k/s |
| Temporal queries | Fold up to a version or time; bi-temporal with occurred + recorded times |
| Publishing | Store is the outbox; translate; message ID from source event; checkpoint after ack |
| Read-your-writes | Return version; live fold / `FetchLatest`; inline for own views; consistency tokens |
| Versioning rule | New version must be convertible from old — else it's a new event |
| Versioning tools | Weak schema → upcasting → copy-and-transform → (last resort) rewrite |
| Deployments | Expand/contract: readers before writers; old code ignores unknown events |
| Guarding | Golden JSON samples, approval tests, registry tests in CI |
| GDPR | Decide on day one: keep PII out, crypto-shred, or mask; propagate; DPO decides sufficiency |
| Crypto-shredding traps | Keys in backups, plaintext copies, long key caches, legal status |
| Retention | Close, archive, compact; per-projection decision on archived history |
| Restore | Rewind projections; reconcile with already-published events |
| Security | Append-only permissions; ledger tables or hash chains; encryption; audit admin ops |
| Multi-tenancy | Tenant in stream identity, set by infrastructure; test rewrite paths with colliding IDs |
| Arithmetic | ~5–20 µs per event loaded; storage/year = cmds × events × bytes × 365 × overhead |
| Migration in | One context; import or reconstruct, marked; strangle; compare in parallel |
| Migration out | Persist the fold; deciders and integration events keep the door open |
| Architect round | Scope, price, de-risk, ADR — "where the history is the product" |

---

This module closes the loops it was created to close:

- **Module 23's deferred thread** — "event sourcing, where the write side *is* the event log, projections become the only way to query, rebuilds become replays, and events carry personal data that must be erasable" — is now complete: Part D is the write side as a log, Part E makes projections the only read path, Concept 61 turns rebuilds into replays, and Part G handles erasure.
- **Module 22's promise** that "choosing event sourcing later doesn't change the domain code" is kept by the generic decider handler (Concept 46), and its Dynamic Consistency Boundary (Concept 74) now has its full mechanics and its concurrency pitfall (Concepts 35–37).
- **Module 12's write skew and phantom anomalies** reappear as the reason a naïve DCB append condition is unsafe (Concept 36), and its append-only storage structures as the relational event store (Concepts 38–39).
- **Module 11's separation of event sourcing from event-driven architecture** now has its complete argument (Concepts 5–6, 43, 97), and its outbox becomes "the store is the outbox" (Concept 65).
- **Module 7's per-object linearizability** is exactly what append-if-version gives a stream (Concept 28); **Module 9's logs and fencing** reappear in the gap problem and in single-writer Kafka processing (Concepts 39, 43).

Threads left open on purpose:

- **Resilience with Polly** — retrying concurrency conflicts by re-running the decision without multiplying retries across layers (Concept 49), and resilience for subscriptions and publishers — is **Module 25**.
- **Cosmos DB partitioning and RU economics** for event containers and hierarchical partition keys (Concepts 42, 87) — **Module 27**.
- **Observability** — tracing across appends and subscriptions with span links, lag SLOs and error budgets (Concept 88) — **Module 28**.
- **Security architecture** — Key Vault and Managed HSM for crypto-shredding keys, managed identities, least privilege per consumer (Concepts 81, 86) — **Module 29**.
- **ADRs** for the per-context persistence map, versioning policy and retention policy (Concepts 78, 84, 96) — **Module 31**.
- **Brownfield migration** with the strangler fig (Concept 93) — **Module 32**.
- **Build-vs-buy and licence risk** in executive terms (Concepts 99–100) — **Module 33**.

Next in the curriculum: **Module 25 — Resilience in .NET with Polly**: retry, circuit breaker, timeout, rate limiter and hedging in Polly v8's resilience pipelines and `Microsoft.Extensions.Resilience`; how to compose them without multiplying attempts; and where each belongs relative to EF Core's execution strategy, the command pipeline of Module 23 and the concurrency retries of this module.
