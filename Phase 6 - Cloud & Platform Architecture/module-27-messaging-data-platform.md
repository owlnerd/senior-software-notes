# Module 27 — Messaging & Data Platform on Azure: Service Bus, Event Grid, Event Hubs and Cosmos DB
*Phase 6: Cloud & Platform Architecture · Senior/Architect Interview Prep for .NET & C#*

> **Platform state verified on October 5, 2026.** Messaging and data services change more slowly than compute, but several facts in this module are recent enough that a stale answer is a visible signal:
>
> - **Service Bus:** the legacy .NET libraries `WindowsAzure.ServiceBus` and `Microsoft.Azure.ServiceBus` and the **SBMP protocol were retired on September 30, 2026 — five days ago.** `Azure.Messaging.ServiceBus` (AMQP) is the only supported .NET client; the current stable release is **7.21.0** (September 24, 2026). **Geo-Replication** for Premium (metadata *and* data, synchronous or asynchronous, planned or forced promotion, one secondary region) has been **GA since December 2025**. The **Service Bus emulator 2.0.0** (January 2026) added native support for the .NET administration client.
> - **Event Hubs:** **Geo-Replication** for Premium and Dedicated has been **GA since July 2025**, with synchronous or asynchronous replication and replicated consumer offsets. **Kafka transactions** are in **public preview** on Premium and Dedicated; **Kafka compression** is supported only on Premium and Dedicated; **log compaction** is not available on Basic.
> - **Event Grid:** two tiers — **Basic** (push delivery from custom, system, domain and partner topics) and **Standard** (**namespaces**: namespace topics with **pull** and **push** delivery, plus a managed **MQTT broker**).
> - **Cosmos DB (Build 2026, June 2):** **Global Secondary Indexes — GA**; **Per-Partition Automatic Failover (PPAF) — GA** (affected partitions fail over in about **3 minutes at p99**, part of the **Business Critical** tier); **distributed transactions — public preview** (atomic writes and snapshot reads across partitions, containers and databases in one account and region, up to **100 operations / 2 MB**); **Azure Backup for Cosmos DB — preview**; the **Linux emulator — GA**; **changing a container's partition key — GA**; **all versions and deletes change feed mode — GA**. Accounts are now described by two **service tiers**: **General Purpose** (availability SLA up to 99.995%) and **Business Critical** (up to 99.999%, the tier formerly known as "multi-region write accounts"). The .NET SDK `Microsoft.Azure.Cosmos` is at **3.63.2** (September 30, 2026).
> - **Renames and retirements to know:** **Azure DocumentDB** is the new name (November 18, 2025) of what was *vCore-based Azure Cosmos DB for MongoDB* — a separate product from Cosmos DB. **Azure Synapse Link for Cosmos DB is no longer supported for new projects**; **Cosmos DB Mirroring for Microsoft Fabric** is the replacement. And from Module 26: **.NET 8 and 9 reach end of support on November 10, 2026**, the same day as the Functions in-process model.
>
> Prices quoted here are approximate US list prices used for **arithmetic**, not procurement. Re-check the pricing pages before you quote a number to anyone who will hold you to it.

## Orientation

Here is the sentence to carry through the whole module: **each Azure messaging and data service is a contract — a specific set of guarantees about delivery, ordering, retention, consistency and throughput, sold in a specific capacity unit. The architect's job is to give every flow and every dataset the service whose guarantees are native to it, then size, partition, secure and replicate that service for the workload's whole life.**

Module 26 ended with a claim worth repeating here: *compute is usually the least locked-in layer of an Azure system; the data and messaging services are where portability really gets decided.* This module is about that most committed layer. You can move a container from App Service to Container Apps in a week. Moving an order history out of Cosmos DB, or rewiring forty services from Service Bus topics to Kafka, is a quarter of work and a migration risk. So these choices deserve more deliberation than any compute choice — and interviewers know it.

This module also sits on a lot of earlier theory, and deliberately doesn't repeat it:

- **Module 11** taught messaging from first principles — delivery semantics, idempotency, the outbox, queues vs logs — and introduced Service Bus, Event Hubs and Event Grid as brokers.
- **Module 7** derived Cosmos DB's five consistency levels and session tokens from the consistency spectrum.
- **Module 8** covered partitioning, hot keys, hierarchical partition keys and the partition-aware `CosmosClient`.
- **Module 12** covered Cosmos data modeling, the 20 GB / 10,000 RU/s ceilings, and single-table design.
- **Module 5** gave you the limits table and the partition-count formula.
- **Modules 24, 25 and 26** covered event sourcing, SDK resilience composition and the compute that hosts consumers.

What those modules did *not* do is treat these services as **platforms you operate**: how capacity is bought and sized, how topologies are designed, what each service does during a zone or region failure, which knobs in the .NET SDK actually matter, how the services compose into a system, and what all of it costs. That's this module.

Why it matters in an interview: Azure-heavy loops almost always contain a question of the form *"Service Bus, Event Grid or Event Hubs — and why?"*, followed by a Cosmos DB deep dive. Weak answers recite feature lists. Strong answers name the **contract** each flow needs, size the **capacity unit** with numbers, explain **what happens on failure** (lock expiry, checkpoint replay, partition failover), and know **what changed recently**. Current loops ask: *"Why does our Event Grid handler see the same event three times?"*, *"Our Cosmos container throttles at 30% utilization — why?"*, *"How would you make this pipeline survive a region outage, and what's the RPO?"*, *"Would you use the new distributed transactions or a saga?"*, *"We still have `Microsoft.Azure.ServiceBus` in production — what now?"*, and *"How do you get events out of Cosmos reliably?"*

This module has eight jobs:

1. **Build a first-principles model of the platform** — four services as four contracts, a guarantee matrix, partitions as the universal unit, capacity units, the namespace/account as a boundary, and push vs pull — so every product decision is a point in a space.
2. **Teach Service Bus as an operated platform** — Premium capacity and messaging units, entity topology design, message lifecycle settings, sessions at scale, SDK throughput tuning, transactions, multi-region options and the September 2026 retirement.
3. **Teach Event Grid precisely** — Basic vs namespaces, system topics, push delivery mechanics and retry, pull delivery, the MQTT broker, domains, and the traps that come from "it's not a queue."
4. **Teach Event Hubs as an operated platform** — tiers, partition and capacity sizing, producers, processors and checkpoints, the Kafka endpoint's real compatibility, Capture and Schema Registry, and geo-replication.
5. **Teach Cosmos DB as an operated platform** — account-level one-way doors, RU economics, throughput modes, partition operations, GSIs, consistency in production, transactions old and new, the change feed as an integration backbone, multi-region and PPAF, backup, SDK configuration and security.
6. **Compose the services** — canonical topologies, claim check, ordering and idempotency end to end, backpressure across a pipeline, and multi-region and security baselines across all four.
7. **Make .NET do it right** — client lifetime and DI, Aspire integrations and emulators, hosting consumers, retries without multiplication, contracts and serialization, observability and testing.
8. **Make the decision defensible** — decision axes, capacity and cost arithmetic, hidden costs, lifecycle risk, anti-patterns, and a review checklist.

Seven framings to carry through:

1. **Every service is a contract.** Read a service by what it guarantees — delivery, order, retention, consistency, maximum size, replay — and, equally, by what it explicitly does not.
2. **Partitions are the unit of everything.** In all four services, the partition decides ordering, parallelism, throughput ceilings and hot spots. Choose partition keys as deliberately as you'd choose a primary key.
3. **Capacity is a unit you buy.** Messaging units, throughput/processing/capacity units, Event Grid throughput units and RU/s are all the same idea: a slice of a server. Size them from the workload, with headroom for the failure you're designing for.
4. **Push vs pull decides who controls the pace.** A pushed consumer must keep up or be retried; a pulling consumer sets its own rate. Backpressure design starts with which one you have.
5. **"Multi-region" is a different feature in every service** — with a different RPO, RTO and promotion model. Know which is which, and compose them deliberately.
6. **Identity and network are per-service decisions.** Data-plane RBAC roles, disabling key/SAS authentication, private endpoints — each service has its own version, and each must be set.
7. **Compose by strengths.** Event Grid routes facts, Service Bus carries commands and workflows, Event Hubs carries streams, Cosmos DB stores state and emits its own change stream. Most designs that feel awkward are using one service for another's job.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Four services, four contracts | Router, broker, log and database — four different promises, not four flavors of one |
| 2 | The guarantee matrix | Delivery, ordering, retention, consumption model, size and replay, side by side |
| 3 | Partitions everywhere | The partition is the unit of order, parallelism, throughput and hot spots in all four |
| 4 | Capacity units | MU, TU/PU/CU, Event Grid TU and RU/s are all slices of servers; per-operation billing sits on top |
| 5 | The namespace and the account | The boundary for quotas, networking, identity, failover and blast radius |
| 6 | Push vs pull | Who controls the pace — and therefore where backpressure lives |
| 7 | Service Bus in 2026 | Basic, Standard, Premium — and the end of the legacy SDKs and SBMP |
| 8 | Premium capacity | Messaging units 1–16, CPU as the scaling signal, autoscale, partitioned namespaces |
| 9 | Entity topology design | Queues, topics, subscriptions, filters, forwarding — and how to shape them for a domain |
| 10 | Message lifecycle settings | Lock duration, max delivery count, TTL, dead-letter reasons, auto-delete, duplicate detection |
| 11 | Sessions at scale | Ordered, exclusive processing per key — and its throughput price |
| 12 | Throughput engineering in the SDK | One client, cached senders, batching, prefetch and concurrency — tuned together |
| 13 | Transactions and the outbox relay | Send-via, MessageId + duplicate detection, and the relay that feeds the broker |
| 14 | Service Bus reliability | Zones, Geo-DR (metadata), Geo-Replication (data) — three different promises |
| 15 | Service Bus security | Data-plane roles, local auth off, private endpoints, CMK |
| 16 | Service Bus traps | Lock loss, retry multiplication, giant topics, connection churn, the retired SDKs |
| 17 | Event Grid in 2026 | Basic push topics and Standard namespaces; CloudEvents as the envelope |
| 18 | System topics and resource events | Azure telling you what happened, for free, with subject filtering |
| 19 | Push delivery mechanics | Validation, 30-second responses, exponential retry for 24 hours, dead-letter to Blob |
| 20 | Namespaces and pull delivery | Queue-like receive, acknowledge, release and reject over HTTP |
| 21 | The MQTT broker | Devices publishing and subscribing, routed into the rest of Azure |
| 22 | Domains, fan-out and limits | Multi-tenant publishing, partner events, throughput units, operation counts |
| 23 | Event Grid traps and fit | Not a queue, no ordering, duplicates by design, fan-out multiplies cost |
| 24 | Event Hubs tiers in 2026 | Basic, Standard, Premium, Dedicated — partitions, retention, capacity per tier |
| 25 | Sizing partitions and capacity | max(ingress ÷ 1, egress ÷ 2) partitions; TUs from MB/s and events/s; headroom for consumers |
| 26 | Producers | Partition keys vs round robin, batching, the buffered producer, ordering costs |
| 27 | Processors and checkpoints | Ownership, load balancing, checkpoint cadence, lag, one reader per partition |
| 28 | The Kafka endpoint | Real Kafka clients, real gaps: transactions preview, compression tiers, config gotchas |
| 29 | Capture, Schema Registry, stream processing | Archive without code, schemas with governance, Stream Analytics and Fabric |
| 30 | Event Hubs reliability | Zones, Geo-DR metadata pairing, Geo-Replication with replicated offsets |
| 31 | Event Hubs traps and fit | Fixed partitions on Standard, tiny events, swallowed failures, checkpoint storms |
| 32 | Cosmos DB as a platform | What earlier modules established, and what operating it adds |
| 33 | Account-level one-way doors | API, tier, capacity mode, regions, zones, consistency, backup — decide them early |
| 34 | RU economics | Reads, writes, queries and indexes priced in one currency you can measure per request |
| 35 | Throughput modes | Manual, autoscale, dynamic scaling, serverless, shared, burst, buckets and priority |
| 36 | Partition operations | Splits, RU distribution, hot-partition diagnosis, redistribution, merge, partition key change |
| 37 | Hierarchical partition keys in practice | Prefix routing, `/id` as the last level, and what HPK rules out |
| 38 | Global secondary indexes | Auto-synced read-only containers keyed differently — at a write-cost premium |
| 39 | Consistency in production | Account default, per-request levels, session tokens across instances, measured staleness |
| 40 | Transactions | Batch, ETags, patch, stored procedures — and the new distributed transactions |
| 41 | The change feed as a backbone | Latest-version vs all-versions-and-deletes, processor leases, estimator, the Cosmos outbox |
| 42 | Multi-region and failover | Read regions, PPAF, forced vs service-managed failover, multi-region writes and conflicts |
| 43 | Backup and restore | Periodic vs continuous, point-in-time restore into a new account, Azure Backup preview |
| 44 | The .NET SDK, configured | Singleton, direct mode, preferred regions, hedging, bulk, STJ, diagnostics, 429 retries |
| 45 | Cosmos security | Data-plane RBAC, key auth off, private endpoints, CMK |
| 46 | Analytics, search and AI | Fabric mirroring, vector and full-text search, and what not to run on the OLTP container |
| 47 | Cosmos traps | Per-request clients, cross-partition everything, default indexing, storage-driven throughput floors |
| 48 | The rest of the data menu | Azure SQL, PostgreSQL, Azure DocumentDB, Managed Redis, Table Storage — when Cosmos isn't it |
| 49 | Choosing per flow | Fact, command or stream? Then the service whose contract is native |
| 50 | Canonical topologies | Grid→Bus buffering, change feed→Bus outbox, Hubs→processing→Cosmos, MQTT→Grid→Hubs |
| 51 | Claim check | Large payloads in Blob, references in messages |
| 52 | Ordering end to end | Where order survives a pipeline, and where it silently dies |
| 53 | Idempotency end to end | Message IDs, broker dedup windows, and Cosmos as the dedup store |
| 54 | Backpressure across a pipeline | Who throttles whom: 429s, max concurrency, lag, and the slowest stage |
| 55 | Multi-region across the platform | Composing four different DR features into one RPO/RTO story |
| 56 | The security baseline | One table: identity, network, keys and policy for all four services |
| 57 | Clients, DI and Aspire | `Microsoft.Extensions.Azure`, singletons, credentials, Aspire integrations and emulators |
| 58 | Hosting consumers | Processor, event processor and change feed processor as `BackgroundService`s |
| 59 | Retries without multiplication | SDK retry options, Polly, 429s, and one retry owner per failure |
| 60 | Contracts and serialization | CloudEvents, schema registry, STJ source generation, tolerant readers |
| 61 | Observability per service | The handful of metrics that matter, and SDK traces through OpenTelemetry |
| 62 | Testing | Emulators, Testcontainers, Aspire topology, and failover drills |
| 63 | Decision axes | The questions that separate the services, and which ones decide |
| 64 | Capacity and cost, worked | MU, TU vs PU, Event Grid operations, Cosmos manual vs autoscale vs serverless |
| 65 | Hidden costs | Replicated capacity, regions × RU, GSIs, change feed reads, private endpoints, logs |
| 66 | Lifecycle risk | Retirements and renames as architecture inputs |
| 67 | Anti-patterns | The mistakes that make the platform the problem |
| 68 | The platform decision review, and when not to | A checklist to narrate, and the restraint that signals seniority |

---

# Part A — First principles: the platform as a set of contracts

## Concept 1 — Four services, four contracts

Strip away the product pages and the four services in this module are four fundamentally different kinds of system. Treating them as four flavors of "messaging" is the root of most bad designs.

| Service | What it fundamentally is | The question it answers | The unit you reason about |
|---|---|---|---|
| **Event Grid** | An **event router**: it receives small notifications and delivers each to every interested subscriber | "Who needs to know that *this* happened?" | The event and its subscriptions |
| **Service Bus** | A **message broker**: durable queues and topics with per-message locks, settlement and dead-lettering | "Who will *do* this piece of work, exactly once in effect?" | The message and its lock |
| **Event Hubs** | A **partitioned append-only log**: high-throughput ingestion that consumers read at their own offsets and can replay | "What is the *stream* of everything that happened, in order per key?" | The partition and the offset |
| **Cosmos DB** | A **partitioned, replicated database** that also exposes its own change log | "What is the *current state*, and how did it change?" | The item, the logical partition and the RU |

Three consequences follow, and they're the backbone of every later decision:

**1. The unit of *consumption* differs.** In Service Bus the broker tracks each message: it's locked to one receiver, completed, abandoned or dead-lettered. In Event Hubs the broker tracks nothing per event: the consumer owns a position (an offset) per partition, and "processing" a bad event means *moving past it*. In Event Grid Basic the broker tracks each *delivery attempt* to each subscriber and retries until it gets a success or gives up. Cosmos's change feed is like Event Hubs: a per-partition log that consumers read with their own continuation state (leases). Knowing which model you have tells you where failure handling lives: in the broker (Service Bus, Event Grid) or in your code (Event Hubs, change feed).

**2. The relationship between producer and consumer differs.** A Service Bus message is usually a **command or a work item** — "reserve stock for order 42" — with an implied owner. An Event Grid event is a **notification of a fact** — "blob X was created" — with no assumption about who cares. An Event Hubs event is a **record in a stream** — "temperature at 12:00:01 was 21.4" — where the value is in the aggregate. Module 11 called these commands, events and stream records; the services are optimized for exactly one each.

**3. Their economics differ.** Service Bus and Event Hubs sell you **capacity** (Standard and Basic add per-operation charges); Event Grid Basic sells you **operations**; Cosmos sells you **throughput (RU/s) and storage**. A design that pushes a million tiny telemetry events per second through Service Bus is wrong technically *and* financially; a design that uses Event Grid as a work queue is wrong technically *and* will surprise you on delivery semantics.

A fifth "service" deserves mention because it's often the right answer: **Azure Storage queues** — a cheap, simple, durable queue with 64 KB messages, visibility timeouts and no ordering, topics or sessions. Module 11 (Concept 35) covered it; here it's the baseline that Service Bus has to beat.

**The interview-grade sentence:** *"I treat the four services as four contracts: Event Grid is a router for facts, Service Bus is a broker for commands and workflows with per-message locks and dead-lettering, Event Hubs is a partitioned log for streams that consumers replay from their own offsets, and Cosmos DB is a partitioned database with its own change log. Most awkward designs come from using one of them for another's job."*

---

## Concept 2 — The guarantee matrix

Before choosing, put the contracts side by side. This table is worth being able to reproduce from memory, at least in shape:

| Property | Event Grid Basic | Event Grid namespaces | Service Bus | Event Hubs | Cosmos DB change feed |
|---|---|---|---|---|---|
| **Delivery** | At-least-once push, retried for up to 24 h | At-least-once; pull (queue-like) or push | At-least-once (peek-lock); at-most-once (receive-and-delete) | At-least-once from the consumer's checkpoint | At-least-once from the processor's lease |
| **Ordering** | None | None across events | FIFO per **session**; best-effort otherwise | Strict per **partition** | Per **logical partition key** |
| **Consumption model** | Push to handlers | Pull with lock/acknowledge, or push | Pull (AMQP long-lived links) with locks | Pull by offset, per consumer group | Pull by continuation, per lease |
| **Per-message failure handling** | Retry schedule, then dead-letter to Blob | Release/reject; max delivery count; dead-letter | Abandon, defer, dead-letter queue, max delivery count | None — the consumer must park failures | None — the consumer must park failures |
| **Retention** | Until delivered or TTL (≤ 24 h) | Up to 7 days in a namespace topic | Until consumed or TTL expires | Time-based: 1 / 7 / 90 days by tier (or compaction) | As long as the item exists (latest-version); bounded window for all-versions-and-deletes |
| **Replay** | No | No (beyond redelivery) | No (once completed it's gone) | **Yes** — any consumer group from any retained offset | **Yes** — from the beginning, a point in time, or "now" |
| **Max size** | 1 MB per event (billed per 64 KB) | 1 MB | 256 KB (Standard); 1 MB default, up to 100 MB (Premium) | 1 MB per event (256 KB on Basic) | 2 MB per item |
| **Fan-out** | Built in: many subscriptions | Many event subscriptions per topic | Topics + up to 2,000 subscriptions per topic | Consumer groups (20 on Standard, 100 on Premium) | Multiple processors with separate lease containers |
| **Filtering at the service** | Event type, subject prefix/suffix, advanced filters | Event type, advanced filters | SQL and correlation filters per subscription | None | None (filter in the consumer, or use a GSI) |
| **Transactions** | No | No | Within a namespace, including send-via | Kafka transactions (preview, Premium/Dedicated) | Single partition (batch); cross-partition in preview |

Three reading notes:

- **"At-least-once" is universal.** There is no exactly-once delivery anywhere in this table — only exactly-once *effects*, built from at-least-once delivery plus idempotent handling (Module 11, Concept 9). Interviewers listen for whether you say this unprompted.
- **Ordering always has a scope.** "Is it ordered?" is the wrong question; "ordered *per what*?" is the right one. Session, partition, logical partition key — and nothing across them.
- **Replay is a property of logs, not queues.** If a new consumer next quarter will need last month's data, that requirement alone pushes you toward Event Hubs (with long retention or Capture) or Cosmos's change feed, not Service Bus.

**The interview-grade sentence:** *"Side by side, all four are at-least-once; ordering exists only per session in Service Bus, per partition in Event Hubs and per partition key in the change feed; per-message failure handling lives in the broker for Service Bus and Event Grid and in my code for the logs; and only the logs — Event Hubs and the change feed — can be replayed. Those four properties usually choose the service before features do."*

---

## Concept 3 — Partitions everywhere

Module 8 established that a partition is the unit of **ordering**, **parallelism**, **throughput** and **failure**. All four services in this module are partitioned, and the same four consequences appear in each — with different names and different degrees of control.

| Service | The partition | Who chooses the key | What a partition bounds |
|---|---|---|---|
| **Event Hubs** | A partition of an event hub (a separate ordered log) | The producer: partition key, explicit partition ID, or round robin | Ordering scope; one active reader per consumer group; ~1 MB/s in / ~2 MB/s out on Standard |
| **Service Bus** | An internal partition of a partitioned entity (Standard) or namespace (Premium) — the broker's choice | The broker hashes `PartitionKey`, `SessionId` or `MessageId`; otherwise spreads | Ordering (sessions pin to one partition); a partition's own throughput |
| **Event Grid** | Internal and invisible | The service | Nothing you control — which is why it gives no ordering |
| **Cosmos DB** | A **logical** partition (all items with one key value) inside a **physical** partition (a replica set) | You, at container creation — the most important decision in the account | 20 GB per logical partition; 10,000 RU/s per physical partition; transaction scope; change-feed ordering scope |

The patterns that follow:

**Ordering and parallelism are the same knob turned in opposite directions.** More partitions (or sessions, or partition-key values) means more parallel consumers and less global order. One partition means perfect order and one consumer. Every design either picks a key whose scope matches the ordering requirement exactly — per order, per device, per tenant — or pays for over-ordering in throughput.

**The hottest key sets the ceiling, not the average.** A tenant producing 30% of all events lands 30% of the traffic on one Event Hubs partition, one Service Bus session, or one Cosmos logical partition — and every one of those has a fixed per-partition ceiling. Adding partitions doesn't help a hot *key*; only changing the key (adding a suffix, a hierarchy, or a time bucket) does (Module 5, Concept 23).

**Partition counts are sometimes one-way doors.** Event Hubs Standard partitions are **immutable** after creation; Premium and Dedicated can add partitions but not remove them (and adding them changes the key-to-partition mapping for new events). A Service Bus Premium namespace's partitioning is chosen **at namespace creation**. A Cosmos partition key could not be changed at all until the partition-key-change feature reached GA in June 2026 — and even now it's a migration, not a setting. Treat each as a schema decision.

**Cross-partition operations cost more everywhere.** A Cosmos query without the partition key fans out to every physical partition (roughly 2–3 extra RU per partition plus latency). An Event Hubs consumer that needs global order must re-sort across partitions. A Service Bus transaction can't span partitions of a partitioned entity in the way you'd hope. The platform is cheapest when the partition key is present in the dominant operation.

**The interview-grade sentence:** *"In all four services the partition is the unit of ordering, parallelism and throughput, so the partition key is a schema decision: ordering only exists within it, the hottest key sets the ceiling, cross-partition work costs more, and the counts or keys are often one-way doors — immutable on Event Hubs Standard, fixed at creation for a Service Bus Premium namespace, and a migration even now for Cosmos."*

---

## Concept 4 — Capacity units: buying slices of servers

Every service in this module sells capacity in a unit that, underneath, is a slice of CPU, memory, disk and network on shared or dedicated machines. Reading the unit correctly is how you size, budget and predict throttling.

| Service / tier | Capacity unit | What one unit roughly gives you | Billing on top of capacity | Scaling the unit |
|---|---|---|---|---|
| **Service Bus Basic/Standard** | None (shared) | Shared multi-tenant capacity with throttling under contention | Base charge (Standard) + per-million operations | Not applicable — throttled when the shared cluster is busy |
| **Service Bus Premium** | **Messaging unit (MU)** — 1, 2, 4, 8 or 16 per namespace | Dedicated, isolated CPU and memory; throughput depends heavily on message size and pattern | None per operation | Manual or autoscale on the namespace CPU metric |
| **Event Hubs Basic/Standard** | **Throughput unit (TU)** — up to 40 | Ingress 1 MB/s **or** 1,000 events/s (whichever first); egress 2 MB/s or 4,096 events/s; 84 GB of retention storage | Per-million ingress events (metered in 64 KB units) | Manual, or **auto-inflate** (scales up only, never down) |
| **Event Hubs Premium** | **Processing unit (PU)** — up to 16 | Isolated resources; guidance of roughly 5–10 MB/s ingress per PU; 1 TB retention per PU | None per event | Manual |
| **Event Hubs Dedicated** | **Capacity unit (CU)** | Single-tenant cluster capacity; 10 TB retention per CU | None per event | Cluster scaling |
| **Event Grid Basic** | None | Shared, regional | Per-million operations (first 100,000 per month free) | Not applicable |
| **Event Grid namespaces** | **Throughput unit (TU)** | Per TU, about 1,000 publish requests/s or 1 MB/s inbound, and up to 2,000 events/s or 2 MB/s egress; 10,000 MQTT sessions | Per-million operations | Manual |
| **Cosmos DB provisioned** | **RU/s** per container or shared database | A normalized blend of CPU, IOPS and memory per second; 10,000 RU/s per physical partition | Storage per GB; regions multiply the RU charge | Manual, autoscale, dynamic scaling (Concept 35) |
| **Cosmos DB serverless** | None provisioned | Burst capacity that scales with the number of physical partitions | Per-million RUs consumed + storage | Automatic, no guarantees |

Three ideas make this table useful rather than trivia:

**1. Capacity units are where throttling comes from.** "Server busy" from Service Bus, "quota exceeded" from Event Hubs, HTTP 429 from Cosmos DB — each is the service telling you that the demand in some interval exceeded the unit you bought (or the share of a shared cluster you get). A design isn't sized until you can say **which unit will throttle first under peak load** and what the client does when it happens.

**2. "Per-operation" and "per-capacity" billing reward different behavior.** On Service Bus Standard every send, receive, renew, peek and abandon is an operation, so chatty clients and long polling choices show up on the bill. On Event Hubs Standard events are metered in **64 KB** units, so 10,000 events of 100 bytes cost the same ingress meter as 10,000 events of 64 KB — batching small events into larger ones is a cost lever. On Premium tiers, operations are free but capacity is not: the question becomes utilization.

**3. Size for the failure, not just the peak.** If a Cosmos account loses a region and its traffic moves to the remaining one, that region needs the capacity for both. If a Service Bus Premium namespace is geo-replicated, the secondary runs **the same number of MUs** as the primary — and you pay for them. If an Event Hubs consumer group falls behind during an incident, catching up needs egress headroom above steady state. Capacity planning is peak × (failure scenario) × (catch-up factor), not average × 1.

**The interview-grade sentence:** *"Every service sells a slice of servers — messaging units on Service Bus Premium, throughput, processing or capacity units on Event Hubs, throughput units on Event Grid namespaces, RU/s on Cosmos — and throttling is just demand exceeding that slice. So I size from the workload, name which unit throttles first, account for per-operation or 64 KB metering where it applies, and add headroom for the failure scenario and catch-up, not just the peak."*

---

## Concept 5 — The namespace and the account: boundaries that matter

Each service has a top-level resource — a **namespace** (Service Bus, Event Hubs, Event Grid namespaces), a **topic** or **domain** (Event Grid Basic), or an **account** (Cosmos DB). It's easy to treat it as a folder. It isn't. It's the boundary for most of the decisions that matter operationally:

| Boundary | Service Bus namespace | Event Hubs namespace | Event Grid namespace / topic | Cosmos DB account |
|---|---|---|---|---|
| **Tier and capacity** | Tier; MUs on Premium | Tier; TUs/PUs (or a Dedicated cluster) | Tier; TUs on namespaces | Capacity mode (provisioned vs serverless); service tier |
| **Quotas** | Entity counts, connections, filters | Event hub count, partitions, consumer groups | Topics, subscriptions, TUs | Containers, throughput limits, regions |
| **Network** | Public access, IP rules, private endpoints | Same | Same | Same, plus region list |
| **Identity** | Local (SAS) auth on/off; RBAC scope | Same | Same (access keys / Entra) | Key auth on/off; data-plane RBAC scope |
| **Encryption** | CMK (Premium) | CMK (Premium/Dedicated) | — | CMK (account-wide, at creation for some settings) |
| **Failover** | Geo-DR alias or Geo-Replication — **the whole namespace** | Geo-DR or Geo-Replication — the whole namespace | Regional; you design multi-region | Regions, failover type, PPAF — **the whole account** |
| **Blast radius** | A noisy entity can consume a Premium namespace's MUs | A busy event hub can consume a namespace's TUs | Shared regional infrastructure | Account-level control-plane operations and settings |

Design rules that come out of the table:

- **Split namespaces by capacity isolation and failover unit, not by team politics.** If the payments flows must fail over independently of telemetry, or must never be starved by telemetry, they need different namespaces. Two workloads with different DR requirements in one namespace inherit the stricter one's cost.
- **Don't over-split either.** Every namespace or account is another private endpoint, another set of role assignments, another alert set, another capacity line. A common, sane layout: one Service Bus namespace per *bounded context group* and environment, one Event Hubs namespace per *stream domain* (telemetry vs CDC), one Cosmos account per *product* and environment with multiple containers.
- **Account-level settings are global.** In Cosmos, the default consistency level, the region list, the failover configuration, the backup mode, the service tier and features like dynamic scaling or PPAF are **account-wide**. Two applications that need different answers need different accounts.
- **Use the namespace as the tenancy boundary only when it must be.** Per-tenant namespaces or accounts give the strongest isolation and the worst operational scaling. Multitenant guidance from Microsoft (for Service Bus, Event Hubs and Cosmos) describes the spectrum: shared entity with tenant key → entity per tenant → namespace/account per tenant. Most SaaS designs live in the first two and reserve the third for large or regulated tenants (Module 8's cells).

**The interview-grade sentence:** *"The namespace or account isn't a folder — it's the boundary for tier and capacity, quotas, networking, identity, encryption and failover. So I split by capacity isolation and DR unit — payments must not fail over with, or be starved by, telemetry — keep account-wide Cosmos settings like consistency and regions in mind, and avoid splitting further than those reasons justify because every boundary adds endpoints, roles and alerts."*

---

## Concept 6 — Push vs pull: who controls the pace

The last first principle decides where backpressure lives.

**Pull** (Service Bus, Event Hubs, Event Grid namespace pull delivery, the Cosmos change feed): the consumer asks for work when it's ready. If it slows down, work accumulates **in the broker** — queue depth, consumer lag — which is visible, durable, and the natural input to autoscaling (KEDA, Functions target-based scaling: Module 26, Concepts 20 and 31). The consumer controls concurrency directly.

**Push** (Event Grid Basic, Event Grid namespace push, webhooks in general): the service decides when to deliver. If the handler slows down, the service sees failures or timeouts and **retries with backoff** — the handler's slowness becomes retry traffic. Backpressure exists, but it's indirect, and the handler must be able to absorb the arrival rate or fail cleanly. Push is also the only model that needs the consumer to be **reachable** (a public endpoint, or an Azure service destination).

Consequences that interviewers probe:

1. **Push into pull is the standard buffering move.** Route Event Grid events into a **Service Bus queue** or an **Event Hub** (both are native Event Grid destinations) and consume from there. You keep Event Grid's routing and filtering, and gain a durable buffer your consumer drains at its own pace — with dead-lettering, sessions or replay as needed. Event Grid namespaces' pull delivery achieves something similar inside Event Grid itself.
2. **Push needs fast, idempotent acknowledgment.** A handler that does 20 seconds of work before returning 200 will see timeouts, retries and duplicates under load. The robust push handler validates, records the event (or enqueues it), returns success, and does the work asynchronously.
3. **Pull needs bounded concurrency.** A pull consumer can ask for as much as it wants — which is how a Service Bus processor with `MaxConcurrentCalls = 200` across 30 replicas becomes 6,000 concurrent database calls. The pace you control is the pace you must limit (Concept 54).
4. **Scaling signals differ.** Pull models scale on **backlog** (active messages, unprocessed events, change-feed estimator lag). Push models scale on **request concurrency** at the handler, which is a weaker signal of how much work is waiting.

**The interview-grade sentence:** *"Pull lets the consumer set the pace and keeps backlog visible in the broker, which is the right input for autoscaling; push makes the service set the pace and turns slowness into retries, so push handlers must acknowledge fast and idempotently. The standard move is to route pushed events into a Service Bus queue or an Event Hub and pull from there — and with pull, I have to bound concurrency myself because nothing else will."*

---
# Part B — Azure Service Bus as an operated platform

## Concept 7 — Service Bus in 2026: tiers, and the end of the legacy SDKs

Module 11 (Concept 32) introduced the feature set: queues, topics and subscriptions, peek-lock settlement, sessions, scheduled messages, deferral, duplicate detection, dead-lettering, transactions and auto-forwarding. Here's the operational picture that sits on top.

**Three tiers, two worlds:**

| | **Basic** | **Standard** | **Premium** |
|---|---|---|---|
| Entities | Queues only | Queues, topics, subscriptions | Queues, topics, subscriptions |
| Features | No sessions, no transactions, no duplicate detection, no topics | Sessions, transactions, duplicate detection, filters, forwarding | All of Standard plus JMS 2.0, large messages, CMK, Geo-DR and Geo-Replication |
| Max message size | 256 KB | 256 KB | 1 MB default, up to **100 MB** over AMQP |
| Capacity | Shared | Shared, with throttling under contention | **Dedicated messaging units** (isolated CPU/memory) |
| Networking | Public (IP rules) | Public (IP rules) | **Private endpoints**, VNet rules |
| Availability zones | Yes (where the region supports them) | Yes | Yes |
| Billing | Per operation | Base charge + per operation | Per messaging-unit hour |
| Latency predictability | Variable | Variable under neighbor load | Predictable for a given MU load |

The architecture rule of thumb: **Standard for development, low-volume integration and cost-sensitive work where latency variance is acceptable; Premium for production systems that need predictable latency, private networking, large messages, or any of the multi-region features.** The jump in price is large (from roughly $10 a month plus operations to hundreds of dollars per MU per month), so the decision is real — but "production, private networking required" alone settles it for most enterprises.

**The retirement that just happened.** On **September 30, 2026**, Microsoft retired the `WindowsAzure.ServiceBus` and `Microsoft.Azure.ServiceBus` libraries (and Java's `com.microsoft.azure.servicebus`) and **ended support for the SBMP protocol**, which only `WindowsAzure.ServiceBus` on .NET Framework used. Practically:

- Code on **SBMP** (the old `WindowsAzure.ServiceBus` with its default `NetMessaging` transport) **no longer works** against the service.
- Code on **`Microsoft.Azure.ServiceBus`** keeps working over AMQP for now but receives **no support or updates** — an unpatched dependency, and a finding in any security review.
- Event Hubs code still using `WindowsAzure.ServiceBus` is affected too — it migrates to `Azure.Messaging.EventHubs`.
- Azure Functions apps must be on the **Service Bus extension 5.x**, which is built on `Azure.Messaging.ServiceBus` (bindings to `ServiceBusReceivedMessage` and `ServiceBusMessageActions`).

The migration is mostly mechanical — one `ServiceBusClient` per process creating senders, receivers and processors — with two places that bite: **serialization** (the old `BrokeredMessage` used a DataContract binary serializer for object bodies; the new library carries raw bytes, so you must deserialize those old payloads explicitly while both versions run) and **settlement** (the processor's auto-complete behavior and lock renewal options differ). Microsoft's migration guides for both legacy libraries are in the resources.

**The interview-grade sentence:** *"Service Bus has three tiers: Basic for simple queues, Standard for the full feature set on shared capacity, and Premium for dedicated messaging units, private endpoints, 100 MB messages, CMK and the geo features — which is where production enterprise workloads land. And as of September 30, 2026 the legacy WindowsAzure.ServiceBus and Microsoft.Azure.ServiceBus libraries are retired and SBMP is gone, so anything still on them is either broken or unsupported and moves to Azure.Messaging.ServiceBus, with care around old DataContract-serialized bodies."*

---

## Concept 8 — Premium capacity: messaging units, CPU, autoscale and partitioned namespaces

A Premium namespace is a slice of dedicated broker infrastructure measured in **messaging units (MUs)**: **1, 2, 4, 8 or 16** per namespace. Each MU is isolated CPU and memory; nothing else in Azure shares it.

**There is no fixed "messages per second per MU."** Throughput depends on message size, sessions, filters, transactions, the number of entities, receive mode, prefetch and batching. Microsoft deliberately publishes guidance rather than a number, and tells you to benchmark your own pattern. What you *can* rely on is the signal: **the namespace's CPU usage metric** (and memory usage). It's the Premium equivalent of RU utilization.

**Sizing method:**

1. **Benchmark one MU** with your real message size, entity layout and client settings, at increasing load, until CPU sustains around 70–75% or latency degrades. That's your per-MU throughput for *this* pattern.
2. **Divide peak demand** by that number, round up to the next valid MU count, and add headroom — at least for catch-up after an outage (a backlog drains only as fast as spare capacity allows).
3. **Configure autoscale** on the namespace: Azure Monitor autoscale rules on CPU (scale up around 70–75%, down around 20–25%, asymmetric cool-downs) between a minimum and maximum MU count. Scaling MUs is an online operation.
4. **Re-benchmark when the pattern changes** — introducing sessions, large messages or complex SQL filters can halve per-MU throughput.

**Partitioned Premium namespaces.** Premium supports **partitioning at the namespace level**, chosen when the namespace is created: the namespace is split into a number of partitions (1, 2 or 4), the messaging units are distributed across them, and **every queue and topic in the namespace is partitioned**. You can't change it later. Partitioning raises the ceiling for very high throughput and improves availability (a failing internal partition affects only its share), but adds constraints familiar from Standard partitioned entities: transactions and sessions must stay within one partition (the broker uses `PartitionKey`/`SessionId` to place them), and ordering across partitions is lost unless you use sessions. Most workloads don't need it; the ones that do usually know from the benchmark.

**Standard tier capacity, by contrast,** is shared. You can't buy more of it; you get throttled (`ServerBusyException` / `ServiceBusFailureReason.ServiceBusy`) when the shared cluster is under pressure or you exceed per-namespace quotas, and the client backs off. Standard *partitioned entities* (set at entity creation) spread an entity across multiple brokers and message stores for throughput and availability.

**Quotas that shape designs** (check the quotas page — they change):

| Quota | Basic / Standard | Premium |
|---|---|---|
| Queues + topics per namespace | 10,000 | 1,000 per MU |
| Subscriptions per topic | 2,000 | 2,000 |
| SQL filters per topic | 2,000 | 2,000 |
| Correlation filters per topic | 100,000 | 100,000 |
| Message size | 256 KB | 1 MB default; up to 100 MB |
| Lock duration | up to 5 minutes | up to 5 minutes |

**The interview-grade sentence:** *"Service Bus Premium is sold in messaging units — 1, 2, 4, 8 or 16 — with no published messages-per-second figure because throughput depends on size, sessions, filters and batching, so I benchmark one MU with the real pattern, watch the namespace CPU metric, size peak plus catch-up headroom, and autoscale MUs on CPU. Namespace-level partitioning is a creation-time choice that raises the ceiling but scopes transactions and ordering to a partition, so I only use it when a benchmark says I need it."*

---

## Concept 9 — Entity topology design

Topology is the Service Bus equivalent of a schema: queues, topics, subscriptions, rules and forwarding chains. It outlives the code that created it, so design it on purpose.

**Queue or topic?**

- A **queue** is point-to-point: one logical consumer (possibly many competing instances) per message. Use it for **commands** — "charge card for order 42" — that have exactly one owner.
- A **topic** with **subscriptions** is publish-subscribe: each subscription gets its own copy and behaves like a queue for its consumers. Use it for **events** — "order 42 placed" — that several bounded contexts react to independently.

A good default is: **commands to queues owned by the receiving service; events to topics owned by the publishing service, one subscription per consuming service.** Ownership matters for evolution: the publisher owns the topic's existence and the event contract; each consumer owns its subscription, its rules and its dead-letter queue.

**How many topics?** Three common layouts, each with a real trade-off:

| Layout | Example | Pros | Cons |
|---|---|---|---|
| **Topic per event type** | `order-placed`, `order-cancelled` | Simple subscriptions (no filters), clear ownership, per-type metrics | Many entities; consumers needing several types need several subscriptions; order across types isn't preserved |
| **Topic per publisher / bounded context** | `orders-events` with a `type` property | Few entities; one subscription can receive several types with filters; sessions can order per aggregate across types | Every subscription needs filters; a firehose topic can grow huge; per-type visibility needs properties in metrics |
| **One giant shared topic** | `all-events` | Looks simple on day one | Filter explosion, noisy neighbor between flows, unclear ownership — Module 11's "giant shared topic" anti-pattern |

For most .NET systems, **topic per bounded context with a correlation filter per subscription** is the pragmatic middle, and it's also the shape most frameworks (MassTransit, NServiceBus, Wolverine) produce or support.

**Filters and rules.** Each subscription has one or more **rules**; each rule has a **filter** and an optional **action**:

- **Correlation filters** match on system and user properties by equality (`Subject`, `CorrelationId`, `ContentType`, user properties). They're evaluated with hashing — cheap even at scale — and the namespace allows up to 100,000 per topic. **Prefer them.**
- **SQL filters** evaluate SQL-92-like expressions (`Priority > 3 AND Region = 'EU'`). They're more expressive and **more CPU-expensive** (each is evaluated per message), limited to 2,000 per topic, and complex ones measurably reduce throughput.
- **SQL actions** modify properties when a rule matches — useful for routing tags, sparingly.
- Remember that **a new subscription has a default `TrueFilter`** (everything). Creating it and then replacing the rule leaves a window where it receives everything — create subscriptions with their rules in one operation (or via infrastructure as code).

**Filters run on properties, never on the body.** Put routing data — event type, tenant, region, priority, aggregate ID — in **application properties** at send time. This is also what makes a message routable without deserializing it.

**Forwarding and chaining.** **Auto-forwarding** moves messages from a queue or subscription into another queue or topic in the same namespace, transactionally. Uses: fan-in (many subscriptions into one processing queue), per-tenant queues forwarding into a shared processor, or isolating a consumer's dead-letter queue by forwarding it to a central DLQ handler. Chains are limited (a message can traverse up to four hops), and each hop is billed as an operation on Standard.

**Topology belongs in infrastructure as code.** Create entities, subscriptions and rules with Bicep/Terraform or a dedicated provisioning step with `ServiceBusAdministrationClient` — **not** from application startup with management rights. Applications that hold `Manage` rights to create their own topology are a security finding (Concept 15) and a source of drift.

```csharp
// A provisioning step (pipeline job or migration tool) — not application startup.
var admin = new ServiceBusAdministrationClient("sb-orders-prod.servicebus.windows.net", credential);

if (!await admin.TopicExistsAsync("orders-events"))
    await admin.CreateTopicAsync(new CreateTopicOptions("orders-events")
    {
        RequiresDuplicateDetection          = true,
        DuplicateDetectionHistoryTimeWindow = TimeSpan.FromHours(1),
        DefaultMessageTimeToLive            = TimeSpan.FromDays(7)
    });

// Subscription + its rule in one call: no window with the default TrueFilter.
if (!await admin.SubscriptionExistsAsync("orders-events", "billing"))
    await admin.CreateSubscriptionAsync(
        new CreateSubscriptionOptions("orders-events", "billing")
        {
            MaxDeliveryCount                 = 10,
            LockDuration                     = TimeSpan.FromSeconds(60),
            DeadLetteringOnMessageExpiration = true,
            RequiresSession                  = true          // per-order ordering for billing (Concept 11)
        },
        new CreateRuleOptions("order-lifecycle",
            new CorrelationRuleFilter { ApplicationProperties = { ["eventFamily"] = "order-lifecycle" } }));
```

**The interview-grade sentence:** *"My default topology is commands to queues owned by the receiver and events to a topic per bounded context owned by the publisher, with one subscription per consumer filtered by correlation filters on application properties — cheap, up to 100,000 per topic — rather than SQL filters, which cost CPU per message. Routing data goes in properties, never the body; subscriptions are created with their rules atomically to avoid the default TrueFilter window; and the whole topology lives in infrastructure as code, not in application startup with Manage rights."*

---

## Concept 10 — Message lifecycle settings

Six entity settings decide how a message lives and dies. Their defaults are reasonable for demos and frequently wrong for production.

| Setting | Default | Range | What it really controls | How to set it |
|---|---|---|---|---|
| **Lock duration** | 60 s | up to 5 min | How long a receiver owns a message before the broker assumes it died and redelivers | ≥ p99 processing time with margin; use **lock renewal** for longer work, not a giant lock |
| **Max delivery count** | 10 | ≥ 1 | How many failed deliveries (abandons, lock expiries) before automatic dead-lettering | Low (3–5) for poison-prone work with in-process retries; default for broker-driven retry |
| **Default message TTL** | Effectively unlimited | ≥ ~1 s | When an unconsumed message expires | Set it — stale commands are dangerous ("ship order" two weeks late) |
| **Dead-lettering on expiration** | Off | — | Whether expired messages go to the DLQ or vanish | On for anything business-relevant |
| **Duplicate detection window** | 10 min (when enabled) | 20 s – 7 days | How long the broker remembers `MessageId`s to drop duplicate sends | Cover your producer's realistic retry horizon (relay restarts, outages) — often hours |
| **Auto-delete on idle** | Off | ≥ 5 min | Deletes an entity unused for the interval | Only for genuinely temporary entities (per-session reply queues) |

**How redelivery actually happens** — the mechanics behind the at-least-once contract:

1. A receiver gets a message under a **lock** (peek-lock mode). The message's `DeliveryCount` increments.
2. The receiver **completes** it (gone), **abandons** it (unlocked immediately for redelivery), **defers** it (set aside, retrievable only by sequence number), or **dead-letters** it (moved to the entity's `$DeadLetterQueue` with a reason and description).
3. If the receiver crashes, or processing outlives the lock without renewal, **the lock expires** and the message becomes available again — even if your handler is still running and later succeeds. That second execution is the most common source of duplicate processing in Service Bus systems.
4. When `DeliveryCount` exceeds `MaxDeliveryCount`, the broker dead-letters it with reason `MaxDeliveryCountExceeded`.

**Dead-letter reasons you should know:** `MaxDeliveryCountExceeded`, `TTLExpiredException` (with expiration dead-lettering on), `HeaderSizeExceeded`, filter-evaluation errors on subscriptions, session ID problems, and whatever **your** code sets when it dead-letters explicitly. Always set `deadLetterReason` and `deadLetterErrorDescription` when your code dead-letters — the DLQ is an operational tool only if each message says why it's there.

**DLQ operations are part of the design, not an afterthought:**

- **Alert** on dead-lettered message count per entity (it's a native metric).
- **Inspect** with Service Bus Explorer in the portal or the open-source Service Bus Explorer tool.
- **Repair and resubmit** with a controlled tool or job — never by bulk-moving the whole DLQ back to the main queue without understanding why it failed (that re-poisons the consumer).
- **Bound retention** with a TTL on the DLQ's forwarded destination if you centralize, and make sure someone owns it.

**The interview-grade sentence:** *"I set lock duration to cover p99 processing time and renew locks for longer work rather than making the lock huge, choose max delivery count according to whether retries are in-process or broker-driven, always set a TTL with dead-lettering on expiration for business messages, and size the duplicate-detection window to the producer's real retry horizon — often hours. Lock expiry while a handler is still running is the classic cause of duplicates, and the DLQ needs alerts, explicit reasons and a repair-and-resubmit tool."*

---

## Concept 11 — Sessions at scale

**Sessions** give Service Bus its strongest ordering guarantee: all messages with the same `SessionId` are delivered **in order** and **exclusively** to one receiver at a time, which holds a **session lock**. They also offer **session state** — a small blob the broker stores per session — so a handler can resume a multi-message workflow after a crash on a different instance.

What sessions buy you, concretely:

- **Per-key FIFO**: "process events for order 42 in order" without global ordering.
- **Mutual exclusion per key**: two instances never process order 42 concurrently — a free distributed lock scoped to the key, which removes a whole class of race conditions in handlers (Module 9's locks, for free, with the broker as coordinator).
- **Resumable state**: the session-state blob for long-running correlations (aggregating a multi-part message, a request/reply conversation).

What they cost:

- **Throughput per key is serial.** If one order produces 1,000 events, they're processed one at a time. A single hot session is a single-threaded bottleneck — the Service Bus version of a hot partition.
- **Head-of-line blocking within a session.** A poison message blocks every later message in its session until it's dead-lettered (after `MaxDeliveryCount` attempts), so failing fast matters more than usual.
- **Session acquisition overhead.** Receivers must find and lock sessions; with very many short sessions, the acquire/release cycle is a meaningful share of the work.
- **All-or-nothing**: a session-enabled entity accepts only messages with a `SessionId`.

**The .NET side** — the session processor:

```csharp
ServiceBusSessionProcessor processor = client.CreateSessionProcessor("orders-events", "billing",
    new ServiceBusSessionProcessorOptions
    {
        MaxConcurrentSessions        = 32,                       // sessions handled in parallel by this instance
        MaxConcurrentCallsPerSession = 1,                        // keep 1 to preserve order within a session
        SessionIdleTimeout           = TimeSpan.FromSeconds(5),  // release an idle session quickly to pick up others
        AutoCompleteMessages         = false,
        MaxAutoLockRenewalDuration   = TimeSpan.FromMinutes(5),
        PrefetchCount                = 0
    });

processor.ProcessMessageAsync += async args =>
{
    BillingState state = await LoadStateAsync(args);            // from session state (or your own store)
    state = await billing.ApplyAsync(state, args.Message, args.CancellationToken);  // idempotent per MessageId
    await args.SetSessionStateAsync(BinaryData.FromObjectAsJson(state), args.CancellationToken);
    await args.CompleteMessageAsync(args.Message, args.CancellationToken);
};
processor.ProcessErrorAsync += args => { logger.LogError(args.Exception, "Session processing error"); return Task.CompletedTask; };
```

Three tuning facts that interviewers like:

1. **`SessionIdleTimeout` matters more than people expect.** With a long idle timeout, a receiver holds an empty session waiting for more messages while other sessions with backlog wait. Short timeouts keep receivers moving; very short ones add acquisition churn. Measure.
2. **Total parallelism = instances × `MaxConcurrentSessions`**, and it's capped by the number of sessions with active messages. Ten instances with 32 sessions each will idle if only 50 sessions have work.
3. **Choose the session key exactly at the ordering scope you need** — order ID, not customer ID, if you only need per-order ordering. A coarser key serializes unrelated work.

**When not to use sessions:** if the consumer is naturally idempotent and commutative (setting a status to the newest version by timestamp, or using optimistic concurrency against the store — Module 19), you often don't need broker ordering at all; the store enforces correctness. Sessions are for when the *processing* must be ordered, not just the outcome.

**The interview-grade sentence:** *"Sessions give per-key FIFO, exclusive processing per key — effectively a distributed lock the broker manages — and resumable session state, at the price of serial throughput per key, head-of-line blocking from poison messages, and acquisition overhead. I key sessions at exactly the ordering scope I need, keep one concurrent call per session, tune MaxConcurrentSessions and a short SessionIdleTimeout, and skip sessions entirely when an idempotent, version-checked write already makes order irrelevant."*

---

## Concept 12 — Throughput engineering in the SDK

Most Service Bus performance problems are client configuration problems. The knobs interact, so tune them together.

**1. Client lifetime.** One `ServiceBusClient` per process (per namespace), registered as a singleton; it owns one AMQP connection. Senders, receivers and processors are cheap links on that connection — **cache senders** too (one per entity). Creating a client per message means a TLS handshake, AMQP connection setup and token acquisition per message: latency, CPU, connection quota exhaustion and SNAT pressure (Module 26, Concept 14).

**2. Sending.**

- **Batch** with `CreateMessageBatchAsync` + `TryAddMessage` — one network round trip for many messages, respecting the size limit. On Standard, a batch counts as one operation per message for billing, but saves round trips and CPU on both sides.
- **Send concurrently** if a single sender can't keep up — senders are thread-safe.
- **Set `MessageId` deterministically** (from the business event's ID) so duplicate detection can work (Concept 13).
- **Large payloads**: on Standard, anything near 256 KB belongs in Blob Storage with a reference in the message (claim check, Concept 51); on Premium, large messages up to 100 MB work but are slower and consume MU capacity disproportionately — use them for legacy JMS migrations, not as a design default.

**3. Receiving.** The `ServiceBusProcessor` is the right default: it runs a receive loop, invokes your handler with concurrency, renews locks, and abandons on unhandled exceptions.

```csharp
var processor = client.CreateProcessor("charge-card", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls         = 16,                         // per instance; I/O-bound handlers can go higher
    PrefetchCount              = 0,                          // start at 0; raise only after measuring (see below)
    AutoCompleteMessages       = false,                      // settle explicitly after the work is committed
    MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(10),   // covers slow paths; the lock itself stays short
    ReceiveMode                = ServiceBusReceiveMode.PeekLock
});
```

**Prefetch** is the knob most often set wrong. It makes the client pull messages *ahead* of the handler into a local buffer, under lock. That cuts latency for fast handlers — and causes **lock expiry** for slow ones, because prefetched messages' lock clocks are already ticking while they wait in the buffer (and **auto-renewal doesn't cover messages still sitting in the prefetch buffer**). Rule of thumb from Microsoft's guidance: if you use prefetch, size it to what the instance can process within one lock duration — roughly `MaxConcurrentCalls × (lock duration ÷ average processing time)` as an upper bound, and usually much less. For slow or variable handlers, keep it at 0.

**Concurrency** (`MaxConcurrentCalls`) is per instance; total concurrency against downstream systems is **instances × MaxConcurrentCalls**. I/O-bound handlers can run dozens per instance; CPU-bound handlers should match cores. On Functions and KEDA-scaled Container Apps, multiply by the maximum instance count before you deploy (Module 26, Concept 26).

**Receive mode.** `ReceiveAndDelete` removes the message on delivery — at-most-once, faster, and correct only when losing a message is acceptable (telemetry-like data on a queue, which usually means you wanted Event Hubs).

**4. What "fast" looks like.** Rough orders of magnitude to calibrate expectations (measure your own): a well-tuned client on Premium can move thousands to tens of thousands of small messages per second per MU with batching; per-message send without batching tops out much lower per sender because each send is a round trip; a single session processes at the speed of one handler.

**The interview-grade sentence:** *"Service Bus throughput is mostly client configuration: one ServiceBusClient per process with cached senders, batched sends with deterministic MessageIds, the processor with explicit settlement and auto lock renewal, MaxConcurrentCalls sized against downstream capacity across all instances, and prefetch at zero unless the handler is fast — because prefetched messages' locks are already ticking and auto-renewal doesn't cover the buffer."*

---

## Concept 13 — Transactions, the outbox relay, and duplicate detection

Service Bus offers **transactions within a namespace**: a group of operations that commit or roll back together. The important one is **send-via** (cross-entity transactions): receive from entity A, then *complete* the input and *send* outputs to B and C in one transaction routed through A — so a message is never consumed without its outputs being sent, nor sent twice because the completion failed.

```csharp
// Requires: ServiceBusClientOptions.EnableCrossEntityTransactions = true, and the first operation
// in the transaction must be on the "via" entity (the one you received from).
await using ServiceBusReceiver receiver = client.CreateReceiver("payments-in");
ServiceBusSender ledgerSender  = client.CreateSender("ledger");
ServiceBusSender receiptSender = client.CreateSender("receipts");

ServiceBusReceivedMessage msg = await receiver.ReceiveMessageAsync();
using (var ts = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    await receiver.CompleteMessageAsync(msg);                     // first op: on the via entity
    await ledgerSender.SendMessageAsync(new ServiceBusMessage(Map(msg)) { MessageId = $"{msg.MessageId}:ledger" });
    await receiptSender.SendMessageAsync(new ServiceBusMessage(Receipt(msg)) { MessageId = $"{msg.MessageId}:receipt" });
    ts.Complete();
}
```

**What it doesn't do — and the interview trap.** A Service Bus transaction **cannot include your database**. There's no distributed transaction between SQL or Cosmos and Service Bus (and `TransactionScope` won't escalate one — Module 7's `TransactionScope` warnings apply). So "update the database and publish an event" is still the **dual-write problem**, and the answer is still the **transactional outbox** (Module 11): commit the business change and an outbox record in the same database transaction, and let a **relay** publish outbox records to Service Bus.

**The relay's job** — and where Service Bus features help:

1. Read unpublished outbox rows (or, with Cosmos, read the change feed — Concept 41).
2. Send each as a message whose **`MessageId` is the outbox record's ID** (deterministic).
3. Mark the row published (or advance the change-feed lease).

If the relay crashes between steps 2 and 3, it will send the same record again on restart. **Duplicate detection** on the destination entity drops the second send *if it arrives within the detection window* — which is why the window should cover the relay's realistic restart-and-catch-up time (hours, not the 10-minute default). Duplicate detection is a broker-side **producer** guard; it doesn't remove the need for **idempotent consumers**, because consumers still see redeliveries from lock expiry (Concept 10) and because duplicates outside the window still happen.

**Batch sends and transactions:** a batch is sent atomically (all or nothing) on non-partitioned entities; on partitioned entities, messages in one batch or transaction must share a `PartitionKey` (or `SessionId`) so they land in the same partition.

**Frameworks.** MassTransit, NServiceBus and Wolverine implement the outbox (and inbox for consumer idempotency) for you, including the relay; Module 11, Concept 42 covered the landscape and the MassTransit v9 licensing change. The design point stands with or without a framework.

**The interview-grade sentence:** *"Service Bus transactions cover operations inside one namespace — send-via lets me complete an input and send outputs atomically — but they can't include my database, so database-plus-publish is still an outbox with a relay. The relay sends with the outbox record ID as MessageId so duplicate detection drops replays within the window, which I size to the relay's restart horizon, and consumers stay idempotent anyway because lock expiry still redelivers."*

---

## Concept 14 — Service Bus reliability: zones, Geo-DR and Geo-Replication

Three features, three different promises. Being able to separate them is one of the best Azure-architect signals in this module.

| | **Availability zones** | **Geo-Disaster Recovery (Geo-DR)** | **Geo-Replication** |
|---|---|---|---|
| Protects against | Datacenter (zone) failure within a region | Region loss — **configuration only** | Region loss — **configuration and data** |
| Tiers | All tiers, in regions with zones | Premium | Premium (GA since December 2025) |
| What's replicated | Everything, synchronously across zones | **Metadata**: entities, subscriptions, rules, settings — **not messages** | Metadata **and** messages **and** message state changes (completions, lock state, sequence) |
| Endpoint | Unchanged | An **alias** FQDN that points to primary or secondary | The namespace's own hostname; promotion repoints it |
| Failover | Automatic, transparent | One-way, manual **failover** that **breaks the pairing**; re-pair afterwards | **Promotion** — **planned** (waits for replication to catch up, no loss) or **forced** (immediate, may lose unreplicated data); roles swap, pairing continues |
| RPO | 0 | Messages in the old primary are **unavailable** until it recovers (or lost) | 0 with **synchronous** replication; up to the configured max lag with **asynchronous** |
| Latency cost | None noticeable | None | Synchronous: each write waits for the secondary (cross-region latency on every send/settle); asynchronous: none |
| Cost | Included | Second namespace (Premium MUs) | The secondary runs **the same MUs** as the primary |
| Limits | — | One secondary | **One secondary region** currently |

How to choose:

- **Zones**: always (they're free where supported). They handle the common failure.
- **Geo-DR**: when the requirement is "keep the *topology* available elsewhere" and in-flight messages can be lost or recovered later — e.g. senders can replay from their outbox, or messages are transient notifications. It's cheaper conceptually but leaves a data gap.
- **Geo-Replication**: when in-flight messages must survive a region loss. Choose **synchronous** for RPO 0 if the cross-region latency on every operation is acceptable (pair regions close together); choose **asynchronous** with a bounded max replication lag for latency-sensitive flows, accepting that a **forced** promotion can lose up to that lag.

**Promotion is a decision, not an event.** Neither Geo-DR failover nor Geo-Replication forced promotion happens automatically by default; someone (or automation keyed on health metrics) decides. Write the runbook: who decides, on what signal, what happens to consumers in the old region, and how you fail back. Geo-Replication can be promoted from automation using its replication-lag and health metrics; make sure the automation can't flap.

**Client behavior during failover.** The `ServiceBusClient` reconnects using its retry policy; with Geo-Replication the hostname doesn't change, so clients follow the promotion after DNS and connection re-establishment. Messages being processed during the promotion may be redelivered — idempotent consumers make that harmless.

**The interview-grade sentence:** *"Service Bus has three separate resilience features: availability zones for in-region datacenter loss on every tier; Geo-DR, which pairs Premium namespaces behind an alias and replicates only metadata, so in-flight messages are stranded on failover; and Geo-Replication, GA since December 2025, which replicates metadata and messages to one secondary, synchronously for RPO zero at the cost of cross-region latency per operation, or asynchronously with a bounded lag, with planned or forced promotion. Promotion is a runbook decision, and the secondary costs the same MUs."*

---

## Concept 15 — Service Bus security

The secure baseline for a production namespace:

1. **Microsoft Entra ID for every client** — managed identity or workload identity (Module 26, Concept 52) with **data-plane roles** scoped as tightly as possible:
   - **Azure Service Bus Data Sender** — send to entities;
   - **Azure Service Bus Data Receiver** — receive from entities;
   - **Azure Service Bus Data Owner** — full data-plane access (rarely needed by applications).
   Scope assignments at the **entity** level (a queue, a topic) where practical, so a compromised consumer can't read other services' queues.
2. **Disable local authentication** (`disableLocalAuth = true`) so SAS keys and connection strings stop working entirely. SAS policies are shared secrets that end up in configuration files, CI logs and laptops; Entra tokens expire and are tied to an identity.
3. **No `Manage` rights in applications.** Topology is provisioned by pipelines (Concept 9). Applications that can create, delete or reconfigure entities are a lateral-movement risk.
4. **Private endpoints** (Premium) with public network access disabled, or at least IP rules; private DNS zones (`privatelink.servicebus.windows.net`) wired correctly — the classic failure is a client resolving the public IP and being rejected.
5. **Customer-managed keys** (Premium) where policy requires control over encryption keys; data is encrypted at rest with Microsoft-managed keys by default.
6. **TLS 1.2 minimum**, and Azure Policy to enforce the above across subscriptions (built-in policies exist for local auth, private link and CMK).

A small but real detail: when you disable local auth, **every** client must be on Entra — including tooling, Logic Apps connections, Functions bindings using connection strings (switch to identity-based connections: `ServiceBusConnection__fullyQualifiedNamespace`), and third-party integrations. Inventory before you flip it.

**The interview-grade sentence:** *"For Service Bus I use Entra identities with Data Sender and Data Receiver roles scoped to individual entities, disable local SAS authentication entirely, keep Manage rights out of applications because topology belongs to the pipeline, and put Premium namespaces behind private endpoints with correct private DNS, adding CMK where policy requires — after inventorying every client, including Functions bindings, that still uses connection strings."*

---

## Concept 16 — Service Bus traps

**1. Lock expiry while the handler is still running.** Long or variable processing plus prefetch plus no renewal → the broker redelivers to another instance while the first is still working → double execution. Fix: renewal (`MaxAutoLockRenewalDuration`), prefetch at 0 for slow handlers, idempotent handlers, and moving genuinely long work elsewhere (a workflow, a job — Module 26, Concept 32).

**2. Retry multiplication.** In-process retries (Polly) × broker redelivery (`MaxDeliveryCount`) × SDK transport retries can turn one failure into dozens of attempts against a struggling dependency. Module 25 (Concept 64) gave the rule: let the broker own retries across time; keep in-process retries tiny and inside the lock budget.

**3. Abandon loops.** Abandoning immediately on a transient failure makes the message available instantly — the same instance (or another) picks it up within milliseconds and fails again, burning delivery count in seconds. For backoff, **schedule a retry** (complete the message and send a scheduled copy with an incremented attempt property), **defer**, or let the lock expire — or use a framework's delayed redelivery.

**4. The giant topic.** One topic, hundreds of subscriptions, dozens of SQL filters each, everything for everyone. Filter evaluation consumes MU CPU on every message; one consumer's backlog makes the entity huge; ownership is unclear. Split by bounded context (Concept 9).

**5. Connection churn.** A new `ServiceBusClient` per request, per message or per function invocation exhausts connection quotas and SNAT ports and adds tens of milliseconds per operation. Singletons (Concept 12).

**6. Unwatched DLQs.** Messages sit in dead-letter queues for months; the DLQ grows; nobody knows a business process silently stopped. Alert on DLQ count per entity, and own the repair tool (Concept 10).

**7. Session starvation and hot sessions.** A long `SessionIdleTimeout` or a few huge sessions serialize everything (Concept 11).

**8. Using Service Bus as a database.** Peeking messages to "check status," deferring messages for days as a storage mechanism, or scheduling millions of messages months ahead. The broker isn't a query engine; keep state in a store and use the broker for movement.

**9. Using it for telemetry.** Millions of small events per second, no per-message workflow, replay needed — that's Event Hubs (Concept 31). Service Bus can do it at a high MU cost and without replay.

**10. Still on retired libraries.** As of five days ago, `Microsoft.Azure.ServiceBus` and `WindowsAzure.ServiceBus` are retired and SBMP no longer works (Concept 7). Check transitive dependencies too — older versions of some frameworks and the Functions Service Bus extension 4.x pulled in `Microsoft.Azure.ServiceBus`.

**The interview-grade sentence:** *"The Service Bus traps I look for are lock expiry with a handler still running, retries multiplied across Polly, redelivery and the transport, abandon loops with no backoff, giant shared topics full of SQL filters, a client per message, unwatched dead-letter queues, hot or starved sessions, the broker used as a database or a telemetry pipe — and, since September 30, anything still depending on the retired libraries, including transitively through old Functions extensions."*

---
# Part C — Azure Event Grid

## Concept 17 — Event Grid in 2026: two tiers and one envelope

Module 11 (Concept 34) positioned Event Grid as Azure's **event notification** backbone. In 2026 it is really two products under one name, and saying so is the first senior signal:

| | **Event Grid Basic** | **Event Grid Standard (namespaces)** |
|---|---|---|
| Resources | **Custom topics**, **system topics** (Azure service events), **domains**, **partner topics** | **Namespaces** containing **namespace topics** and an **MQTT broker** (topic spaces, clients, client groups, permission bindings) |
| Delivery | **Push** to handlers: webhooks, Functions, Service Bus, Event Hubs, Storage queues, Hybrid Connections — and to namespace topics | **Pull** (queue-like: receive, acknowledge, release, reject) and **push** (to Event Hubs, webhooks — check the current destination list) |
| Schemas | Event Grid schema, **CloudEvents 1.0**, or custom input mapping | **CloudEvents 1.0** |
| Capacity | Shared, per-operation billing | **Throughput units** per namespace + per-operation billing |
| Networking | Public with IP rules; private endpoints for publishing to custom topics/domains | Private endpoints for publishing **and** consuming (pull) |
| Typical use | Reacting to Azure resource events; simple fan-out of application events to Azure handlers | Consumers that must pull at their own pace or stay private; MQTT device and vehicle messaging; higher-scale pub/sub |

**CloudEvents** is the envelope for both. It standardizes the metadata every event needs — `id`, `source`, `type`, `subject`, `time`, `specversion`, `datacontenttype` — independent of the transport. Use it for your own events even outside Event Grid: it makes events portable across Event Grid, Event Hubs, Service Bus and Kafka, and the `Azure.Messaging.CloudEvent` type in .NET gives you serialization for free. The design rule from Module 11, Concept 26 holds: the envelope is a stable contract, the `data` payload is versioned.

How to choose between the tiers: **Basic** when the events originate in Azure (system topics) or the handlers are Azure services or reachable webhooks and push is fine; **namespaces** when consumers need to pull (private networks, rate control, no public endpoint), when you need MQTT, or when you want queue-like acknowledgment semantics without moving to Service Bus.

**The interview-grade sentence:** *"Event Grid is two products now: Basic, with custom, system, domain and partner topics that push to handlers like Functions, Service Bus and webhooks; and namespaces on the Standard tier, which add pull delivery with acknowledge and release semantics, push to Event Hubs and webhooks, throughput units, private consumption and an MQTT broker. CloudEvents 1.0 is the envelope across both, and I use it for my own events regardless of transport."*

---

## Concept 18 — System topics and resource events

The most valuable thing Event Grid does is often invisible: **Azure services publish events about themselves**, and you subscribe through a **system topic** without building any producer.

Common sources and what they enable:

| Source | Example event types | What teams build with them |
|---|---|---|
| **Blob Storage / ADLS Gen2** | `Microsoft.Storage.BlobCreated`, `BlobDeleted`, `BlobTierChanged` | Upload processing pipelines, virus scanning, thumbnailing, data-lake ingestion triggers — and the Functions Flex blob trigger (Module 26, Concept 19) |
| **Resource groups / subscriptions** | `ResourceWriteSuccess`, `ResourceDeleteSuccess`, `ResourceActionSuccess` | Governance automation, tagging enforcement, drift alerts, inventory |
| **Key Vault** | `SecretNearExpiry`, `CertificateNewVersionCreated` | Rotation automation and alerting |
| **Service Bus** | `ActiveMessagesAvailableWithNoListeners`, `DeadletterMessagesAvailableWithNoListener` | Waking consumers that scale to zero, DLQ alerting |
| **Container Registry** | `ImagePushed`, `ImageDeleted` | Deployment and scanning triggers |
| **Azure Maps, Communication Services, App Configuration, Entra ID (via Graph), IoT Hub, Media, Machine Learning, API Management** | Domain-specific | Integrations without polling |

**Subject-based filtering** is what makes resource events usable. Blob events carry a subject like `/blobServices/default/containers/uploads/blobs/2026/10/invoice-123.pdf`, so a subscription can filter with **subject begins with** `/blobServices/default/containers/uploads/` and **subject ends with** `.pdf` — no code, no wasted deliveries. **Advanced filters** add conditions on data fields (`data.contentLength` greater than a size, `data.api` in a set), with limits on how many per subscription (check the quotas).

Two subtleties that come up in real systems:

1. **`BlobCreated` fires on commit, and for some APIs more than once in a workflow.** For block blobs written in chunks, the event fires when the blob is committed; filtering on `data.api` (for example `PutBlob`, `PutBlockList`, or `FlushWithClose` for ADLS) avoids reacting to intermediate operations. Handlers must still be idempotent.
2. **Events are notifications, not data.** A `BlobCreated` event tells you a blob exists; it doesn't carry its content. The handler reads the blob — and by then, it may have been overwritten or deleted. Design handlers to check current state rather than trusting the event as a snapshot.

**The interview-grade sentence:** *"Event Grid's biggest win is system topics: Blob Storage, resource groups, Key Vault, Service Bus, Container Registry and many other services publish their own events, which I filter by type and subject prefix or suffix — for example only PDFs in one container — without writing producers. I treat them as notifications rather than snapshots, so handlers re-read current state and stay idempotent."*

---

## Concept 19 — Push delivery mechanics: validation, retry and dead-letter

Push delivery is where Event Grid Basic's behavior surprises teams. Know the mechanics exactly.

**1. Endpoint validation.** Before Event Grid delivers to a webhook, it proves the endpoint wants the events:

- With the **Event Grid schema**, it sends a `Microsoft.EventGrid.SubscriptionValidationEvent` containing a `validationCode`; the endpoint must echo it back (or someone must visit the `validationUrl` within a few minutes for manual validation).
- With **CloudEvents**, it uses the CloudEvents **abuse-protection handshake**: an HTTP `OPTIONS` request with a `WebHook-Request-Origin` header, to which the endpoint replies with `WebHook-Allowed-Origin`.

Azure service handlers (Functions with the Event Grid trigger, Service Bus, Event Hubs, Storage queues) don't need this — Event Grid trusts them via Azure RBAC and managed identity.

**2. Success, timeout and retry.**

- A delivery succeeds when the handler returns a **2xx** status (200–204) **within the response timeout (about 30 seconds)**.
- Otherwise Event Grid **retries with exponential backoff and jitter** — roughly 10 s, 30 s, 1 min, 5 min, 10 min, 30 min, 1 h, 3 h, 6 h, then every 12 h — until either the **maximum delivery attempts** (default **30**, configurable 1–30) or the **event time-to-live** (default **1,440 minutes = 24 hours**, configurable) is reached, whichever comes first.
- Some responses aren't retried at all — notably **400 Bad Request** and **413 Request Entity Too Large** (and authorization failures for some destinations); they go straight to dead-letter if configured.
- Event Grid also **delays delivery to endpoints that keep failing** to protect them, so an unhealthy handler sees fewer, later attempts — and events can arrive well after they happened.

**3. Dead-lettering.** If you configure a **dead-letter destination** (a Blob Storage container), undeliverable events are written there with the reason and attempt history. **Without one, they're dropped.** The dead-letter container is the only record that an event was lost — monitor it, and build a replay tool that reads blobs and republishes.

**4. Batching.** Per subscription you can set **max events per batch** and a **preferred batch size in KB**, so a webhook receives an array of events per request. Batches succeed or fail as a whole.

**5. Delivery identity and destinations.** Deliver to Service Bus, Event Hubs and Storage queues using a **managed identity** on the topic (system or user-assigned) with data-sender roles — no connection strings. Webhooks can authenticate deliveries with **Entra ID** (Event Grid acquires a token for an app registration you specify) — validate it in the handler.

**The robust push handler, in ASP.NET Core:**

```csharp
app.MapPost("/events/uploads", async (HttpRequest request, IUploadInbox inbox, CancellationToken ct) =>
{
    // CloudEvents batch: parse without trusting order or uniqueness.
    BinaryData body = await BinaryData.FromStreamAsync(request.Body, ct);
    CloudEvent[] events = CloudEvent.ParseMany(body);

    foreach (CloudEvent e in events)
    {
        // Record and return fast: dedupe on e.Id + e.Source, do the real work asynchronously.
        await inbox.TryAddAsync(e.Id, e.Source, e.Type, e.Subject, e.Data, ct);
    }
    return Results.Ok();                                  // within the ~30 s window, every time
}).RequireAuthorization("EventGridDelivery");             // validate the Entra token Event Grid sends

// CloudEvents abuse-protection handshake for webhook validation.
app.MapMethods("/events/uploads", ["OPTIONS"], (HttpRequest request, HttpResponse response) =>
{
    response.Headers["WebHook-Allowed-Origin"] = request.Headers["WebHook-Request-Origin"].ToString();
    return Results.Ok();
});
```

**The interview-grade sentence:** *"Event Grid push validates webhooks first — a validation code for its own schema or the CloudEvents OPTIONS handshake — then counts a delivery as successful only on a 2xx within about 30 seconds; otherwise it retries with exponential backoff for up to 30 attempts or a 24-hour TTL, doesn't retry 400s or 413s, slows down for failing endpoints, and drops events unless a Blob dead-letter destination is configured. So my handlers authenticate the delivery, record the event idempotently and return immediately, and someone owns the dead-letter container."*

---

## Concept 20 — Namespaces and pull delivery

Event Grid **namespace topics** turn Event Grid into something that behaves, for consumers, much like a lightweight queue over HTTP. An **event subscription** on a namespace topic has a **delivery mode**: **queue** (pull) or **push**.

**Pull delivery semantics:**

| Operation | What it does |
|---|---|
| **Receive** | Returns a batch of events (with a wait time for long polling), each with a **lock token**; the events are invisible to other receivers for the **lock duration** (configurable, up to 5 minutes) |
| **Acknowledge** | Deletes the events — processing done |
| **Release** | Makes the events available again, optionally after a **delay** — a built-in backoff that Service Bus abandon lacks |
| **Reject** | Dead-letters the events (to the subscription's Blob dead-letter destination, if configured) |
| **Renew locks** | Extends the lock for slow processing |

Events that exceed the subscription's **max delivery count** (default 10) or **retention** (up to 7 days on the topic) are dead-lettered or dropped, as configured. Ordering isn't guaranteed.

```csharp
// Azure.Messaging.EventGrid.Namespaces
var receiver = new EventGridReceiverClient(
    new Uri("https://evg-orders.westeurope-1.eventgrid.azure.net"), "orders", "fulfilment", credential);

ReceiveResult result = await receiver.ReceiveAsync(maxEvents: 50, maxWaitTime: TimeSpan.FromSeconds(10), ct);

var done = new List<string>(); var retryLater = new List<string>(); var poison = new List<string>();
foreach (ReceiveDetails d in result.Details)
{
    try { await fulfilment.HandleAsync(d.Event, ct); done.Add(d.BrokerProperties.LockToken); }
    catch (TransientException) { retryLater.Add(d.BrokerProperties.LockToken); }
    catch (Exception)          { poison.Add(d.BrokerProperties.LockToken); }
}
if (done.Count > 0)       await receiver.AcknowledgeAsync(done, ct);
if (retryLater.Count > 0) await receiver.ReleaseAsync(retryLater, ReleaseDelay.OneMinute, ct);   // delayed retry
if (poison.Count > 0)     await receiver.RejectAsync(poison, ct);
```

**When namespace pull beats the alternatives:**

- **vs Event Grid Basic push**: the consumer has no public endpoint, must run in a private network, or must control its rate.
- **vs Service Bus**: you need fan-out of *notifications* with simple acknowledgment and delayed release, CloudEvents end to end, and MQTT integration — not sessions, transactions, filters on SQL, scheduled messages, or deferral. If you need any of those, Service Bus is the queue.
- **vs Event Hubs**: you need per-event acknowledgment, not a replayable stream.

**The interview-grade sentence:** *"Namespace topics give Event Grid queue-like pull semantics over HTTP: receive a batch under a lock of up to five minutes, then acknowledge, release with an optional delay, reject to dead-letter, or renew — with a max delivery count and up to seven days of retention. I use it when consumers must stay private or set their own pace and the events are notifications; once I need sessions, transactions, scheduled messages or SQL filters, it's Service Bus."*

---

## Concept 21 — The MQTT broker

Event Grid namespaces include a managed **MQTT broker** (MQTT **v3.1.1** and **v5**), which makes Event Grid a pub/sub backbone for **devices, vehicles, factory equipment and apps** that speak MQTT.

The model:

- **Clients** authenticate with **X.509 certificates** (CA-signed or self-signed thumbprints), or with Entra ID / custom OAuth tokens; **client groups** collect them by attributes.
- **Topic spaces** define sets of MQTT topic templates (`vehicles/${client.authenticationName}/telemetry`), and **permission bindings** grant client groups publish or subscribe rights on topic spaces — per-device isolation without per-device configuration.
- Clients publish and subscribe **one-to-many, many-to-one and one-to-one** (commands to a device, telemetry from many devices, broadcasts), with MQTT v5 features such as request/response, message expiry and user properties, plus **retain** (the last message on a topic delivered to new subscribers) and **HTTP publish** for non-MQTT services.
- **Routing** sends MQTT messages to a **namespace topic** or a **custom topic**, from which push or pull subscriptions deliver them into Event Hubs, Functions, Service Bus and the rest of Azure.

Capacity is in **throughput units**: per TU, on the order of **10,000 MQTT sessions** and **1,000 inbound messages per second or 1 MB/s** (check the current quotas).

**Event Grid MQTT vs IoT Hub** is a common question: Event Grid's broker is for **MQTT pub/sub at scale with standard MQTT semantics and routing into Azure**; IoT Hub remains the service for **device management** features (device twins, direct methods, device provisioning workflows, IoT-specific SDKs). Many architectures use both, or choose Event Grid when standard MQTT clients and topic-based pub/sub are the requirement.

**The interview-grade sentence:** *"Event Grid namespaces include an MQTT v3.1.1 and v5 broker: clients authenticate with X.509 or tokens, topic spaces and permission bindings give per-device isolation through templates, MQTT v5 features and retain are supported, and routing pushes messages into namespace or custom topics and on into Event Hubs or Functions — sized in throughput units of roughly 10,000 sessions each. IoT Hub stays the choice when device management, twins and direct methods are the requirement."*

---

## Concept 22 — Domains, fan-out and limits

**Event domains** solve the multitenant publishing problem: one endpoint, one set of credentials, and **thousands of topics** (one per tenant, customer or partner) inside it. Publishers send to the domain with the target topic named in each event; subscribers subscribe only to their topic (with RBAC scoped to it) or to the whole domain. It's the Event Grid shape for "notify each of my 20,000 customers' webhooks about their own events."

**Partner topics** let SaaS providers (and Microsoft services such as Microsoft Graph for Entra ID and Outlook events) publish into your subscription without you running a bridge.

**Limits that shape design** (these change — check the quotas page):

| Limit (Basic) | Order of magnitude |
|---|---|
| Event size | 1 MB (operations billed in 64 KB chunks) |
| Event subscriptions per topic | 500 |
| Topics per domain | 100,000 |
| Publish rate per topic or domain | ~5,000 events/s or 5 MB/s |
| Advanced filters per subscription | 25 |

**Fan-out multiplies operations.** One published event × 1,000 subscriptions = 1,001 operations (one publish, 1,000 deliveries) — plus retries. At $0.60 per million operations, a million events a day to 50 subscribers is about 51 million operations a day, roughly $900 a month. Not expensive, but not free, and retries against a failing webhook farm can double it. Filtering at the subscription (so non-matching events never count as deliveries) is a cost lever as well as a correctness one.

**The interview-grade sentence:** *"Event domains give one publishing endpoint over up to 100,000 tenant topics with RBAC per topic, and partner topics bring SaaS and Microsoft Graph events in without a bridge. The limits I design around are 1 MB events billed per 64 KB, about 500 subscriptions per topic and a few thousand events per second per topic, and I remember that fan-out and retries multiply operations, so subscription filters save money as well as work."*

---

## Concept 23 — Event Grid traps, and where it fits

**1. Treating Basic as a queue.** There's no lock, no "process later," no competing-consumer semantics, no ordering. A slow handler gets retries; a down handler gets 24 hours and then dead-letter (or loss). If the work must be buffered, route it into Service Bus or Event Hubs (Concept 6), or use namespace pull.

**2. Assuming exactly-once or in-order.** Duplicates are normal (a timeout followed by a retry of an event your handler actually processed); order across events isn't guaranteed. Dedupe on `id` + `source`, and design handlers to read current state (Concept 18).

**3. Doing the work inside the push request.** Handlers that take longer than the response window generate their own duplicates. Acknowledge fast, work asynchronously (Concept 19).

**4. No dead-letter destination.** Without one, events that exhaust retries vanish without a trace.

**5. Webhooks without authentication.** Validation proves the endpoint opted in; it doesn't authenticate later deliveries. Use Entra-authenticated delivery and validate the token, or at least a secret in the URL query plus IP restrictions — and prefer Azure service destinations with managed identity.

**6. Large payloads in events.** Events are notifications; put the data in Blob or the database and reference it.

**7. Subscriptions created by hand.** Like Service Bus topology, subscriptions, filters, retry policies and dead-letter settings belong in infrastructure as code.

**Where it fits:** reacting to Azure resource events; fan-out of domain notifications to many independent handlers, including external webhooks and SaaS partners; MQTT device messaging; and lightweight pull queues for notifications. **Where it doesn't:** commands with a single owner and a workflow (Service Bus), high-volume telemetry and replayable streams (Event Hubs), or anything requiring order.

**The interview-grade sentence:** *"The Event Grid traps are using Basic as a queue, assuming exactly-once or ordered delivery, doing slow work inside the push request, running without a dead-letter container, leaving webhooks unauthenticated, putting payloads in events and creating subscriptions by hand. It fits resource events, notification fan-out to many independent handlers and partners, MQTT and light pull queues — not commands, streams or ordered work."*

---

# Part D — Azure Event Hubs as an operated platform

## Concept 24 — Event Hubs tiers in 2026

Module 11 (Concept 33) taught the model: namespaces, event hubs (topics), partitions, consumer groups, offsets and checkpoints, Capture and the Kafka endpoint. Here's the tier lineup that decides what you can build:

| | **Basic** | **Standard** | **Premium** | **Dedicated** |
|---|---|---|---|---|
| Tenancy | Multitenant | Multitenant | Multitenant with **resource isolation** | **Single-tenant** cluster |
| Capacity unit | TU (up to 40) | TU (up to 40) | PU (up to 16) | CU |
| Max event size | 256 KB | 1 MB | 1 MB | 1 MB |
| Consumer groups per event hub | 1 | 20 | 100 | 1,000-scale |
| Max retention | 1 day | 7 days | 90 days | 90 days |
| Retention storage included | 84 GB per TU | 84 GB per TU | 1 TB per PU | 10 TB per CU |
| Partitions per event hub | 32 | 32 | 100 (and 200 per PU across the namespace) | Up to 1,024-scale |
| Event hubs per namespace | 10 | 10 | 100 per PU | Higher |
| Partition count change | **Immutable** | **Immutable** | **Can add** (not remove) | **Can add** |
| Kafka endpoint | ❌ | ✅ | ✅ (with compression; transactions in preview) | ✅ (same) |
| Capture | ❌ | ✅ (charged per TU) | ✅ (included) | ✅ (included) |
| Log compaction | ❌ | ✅ (1 GB per partition) | ✅ (250 GB per partition) | ✅ |
| Private endpoints | — | ✅ | ✅ | ✅ |
| CMK | — | — | ✅ | ✅ |
| Geo-Replication | — | — | ✅ (GA July 2025) | ✅ |
| Billing | TU-hour + ingress events | TU-hour + ingress events | PU-hour | CU-hour (4-hour minimum) |

The decision in practice:

- **Standard** is the default for most production streams up to a few tens of MB/s, with up to 7 days of replay and the Kafka endpoint.
- **Premium** when you need **isolation and predictable latency**, **long retention** (up to 90 days — a genuine replay source), **more partitions or consumer groups**, **dynamic partition scale-out**, **CMK**, or **Geo-Replication**. It's also often cheaper per MB/s than a Standard namespace at the 30–40 TU end, because ingress events aren't metered.
- **Dedicated** for very large or regulated workloads: hundreds of MB/s to GB/s, single tenancy, or many namespaces consolidated on one cluster.
- **Basic** almost never in production: one consumer group and one day of retention remove most of what makes a log useful.

**The interview-grade sentence:** *"Event Hubs Standard — TUs, 32 immutable partitions, 20 consumer groups, 7-day retention, Kafka and Capture — covers most production streams; Premium adds resource isolation, PUs with no per-event charges, up to 90 days of retention, 100 partitions per hub that can be added later, CMK and Geo-Replication; Dedicated is a single-tenant cluster for very large or regulated estates; and Basic's single consumer group and one-day retention rule it out of most production designs."*

---

## Concept 25 — Sizing partitions and capacity

Module 5 (Concept 23) gave the formula; here is the full method, with the consumer side included.

**Step 1 — throughput per partition.** Microsoft's guidance: about **1 MB/s ingress and 2 MB/s egress per partition on Standard**, and about **1–2 MB/s ingress and 2–5 MB/s egress per partition on Premium/Dedicated**.

**Step 2 — partitions for throughput, consumption and growth:**

```
partitions ≥ max( peak ingress MB/s ÷ ingress per partition,
                  peak egress MB/s per consumer group ÷ egress per partition,
                  max consumer instances you want per consumer group,     ← one active reader per partition
                  hottest key's MB/s fits in one partition? )            ← if not, change the key, not the count
          × headroom (and, on Standard, a guess at two years of growth — partitions are immutable)
```

**Step 3 — capacity units (Standard):**

```
TUs ≥ max( ingress MB/s,
           ingress events/s ÷ 1,000,
           total egress MB/s across all consumer groups ÷ 2,
           total egress events/s ÷ 4,096 )
```

Note the **egress multiplies by consumer groups**: three independent consumers reading a 10 MB/s stream need 30 MB/s of egress.

**Worked example.** Clickstream at **12 MB/s** peak ingress, average event 1.5 KB (so ~8,000 events/s), read by **three consumer groups** (real-time personalization, fraud scoring, Capture-like archival via Stream Analytics), target of **up to 24 consumer instances** in the busiest group:

- Partitions for ingress: 12 ÷ 1 = 12. For egress per group: 12 ÷ 2 = 6. For consumers: 24. → **32** partitions (the Standard max, also allowing growth).
- TUs: ingress 12 MB/s → 12; events 8,000 ÷ 1,000 → 8; egress 36 MB/s ÷ 2 → 18 → **18 TUs** at peak, with **auto-inflate** to a maximum of, say, 30 for catch-up after a consumer outage.
- Check the 40-TU ceiling: 30 with catch-up headroom is close to it. If growth is expected, **Premium** with 2–3 PUs (100 partitions per hub, no ingress-event metering, longer retention) is the more comfortable home — price both (Concept 64).
- **Hot key check**: if one partition key (a large customer) produces 2 MB/s, it exceeds one partition's ingress guidance on Standard; give that key a suffix (customer + session hash) if per-customer order isn't required, or accept the ceiling.

**Auto-inflate** raises TUs automatically when you hit throttling, up to a maximum you set — and **never lowers them**. Schedule a job (or use a runbook) to scale back down after peaks, or you'll pay peak TUs forever.

**Partitions aren't billed on Standard** — capacity units are. Over-provisioning partitions modestly costs nothing directly; the indirect costs are more checkpoint writes, more ownership churn in processors, and smaller batches per partition.

**The interview-grade sentence:** *"I size Event Hubs partitions as the max of ingress over about 1 MB/s, egress per consumer group over about 2 MB/s, and the number of parallel consumers I want — because each consumer group gets one active reader per partition — with growth headroom on Standard since partitions are immutable; TUs as the max of ingress MB/s, events over 1,000, and total egress across all consumer groups over 2 MB/s; then I check the hottest key against one partition and remember that auto-inflate never scales back down."*

---

## Concept 26 — Producers

A producer decides **where each event goes**, and that decision fixes the ordering and load-balance properties of everything downstream.

**Three ways to route an event:**

| Routing | How | Ordering | Availability | When |
|---|---|---|---|---|
| **No key (round robin)** | Omit partition key and ID | None across events | Best — the service routes around unavailable partitions | Independent events (most telemetry) |
| **Partition key** | `PartitionKey = deviceId` | All events with the key go to one partition, in order | A key's partition being unavailable fails those sends | Per-entity order (per device, per order, per account) |
| **Explicit partition ID** | `PartitionId = "3"` | Total control | Same as above | Rare: custom sharding, replaying to a specific partition |

**Batching is the throughput lever.** One `SendAsync` of an `EventDataBatch` is one network operation; the batch respects the size limit and must share one partition key (or none):

```csharp
await using var producer = new EventHubProducerClient(
    "evh-telemetry.servicebus.windows.net", "clicks", credential);     // singleton in DI

using EventDataBatch batch = await producer.CreateBatchAsync(new CreateBatchOptions { PartitionKey = sessionId });
foreach (Click c in clicks)
{
    var e = new EventData(JsonSerializer.SerializeToUtf8Bytes(c, ClickContext.Default.Click));
    e.Properties["schema"] = "click.v3";                                 // routing/versioning metadata
    if (!batch.TryAdd(e)) { await producer.SendAsync(batch); /* start a new batch and re-add */ }
}
await producer.SendAsync(batch);
```

**The buffered producer.** `EventHubBufferedProducerClient` accepts events one at a time, groups them by partition key into batches, and sends them in the background — good for high-rate producers that don't want to manage batches. The trade: `EnqueueEventAsync` returns **before** the event is sent, so a process crash loses whatever is buffered. Call `FlushAsync` on shutdown (Module 26, Concept 51), handle `SendEventBatchFailedAsync`, and don't use it where the producer must know an event is durable before acknowledging upstream.

**Event size and metering.** Standard meters ingress in **64 KB units**: events far smaller than 64 KB are inefficient in billing terms; combining many tiny readings into one event (an array of readings per device per second) reduces both cost and per-event overhead — at the cost of finer-grained routing.

**Producer-side duplicates.** A send that times out may have succeeded. With AMQP there's no broker-side duplicate detection like Service Bus's, so consumers must be idempotent (event ID in properties, Concept 53). Kafka clients can use the Kafka idempotent-producer settings where supported by the endpoint — verify for your tier.

**The interview-grade sentence:** *"An Event Hubs producer chooses between round robin, which gives the best availability and no ordering, a partition key for per-entity order at the cost of that key's partition availability, and an explicit partition for rare custom sharding. Throughput comes from batches that share a key; the buffered producer batches for me but returns before the send, so I flush on shutdown; tiny events waste the 64 KB metering unit on Standard; and since a timed-out send may have succeeded, consumers stay idempotent."*

---

## Concept 27 — Processors, ownership and checkpoints

The **`EventProcessorClient`** (package `Azure.Messaging.EventHubs.Processor`) is the production consumer for .NET. It coordinates many instances reading one consumer group: each instance **claims ownership** of some partitions through a **checkpoint store** (Blob Storage), reads them, and records **checkpoints** (offsets) so a restarted or rebalanced reader resumes close to where the last one stopped.

```csharp
var checkpoints = new BlobContainerClient(new Uri("https://stcheckpoints.blob.core.windows.net/clicks-personalize"), credential);

var processor = new EventProcessorClient(checkpoints, "personalize",
    "evh-telemetry.servicebus.windows.net", "clicks", credential,
    new EventProcessorClientOptions
    {
        LoadBalancingStrategy           = LoadBalancingStrategy.Balanced,   // Greedy claims faster at startup
        PrefetchCount                   = 300,
        TrackLastEnqueuedEventProperties = true                              // lets you compute lag per partition
    });

var sinceCheckpoint = new ConcurrentDictionary<string, int>();

processor.ProcessEventAsync += async args =>
{
    if (!args.HasEvent) return;                                            // fires on MaximumWaitTime with no event
    try
    {
        await personalizer.ApplyAsync(args.Data, args.CancellationToken);  // idempotent on event id
    }
    catch (Exception ex)
    {
        await parking.StoreAsync(args.Partition.PartitionId, args.Data, ex, args.CancellationToken); // never let it escape
    }

    // Checkpoint every 500 events per partition — trade replay size for checkpoint cost.
    if (sinceCheckpoint.AddOrUpdate(args.Partition.PartitionId, 1, (_, n) => n + 1) >= 500)
    {
        await args.UpdateCheckpointAsync(args.CancellationToken);
        sinceCheckpoint[args.Partition.PartitionId] = 0;
    }
};
processor.ProcessErrorAsync += args => { logger.LogError(args.Exception, "Partition {P}: {Op}", args.PartitionId, args.Operation); return Task.CompletedTask; };

await processor.StartProcessingAsync(ct);
```

**What to know about each moving part:**

- **Ownership and load balancing.** Instances periodically claim and steal partitions until the load is balanced. With N partitions and M instances, each instance owns about N ÷ M; instances beyond N sit idle. Rebalancing after a scale event takes tens of seconds, during which some events are re-read from the last checkpoint.
- **Checkpoint cadence is a trade-off.** Every checkpoint is a Blob write (latency, cost, storage throttling at high partition counts). Infrequent checkpoints mean more replay after a crash or rebalance. Checkpoint every few hundred events or few seconds per partition — and **always assume replay**, which is why handlers are idempotent (Module 11).
- **Exceptions in the handler are your problem.** The processor doesn't retry an event whose handler threw, and won't dead-letter it — there's no such thing on a log. Catch everything, **park** failures (a Blob, a Service Bus queue, a Cosmos container), emit a metric, and move on. A handler that lets exceptions escape can silently skip events or stall a partition depending on the code path.
- **One active reader per partition per consumer group.** The service allows up to five concurrent readers per partition per group, but the processor uses one; extra readers (and **epoch / exclusive** receivers) are for special cases. Give every independent consumer **its own consumer group** — never share one between two applications.
- **Lag** is the metric that matters: last enqueued sequence number minus the sequence number being processed, per partition. With `TrackLastEnqueuedEventProperties`, the processor exposes the last enqueued event information so you can emit lag yourself; KEDA's Event Hubs scaler computes it from the checkpoint store (Module 26, Concept 31).
- **Batch processing** — for high throughput, the `EventProcessor<TPartition>` base class lets you process batches per partition rather than one event at a time; Functions' Event Hubs trigger is batch-oriented and checkpoints per batch (Module 26, Concept 25).

**The interview-grade sentence:** *"The EventProcessorClient balances partition ownership across instances through a Blob checkpoint store, so useful parallelism is capped by partition count and every independent consumer gets its own consumer group. Checkpoint cadence trades checkpoint writes against replay after a rebalance, so handlers are idempotent and replay is assumed; exceptions are mine to catch and park because a log has no dead-letter queue; and lag per partition — last enqueued minus processed — is the metric I alert and scale on."*

---

## Concept 28 — The Kafka endpoint, honestly

Event Hubs **Standard, Premium and Dedicated** expose a **Kafka-protocol endpoint**: an event hub is a Kafka topic, a namespace is a "cluster," and existing Kafka producers and consumers work by changing configuration. It's one of the strongest arguments for Event Hubs in mixed or migrating estates — and the details are where interviews go.

**What works:**

- Kafka producer and consumer APIs (clients 1.0+), consumer groups (created on demand, up to 1,000 Kafka groups per namespace on Standard/Premium), offsets and commits.
- **Log compaction** (GA) on Standard and above — key-based retention for CDC and table-like topics.
- **Kafka Connect** (source and sink connectors) and **MirrorMaker 2** for migration and replication.
- Protocol interoperability: write with Kafka, read with AMQP (`EventProcessorClient`, Stream Analytics, Functions) and vice versa.
- **Entra ID authentication** via SASL `OAUTHBEARER`, or connection strings via SASL `PLAIN` (username `$ConnectionString`).

**What's partial or different:**

- **Transactions** (exactly-once producer/consumer semantics) are in **public preview** and only on **Premium and Dedicated**.
- **Compression** (gzip, snappy, lz4, zstd at the client) is supported only on **Premium and Dedicated**.
- **Topic administration** is done through Azure (portal, CLI, Bicep), not reliably through Kafka admin APIs; partition counts follow tier rules (immutable on Standard).
- **Retention** is time-based by tier (and compaction), not size-based per topic.
- Kafka Streams and some advanced features have been in preview — check current status before relying on them.
- There are no brokers to tune; capacity is TUs/PUs/CUs, and throttling appears as Kafka quota errors.

**Client configuration gotchas** (Microsoft's recommended configuration page is the source of truth):

```properties
bootstrap.servers=evh-telemetry.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER                 # Entra ID; or PLAIN with $ConnectionString
request.timeout.ms=60000                   # the service can take longer than Kafka's default to respond under load
metadata.max.age.ms=180000                 # stay below the service's idle-connection close (~4 minutes)
connections.max.idle.ms=180000
max.request.size=1000000                   # 1 MB event limit (Standard and above)
linger.ms=5                                # small linger to batch
```

The classic production incident is **idle connections being closed by the service** after a few minutes without traffic, followed by producers that hang on a stale connection until a long timeout — which is why the idle and metadata settings matter.

**When the Kafka endpoint is the right answer:** you have Kafka clients, connectors or skills and want a managed service without operating brokers; you're migrating from self-managed Kafka (MirrorMaker 2); or you want one stream consumed by both Kafka-native and Azure-native tools. **When it isn't:** you depend on Kafka features in preview or unsupported on your tier, on broker-level configuration, or on the wider Kafka ecosystem (Streams, ksqlDB, specific connectors) at a depth where Confluent Cloud or a managed Kafka service is the honest choice.

**The interview-grade sentence:** *"Event Hubs Standard and above speak the Kafka protocol on port 9093, so Kafka clients, compaction, Kafka Connect and MirrorMaker 2 work by configuration — with Entra via OAUTHBEARER — and I can mix Kafka producers with AMQP consumers. The gaps are that transactions are still in preview and, like compression, only on Premium and Dedicated, topics are administered through Azure with tier partition rules, and clients need tuned timeouts and idle settings because the service closes idle connections after a few minutes."*

---

## Concept 29 — Capture, Schema Registry and stream processing

**Capture** writes the stream automatically to **Blob Storage or ADLS Gen2** in **Avro** files, on a time window (minutes) or size window (MB), whichever comes first — per partition, without writing a consumer. It's the cheapest way to build a **cold path** (data lake, replay beyond retention, audit) and to backfill new consumers. It's **included on Premium and Dedicated** and charged per TU on Standard. Parquet output is available through the Stream Analytics no-code editor. Capture keeps up with the stream independently of your consumers — but it's an at-least-once archive, so downstream readers should tolerate duplicates at window boundaries.

**Schema Registry** lives inside an Event Hubs namespace: a central store of **schema groups** with compatibility rules (backward, forward, none) for **Avro** (and JSON Schema in supported clients). Producers serialize with a schema ID header; consumers resolve the schema by ID. It's free with the namespace and works with both AMQP and Kafka clients. Use it when several teams produce and consume the same stream and you want compatibility *enforced* rather than hoped for (Module 11, Concept 27).

**Stream processing** — computing over the stream, not per event — has three main Azure homes:

| Option | Model | Best for |
|---|---|---|
| **Azure Stream Analytics** | SQL-like streaming queries with windows (tumbling, hopping, sliding, session), reference data joins, outputs to Cosmos, SQL, Blob, Power BI, Functions; a no-code editor | Aggregations, filtering, alerting and materialized views with minimal code |
| **Microsoft Fabric Real-Time Intelligence** (Eventstreams, Eventhouse/KQL) | Ingest Event Hubs into Fabric, transform, store in Eventhouse, query with KQL | Analytics, dashboards and investigation over streams, within a Fabric estate |
| **Your own code** (`EventProcessorClient`, Functions, Kafka clients in containers) | Imperative, per event or per batch | Business logic per event, custom stateful processing |

**The interview-grade sentence:** *"Capture archives every partition to Blob or ADLS in Avro on a time or size window — included on Premium — which gives me a cold path and backfill without a consumer; Schema Registry in the namespace enforces Avro compatibility for both AMQP and Kafka clients; and for computing over the stream I'd use Stream Analytics for windowed SQL, Fabric Real-Time Intelligence for analytics with KQL, and my own processors for per-event business logic."*

---

## Concept 30 — Event Hubs reliability

Event Hubs has the same three-layer resilience story as Service Bus, and the same need to keep the layers apart:

| | **Availability zones** | **Geo-DR** | **Geo-Replication** |
|---|---|---|---|
| Protects against | Zone failure | Region loss — **configuration** | Region loss — **configuration, events and consumer offsets** |
| Tiers | All, in regions with zones | Standard, Premium, Dedicated | **Premium and Dedicated** (GA since July 2025) |
| What's replicated | Everything, synchronously within the region | Metadata (event hubs, consumer groups, settings) via an **alias** | Metadata, event data and **consumer offsets** (and Capture configuration) |
| Failover | Transparent | Manual, one-way, breaks the pairing | **Promotion** (planned or forced); the namespace hostname follows the new primary |
| RPO | 0 | Events in the old primary unavailable until recovery | 0 with **synchronous** replication; bounded by max lag with **asynchronous** |

Three Event Hubs–specific points:

1. **Replicated offsets** are what make Geo-Replication useful for consumers: after promotion, Kafka consumers resume from replicated committed offsets. For AMQP consumers, your **checkpoint store** (Blob) is separate — make it zone-redundant and decide how it fails over (geo-redundant storage, or rebuild checkpoints from a time offset and tolerate replay).
2. **Sequence numbers and offsets may not be identical across regions** in every scenario, so consumers should be prepared to resume by time or replay a little — idempotency again.
3. **Producers** using a partition key during a partial outage fail for that key's partition; round-robin producers route around it. If per-key order isn't critical during incidents, consider falling back to no key.

**The interview-grade sentence:** *"Event Hubs mirrors Service Bus: zones for in-region failures on every tier, Geo-DR pairing for metadata only, and Geo-Replication on Premium and Dedicated — GA since July 2025 — replicating events and consumer offsets synchronously or asynchronously with planned or forced promotion. For AMQP consumers the Blob checkpoint store is a separate DR decision, so I make it zone-redundant and design consumers to resume by time and tolerate replay."*

---

## Concept 31 — Event Hubs traps, and where it fits

**1. Too few partitions on Standard.** Immutable, so the consumer you add next year can't parallelize beyond today's count. Over-provision modestly (Concept 25).

**2. Sharing a consumer group between applications.** Two apps in one group steal partitions from each other and each sees half the stream. One group per independent consumer.

**3. Swallowed or escaping exceptions.** Either the handler throws and events are skipped, or it catches and logs without parking — both lose data silently. Park failures and alert (Concept 27).

**4. Checkpoint storms.** Checkpointing after every event across hundreds of partitions hammers the Blob account and throttles. Batch checkpoints; use a dedicated storage account for checkpoints.

**5. Tiny events on Standard.** 64 KB metering and per-event overhead make 100-byte events expensive per byte; aggregate.

**6. Auto-inflate as autoscale.** It only scales up (Concept 25).

**7. Using it as a queue.** No per-message settlement, no dead-letter, no scheduled delivery, no sessions — if the work items need those, it's Service Bus.

**8. Expecting global order.** Order exists per partition only; a consumer that needs global order must re-sequence (by event time with a watermark) — usually a sign the requirement is really per-entity order.

**9. Retention as a database.** Seven or even ninety days of retention is a replay window, not a system of record; use Capture or a store for anything longer.

**Where it fits:** telemetry, clickstream, logs, IoT ingestion, CDC streams, anything where multiple independent consumers read the same high-volume data and replay matters, and Kafka-compatible workloads without brokers to operate. **Where it doesn't:** per-message workflows, low-volume commands, and anything requiring broker-side retry and dead-lettering.

**The interview-grade sentence:** *"The Event Hubs traps are under-partitioning an immutable Standard hub, two apps sharing a consumer group, handlers that let exceptions escape or swallow them without parking, checkpointing every event, tiny events under 64 KB metering, treating auto-inflate as autoscale, using it as a queue or a database, and expecting global order. It fits high-volume streams with multiple replaying consumers — not per-message workflows."*

---
# Part E — Azure Cosmos DB as an operated platform

## Concept 32 — What earlier modules established, and what operating it adds

Cosmos DB has appeared in this curriculum more than any other product, so start by naming what you already have:

- **Module 7:** the five consistency levels (strong, bounded staleness, session, consistent prefix, eventual), their RU and latency costs, the K/T minimums, the 5,000-mile strong-consistency block, session tokens across instances, and `ReadConsistencyStrategy`.
- **Module 8:** logical vs physical partitions, the partition-aware SDK and its routing cache, the five partition-key criteria, hierarchical partition keys, and the change feed as the materialized-view mechanism.
- **Module 12:** the account → database → container → logical partition → physical partition hierarchy; 20 GB per logical partition, ~50 GB and 10,000 RU/s per physical partition, 2 MB items, `TransactionalBatch` limits; access-pattern-first modeling; single-table design; schema evolution without DDL.
- **Module 5:** the throughput floor that storage imposes, and the partition-count arithmetic where **storage often drives the number of physical partitions**, quietly lowering RU per partition.
- **Module 19:** the EF Core Cosmos provider and its limits.
- **Module 24:** event stores and projections — for which the change feed is the natural Cosmos mechanism.

What none of those covered is **running** Cosmos: the account-level decisions you can't easily undo, how RUs are spent and measured, the throughput modes and when each wins, operating partitions (splits, hot partitions, redistribution, merge, partition-key changes), the 2026 features (GSIs, PPAF, distributed transactions), the change feed as an *integration* backbone, multi-region failover behavior, backup, SDK configuration, security and the traps. That's this part.

**The interview-grade sentence:** *"I treat Cosmos DB as two layers: the data-model layer — partition key, item design, consistency per operation — which the theory covers, and the operating layer — account-level choices, RU spend, throughput mode, partition operations, failover, backup, SDK configuration and security — which decides whether a good model survives production."*

---

## Concept 33 — Account-level decisions and one-way doors

Some Cosmos settings are easy to change; others are one-way doors or expensive migrations. Decide the second kind early and on purpose.

| Decision | Options | Changeable later? | How to decide |
|---|---|---|---|
| **API** | **NoSQL** (native), MongoDB (RU), Cassandra, Gremlin, Table | **No** — new account + migration | NoSQL unless you're migrating a workload that speaks Mongo/Cassandra wire protocols. New features ship to NoSQL first (GSIs, distributed transactions, vector/full-text search, PPAF) |
| **Product** (not just API) | Cosmos DB vs **Azure DocumentDB** (former vCore Mongo) vs PostgreSQL options | — | Azure DocumentDB is a separate, vCore-priced MongoDB-compatible service; for distributed PostgreSQL, Microsoft's guidance steers new projects to Azure Database for PostgreSQL (elastic clusters) — check current status (Concept 48) |
| **Capacity mode** | Provisioned (manual/autoscale) or **serverless** | Serverless → provisioned is supported (one-way) | Serverless for dev/test and spiky, low-average workloads without latency/throughput guarantees; provisioned for production with SLAs |
| **Service tier** | **General Purpose** (SLA up to 99.995%) or **Business Critical** (up to 99.999%; multi-region writes; PPAF) | Yes, with cost implications | Business Critical when you need multi-region writes or per-partition automatic failover |
| **Regions** | One or many; which one is the write region | Add/remove online; change write region when healthy | Users' locations, data residency, failover pairs, latency budget (Module 5) |
| **Zone redundancy per region** | On/off | **Only when a region is added** (workaround: add a temporary region); serverless only at creation | **On** wherever supported, especially single-region accounts — higher SLA, no latency penalty |
| **Default consistency** | Five levels | Yes (account-wide; per-request relaxation and stronger reads available) | Session, unless a concrete requirement says otherwise (Module 7) |
| **Backup mode** | **Periodic** or **continuous** (7-day or 30-day point-in-time restore) | Periodic → continuous supported (one-way) | Continuous for production: point-in-time restore, and required by several features (Fabric mirroring, all-versions-and-deletes change feed) |
| **Partition key** (per container) | Any path, or hierarchical (up to 3 levels) | Now possible (GA, June 2026) — but it's a copy-and-cutover migration | The single most important modeling decision (Modules 8, 12) |
| **Unique keys** (per container) | Unique constraints within a logical partition | Only at container creation | Define up front if needed |
| **Network and identity** | Public/private endpoints, key auth on/off, RBAC | Yes | Private endpoints, Entra RBAC, key auth off (Concept 45) |

**One account or several?** Account-wide settings drive the split (Concept 5): different consistency defaults, region sets, tiers, backup modes or network boundaries need different accounts. Otherwise, fewer accounts with several containers are simpler to secure and monitor. For multitenant SaaS, Cosmos supports partition-key-per-tenant (densest), container-per-tenant (isolation of throughput and indexing), and account-per-tenant (strongest isolation, with **fleets** to manage many accounts) — most systems use the first, with the others for large or regulated tenants.

**The interview-grade sentence:** *"In Cosmos the one-way doors are the API — NoSQL unless I'm migrating a wire protocol, since new features land there first — serverless versus provisioned, zone redundancy, which can only be set when a region is added, the backup mode, unique keys and, despite the new partition-key change feature, the partition key. Service tier, regions, consistency default, network and identity can change later. Account-wide settings decide how many accounts I need; tenants usually share containers by partition key."*

---

## Concept 34 — RU economics

The **request unit** is Cosmos's normalized currency for CPU, IOPS and memory. Module 12 called it "the rare architectural mistake with a directly observable price." Here's how the price is actually set.

**What things cost (NoSQL API, rough orders of magnitude):**

| Operation | Approximate cost | What changes it |
|---|---|---|
| **Point read** (`ReadItemAsync` with id + partition key) of a 1 KB item | **1 RU** (session/eventual); **2 RU** at strong or bounded staleness | Item size (roughly linear) |
| **Write** (create/replace/upsert) of a 1 KB item, default indexing | **~5–6 RU** | Item size, **number of indexed paths**, triggers, GSIs on the container |
| **Patch** | Usually less than a full replace for large items | Number of operations, indexed paths changed |
| **Delete** | Similar to a write | Indexing, GSIs |
| **Query** | Highly variable: from ~2–3 RU for an index seek to thousands for scans | Index utilization, result size, number of physical partitions touched (≈2–3 RU each for cross-partition fan-out), `ORDER BY` without a composite index, aggregates |
| **Change feed read** | Charged per page read | Batch size, item size |

**Three rules follow:**

1. **Point reads beat queries.** If you know the id and partition key, `ReadItemAsync` is the cheapest, fastest, most predictable operation in the database. A `SELECT * FROM c WHERE c.id = @id` *query* for the same item costs several times more. Model so the hot paths are point reads (Module 12, Concept 33).
2. **Indexing is a write tax you choose.** By default every property is indexed. Each indexed path makes every write more expensive and the index larger. **Exclude paths you never filter or sort on** (`/*` excluded, then include only queried paths, for write-heavy containers), add **composite indexes** for multi-property `ORDER BY` and filter+sort combinations (they turn expensive queries into cheap ones), and use **vector** and **full-text** index types only where those queries exist.
3. **Every response tells you the price.** `ItemResponse<T>.RequestCharge`, `FeedResponse<T>.RequestCharge`, and the diagnostics string carry the RU cost. Log it for hot paths, assert on it in performance tests, and use **index metrics** (`QueryRequestOptions.PopulateIndexMetrics = true`) to see which indexes a query used and which it wished it had.

```csharp
// Measure, don't guess: RU per operation in tests and telemetry.
ItemResponse<Order> read = await orders.ReadItemAsync<Order>(id, new PartitionKey(customerId));
metrics.RecordRu("orders.read", read.RequestCharge);             // ~1 RU for a 1 KB item at session

var q = orders.GetItemQueryIterator<OrderSummary>(
    new QueryDefinition("SELECT c.id, c.total FROM c WHERE c.customerId = @c AND c.status = @s ORDER BY c.createdAt DESC")
        .WithParameter("@c", customerId).WithParameter("@s", "Open"),
    requestOptions: new QueryRequestOptions
    {
        PartitionKey          = new PartitionKey(customerId),   // single-partition: no fan-out
        MaxItemCount          = 50,
        PopulateIndexMetrics  = true                             // dev/test: shows used and potential composite indexes
    });
FeedResponse<OrderSummary> page = await q.ReadNextAsync();
logger.LogDebug("Query RU {Ru}; index metrics {Metrics}", page.RequestCharge, page.IndexMetrics);
```

```jsonc
// Indexing policy for a write-heavy orders container: index only what's queried, plus a composite for the hot query.
{
  "indexingMode": "consistent",
  "includedPaths": [ { "path": "/customerId/?" }, { "path": "/status/?" }, { "path": "/createdAt/?" } ],
  "excludedPaths": [ { "path": "/*" } ],
  "compositeIndexes": [[ { "path": "/status", "order": "ascending" }, { "path": "/createdAt", "order": "descending" } ]]
}
```

**From RU to money.** Provisioned manual throughput costs about **$0.008 per 100 RU/s per hour** per region (≈ $5.84 per 100 RU/s per month), autoscale **1.5×** that per RU/s of the hourly peak, serverless about **$0.25 per million RUs consumed**, and storage about **$0.25 per GB-month** (transactional). So a 1,000 RU/s container is about $58/month per region; a workload that averages 20,000 RU/s with peaks of 50,000 is a few thousand dollars per region per month — which is why RU optimization is an architecture conversation.

**The interview-grade sentence:** *"RUs price every operation: a 1 KB point read is about 1 RU, a 1 KB write about 5 to 6 with default indexing, and queries range from a few RU for an index seek to thousands for scans, plus 2 to 3 RU per physical partition when they fan out. So I make hot paths point reads, treat indexing as a write tax — exclude unused paths and add composite indexes for the hot sorts — and measure RequestCharge and index metrics on every hot path, because at roughly $5.84 per 100 RU/s per region per month, a bad query is a line item."*

---

## Concept 35 — Throughput modes: manual, autoscale, dynamic scaling, serverless, and the extras

Cosmos gives you several ways to buy RUs. Choosing correctly is one of the biggest cost levers in an Azure estate.

**1. Manual (standard) provisioned throughput.** You set RU/s on a container (or a database). You pay for it every hour whether used or not; requests beyond it get **429 (Too Many Requests)**. Minimum is 400 RU/s, and the floor rises with storage and with the highest RU/s ever set (Module 5). Best for **steady, predictable** load running near its provisioned level.

**2. Autoscale.** You set a **maximum** `Tmax`; Cosmos scales instantly between **10% and 100% of `Tmax`** based on usage, and bills each hour for **the highest RU/s it scaled to in that hour**, at **1.5×** the manual rate (single-write-region accounts). Break-even from first principles: if `u` is the average over hours of (hourly peak ÷ `Tmax`), autoscale costs `1.5 × u × Tmax` versus `1.0 × Tmax` for manual provisioned at the peak — so **autoscale is cheaper whenever the hourly peaks average below about two-thirds of the maximum**, which is true for most business workloads with daily cycles. It's worse for workloads that sit near the peak all day.

**3. Dynamic scaling (per-region and per-partition autoscale).** Classic autoscale scales *all* physical partitions in *all* regions to the level needed by the hottest partition in the hottest region. **Dynamic scaling** lets each partition and region scale independently and bills the **sum of each one's hourly peak** — much cheaper for skewed partitions and for multi-region accounts where a read region is quiet. It's **enabled by default for accounts created after September 25, 2024**; older accounts can enable it in the portal. Customers report double-digit percentage savings with no code change. If you see an autoscale account with uneven partitions and dynamic scaling off, that's a one-click cost finding.

**4. Serverless.** No provisioned capacity; you pay per million RUs consumed plus storage. Each container's throughput scales with its physical partitions (**about 5,000 RU/s per physical partition** in current documentation), with **no guarantees of predictable throughput or latency** and no throughput SLA. Use it for development and test, prototypes, low-traffic internal apps, and spiky workloads whose average is a tiny fraction of their peak. Don't use it for latency-SLO-bound production paths. Moving from serverless to provisioned is supported; the reverse isn't.

**5. Shared (database-level) throughput.** RU/s provisioned on a database and shared by up to 25 containers. Cheap for many small containers (container-per-tenant designs, many low-traffic microservice containers), with **noisy-neighbor** risk: one container's burst can starve the others. Containers can opt out with dedicated throughput.

**Extras that shape behavior at the edges:**

- **Burst capacity**: a physical partition provisioned below 3,000 RU/s accumulates up to **5 minutes of idle capacity** and can spend it at up to **3,000 RU/s** to absorb short spikes without 429s. Free; enable it on the account.
- **Priority-based execution**: tag requests as **high** or **low** priority (`RequestOptions.PriorityLevel`); when a partition is over its RU budget, **low-priority requests are throttled first**. Perfect for keeping user-facing reads alive while a backfill or ETL runs against the same container.
- **Throughput buckets** (check current status — preview at last look): carve a container's throughput into buckets with a maximum share each (for example, "ETL may use at most 20%"), and tag requests with a bucket — server-side isolation between workloads sharing a container.
- **Fleets**: centralized management of many accounts (account-per-tenant SaaS), including pooled throughput across accounts — check current feature status for your design.

**Choosing, in one table:**

| Workload shape | Mode |
|---|---|
| Steady, high utilization (average ≥ ~66% of peak) | **Manual**, ideally with reserved capacity |
| Daily cycles, business hours, unpredictable growth | **Autoscale** with **dynamic scaling** |
| Skewed partitions or quiet read regions | **Autoscale + dynamic scaling** (default for new accounts) |
| Dev/test, prototypes, bursty and mostly idle | **Serverless** |
| Many small containers, tolerant of shared fate | **Shared database throughput** |
| Mixed user-facing and batch on one container | Any provisioned mode + **priority-based execution** (and throughput buckets where available) |

**The interview-grade sentence:** *"Manual throughput wins for steady load near its peak; autoscale scales between 10% and 100% of its max and bills the hourly peak at 1.5 times the rate, so it wins whenever hourly peaks average below about two-thirds of the max; dynamic scaling, on by default for accounts since September 2024, scales each partition and region independently and fixes the hot-partition and quiet-region waste; serverless is per-RU with no guarantees, for dev and spiky low-average work. Burst capacity absorbs short spikes, and priority-based execution keeps user traffic alive while batch jobs run on the same container."*

---

## Concept 36 — Partition operations: splits, hot partitions, redistribution and merge

You never address physical partitions directly, but you operate them indirectly every day.

**How partitions are created.** A container starts with a number of physical partitions determined by its initial RU/s (roughly one per 10,000 RU/s) and grows by **splitting**:

- when a physical partition's data exceeds about **50 GB**, it splits into two, each holding roughly half;
- when you raise RU/s beyond **current partitions × 10,000 RU/s**, Cosmos must split to have enough partitions — that scale-up is **asynchronous** (it can take from minutes to hours), whereas a scale-up within current capacity is instant.

Splits are online and invisible to correct clients (the SDK handles routing changes — Module 8's `410 Gone` / partition-split handling). Their side effect matters: **provisioned RU/s are divided evenly across physical partitions**, so more partitions means **less RU per partition**.

**Diagnosing "429s at 30% utilization."** This is the most common Cosmos production complaint, and it's almost always one of three things:

1. **A hot logical partition key** — one key receiving a large share of traffic, capped by its physical partition's share of RU/s (and an absolute 10,000 RU/s).
2. **A skewed physical partition** — several busy keys happen to share one physical partition.
3. **Bursty traffic** — per-second spikes far above the per-hour average the dashboard shows.

The tools:

| Signal | Where | What it tells you |
|---|---|---|
| **Normalized RU Consumption** by `PartitionKeyRangeId` | Azure Monitor metrics | The busiest physical partition's utilization (0–100%) — the number that actually throttles |
| **429 rate** (Total Requests by status code 429, substatus **3200** = rate limited) | Metrics, SDK diagnostics | How often you're throttled and on which operations |
| **Partition key statistics / top keys by RU** | Diagnostic logs (`CDBPartitionKeyRUConsumption`, `CDBPartitionKeyStatistics`) in Log Analytics | Which **logical** keys consume the RU and storage |
| **SDK diagnostics** (`ItemResponse.Diagnostics`) | Your logs for slow or throttled requests | Retries, regions contacted, backend latency |

A useful rule of thumb from Microsoft's guidance: a **small, steady rate of 429s (on the order of 1–5% of requests) with acceptable end-to-end latency is healthy** — it means you're using what you pay for, and the SDK's automatic retries absorb it. Sustained high 429 rates, or 429s on a single partition while others idle, are design problems.

**The fixes, in order of preference:**

1. **Fix the key** (Module 8): a more granular key, a hierarchical key (Concept 37), or a synthetic suffix for write-hot keys. Throughput won't fix a hot key.
2. **Enable dynamic scaling** (Concept 35) so the hot partition scales without paying for every partition.
3. **Redistribute throughput across physical partitions** (preview feature, CLI/PowerShell): give a hot *physical* partition more of the container's RU/s (still capped at 10,000 per partition) — useful for skew among several keys, useless for one key.
4. **Partition merge** (check current status): after you lower RU/s or delete data, a container may be left with many sparse partitions — which spreads RU thin and makes cross-partition queries fan out more. Merge reduces the partition count.
5. **Change the partition key** (GA since June 2026): a service-managed copy into a container with the new key and a cutover — powerful, but a migration with a plan, not a setting.
6. **Add a GSI** (Concept 38) when the problem is *cross-partition reads* rather than *hot writes*.

**The interview-grade sentence:** *"Physical partitions split at about 50 GB or when I raise RU/s beyond partitions times 10,000, and provisioned RU/s divide evenly across them — so '429s at 30% utilization' usually means one partition is at 100% while the average looks fine. I diagnose with normalized RU consumption per partition key range, 429 substatus 3200 and the partition-key RU logs, accept a small steady 429 rate as healthy, and fix hot keys by changing the key, with dynamic scaling, throughput redistribution and merge as tools for skew and sparse partitions."*

---

## Concept 37 — Hierarchical partition keys in practice

Modules 8 and 12 made the case for **hierarchical partition keys (HPK)**: up to **three levels** — typically `/tenantId` → `/userId` → `/id` — so a first-level value can exceed 20 GB and 10,000 RU/s while queries on a **prefix** are routed only to the partitions holding it.

The operational details that interviews reach:

- **Routing works on prefixes in order.** A query filtering on `tenantId` (level 1), or `tenantId` and `userId` (levels 1–2), is routed to the subset of physical partitions holding that prefix. A query filtering only on `userId` (level 2) **without** `tenantId` can't use the hierarchy and fans out.
- **Point reads need the full key.** `ReadItemAsync` takes a `PartitionKey` built with all levels (`PartitionKeyBuilder().Add(t).Add(u).Add(id)`).
- **Ending with `/id`** gives effectively unlimited storage per tenant (each logical partition is a single item), at the cost that transactional batches are limited to items sharing the *full* key — which for `/id` is one item. If you need multi-item transactions within a user, end with `/userId` instead and live with the 20 GB per user.
- **It's chosen at container creation** (or via the new partition-key change migration).
- **Some features don't support HPK yet** — at the time of writing, both the **distributed transactions preview** and **Azure Backup for Cosmos DB (preview)** list hierarchical partition keys as unsupported. Check the feature matrix before combining HPK with new features.
- **SDK support** has been in the .NET SDK since 3.33; use a current version.

```csharp
var props = new ContainerProperties("activity", new List<string> { "/tenantId", "/userId", "/id" });
Container activity = await db.CreateContainerIfNotExistsAsync(props, ThroughputProperties.CreateAutoscaleThroughput(20_000));

// Point read: full key.
var pk = new PartitionKeyBuilder().Add(tenantId).Add(userId).Add(activityId).Build();
var item = await activity.ReadItemAsync<Activity>(activityId, pk);

// Prefix query: routed to the partitions holding this tenant only.
var tenantPrefix = new PartitionKeyBuilder().Add(tenantId).Build();
var it = activity.GetItemQueryIterator<Activity>(
    "SELECT * FROM c WHERE c.type = 'login' ORDER BY c.at DESC",
    requestOptions: new QueryRequestOptions { PartitionKey = tenantPrefix });
```

**The interview-grade sentence:** *"With hierarchical partition keys like tenantId, userId, id, queries are routed by prefix in order — tenant, or tenant plus user — but a filter on the second level alone fans out; point reads need the full key; ending with id makes per-tenant storage effectively unlimited but limits transactional batches to one item, so I end with userId if I need per-user transactions. It's a creation-time choice, and I check the feature matrix because some previews, like distributed transactions and Azure Backup, don't support HPK yet."*

---

## Concept 38 — Global secondary indexes

A container has exactly one partition key, but real systems query by several keys: orders by customer *and* by order number *and* by merchant. Before 2026 you solved that by writing a **change-feed-driven projection** into a second container yourself (Module 12, Concept 33's "duplicate it" step). **Global secondary indexes (GSIs)** — **GA since June 2026**, the evolution of the earlier "materialized views" preview — make Cosmos maintain that second container for you.

**How they work:**

- A GSI is a **read-only container** defined from a **source container** plus a **definition query** (a projection: all properties or a subset) and its **own partition key, throughput and indexing policy**.
- Cosmos keeps it in sync using the **change feed**, through a managed job; reads from the change feed are charged to the **source**, writes into the GSI to the **GSI's** throughput.
- GSI containers must use **autoscale** throughput (so they can absorb bursts without falling far behind).
- The source container and definition query **can't be changed** after creation.
- Sync is **eventually consistent**; monitor the **Global Secondary Index Propagation Latency** metric.
- There's a write-side tax: once a container has GSIs, **replace and delete operations on the source cost noticeably more RU** (Microsoft cites roughly 50–100% more; creates aren't affected).

**GSI or hand-built projection?**

| | **GSI** | **Your own change-feed projection** |
|---|---|---|
| Effort | Configuration | Code, hosting, leases, monitoring, replays |
| Shape | Same items, re-keyed, projected (subset of properties) | Anything: joins, aggregates, denormalized views, other stores (Search, SQL, Redis) |
| Freshness | Managed, eventual; latency metric | Yours to manage |
| Cost | Source write premium + GSI RU | Change feed reads + compute + target writes |
| Failure handling | Managed | Yours (Concept 41) |

Use a **GSI** when you need the *same data* addressable by *another key* — the classic alternate-lookup problem, or isolating vector/full-text search load from the transactional container. Use your **own projection** when the read model is genuinely different (aggregates, joins, different store) — Module 23's CQRS read models.

**The interview-grade sentence:** *"Global secondary indexes, GA since June 2026, are read-only containers Cosmos keeps in sync from a source container through the change feed, each with its own partition key, autoscale throughput and indexing policy, so a cross-partition query becomes a single-partition lookup — eventually consistent, with a propagation-latency metric, and with replaces and deletes on the source costing roughly 50 to 100% more RU. I use them for alternate keys over the same data and build my own change-feed projections when the read model is genuinely different."*

---

## Concept 39 — Consistency in production

Module 7 derived the levels; operating them adds a few practical rules.

**1. Session is the default for a reason** — and it's per *client session*, which in a web farm means per *SDK client instance* unless you flow tokens. Writes return a session token; a later read on a *different* instance won't see the write unless you pass the token (`ItemRequestOptions.SessionToken`), which Module 7 showed flowing through a cookie. Two practical refinements:

- **Session tokens are per partition key range**, so the token for one user's partition doesn't cover another's — flow the token for the data the user just wrote, scoped to that operation.
- If flowing tokens is impractical, route the user's follow-up reads to the **write region** with the **same client**, or accept "read-your-writes within a request" and design the UI around it (optimistic UI after a write).

**2. Relax per request, strengthen deliberately.** Per-request `ConsistencyLevel` can make a read *weaker* (eventual for a dashboard counter). Since SDK 3.46, `ReadConsistencyStrategy` can request *stronger* read guarantees per read without changing the account default — for the few reads that must be up to date (an authorization check right after a permission change).

**3. Measure staleness instead of guessing.** Azure Monitor exposes **replication latency** between regions and the **probabilistically bounded staleness (PBS)** metric — how often reads at session or eventual actually return stale data. In most single-region accounts, it's nearly never; across regions, it's a function of distance.

**4. Region choice and consistency interact.**

- **Strong** across regions makes every write wait for a majority of regions (latency = round trip to them), doubles read RU cost, is blocked by default for regions more than 5,000 miles apart, and isn't available with multi-region writes. It gives **RPO 0**.
- **Bounded staleness** across regions has minimums of **100,000 operations or 300 seconds** — and that bound is effectively your **RPO** for a region loss.
- **Session, consistent prefix and eventual** have an RPO under about **15 minutes** for a region outage according to Microsoft's reliability guidance (in practice usually much less).

**5. Consistency and multi-region writes.** With multiple write regions (Business Critical), conflicts become possible and session guarantees hold per region; design with conflict resolution (Concept 42) or route each entity's writes to a home region.

**The interview-grade sentence:** *"In production I keep session as the default, flow session tokens per user for read-your-writes across instances — remembering they're scoped per partition key range — relax individual reads to eventual where staleness is harmless and use ReadConsistencyStrategy for the few reads that must be strong. I measure staleness with replication latency and PBS rather than guessing, and I tie consistency to RPO across regions: strong gives zero, bounded staleness gives its K and T, and the others are under about fifteen minutes."*

---

## Concept 40 — Transactions: batch, ETags, patch — and distributed transactions

Cosmos has always offered transactions **within one logical partition**. In 2026 it added a preview of transactions **across** partitions. Know both, and know when neither is the right tool.

**Within a logical partition:**

| Mechanism | What it gives you | Limits |
|---|---|---|
| **`TransactionalBatch`** | ACID, all-or-nothing operations (create, replace, upsert, patch, read, delete) on items sharing one partition key value | Up to **100 operations** and **2 MB** per batch; single logical partition |
| **Optimistic concurrency with `_etag`** | `IfMatchEtag` on a replace/patch/delete fails with **412 Precondition Failed** if the item changed since you read it | Per item; you implement the retry (Module 19's pattern) |
| **Patch (partial document update)** | Up to **10** operations (set, add, replace, remove, increment, move) applied atomically to one item, optionally conditional on a **filter predicate** (`FROM c WHERE c.status = 'Pending'`) | Single item |
| **Stored procedures and triggers** | JavaScript run inside the partition with ACID semantics | Single logical partition; bounded execution time; largely superseded by batch + patch for new code |

```csharp
// Aggregate and its outbox entry, atomically, in one logical partition (orderId as partition key).
TransactionalBatchResponse r = await orders.CreateTransactionalBatch(new PartitionKey(order.Id))
    .ReplaceItem(order.Id, order, new TransactionalBatchItemRequestOptions { IfMatchEtag = order.ETag })
    .CreateItem(new OutboxEntry(order.Id, "OrderConfirmed", Guid.NewGuid()))     // same partition key value
    .ExecuteAsync(ct);

if (r.StatusCode == HttpStatusCode.PreconditionFailed) throw new ConcurrencyConflictException(order.Id);
if (!r.IsSuccessStatusCode) throw new CosmosBatchFailedException(r.StatusCode, r.ErrorMessage);

// Conditional increment without read-modify-write.
await stock.PatchItemAsync<StockItem>(sku, new PartitionKey(sku),
    [PatchOperation.Increment("/reserved", qty)],
    new PatchItemRequestOptions { FilterPredicate = $"FROM s WHERE s.onHand - s.reserved >= {qty}" });
```

Note what the batch example achieves: the **outbox in the same partition as the aggregate** gives you a transactional outbox *inside Cosmos* — and the change feed is the relay (Concept 41). That design depends entirely on putting the outbox entry under the aggregate's partition key.

**Distributed transactions (public preview, since June 2026).** The .NET SDK adds `CosmosClient.CreateDistributedWriteTransaction()` and `CreateDistributedReadTransaction()`: atomic writes, and point-in-time-consistent snapshot reads, **across logical partitions, containers and databases** within one account and region.

```csharp
DistributedTransactionResponse response = await client.CreateDistributedWriteTransaction()
    .ReplaceItem("banking", "accounts", new PartitionKey("account-A"), debited)
    .ReplaceItem("banking", "accounts", new PartitionKey("account-B"), credited)
    .CreateItem ("banking", "ledger",   new PartitionKey("2026-10"),   ledgerEntry)
    .CommitTransactionAsync(ct);
```

What the preview documentation says, and what an architect should do with it:

- **Limits:** up to **100 operations** and **2 MB** per transaction.
- **Scope:** NoSQL API, **provisioned** throughput, **single write region**; in multi-region accounts, atomic **within the write region only** — replication to read regions is asynchronous **per partition**, so readers in other regions can briefly see partial results.
- **Incompatible (in preview) with:** customer-managed keys, **per-partition automatic failover**, **continuous backup**, long-term retention, partition merge, **hierarchical partition keys**, Fabric native databases, serverless and multi-region-write accounts. That's a long list — several items (continuous backup, PPAF, HPK) are things this module recommends for production.
- **Status:** preview, enrollment by form, no SLA; behavior and limits may change.

**So when do you use what?**

- **Single aggregate or aggregate + outbox:** batch or patch in one logical partition — the design target (Module 22's aggregate as consistency boundary maps onto the logical partition).
- **A small number of items across partitions in one account, where atomicity truly matters, and the preview's constraints are acceptable:** distributed transactions — after GA, for production.
- **Across services, stores or long-running steps:** a **saga** with compensation (Module 12) — no database transaction spans a Service Bus queue, a payment provider and Cosmos.
- **Most cross-partition "transactions" in practice** are better redesigned so the invariant lives in one partition, or made idempotent and eventually consistent.

**The interview-grade sentence:** *"Within a logical partition Cosmos gives me ACID TransactionalBatch up to 100 operations and 2 MB, ETag optimistic concurrency, and conditional patch — enough to commit an aggregate and its outbox entry atomically when they share a partition key. Distributed transactions across partitions and containers arrived in preview in June 2026 with the same 100-operation limit, but only in the write region, without an SLA, and incompatible today with continuous backup, PPAF, hierarchical keys and CMK — so for production I still put invariants in one partition and use sagas across services, and I'd revisit distributed transactions at GA."*

---
## Concept 41 — The change feed as an integration backbone

Every Cosmos container exposes a **change feed**: a persistent, **per-partition ordered** log of changes. Module 8 introduced it as the materialized-view mechanism; in a platform design it's much more — the way Cosmos **publishes**, the relay for the Cosmos-native outbox, the source for GSIs, search indexes, caches, analytics and event-driven integration.

**Two modes:**

| | **Latest version** (default) | **All versions and deletes** (GA since June 2026) |
|---|---|---|
| What you see | Inserts and updates — the **latest version** of each changed item (intermediate updates between reads may be collapsed) | **Every** create, update and **delete**, each as its own change with operation metadata |
| Deletes | **Not included** — use soft delete (`isDeleted` flag) plus TTL if consumers must learn about deletions | Included |
| History | From the beginning of the container | Only within the **continuous backup retention window** (7 or 30 days) — continuous backup is a prerequisite |
| Start from | Beginning, a point in time, or now | A point within the retention window, or now |
| Typical use | Projections, caches, search sync, outbox relay | Auditing, exact replication, CDC to other stores, compliance, consumers that must see each intermediate state |

**Ordering and delivery.** Changes are ordered **within a logical partition key** (by modification time, `_lsn`) and not across keys. Delivery is **at-least-once**: a consumer can see a change again after a crash or rebalance. Exactly the same discipline as Event Hubs applies — idempotent handlers keyed on item id and version (`_etag` or `_lsn`).

**Three ways to consume it in .NET:**

1. **Change Feed Processor (CFP)** — the push-style library in the SDK. Instances share a **lease container**; each lease covers a **feed range** (a set of partition key ranges); instances acquire and balance leases automatically as they scale. It's the right default for services.
2. **Pull model** (`GetChangeFeedIterator` with `FeedRange`) — you control iteration and parallelism yourself; useful for jobs, migrations and custom scheduling.
3. **Azure Functions Cosmos DB trigger** — CFP hosted by Functions, with leases managed for you and target-based scaling (Module 26, Concept 25).

```csharp
Container leases = db.GetContainer("leases");                       // partition key /id; cheap, can be shared

ChangeFeedProcessor processor = orders
    .GetChangeFeedProcessorBuilder<OrderDocument>("orders-to-servicebus", HandleChangesAsync)
    .WithInstanceName(Environment.MachineName)
    .WithLeaseContainer(leases)
    .WithStartTime(DateTime.UtcNow.AddMinutes(-5))                    // first run only; afterwards leases decide
    .WithMaxItems(100)
    .WithPollInterval(TimeSpan.FromSeconds(1))
    .Build();

async Task HandleChangesAsync(ChangeFeedProcessorContext context, IReadOnlyCollection<OrderDocument> changes, CancellationToken ct)
{
    foreach (OrderDocument doc in changes)
    {
        try
        {
            if (doc.Type != "outbox") continue;                         // relay only outbox entries
            await sender.SendMessageAsync(new ServiceBusMessage(doc.Payload)
            {
                MessageId = doc.Id,                                     // duplicate detection drops replays (Concept 13)
                Subject   = doc.EventType,
                ApplicationProperties = { ["aggregateId"] = doc.AggregateId }
            }, ct);
        }
        catch (Exception ex) when (IsPermanent(ex))
        {
            await poison.StoreAsync(context.LeaseToken, doc, ex, ct);  // park, don't block the lease
        }
        // Transient exceptions propagate: the CFP retries this batch on the same lease.
    }
}

// Lag: how far behind each lease is — alert and scale on it.
ChangeFeedEstimator estimator = orders.GetChangeFeedEstimator("orders-to-servicebus", leases);
```

**The behavior that differs from Event Hubs** — and matters: when your CFP delegate **throws**, the processor **retries the same batch on the same lease** rather than skipping it. That's safer than silent skipping, but a **poison item blocks its lease** (and everything behind it in those partition key ranges) indefinitely. So: let transient errors throw (retry is what you want), and catch and **park** permanent errors.

**The Cosmos-native outbox.** Combine Concept 40 and this one:

1. In one `TransactionalBatch`, write the aggregate change **and** an outbox item **in the same logical partition**.
2. A CFP (or a Functions Cosmos trigger) reads the change feed and publishes outbox items to Service Bus or Event Hubs with **`MessageId = outbox id`**.
3. Duplicate detection on the destination absorbs relay replays within its window; consumers stay idempotent.
4. Remove outbox items with **TTL** (they've done their job once the lease moves past them) — or keep them for audit.

No polling, no second database, no dual write. The alternative — publishing **every aggregate change** as an event straight from the change feed — works too, but couples your event contract to your storage schema; an explicit outbox item lets you shape the integration event deliberately (Module 11's "events are contracts").

**Operational notes.** Change feed reads consume **RU from the source container** (budget for them); give each independent consumer its **own processor name** (they can share one lease container); the **estimator** gives per-lease lag; and starting a new processor from the beginning on a large container is a **backfill** — schedule it with **low priority** (Concept 35) so it doesn't throttle user traffic.

**The interview-grade sentence:** *"The change feed is Cosmos's built-in, per-partition ordered, at-least-once change log — latest-version mode for inserts and updates, and since June 2026 all-versions-and-deletes, which needs continuous backup and only reaches back through its retention window. I consume it with the change feed processor, which balances leases across instances and retries a batch on the same lease when my delegate throws — so I park permanent failures to avoid blocking a lease — and I use it as the relay for a Cosmos-native outbox: aggregate and outbox item in one transactional batch, published with the outbox id as MessageId."*

---

## Concept 42 — Multi-region, failover and multi-region writes

Cosmos makes multi-region data distribution a configuration change. Making the *application* survive a region loss still takes design. Microsoft's reliability guidance gives the full behavior matrix; the architect's version:

**The configurations:**

| Configuration | Write availability during a write-region outage | RPO (region loss) | Tier |
|---|---|---|---|
| **Single region** (zone-redundant) | Lost until the region recovers | 0 for zone loss; region loss → wait (or restore from backup elsewhere) | General Purpose |
| **One write region + read regions**, no automatic failover | Lost until you **force a failover** (take the region offline) | 0 with strong; otherwise unreplicated writes may be lost (session/prefix/eventual: under ~15 min) | General Purpose |
| … with **service-managed failover** | Restored when **Microsoft declares the outage and fails over — which can take an hour or more** | Same | General Purpose |
| … with **per-partition automatic failover (PPAF)** | **Affected partitions fail over automatically, about 3 minutes at p99**; healthy partitions stay put | Same as above per consistency level | **Business Critical** |
| **Multiple write regions** | Writes continue in other regions; the SDK reroutes | Recent writes in the failed region may be temporarily unavailable; reconciled via conflict resolution after recovery | **Business Critical** |

**What each failover type means operationally:**

- **Forced failover (offline region)** — your action. Fast to execute (seconds), but you must detect the outage and decide. **Microsoft recommends it over waiting for service-managed failover** when you need write availability back quickly. Possible data loss of unreplicated writes (unless strong).
- **Service-managed failover** — Microsoft's action, triggered when the outage is declared; slow by design. Treat it as a backstop.
- **PPAF** — the 2026 answer to "we want single-region-write simplicity with automatic, fast failover": a partition-level failover manager detects affected partitions and moves their writes to a secondary region within minutes, without application changes (the SDK follows redirects). Supported consistency levels at GA: strong, session, consistent prefix and eventual. It's part of the **Business Critical** tier — price that in.
- **Change write region** — planned and loss-free, but only when both regions are healthy; **not** an outage tool.

**Two outage rules from Microsoft's guidance that belong in every runbook:**

1. **Don't perform control-plane operations on the affected account during an outage** — changing write regions, failover priorities, consistency, networking, throughput or enabling multi-region writes can leave the account inconsistent and delay recovery.
2. **The recovered region does not automatically become the write region again.** Failing back is your decision, once it's safe; unreplicated writes from before the failover can be read from the **conflict feed**.

**Multi-region writes and conflicts.** With multiple write regions, two regions can update the same item concurrently. Conflict resolution policies:

- **Last writer wins (LWW)** — default, on `_ts` or a custom numeric property (a version or a business timestamp). Simple; loses the "losing" write silently.
- **Custom** — a merge stored procedure registered on the container, or **no procedure**, in which case conflicts go to the **conflict feed** for your application to resolve.

Module 7's lesson applies: LWW loses data, so prefer **structural single-writer** designs — each entity has a **home region** that takes its writes (route by user's home region, by tenant, by partition key), with multi-region writes used for availability and for entities that are genuinely commutative. Microsoft also warns that **updating or recreating the same item id very frequently across regions** degrades replication performance because of conflict volume.

**The client side — where most "multi-region" designs are actually decided:**

```csharp
var options = new CosmosClientOptions
{
    ApplicationName = "orders-api",
    // Order matters: nearest first. Set per deployment region (or use ApplicationRegion for automatic proximity ordering).
    ApplicationPreferredRegions = ["West Europe", "North Europe"],
    // Hedging: if the first region hasn't answered within 500 ms, also try the next one; first response wins.
    AvailabilityStrategy = AvailabilityStrategy.CrossRegionHedgingStrategy(
        threshold: TimeSpan.FromMilliseconds(500), thresholdStep: TimeSpan.FromMilliseconds(100)),
    ConnectionMode = ConnectionMode.Direct
};
```

- **Preferred regions** decide where reads (and, with multi-region writes, writes) go, and the failover order the SDK uses when a region fails. Unconfigured, everything goes to the write region — so a "multi-region" account read from Asia may still be served from Europe.
- **Hedging** (the threshold-based availability strategy) sends a parallel request to the next region when the first is slow — Module 25's hedging, built into the SDK — trading extra RUs for tail latency during regional degradation.
- **Partition-level circuit breaker** support in the SDK can steer requests for a failing partition to another region temporarily (enabled by configuration in recent SDK versions — check the docs for the switch in your version).
- **Excluded regions** can be set per request, useful in drills and for routing specific operations.

**Capacity for failover.** When a region fails, its traffic lands on the others. Provision (or autoscale) the surviving regions for the combined load — dynamic scaling (Concept 35) makes standby read regions cheaper because each region scales on its own usage.

**The interview-grade sentence:** *"For Cosmos multi-region I choose between one write region with read regions — where write availability after a write-region outage comes from a forced failover I trigger, a service-managed failover that can take an hour or more, or per-partition automatic failover, GA in June 2026, which moves affected partitions in about three minutes on the Business Critical tier — and multi-region writes, where conflicts resolve last-writer-wins, by a merge procedure or through the conflict feed, so I prefer a home region per entity. The SDK's preferred regions, hedging and circuit breaker decide what the application actually experiences, and the runbook says no control-plane changes during an outage and no automatic failback."*

---

## Concept 43 — Backup and restore

Replication protects against infrastructure failure; it faithfully replicates **your mistakes** too. A bad deployment that corrupts documents corrupts them in every region within seconds. That's what backup is for.

| | **Periodic backup** | **Continuous backup** |
|---|---|---|
| How | Full snapshots at an interval (default every 4 hours, keeping 2 copies — both configurable) | Continuous, enabling **point-in-time restore (PITR)** to any second in the window |
| Retention | Configurable (hours to days) | **7 days** or **30 days** tier |
| Restore | Request through support (historically) into a new account | **Self-service**, to a **new account**, at a chosen timestamp; restoring **deleted** databases/containers is also supported |
| Storage redundancy | Geo, zone or locally redundant backup storage | Managed |
| Prerequisite for | — | **Fabric mirroring**, **all-versions-and-deletes change feed**, and some other features |
| Migration | Periodic → continuous supported (one-way) | — |

**Azure Backup for Cosmos DB (preview since June 2026)** adds vault-based backups — **immutable** backups and **long-term retention** beyond the continuous window — for accounts on continuous backup, with preview limitations (for example, no hierarchical partition keys or PPAF accounts, no cross-region restore). It addresses the "ransomware and 7-year retention" requirement that PITR alone doesn't.

**Operational truths about restore:**

- **Restore creates a new account.** Your application's connection configuration, RBAC assignments, private endpoints and networking must be re-pointed or re-created — rehearse it. "We have backups" is a hypothesis until someone has restored one and an application has read from it.
- **Restore time** scales with data size; for large accounts it's hours. Factor it into RTO for the "logical corruption" scenario.
- **Partial recovery** is common: restore to a new account, then copy the affected items back (a small tool using the SDK or a container copy job), rather than cutting the whole application over.
- **Design for logical-error recovery in the data model too**: soft deletes, versioned documents, and event-sourced aggregates (Module 24) make many "restores" unnecessary.

**The interview-grade sentence:** *"Replication copies mistakes, so production Cosmos accounts use continuous backup with 7- or 30-day point-in-time restore — which is also a prerequisite for Fabric mirroring and the all-versions change feed — and restore always lands in a new account, so I rehearse re-pointing applications and often copy back only the damaged items. For immutability and long-term retention there's Azure Backup for Cosmos DB, in preview since June 2026 with feature limitations."*

---

## Concept 44 — The .NET SDK, configured

Most Cosmos performance and reliability problems in .NET services come from a handful of client settings. Here is a production-shaped registration, then why each line is there.

```csharp
builder.Services.AddSingleton(sp =>
{
    TokenCredential credential = sp.GetRequiredService<TokenCredential>();          // managed/workload identity (Concept 45)
    var options = new CosmosClientOptions
    {
        ApplicationName                     = "orders-api",
        ApplicationPreferredRegions         = builder.Configuration.GetSection("Cosmos:Regions").Get<string[]>(),
        ConnectionMode                      = ConnectionMode.Direct,                // default; Gateway for locked-down networks
        MaxRetryAttemptsOnRateLimitedRequests = 9,                                  // SDK retries 429s honoring retry-after
        MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(10),           // cap total wait inside a request budget
        EnableContentResponseOnWrite        = false,                                // don't send the item back on writes
        UseSystemTextJsonSerializerWithOptions = new JsonSerializerOptions(JsonSerializerDefaults.Web),
        AvailabilityStrategy                = AvailabilityStrategy.CrossRegionHedgingStrategy(
                                                  TimeSpan.FromMilliseconds(500), TimeSpan.FromMilliseconds(100)),
        CosmosClientTelemetryOptions        = new CosmosClientTelemetryOptions
        {
            DisableDistributedTracing = false,                                      // OpenTelemetry: Azure.Cosmos.Operation
            CosmosThresholdOptions    = new CosmosThresholdOptions { PointOperationLatencyThreshold = TimeSpan.FromMilliseconds(100) }
        }
    };

    // Warm up: open connections and fetch routing info for the containers this service uses, before taking traffic.
    return CosmosClient.CreateAndInitializeAsync(
        builder.Configuration["Cosmos:Endpoint"], credential,
        [("shop", "orders"), ("shop", "customers")], options).GetAwaiter().GetResult();
});
```

| Setting | Why |
|---|---|
| **Singleton `CosmosClient`** | It owns connections, the partition routing map and region health state; per-request clients are the #1 Cosmos performance bug (Module 8) |
| **Direct mode** | TCP connections straight to replicas — lowest latency; needs outbound ports in the 10000–20000 range. **Gateway mode** (HTTPS 443 via the gateway) for restrictive firewalls, some serverless hosts, or very large fleets where connection counts matter |
| **Preferred regions** | Reads (and multi-region writes) go to the nearest healthy region in your order (Concept 42) |
| **429 retry settings** | The SDK retries throttled requests automatically, honoring the server's retry-after. Cap the wait so it fits inside your request's latency budget, and **don't wrap it in a Polly retry for 429s** — that multiplies attempts (Module 25) |
| **`EnableContentResponseOnWrite = false`** | Writes otherwise return the full item: wasted bandwidth and CPU for most services |
| **System.Text.Json** | Use STJ (with source-generated contexts where possible — Module 17) instead of the legacy Newtonsoft default |
| **Hedging** | Tail-latency protection across regions at the cost of extra RUs (Concept 42) |
| **Distributed tracing and thresholds** | OpenTelemetry spans from the `Azure.Cosmos.Operation` source, plus automatic diagnostics logging for slow or failed operations |
| **`CreateAndInitializeAsync`** | Removes the first-request cold start (Module 26, Concept 49) |

**Bulk mode.** `AllowBulkExecution = true` makes the SDK group many concurrent point operations into per-partition batches — dramatically higher throughput for ingestion jobs and migrations, at the cost of per-operation latency. Use a **separate client** for bulk work (so the API's latency isn't affected), launch operations concurrently (`Task.WhenAll` over many `CreateItemAsync` calls), and mark them **low priority** (Concept 35).

**Diagnostics are the debugging tool.** For a slow or failed request, `response.Diagnostics` (or `CosmosException.Diagnostics`) shows retries, regions contacted, backend latency, connection state and 429 handling. Log it **only** for slow or failed operations (thresholds) — it's verbose.

**EF Core's Cosmos provider** (Module 19) is fine for simple aggregates with straightforward queries; it doesn't expose the change feed, bulk, patch-with-predicate, distributed transactions or many SDK knobs. For platform-heavy services, use the SDK directly behind a repository.

**The interview-grade sentence:** *"In .NET I register one CosmosClient per process, created with CreateAndInitializeAsync to remove the first-request cold start, in direct mode unless the network forces gateway, with preferred regions, cross-region hedging, content response on writes turned off, System.Text.Json, distributed tracing, and 429 retries left to the SDK with a capped wait — never doubled by Polly. Bulk ingestion gets its own client with AllowBulkExecution and low priority, and diagnostics are logged only above a latency threshold."*

---

## Concept 45 — Cosmos DB security

Cosmos has **two** separate authorization planes, and confusing them is a common audit finding.

| Plane | What it controls | How |
|---|---|---|
| **Control plane** (Azure Resource Manager) | Accounts, databases, containers, throughput, regions, keys, networking | **Azure RBAC** (for example *DocumentDB Account Contributor*, *Cosmos DB Operator*) |
| **Data plane** | Reading and writing items, queries, change feed | **Cosmos DB data-plane RBAC** with built-in roles **Cosmos DB Built-in Data Reader** and **Cosmos DB Built-in Data Contributor** (or custom role definitions), assigned to Entra identities at account, database or container scope — managed with Cosmos-specific commands (`az cosmosdb sql role assignment create`), not regular Azure role assignments |

The production baseline:

1. **Entra identities with data-plane RBAC** (managed or workload identity, Module 26 Concept 52), scoped to the containers each service uses.
2. **Disable key-based authentication** (`disableLocalAuth = true`) so primary/secondary keys stop working — keys grant full data access to the whole account and are the most common credential leak. Also set **`disableKeyBasedMetadataWriteAccess`** so even key holders can't change databases and containers through the data plane.
3. **No control-plane rights in applications.** Containers, indexing policies and throughput are managed by infrastructure as code, not created at application startup with `CreateContainerIfNotExistsAsync` in production (fine in development and tests).
4. **Private endpoints** per region (`privatelink.documents.azure.com`), public network access disabled; mind private DNS after failovers (Microsoft has a specific note on private endpoints and failover).
5. **Customer-managed keys** where policy requires; **TLS 1.2+**; Azure Policy to enforce local-auth-off and private-link-only across subscriptions.
6. **Network Security Perimeter**, where supported, as an additional boundary for PaaS-to-PaaS traffic.

**The interview-grade sentence:** *"Cosmos has two planes: Azure RBAC for the control plane and Cosmos's own data-plane RBAC — built-in Data Reader and Data Contributor roles assigned to Entra identities at account, database or container scope. My baseline is data-plane RBAC with managed identities, key authentication disabled along with key-based metadata writes, no container creation from production applications, private endpoints per region with DNS that survives failover, and CMK and policy enforcement where required."*

---

## Concept 46 — Analytics, search and AI on Cosmos data

The operational container should serve the operational workload. Analytics, search and AI retrieval each have a better home or a better pattern.

**Analytics: Fabric mirroring, not the OLTP container.**

- **Azure Synapse Link for Cosmos DB is no longer supported for new projects** (existing enablement keeps working). The replacement is **Cosmos DB Mirroring for Microsoft Fabric** (GA): a near-real-time, no-ETL replica of containers into **OneLake** in Delta Parquet, queryable with T-SQL, Spark and Power BI — without consuming the container's RU for analytical queries. It requires **continuous backup**.
- **Cosmos DB in Fabric** (a native Fabric database) exists for Fabric-first estates.
- Running large analytical queries (aggregations over the whole container) against the transactional container is the classic way to throttle your own users.

**Search: vector, full-text and hybrid inside Cosmos (NoSQL API).**

- **Vector search** with vector indexes (flat, quantized flat, **DiskANN**) stores embeddings next to the operational data under the same partition key — retrieval for RAG and agent memory without a separate vector database. Partition-aware filtering (for example per tenant) is where it shines.
- **Full-text search** and **hybrid search** (vector + BM25 ranked with reciprocal rank fusion) are available in the query language; **semantic reranking** is in preview (Build 2026).
- For heavy search or AI workloads, isolate them: a **GSI** with its own throughput and vector/full-text indexes (Concept 38), or **Azure AI Search** with an indexer over Cosmos when you need its richer search features.

**The interview-grade sentence:** *"I keep analytics off the transactional container: Synapse Link is closed to new projects, so new designs use Cosmos DB Mirroring for Fabric, which needs continuous backup and doesn't consume container RU. For AI retrieval, vector search with DiskANN plus full-text and hybrid search lets embeddings live next to operational data under the same partition key, and I isolate heavy search load with a GSI or Azure AI Search rather than letting it compete with user traffic."*

---

## Concept 47 — Cosmos traps

**1. A `CosmosClient` per request.** Connection storms, lost routing caches, cold starts on every call (Concept 44).

**2. Cross-partition everything.** A partition key that's absent from the dominant queries turns every read into a fan-out — high RU, high latency, and RU that grows with partition count (Concepts 3, 34).

**3. Default indexing on write-heavy containers.** Indexing every path of large documents makes every write expensive (Concept 34).

**4. Storage-driven throughput floors and thin partitions.** Large containers have many physical partitions, so per-partition RU is small and the minimum throughput rises with storage (Module 5). Archival data in Cosmos is expensive; move cold data out (TTL + archive to Blob via the change feed) or into a separate low-throughput container.

**5. Polly around 429s.** Multiplied retries on top of the SDK's own (Concept 44; Module 25).

**6. Unbounded logical partitions.** A key whose data grows forever (a per-tenant log, a device's lifetime telemetry) eventually hits 20 GB — a design bug, fixed with HPK or time bucketing (Concept 37).

**7. Ignoring the RU charge.** Teams that never look at `RequestCharge` discover cost in the invoice instead of in code review.

**8. Hot keys hidden by averages.** "We're at 30% utilization" while one partition throttles (Concept 36).

**9. Multi-region in the portal, single-region in the code.** No preferred regions configured, so reads cross an ocean and failover behavior is accidental (Concept 42).

**10. Analytics and backfills on the OLTP container at normal priority.** Use Fabric mirroring, GSIs, and low priority (Concepts 35, 46).

**11. Counting on preview features in production designs.** Distributed transactions and Azure Backup are previews with compatibility limits (Concepts 40, 43). Design so you can adopt them at GA, not depend on them now.

**12. Cosmos as a queue.** Polling a container for "pending" items with frequent queries; use the change feed, or a real broker.

**The interview-grade sentence:** *"The Cosmos traps are a client per request, cross-partition queries on the hot path, default indexing on write-heavy containers, storage-driven throughput floors from keeping cold data, Polly multiplying 429 retries, unbounded logical partitions, never reading RequestCharge, averages hiding a hot partition, multi-region accounts with no preferred regions in code, analytics and backfills at normal priority on the OLTP container, production designs that depend on previews, and polling a container as if it were a queue."*

---

## Concept 48 — The rest of the data menu: when Cosmos isn't the answer

Cosmos is excellent at **partitioned, low-latency, high-scale key/document access, global distribution and change streams**. It's not the right default for everything. A senior answer knows the alternatives and the triggers:

| Store | Choose it when | Avoid it when |
|---|---|---|
| **Azure SQL Database** (incl. **Hyperscale**, up to ~128 TB) / **SQL Managed Instance** | Relational data with rich ad-hoc queries, joins, reporting, strong multi-row transactions and constraints; teams with SQL skills; EF Core-heavy domains | You need multi-region active-active writes, or partitioned scale beyond a single primary's write rate |
| **Azure Database for PostgreSQL – Flexible Server** (and **elastic clusters** for distributed Postgres) | Relational + JSONB + extensions (PostGIS, pgvector), open-source portability; distributed Postgres for multitenant scale-out | Same as SQL for global multi-write |
| **Azure DocumentDB** (former vCore-based Cosmos DB for MongoDB) | MongoDB-compatible workloads and drivers, vCore pricing, MongoDB tooling, hybrid/multicloud via the open-source DocumentDB engine | You need Cosmos-specific features (RU-based autoscale, PPAF, change feed semantics, GSIs) |
| **Cosmos DB for MongoDB (RU)** | An existing Mongo application that wants Cosmos's RU model and global distribution | Greenfield (prefer NoSQL API or Azure DocumentDB) |
| **Azure Managed Redis** (Module 10) | Caching, sessions, leaderboards, rate limiting, ephemeral state with sub-millisecond latency | As a system of record |
| **Azure Table Storage** | Very cheap, simple key/value with partition and row keys, no secondary indexes | Rich queries, global distribution, predictable low latency at scale |
| **Blob Storage / ADLS** | Large objects, files, archives, data-lake storage | Item-level transactional access |
| **Fabric Eventhouse / Azure Data Explorer** | Time-series, logs and telemetry analytics with KQL | Transactional workloads |

The decision questions (Module 12's axes, applied to Azure): Do the queries need joins and ad-hoc filtering? → relational. Is the access pattern known, key-based, high-scale or globally distributed? → Cosmos. Is it MongoDB-compatibility you need? → Azure DocumentDB. Is it ephemeral? → Redis. Is it analytics? → Fabric/ADX. And polyglot persistence has an operating cost per store — cap the number of stores like you cap compute platforms (Module 26, Concept 60).

**The interview-grade sentence:** *"Cosmos is my answer for partitioned, low-latency, key-based access at scale, global distribution and change streams; for relational queries, joins and constraints I'd pick Azure SQL or PostgreSQL Flexible Server, for MongoDB compatibility Azure DocumentDB — the renamed vCore Mongo service — for ephemeral state Managed Redis, for cheap simple key/value Table Storage, and for telemetry analytics Fabric Eventhouse or Data Explorer, while keeping the number of stores small because each one has an operating cost."*

---
# Part F — Composing the platform

## Concept 49 — Choosing per flow

Systems don't choose "a messaging service"; each **flow** chooses. Module 11 (Concept 38) gave three questions that settle broker choice in 30 seconds; here they are extended to include the data platform:

**Question 1 — What is the thing being moved?**

- A **fact** that has happened, which any number of parties may care about → **Event Grid** (notification), or a **Service Bus topic** (if consumers need durable, per-consumer work queues with dead-lettering).
- A **command** or **work item** with one owner and a workflow (retries, ordering per key, scheduling, dead-letter) → **Service Bus queue**.
- A **stream** of records valuable in aggregate, consumed by several independent readers, possibly replayed → **Event Hubs**.
- **State** that must be queried, and whose changes others need → **Cosmos DB**, with the **change feed** as the outbound stream.

**Question 2 — Who controls the pace, and what must survive?**

- Consumer must pull at its own pace, or stay private → anything but Event Grid Basic push (or route push into a pull service).
- The message must survive the consumer being down for hours → Service Bus or Event Hubs (or namespace pull, up to 7 days).
- Data must be replayable by a consumer that doesn't exist yet → Event Hubs (with retention or Capture) or the change feed.

**Question 3 — What scope of ordering, and at what volume?**

- Order per entity at modest volume with per-message handling → Service Bus **sessions**.
- Order per entity at high volume → Event Hubs **partition keys**, or the change feed's per-partition order.
- No ordering needed → whatever the first two questions chose.

A compact decision table to narrate:

| Requirement | Default choice |
|---|---|
| React to Azure resource events | Event Grid system topics |
| Notify many independent services of a domain event | Service Bus topic (durable per-consumer) or Event Grid (lightweight fan-out, webhooks, partners) |
| Commands with retries, scheduling, DLQ | Service Bus queue |
| Ordered per-key workflow processing | Service Bus sessions |
| Telemetry, clickstream, logs, CDC at volume | Event Hubs |
| Kafka clients without running Kafka | Event Hubs Kafka endpoint |
| Device messaging over MQTT | Event Grid namespaces MQTT broker |
| Publish changes from your operational store | Cosmos change feed (+ outbox items) |
| Cheap, simple background work queue | Storage queues |

**The interview-grade sentence:** *"I choose per flow with three questions: is it a fact, a command, a stream or state; who controls the pace and what must survive or be replayed; and what ordering scope at what volume. Facts go to Event Grid or a Service Bus topic, commands to Service Bus queues, ordered per-key work to sessions, high-volume streams to Event Hubs, MQTT to Event Grid namespaces, and state to Cosmos with the change feed as its outbound stream."*

---

## Concept 50 — Canonical topologies

Most well-designed Azure systems are compositions of a few recurring shapes. Being able to draw these quickly is a strong interview move.

**Topology 1 — Durable reaction to resource events (Event Grid → Service Bus).**

```
Blob Storage ──(BlobCreated)──► Event Grid system topic ──filter: /uploads/*.pdf──► Service Bus queue "pdf-ingest"
                                                                                          │ (DLQ, retries, sessions if needed)
                                                                                          ▼
                                                                           Container Apps worker (KEDA on queue length)
```
Event Grid filters and routes; Service Bus buffers, retries and dead-letters; the worker scales on backlog. This removes push-handler fragility and lets the worker scale to zero.

**Topology 2 — Cosmos-native outbox (Cosmos → change feed → Service Bus).**

```
API ──TransactionalBatch(aggregate + outbox item, same PK)──► Cosmos container
                                                                    │ change feed (at-least-once, per-PK order)
                                                                    ▼
                                               Relay (Change Feed Processor / Functions Cosmos trigger)
                                                                    │ MessageId = outbox id (duplicate detection)
                                                                    ▼
                                         Service Bus topic "orders-events" ──► subscriptions per consumer
```
No dual write, no polling, per-aggregate order preserved into a **session-enabled** subscription if you set `SessionId = aggregateId`.

**Topology 3 — Hot and cold paths for streams (Event Hubs).**

```
Producers ──► Event Hubs (partition key = deviceId)
                 ├─ consumer group "realtime"  ──► processors ──► Cosmos (latest state per device) / alerts
                 ├─ consumer group "analytics" ──► Fabric Eventstream / Stream Analytics ──► Eventhouse / dashboards
                 └─ Capture ──────────────────────► ADLS (Avro) ──► batch, replay, audit
```
One stream, independent consumers each with their own group and pace, and a cold path that needs no consumer code.

**Topology 4 — Devices over MQTT (Event Grid namespaces → Event Hubs).**

```
Devices ──MQTT──► Event Grid namespace (topic spaces, X.509) ──routing──► namespace topic ──push──► Event Hubs ──► Topology 3
          ◄──commands (MQTT topics per device)──
```
Event Grid handles device connectivity, authentication and per-device topics; Event Hubs handles the volume and replay.

**Topology 5 — Command processing with ordered updates (Service Bus sessions → Cosmos).**

```
Producers ──SessionId = accountId──► Service Bus queue (sessions) ──► session processor ──► Cosmos (ETag-guarded writes)
```
The session gives per-account order and exclusivity; ETags guard the store anyway, so a redelivered or reordered message can't corrupt state.

**Topology 6 — Fan-out to external parties (Event Grid domains).**

```
Your services ──► Event Grid domain (topic per customer) ──► customers' webhooks (Entra-authenticated, retried, dead-lettered)
```

**The interview-grade sentence:** *"I build from a few shapes: Event Grid system events routed into a Service Bus queue so a KEDA-scaled worker can buffer, retry and dead-letter; a Cosmos-native outbox relayed from the change feed into a Service Bus topic with the outbox id as MessageId; Event Hubs with one consumer group per independent reader plus Capture for the cold path; MQTT devices into Event Grid namespaces and on to Event Hubs; sessions into ETag-guarded Cosmos writes; and Event Grid domains for customer webhooks."*

---

## Concept 51 — Claim check: large payloads

Every service in this module has a size limit (Concept 2), and large messages are slow and expensive even where they're allowed. The **claim check** pattern stores the payload in **Blob Storage** and sends a **reference** in the message:

1. The producer uploads the payload to Blob (`claims/{messageId}.json`), optionally compressed and encrypted.
2. It sends a small message containing the blob URI, a content hash, size, and metadata in properties.
3. The consumer downloads the blob (with its managed identity — never a long-lived SAS in the message), verifies the hash, processes, and the blob is deleted by lifecycle policy (or kept for audit).

Practical details:

- **Event Grid does part of this for you**: a `BlobCreated` event *is* a claim check. Uploading the payload and letting Event Grid notify is often simpler than sending your own message.
- **Ordering between upload and send** matters: upload first, then send; if the send fails, the orphan blob is cleaned up by lifecycle policy.
- **Idempotency**: name the blob by message id so a retried producer overwrites the same blob rather than creating duplicates.
- **Security**: the blob container's access is now part of the message's security boundary — consumers need read on it, producers write.

**The interview-grade sentence:** *"For payloads near or above the limits — 256 KB on Service Bus Standard, 1 MB on Event Hubs and Event Grid — I use a claim check: upload to Blob named by message id, send a reference with a hash in a small message, let consumers read with their managed identity, and expire blobs with lifecycle rules. When the payload starts life as a blob, Event Grid's BlobCreated event already is the claim check."*

---

## Concept 52 — Ordering end to end

Ordering guarantees exist **per hop**, and a pipeline only preserves order if **every hop** preserves it for the same key. The places it silently dies:

| Hop | Order preserved? | How it breaks |
|---|---|---|
| Producer → Event Hubs | Per partition key | Sending without a key; retries reordering in-flight batches |
| Event Hubs → processor | Per partition | Processing events of one partition **concurrently** inside the handler |
| Producer → Service Bus | Per session | Non-session entity (best effort only); multiple senders racing for one key |
| Service Bus → handler | Per session (with `MaxConcurrentCallsPerSession = 1`) | Concurrent calls per session; abandons and redelivery changing order on non-session entities |
| Cosmos write → change feed | Per logical partition key | Writes for one entity spread over several keys |
| Change feed → relay → broker | Per lease → only if the relay preserves it and the broker hop does | Relay processing items in parallel; sending to a non-session entity |
| Event Grid | **Never** | — |
| Any retry with backoff | Can reorder | A failed message retried later lands after newer ones |

Three design responses:

1. **Pick one ordering key and carry it through every hop** — partition key in Event Hubs, `SessionId` in Service Bus, partition key in Cosmos — with the same value (aggregate id).
2. **Make order unnecessary where you can**: carry a **version** or sequence number in each event and have consumers apply only newer versions (ETag- or version-guarded writes). Then reordering and duplicates become harmless — the most robust option, and the one Module 11 recommends.
3. **Know where you deliberately give it up** (Event Grid, fan-out across topics, retries with delay) and say so in the design.

**The interview-grade sentence:** *"Ordering only survives a pipeline if every hop preserves it for the same key — Event Hubs partition key, Service Bus SessionId with one call per session, Cosmos partition key, and a relay that doesn't parallelize within a lease — and it dies at Event Grid, at retries with delay and at any unkeyed hop. So I carry one key end to end where I truly need order, and otherwise put versions in events so consumers apply only newer state and order stops mattering."*

---

## Concept 53 — Idempotency end to end

Every service here is at-least-once, so every consumer must be idempotent. What the platform gives you to make that cheap:

| Layer | Mechanism | Scope and limits |
|---|---|---|
| **Producer → Service Bus** | Duplicate detection on `MessageId` | Within the configured window (up to 7 days); per entity |
| **Producer → Event Hubs** | None at the broker for AMQP | Consumer-side dedup required |
| **Event Grid** | Event `id` (+ `source`) is stable across retries | Consumer-side dedup required |
| **Cosmos as the target** | Deterministic item `id` + `CreateItemAsync` → **409 Conflict** on replay; or `UpsertItemAsync` for idempotent overwrite; ETag/version checks | Exactly-once *effect* on the item |
| **Cosmos as the dedup store** | An "inbox" container: `id = messageId`, partition key = consumer + message id, **TTL** = dedup horizon | Cheap point write per message; TTL cleans up |

**The inbox pattern on Cosmos** — the consumer-side mirror of the outbox:

```csharp
// Record the message id and apply the effect atomically in one partition (consumer state keyed the same way),
// or — when the effect lives elsewhere — record first and treat 409 as "already processed".
try
{
    await inbox.CreateItemAsync(new InboxEntry(id: message.MessageId, consumer: "billing", ttl: 7 * 24 * 3600),
                                new PartitionKey(message.MessageId), cancellationToken: ct);
}
catch (CosmosException e) when (e.StatusCode == HttpStatusCode.Conflict)
{
    await args.CompleteMessageAsync(message, ct);   // duplicate: already handled
    return;
}
await billing.ApplyAsync(message, ct);              // if this fails, delete the inbox entry or rely on effect idempotency
await args.CompleteMessageAsync(message, ct);
```

The honest caveat: "record, then apply" isn't atomic unless both writes are in the same transaction (same store, same partition). The strongest forms are (a) the effect itself is idempotent (deterministic ids, upserts, version checks), or (b) inbox entry and effect are in one `TransactionalBatch`. Frameworks with an inbox (NServiceBus, MassTransit, Wolverine) implement exactly these trade-offs.

**The interview-grade sentence:** *"Service Bus duplicate detection guards producers within its window, but Event Hubs and Event Grid leave dedup to consumers, so I make effects idempotent with deterministic Cosmos ids — a 409 on replay — upserts and version checks, and where the effect isn't naturally idempotent I use a Cosmos inbox container keyed by message id with a TTL, ideally written in the same transactional batch as the effect."*

---

## Concept 54 — Backpressure across a pipeline

A pipeline runs at the speed of its slowest stage; the question is **where the queue builds up** when one stage slows down, and whether that's where you want it.

**Who throttles whom:**

| Stage overloaded | Symptom | Where pressure goes | What should happen |
|---|---|---|---|
| Cosmos container (RU) | 429s; SDK retries; rising latency | Into the consumer's handler time → lock durations → broker backlog | Consumers slow down (bounded concurrency), backlog grows in the broker, autoscale **doesn't** add more consumers blindly |
| Service Bus Premium (MU CPU) | Server busy; send latency | Producers' send calls | Producers back off; outbox accumulates; scale MUs |
| Event Hubs (TU) | Quota exceeded on send | Producers | Auto-inflate or more PUs; buffered producers back off |
| Downstream API | Timeouts, 5xx | Handler time, retries | Circuit breaker (Module 25), release/abandon with delay, backlog in broker |

**The anti-pattern: autoscaling consumers into a saturated dependency.** KEDA sees a growing queue and adds replicas; each replica adds concurrency against a Cosmos container that's already throttling; 429s rise; handlers slow; the queue grows faster; KEDA adds more. The **maximum replica count × per-replica concurrency** must be sized against the dependency's capacity (Module 26, Concept 26), and the scaling signal should be paired with a dependency-health guard (cap or pause scaling when the dependency throttles).

**Design rules:**

1. **Let the broker hold the backlog.** It's durable, observable and cheap; your process memory isn't.
2. **Bound concurrency at every consumer** against its downstream budget (RU/s, connections, rate limits).
3. **Prefer adaptive back-off** — when the dependency returns 429/503, slow the consumer (reduce concurrency, pause the processor briefly) instead of retrying harder.
4. **Separate priorities physically or logically**: user-facing writes on high priority (Concept 35), backfills low; separate queues for interactive vs batch work.
5. **Alert on age, not just depth** — the age of the oldest message (or consumer lag in time) is what users feel.

**The interview-grade sentence:** *"In a pipeline the backlog should build in the broker, where it's durable and visible, so I bound every consumer's concurrency against its dependency — RU/s, connections, rate limits — slow consumers down adaptively on 429s instead of retrying harder, cap autoscaling at max replicas times concurrency the dependency can absorb so KEDA doesn't scale into a throttling database, keep batch work at low priority, and alert on message age and lag in time."*

---

## Concept 55 — Multi-region across the platform

Each service has a different multi-region feature; a system's DR story is their **composition**. Write it as a table and a runbook, not as "we use geo-replication."

| Service | Feature | RPO | RTO driver | Who triggers |
|---|---|---|---|---|
| **Cosmos DB** | Read regions + PPAF (Business Critical) / forced failover / multi-region writes | 0 (strong) to < ~15 min (session); conflicts with multi-write | PPAF ~3 min; forced: your detection time; service-managed: ≥ 1 h | Service (PPAF), you (forced) |
| **Service Bus Premium** | Geo-Replication (sync/async) or Geo-DR (metadata) | 0 (sync) or ≤ max lag (async); Geo-DR: in-flight messages stranded | Promotion decision + DNS/client reconnection | You (or your automation) |
| **Event Hubs Premium/Dedicated** | Geo-Replication with offsets, or Geo-DR | 0 (sync) or ≤ max lag; checkpoint store separate for AMQP | Promotion + consumer restart | You |
| **Event Grid** | Regional service; Microsoft-managed failover for some resources; you design dual-region topics or publish to both | Events in flight during an outage may be delayed or lost | Your design | You |
| **Blob (claim checks, checkpoints, dead-letters)** | ZRS / GZRS, RA-GZRS | Async geo-replication lag | Account failover | You (or Microsoft) |

**Composing them — the questions to answer explicitly:**

1. **What is the system's RPO?** At best the *weakest* component's for the data that matters. An outbox in Cosmos (RPO under 15 minutes on session) relayed to an asynchronously replicated Service Bus namespace (RPO ≤ max lag) means a forced failover can lose both unrelayed outbox items *and* unreplicated messages — unless the outbox and relay are designed to **re-publish** after failover (which idempotent consumers make safe).
2. **Do the components fail over together?** Promote Service Bus but not Cosmos and your consumers in region B write across regions to region A's Cosmos write region. Decide whether compute, messaging and data move as a **stamp** (Module 26, Concept 61) or independently.
3. **Who decides, and how fast?** PPAF is automatic; Service Bus and Event Hubs promotion isn't. A runbook with a named decision-maker, health signals and automation beats an on-call engineer reading docs at 3 a.m.
4. **What about in-flight work?** Messages locked in region A, partitions half-processed, change-feed leases mid-batch: all replay in region B. Idempotency (Concept 53) is what makes failover safe.
5. **How do you fail back?** No service fails back automatically. Plan the reverse promotion and the data reconciliation (Cosmos conflict feed, re-publishing outbox items).
6. **Have you rehearsed it?** Cosmos supports forced-failover drills and PPAF fault simulation; Service Bus and Event Hubs support planned promotion. A DR design that has never been exercised is a hypothesis (Module 13).

**The interview-grade sentence:** *"Multi-region is a composition of different features — Cosmos PPAF or forced failover, Service Bus and Event Hubs Geo-Replication with planned or forced promotion, Event Grid as a regional service I design around, and geo-redundant storage for claim checks and checkpoints — so the system's RPO is the weakest link for the data that matters. I decide whether components fail over together as a stamp, who triggers promotion, how in-flight work replays safely through idempotent consumers and re-publishable outboxes, how we fail back, and I rehearse it."*

---

## Concept 56 — The security baseline across all four services

One table to carry into any design review:

| Control | Service Bus | Event Hubs | Event Grid | Cosmos DB |
|---|---|---|---|---|
| **Identity for apps** | Entra: *Azure Service Bus Data Sender / Receiver* (entity scope) | Entra: *Azure Event Hubs Data Sender / Receiver* (event hub scope) | Entra: *EventGrid Data Sender* (publish); namespace *Data Receiver* for pull; managed identity for delivery | Entra: *Cosmos DB Built-in Data Reader / Contributor* (data-plane RBAC, container scope) |
| **Disable shared keys** | `disableLocalAuth` (SAS off) | `disableLocalAuth` | Disable access-key auth on topics/namespaces | `disableLocalAuth` + `disableKeyBasedMetadataWriteAccess` |
| **Private networking** | Private endpoints (Premium) | Private endpoints (Standard+) | Private endpoints for publish (Basic), publish **and** pull (namespaces) | Private endpoints per region |
| **Encryption** | Microsoft-managed; CMK on Premium | CMK on Premium/Dedicated | Microsoft-managed | Microsoft-managed; CMK |
| **Delivery/consumer auth** | — | — | Entra-authenticated webhook delivery; managed identity to Azure destinations | — |
| **Management rights** | Pipelines only (no `Manage` in apps) | Pipelines only | Pipelines only | Pipelines only (no container creation in prod apps) |
| **Policy** | Azure Policy: local auth off, private link, CMK | Same | Same | Same |
| **Secrets left over** | None — if Functions bindings use identity-based connections | Checkpoint store via identity | Webhook secrets only if you can't use Entra | None |

Two cross-cutting rules: **least privilege per entity** (a consumer reads its own queue, not the namespace) and **no shared keys anywhere** (if a connection string exists, someone will paste it into a ticket). The Functions-specific move is **identity-based connections** (`<Connection>__fullyQualifiedNamespace`, `<Connection>__accountEndpoint`) instead of connection strings (Module 26, Concept 52).

**The interview-grade sentence:** *"Across all four services my baseline is the same: Entra identities with data-plane roles at the narrowest scope — sender and receiver per entity, Cosmos data-plane RBAC per container — shared keys and SAS disabled, private endpoints, CMK where policy requires, management rights only in pipelines, identity-based connections in Functions, Entra-authenticated Event Grid deliveries, and Azure Policy enforcing it all."*

---

# Part G — .NET implementation

## Concept 57 — Clients, DI and Aspire

**Client lifetime rules** — the same for every Azure SDK client in this module: **one client per process per resource, registered as a singleton, disposed on shutdown**. `ServiceBusClient`, `EventHubProducerClient`, `EventProcessorClient`, `EventGridPublisherClient`/`EventGridSenderClient` and `CosmosClient` are all thread-safe and designed for reuse.

**`Microsoft.Extensions.Azure`** registers Azure SDK clients in DI with shared credentials and options:

```csharp
builder.Services.AddAzureClients(clients =>
{
    clients.AddServiceBusClientWithNamespace(builder.Configuration["ServiceBus:Namespace"]);
    clients.AddEventHubProducerClientWithNamespace(builder.Configuration["EventHubs:Namespace"], "clicks");
    clients.AddEventGridPublisherClient(new Uri(builder.Configuration["EventGrid:TopicEndpoint"]!));

    // One explicit production credential (Module 26, Concept 52), not the full DefaultAzureCredential chain.
    clients.UseCredential(builder.Environment.IsDevelopment()
        ? new DefaultAzureCredential()
        : new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(builder.Configuration["AZURE_CLIENT_ID"]!)));

    clients.ConfigureDefaults(o => o.Retry.MaxRetries = 3);     // transport retries; business retries live elsewhere
});

// Named, cached senders per entity.
builder.Services.AddSingleton(sp => sp.GetRequiredService<ServiceBusClient>().CreateSender("charge-card"));
```

**Aspire** (Module 26, Concept 53) models these resources in the AppHost and wires clients in services — including **local emulators**:

```csharp
// AppHost
var bus = builder.AddAzureServiceBus("messaging").RunAsEmulator();       // Service Bus emulator container locally
bus.AddServiceBusQueue("charge-card");
bus.AddServiceBusTopic("orders-events").AddServiceBusSubscription("billing");

var hubs = builder.AddAzureEventHubs("eventhubs").RunAsEmulator();
hubs.AddHub("clicks");

var cosmos = builder.AddAzureCosmosDB("cosmos").RunAsEmulator();        // or the Linux (vNext) emulator option
cosmos.AddCosmosDatabase("shop").AddContainer("orders", "/customerId");

builder.AddProject<Projects.Orders_Api>("orders-api")
       .WithReference(bus).WithReference(cosmos).WithReference(hubs);

// Service
builder.AddAzureServiceBusClient("messaging");
builder.AddAzureCosmosClient("cosmos");
builder.AddAzureEventHubProducerClient("eventhubs", s => s.EventHubName = "clicks");
```

Method names vary across Aspire versions — the shape is what matters: **declare resources and topology in the AppHost, run emulators locally, get Bicep for Azure from `aspire publish`, and get health checks, telemetry and configuration in services from the client integrations**. For production topology, keep in mind Module 26's caveat: generated infrastructure is a starting point that enterprise IaC often refines.

**The interview-grade sentence:** *"Every Azure SDK client here is a thread-safe singleton per process and resource, registered through Microsoft.Extensions.Azure with one explicit production credential and conservative transport retries, with senders cached per entity. Aspire declares Service Bus, Event Hubs and Cosmos resources and their topology in the AppHost, runs the emulators locally, and gives services wired clients with health checks and telemetry."*

---

## Concept 58 — Hosting consumers

All three long-running consumers — `ServiceBusProcessor`, `EventProcessorClient` and the Cosmos `ChangeFeedProcessor` — have the same shape in .NET: a **`BackgroundService`** that starts the processor, waits for shutdown, and stops it gracefully within the platform's grace period (Module 26, Concept 51).

```csharp
public sealed class ChangeFeedRelayService(ChangeFeedProcessor processor, ILogger<ChangeFeedRelayService> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await processor.StartAsync();
        log.LogInformation("Change feed relay started");
        try { await Task.Delay(Timeout.Infinite, stoppingToken); }
        catch (OperationCanceledException) { }
        await processor.StopAsync();          // releases leases so another instance picks them up quickly
    }
}
```

Hosting choices (Module 26's ladder, applied):

| Consumer | Container Apps / AKS (`BackgroundService`) | Azure Functions trigger |
|---|---|---|
| Service Bus | Processor + KEDA `azure-servicebus` scaler | Service Bus trigger (extension 5.x), target-based scaling |
| Event Hubs | `EventProcessorClient` + KEDA `azure-eventhub` scaler (lag from checkpoint store) | Event Hubs trigger — batch checkpointing, catch per event |
| Cosmos change feed | CFP + scale on estimator lag (custom metric) | Cosmos DB trigger with leases |
| Event Grid | Pull receiver loop (namespaces) or HTTP handler | Event Grid trigger |

Three hosting rules that recur:

1. **Stop gracefully**: stop pulling, finish or abandon in-flight work, release leases/ownership — so the next instance resumes quickly.
2. **Scale on backlog and cap on dependency** (Concept 54).
3. **Separate consumers from APIs** where their scaling differs — the same image can run as two apps (Module 26, Concept 4).

**The interview-grade sentence:** *"Each consumer — the Service Bus processor, the EventProcessorClient and the change feed processor — runs as a BackgroundService that starts it, waits for the stopping token and stops it gracefully to release locks, ownership or leases, hosted on Container Apps with a KEDA scaler on backlog or as a Functions trigger, with replicas capped against the dependency and scaled separately from the API even when they share an image."*

---

## Concept 59 — Retries without multiplication

Module 25 established the rule: **each failure kind has exactly one retry owner**. On this platform, there are four candidate owners, and the defaults overlap:

| Layer | What it retries | Default | Recommendation |
|---|---|---|---|
| **Azure SDK transport** (`ServiceBusRetryOptions`, `EventHubsRetryOptions`, `ClientOptions.Retry`) | Connection drops, timeouts, server busy | Exponential, a few attempts | Keep modest; make sure the **try timeout** fits inside the operation's budget |
| **Cosmos SDK** | 429s (honoring retry-after), transient network errors, region failover | 9 attempts for 429s, 30 s max wait | Cap the wait to your latency budget; **never add Polly for 429s** |
| **Broker redelivery** (Service Bus `MaxDeliveryCount`, Event Grid retry schedule, namespace max delivery count) | Handler failures across time | 10 (Service Bus), 30 attempts/24 h (Event Grid) | Owns **retry across time** for message handlers |
| **Your code / Polly** | Business-level or dependency calls inside a handler | — | A **small** in-process retry inside the lock budget, a circuit breaker, a timeout — not a long retry loop |

The multiplication to compute before production: `transport attempts × in-process attempts × broker deliveries`. Three SDK attempts × three Polly attempts × ten deliveries = 90 attempts against a failing dependency per message — enough to keep it down. Pick the owner per failure: transient network → SDK; dependency outage → circuit breaker + broker redelivery with delay; poison message → dead-letter quickly.

**The interview-grade sentence:** *"On this platform the SDK transport, the Cosmos SDK's 429 handling, broker redelivery and Polly can all retry the same failure, and they multiply — three times three times ten is ninety attempts per message — so I give each failure one owner: the SDKs for transient transport errors and 429s with capped waits, the broker for retries across time, and Polly only for a small in-lock retry, a timeout and a circuit breaker."*

---

## Concept 60 — Contracts and serialization across services

A message or event outlives the code that produced it — in a DLQ, in 90 days of Event Hubs retention, in Capture files, in a Cosmos change feed replay. Contracts need the same discipline as public APIs (Module 11, Concept 43):

- **Envelope**: CloudEvents attributes (`id`, `source`, `type`, `subject`, `time`) — natively on Event Grid; as **application properties** on Service Bus and Event Hubs (the CloudEvents AMQP binding maps them to properties). Routing and filtering use the envelope, never the body.
- **Body**: versioned schema (`type = "com.contoso.orders.placed.v2"` or a `schemaVersion` field), **tolerant readers** (ignore unknown fields, default missing ones), additive changes by default.
- **Serialization**: `System.Text.Json` with **source generation** (Module 17) for throughput; Avro with **Schema Registry** for multi-team streams on Event Hubs; avoid binary formats that bind consumers to .NET types.
- **Legacy bodies**: when migrating off the retired Service Bus libraries, old `BrokeredMessage` object bodies were DataContract-serialized — read them explicitly during the transition (Concept 7).
- **Cosmos documents are contracts too** — for the change feed's consumers. An explicit outbox item decouples the integration event from the storage document (Concept 41).

**The interview-grade sentence:** *"Messages outlive code in dead-letter queues, retention and replays, so I use a CloudEvents envelope — native on Event Grid, mapped to application properties on Service Bus and Event Hubs — route only on the envelope, version the body with tolerant readers and additive changes, serialize with source-generated System.Text.Json or Avro with Schema Registry for shared streams, and publish explicit outbox items rather than raw storage documents."*

---

## Concept 61 — Observability per service

Module 28 covers observability in depth; here are the **few metrics per service** that actually predict incidents, plus the tracing hook.

| Service | Metrics to alert on | What they tell you |
|---|---|---|
| **Service Bus** | **Active messages** and **age of oldest message** per entity; **dead-lettered messages**; **server errors / throttled requests**; **namespace CPU and memory** (Premium) | Backlog, poison, capacity |
| **Event Hubs** | **Incoming vs outgoing** bytes/messages; **throttled requests**; **consumer lag** per group (you compute, or KEDA); capture backlog | Capacity, consumer health |
| **Event Grid** | **Delivery failures**, **dead-lettered events**, **matched vs delivered**, **publish failures**, delivery latency | Handler health, lost events |
| **Cosmos DB** | **Normalized RU consumption** per partition; **429 rate** (substatus 3200); **server-side latency**; **replication latency**; **GSI propagation latency**; change feed estimator lag; availability | Hot partitions, capacity, staleness, projection lag |

**Tracing.** All four .NET SDKs emit `Activity`-based distributed tracing that OpenTelemetry collects — add the sources (for example `Azure.Messaging.ServiceBus.*`, `Azure.Messaging.EventHubs.*`, `Azure.Cosmos.Operation`; the Azure Monitor OpenTelemetry distro enables Azure SDK sources), and **trace context propagates through messages** in application properties (`Diagnostic-Id` / `traceparent`) so a request → message → consumer → Cosmos write appears as one trace (Module 11, Concept 44). Use OpenTelemetry **messaging semantic conventions** for any spans you add yourself.

**Logs worth turning on:** Cosmos diagnostic logs (partition key statistics and RU by key) into Log Analytics with a retention budget; Service Bus and Event Hubs **runtime audit/diagnostic logs** where you need to know who did what.

**The interview-grade sentence:** *"I alert on a few metrics per service: Service Bus active messages, oldest-message age, dead-letters and Premium CPU; Event Hubs incoming versus outgoing, throttling and consumer lag; Event Grid delivery failures and dead-letters; and Cosmos normalized RU per partition, 429 rate, replication and GSI propagation latency and change-feed lag. The SDKs' OpenTelemetry sources plus trace context in message properties turn a request, message, consumer and Cosmos write into one trace."*

---

## Concept 62 — Testing

**Emulators** make the platform testable on a laptop and in CI:

| Service | Local option | Notes |
|---|---|---|
| Service Bus | **Service Bus emulator** (container; 2.0.0 since January 2026 with admin-client support) | Queues, topics, subscriptions, filters; not all Premium features |
| Event Hubs | **Event Hubs emulator** (container; AMQP and Kafka) | Uses Azurite for checkpoints |
| Cosmos DB | **Linux (vNext) emulator — GA since June 2026**; the classic Windows emulator | Feature coverage differs from the service — check the emulator's supported-features list |
| Event Grid | No first-party push emulator | Test handlers with recorded CloudEvents; tunnel for end-to-end; namespaces against a real dev namespace |
| Storage | **Azurite** | Claim checks, checkpoints, dead-letter containers |

**Testcontainers for .NET** and **Aspire's testing support** start these emulators per test run, so integration tests exercise real protocol behavior: locks expiring, sessions, change-feed leases, 409s on duplicate ids.

**What to test beyond the happy path:**

1. **Redelivery and duplicates** — process the same message twice; assert one effect (Concept 53).
2. **Lock expiry** — a handler slower than the lock; assert no double effect.
3. **Poison handling** — a malformed message reaches the DLQ with a reason; a bad event is parked, not skipped.
4. **Ordering** — out-of-order versions don't corrupt state (Concept 52).
5. **Throttling** — inject 429s (or run against small RU/s) and confirm backpressure rather than retry storms.
6. **Contracts** — consumer contract tests against recorded events; schema compatibility checks in CI.
7. **Failover drills** — in a test environment: Cosmos forced failover and PPAF fault simulation; Service Bus and Event Hubs planned promotion; measure RTO and data loss against the design (Concept 55).

**The interview-grade sentence:** *"I test against the Service Bus emulator, the Event Hubs emulator with Azurite, and the Cosmos Linux emulator — GA since June 2026 — started by Testcontainers or Aspire, and I specifically test duplicates, lock expiry, poison handling, out-of-order versions, injected 429s and contract compatibility, then run real failover drills in a test environment: Cosmos forced failover and PPAF simulation, Service Bus and Event Hubs planned promotion."*

---
# Part H — Deciding

## Concept 63 — The decision axes

Across the four services, a handful of axes do most of the deciding:

| Axis | Event Grid | Service Bus | Event Hubs | Cosmos DB |
|---|---|---|---|---|
| **1. What moves** | Facts (notifications) | Commands, work items, workflow events | Streams of records | State (and its change stream) |
| **2. Consumption** | Push (Basic); pull or push (namespaces) | Pull with per-message locks | Pull by offset, per consumer group | Point reads/queries; change feed by lease |
| **3. Failure handling** | Retry schedule + Blob dead-letter | Abandon/defer/DLQ, max delivery count | Consumer parks failures | Consumer parks failures (change feed) |
| **4. Ordering** | None | Per session | Per partition | Per logical partition key |
| **5. Replay** | No | No | Yes (retention, Capture) | Yes (change feed) |
| **6. Volume sweet spot** | Thousands of events/s per topic, massive fan-out | Thousands to tens of thousands of messages/s per namespace | MB/s to GB/s | Up to millions of RU/s |
| **7. Max payload** | 1 MB | 256 KB / up to 100 MB (Premium) | 1 MB | 2 MB item |
| **8. Capacity model** | Per operation (Basic); TUs (namespaces) | Shared + per op (Standard); MUs (Premium) | TUs + events (Standard); PUs/CUs | RU/s (manual/autoscale) or per RU (serverless) |
| **9. Multi-region** | Regional; design it | Geo-DR / Geo-Replication (Premium) | Geo-DR / Geo-Replication (Premium/Dedicated) | Regions, PPAF, multi-region writes |
| **10. Protocols** | HTTP, MQTT | AMQP (JMS on Premium) | AMQP, Kafka, HTTPS | HTTPS/TCP (SDK), Mongo/Cassandra wire APIs |

**Which axes usually decide:**

- **Axis 1 (what moves)** decides the service in most cases on its own.
- **Axes 3 and 5 (failure handling and replay)** separate Service Bus from Event Hubs when the volume is ambiguous.
- **Axis 4 (ordering)** decides sessions vs partitions, and rules out Event Grid.
- **Axis 9 (multi-region)** pushes production designs toward Premium tiers and the Business Critical Cosmos tier — with a cost (Concept 64).

**The interview-grade sentence:** *"The axes that decide are what moves — facts, commands, streams or state — then failure handling and replay, which separate Service Bus from Event Hubs, then ordering scope, which picks sessions or partitions and rules out Event Grid, and finally multi-region requirements, which push designs toward Premium messaging tiers and Business Critical Cosmos with the cost that implies."*

---

## Concept 64 — Capacity and cost, worked

As in Module 26, interviewers want to see a **method**: shape → unit → model → break-even. Prices below are approximate US list prices for arithmetic only.

### 64a. Cosmos DB: manual vs autoscale vs serverless

**Workload:** an orders container, single region. Business hours (8 h/day) need **40,000 RU/s** at peak; the other 16 hours peak at **8,000 RU/s**. Actual average consumption is about 30,000 RU/s in business hours and 3,000 RU/s otherwise.

Unit prices: manual **$0.008 per 100 RU/s-hour** (≈ $5.84 per 100 RU/s-month); autoscale **1.5×** that on the hourly peak; serverless **$0.25 per million RU**.

```
Manual, provisioned for peak 24/7:      40,000 RU/s ÷ 100 × $5.84             ≈ $2,336 / month

Autoscale (Tmax = 40,000), billed on each hour's peak:
  time-weighted peak = (8 h × 40,000 + 16 h × 8,000) ÷ 24          = 18,667 RU/s
  cost               = 18,667 ÷ 100 × $5.84 × 1.5                   ≈ $1,635 / month

Manual with scheduled scale up/down (script or automation, 8 h at 40,000, 16 h at 8,000):
  18,667 ÷ 100 × $5.84                                              ≈ $1,090 / month   (but you own the schedule and its failure modes)

Serverless, on consumed RUs:
  per day = 8 h × 3,600 × 30,000 + 16 h × 3,600 × 3,000            ≈ 1.04 billion RU
  per month ≈ 31 billion RU × $0.25 / million                       ≈ $7,780 / month
```

The derivation worth narrating: fully used, provisioned throughput costs `$0.008 ÷ (100 RU/s × 3,600 s)` ≈ **$0.022 per million RU**. Serverless at **$0.25 per million** is about **11× more per consumed RU**, so **serverless wins only when you would use less than roughly 9% of what you'd otherwise provision** — dev/test, prototypes, rarely-used apps. Autoscale wins over manual when hourly peaks average under two-thirds of the maximum (Concept 35). And everything multiplies by the **number of regions** (plus multi-region-write pricing on Business Critical), so a three-region Business Critical account is roughly three times the single-region figure before reserved-capacity discounts.

### 64b. Event Hubs: Standard vs Premium at the high end

**Workload:** the 12 MB/s clickstream from Concept 25 — about **8,000 events/s**, three consumer groups, **18 TUs** at peak (auto-inflate to 30), and **Capture** to ADLS.

```
Standard:
  TUs:       18 TU × ~$22 / TU-month                                 ≈ $   400   (if you scale back down after peaks)
  Ingress:   8,000 events/s × 2.63 M s/month ≈ 21 billion events × ~$0.028 / million  ≈ $   590
  Capture:   ~$73 per TU-month × 18                                  ≈ $ 1,300
                                                                     ≈ $ 2,300 / month
Premium:
  2–3 PU × ~$900 / PU-month (ingress events and Capture included)   ≈ $ 1,800 – 2,700 / month
  + longer retention (up to 90 days), 100 partitions per hub, isolation, Geo-Replication option
```

At this size the two are **in the same range**, and Premium's isolation, partition headroom and longer retention often make it the better buy. At 1–2 MB/s, Standard is far cheaper; at 100+ MB/s, Dedicated enters the conversation. The lesson: **per-event metering and Capture charges make Standard's cost grow faster than its TU count suggests.**

### 64c. Service Bus: Standard vs Premium

```
Standard: ~$10 / month base + operations (roughly $0.80 per million beyond the included allowance; check tiers)
  50 million operations / month (sends + receives + completes + renewals)  ≈ $10 + ~$30 ≈ $40 / month
Premium:  ~$650–700 per MU-month
  2 MU                                                                  ≈ $1,350 / month
  2 MU + Geo-Replication (secondary runs the same 2 MU)                 ≈ $2,700 / month
```

Standard is an order of magnitude cheaper for small and medium workloads; **Premium is bought for isolation, predictability, private networking and the geo features**, not for raw cost efficiency at low volume. Two operational cost notes on Standard: every lock renewal, peek and abandon is an operation, and empty receives during long polling count — tune receive patterns before blaming the tier.

### 64d. Event Grid

```
Basic: ~$0.60 per million operations (first 100,000 / month free)
  1 M events/day × (1 publish + 5 deliveries) × 30 days = 180 M operations   ≈ $108 / month
  Retry storms against a failing webhook can multiply deliveries — filter and fix handlers.
Namespaces: per-TU-hour + per-million operations; size TUs by publish/egress rate and MQTT sessions.
```

**The interview-grade sentence:** *"For Cosmos I compare manual at peak, autoscale on time-weighted hourly peaks at 1.5 times the rate, scheduled manual scaling, and serverless — which costs about eleven times more per consumed RU than fully used provisioned throughput, so it wins only below roughly 9% utilization — and multiply by regions. For Event Hubs at around 12 MB/s, Standard's per-event and Capture charges put it in Premium's range, which then wins on isolation and retention; and Service Bus Premium is bought for isolation and geo features, at roughly $650 to $700 per MU-month, doubled by Geo-Replication."*

---

## Concept 65 — Hidden costs

| Hidden cost | Triggered by | Mitigation |
|---|---|---|
| **Replicated capacity** | Service Bus Geo-Replication (secondary = same MUs); Cosmos regions (RU × regions); Event Hubs Geo-Replication | Replicate only what needs it; separate namespaces/accounts by DR tier |
| **Business Critical / multi-region-write pricing** | Cosmos PPAF or multi-region writes | Use where the SLA justifies it; General Purpose elsewhere |
| **GSI write premium** | Replace/delete on sources with GSIs (+50–100% RU) and GSI throughput | Only the GSIs you query; projections of needed properties |
| **Change feed and backfill RUs** | Every change feed consumer reads from the source container | Fewer, shared processors where possible; low priority for backfills |
| **Indexing** | Default index-everything on write-heavy containers | Tuned indexing policies (Concept 34) |
| **Storage-driven throughput floors** | Large containers with cold data | TTL + archive to Blob; separate cold containers |
| **Event Hubs Capture on Standard** | Per-TU Capture charge | Premium includes it; or archive via your own consumer if cheaper |
| **Auto-inflate never deflating** | Event Hubs Standard after a peak | Scheduled scale-down |
| **Per-operation chatter** | Service Bus Standard renewals, peeks, empty receives; Event Grid retries and fan-out | Tune receive and lock patterns; filter subscriptions; fix failing handlers |
| **Private endpoints** | Per endpoint-hour + per-GB processed; Cosmos needs one per region | Consolidate accounts/namespaces where boundaries allow |
| **Logs** | Cosmos diagnostic logs (per-request), Service Bus/Event Hubs diagnostic logs, SDK diagnostics | Log only what you query; Basic/Auxiliary tiers; retention budgets |
| **Cross-region traffic** | Clients reading from a far region (no preferred regions), cross-region replication egress | Preferred regions; co-locate compute and data |
| **Premium idle capacity** | MUs/PUs sized for peak and never scaled down | Autoscale MUs; right-size PUs after measurement |
| **Dead-letter and claim-check storage** | Never-cleaned DLQs, dead-letter blobs, claim blobs | TTLs and lifecycle policies |

The habit, as in Module 26: **attach each cost to the decision that causes it** — "adding a GSI on `orderNumber` costs its own RU plus roughly half again on our source replaces; we query by order number 40 times a second, so it pays for itself versus cross-partition queries."

**The interview-grade sentence:** *"Beyond list prices I budget replicated capacity — geo-replicated MUs and RU times regions — the Business Critical premium, GSI write premiums, change-feed and backfill RUs, indexing and storage-driven throughput floors, Capture on Standard Event Hubs, auto-inflate that never deflates, per-operation chatter and retry fan-out, private endpoints per region, diagnostic logs and cross-region traffic, and I attach each to the decision that causes it."*

---

## Concept 66 — Lifecycle risk: retirements, renames and previews

| What | Date / status | Action |
|---|---|---|
| `WindowsAzure.ServiceBus`, `Microsoft.Azure.ServiceBus` (and Java `com.microsoft.azure.servicebus`) **retired**; **SBMP** support ended | **September 30, 2026** (done) | `Azure.Messaging.ServiceBus`; Functions Service Bus extension 5.x; migrate old DataContract bodies |
| Event Hubs code on `WindowsAzure.ServiceBus` | Same retirement | `Azure.Messaging.EventHubs` |
| Legacy `Microsoft.Azure.EventHubs` | Deprecated | `Azure.Messaging.EventHubs` |
| Cosmos DB .NET SDK **v2** | Retired (August 2024) | SDK v3 (`Microsoft.Azure.Cosmos`) |
| **.NET 8 and 9** end of support; Functions **in-process** model end of support | **November 10, 2026** | .NET 10, isolated worker (Module 26) |
| **Azure Synapse Link for Cosmos DB** | No longer supported for new projects | Cosmos DB Mirroring for Fabric |
| vCore-based Cosmos DB for MongoDB → **Azure DocumentDB** | Renamed November 18, 2025 | Update docs, IaC and mental models — it's a separate product |
| **Distributed transactions** (Cosmos) | Public preview (June 2026) | Design to adopt at GA; don't depend on it |
| **Azure Backup for Cosmos DB** | Preview | Same |
| **Kafka transactions** (Event Hubs) | Public preview, Premium/Dedicated | Same |
| **Throughput buckets**, **semantic reranking** (Cosmos) | Preview | Same |

Practices (Module 26, Concept 63, unchanged): prefer the option the provider is investing in, track retirements monthly against your **dependency inventory** (including transitive NuGet dependencies), record platform choices in ADRs with review dates, and keep a recurring budget line for migrations.

**The interview-grade sentence:** *"The lifecycle items on this platform right now are the Service Bus legacy libraries and SBMP, retired on September 30 — which also catches Event Hubs code on WindowsAzure.ServiceBus — the November 10 end of .NET 8, .NET 9 and the Functions in-process model, Synapse Link closed to new projects in favor of Fabric mirroring, and the DocumentDB rename; and on the other side, distributed transactions, Azure Backup for Cosmos and Kafka transactions are previews I design to adopt at GA rather than depend on."*

---

## Concept 67 — Anti-patterns catalogue

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Service Bus for telemetry** | High MU cost, no replay, per-message overhead | Event Hubs |
| **Event Hubs as a work queue** | No settlement, no DLQ, no scheduling — failure handling reinvented badly | Service Bus |
| **Event Grid Basic as a queue** | Push retries and 24-hour loss window instead of buffering | Route to Service Bus / Event Hubs, or namespace pull |
| **Dual write: database + publish** | Lost or phantom events | Outbox (SQL) or Cosmos-native outbox via change feed |
| **One giant shared topic** | Filter CPU, noisy neighbors, unclear ownership | Topic per bounded context |
| **Shared consumer groups** | Consumers steal partitions | One group per consumer |
| **Unwatched DLQs and dead-letter blobs** | Silent business failure | Alerts, owners, repair tools |
| **Clients per request** (any SDK) | Connection storms, cold starts, quotas | Singletons |
| **Polly around SDK retries** (429s, transport) | Retry multiplication | One retry owner per failure |
| **Cross-partition hot paths** | RU and latency scale with partitions | Partition key in dominant queries; GSIs |
| **Hot partition keys "fixed" with more RU/s** | Throughput can't fix a hot key | Change the key; HPK; suffixes |
| **Index-everything on write-heavy containers** | Write RU tax | Tuned indexing policy |
| **Analytics on the OLTP container** | Throttles users | Fabric mirroring, GSIs, low priority |
| **"Multi-region" without preferred regions or a runbook** | Accidental latency and failover behavior | SDK region config, PPAF or forced-failover runbook, drills |
| **Geo-replicating everything** | Doubles capacity cost | Replicate by DR tier |
| **Shared keys and connection strings** | Leaks; no least privilege | Entra + data-plane roles; local auth off |
| **Topology created by applications** | Drift; `Manage` rights in apps | IaC / pipeline provisioning |
| **Depending on previews in production** | Compatibility limits, no SLA | Adopt at GA |
| **Scaling consumers into a throttling dependency** | Retry storms | Cap replicas × concurrency; adaptive backoff |

---

## Concept 68 — The platform decision review, and when not to

**A checklist to narrate** for any messaging or data design:

**Flows and contracts**
1. For each flow: fact, command, stream or state? Which service's contract is native (Concept 49)?
2. Ordering scope per flow, and is it preserved at every hop (Concept 52)?
3. Idempotency at every consumer: what's the dedup key and where does it live (Concept 53)?
4. Payload sizes vs limits; claim check where needed (Concept 51)?
5. Contract versioning and envelope (Concept 60)?

**Capacity and cost**
6. The capacity unit for each service, sized from peak, failure and catch-up — and which throttles first (Concept 4)?
7. Partition counts and keys, checked against the hottest key (Concepts 3, 25, 36)?
8. Throughput mode for each Cosmos container and its break-even (Concepts 35, 64)?
9. Hidden costs attached to decisions (Concept 65)?

**Operations and resilience**
10. Failure handling: DLQs, dead-letter containers, parking for logs — with owners and alerts (Concepts 10, 19, 27, 41)?
11. Backpressure: bounded concurrency, autoscale caps, priorities (Concept 54)?
12. Multi-region composition: RPO/RTO per component and for the system, who promotes, drills (Concept 55)?
13. Backup and restore rehearsed (Concept 43)?

**Security and lifecycle**
14. Identity, keys off, private networking, management rights in pipelines only (Concept 56)?
15. Lifecycle: no retired libraries, previews flagged, ADR with review date (Concept 66)?

**When not to:**

- **Don't add a broker for a synchronous need.** If the user is waiting for the answer, a well-protected HTTP call is simpler (Module 11, Concept 47).
- **Don't put every flow on Premium.** Standard Service Bus and Event Hubs are excellent for many production workloads; buy Premium for a named reason (isolation, private networking, geo, retention).
- **Don't choose Cosmos for relational problems** — joins, ad-hoc reporting and strong multi-row constraints belong in SQL or PostgreSQL (Concept 48).
- **Don't go multi-region-write** because it sounds resilient; one write region with PPAF or a forced-failover runbook is simpler and avoids conflicts for most systems.
- **Don't build your own projection framework** before checking whether a GSI does the job — and don't use a GSI when the read model is genuinely different.
- **Don't adopt distributed transactions to avoid modeling.** Put invariants in one partition first.
- **Don't abstract the platform away "in case we switch clouds."** A thin adapter at the edges keeps code testable; a full broker-agnostic layer usually costs more than it ever saves (Module 26, Concept 66).

**The meta-rule, echoing Modules 13 and 26:** *every service, tier, region and feature you add is also a new failure mode, a new cost line and a new thing to secure.* The best platform design gives each flow the service whose contract is native, at the lowest tier whose guarantees it needs for its whole life — with every upgrade to Premium, Business Critical or multi-region justified by a named requirement and written down with a date to revisit it.

**The interview-grade sentence:** *"I review a messaging and data design flow by flow — contract, ordering, idempotency, payload, versioning — then capacity units, partitions, throughput modes and hidden costs, then failure handling, backpressure, multi-region composition and restore drills, then identity, networking and lifecycle. And I hold back: no broker for synchronous needs, Premium and Business Critical only for named reasons, no Cosmos for relational problems, no multi-region writes for their own sake, and no distributed transactions in place of good partition design."*

---

# Putting it together

## Worked example 1 — "Design the messaging and data platform for an order system"

*An e-commerce platform: 2 million orders a day with peaks of 250 orders/s during sales; services for catalog, orders, payments, inventory, shipping and notifications; customers upload product-review photos; the web and mobile apps emit clickstream for personalization (about 10 MB/s at peak); EU users primarily, with a requirement to survive a regional outage for ordering with RPO ≤ 1 minute and RTO ≤ 15 minutes.*

Narrate it in this order:

**1. Classify the flows** (Concept 49):

| Flow | Kind | Service |
|---|---|---|
| `PlaceOrder` → order service | Synchronous command (user waits) | HTTP API (no broker) |
| Order placed / paid / shipped → billing, inventory, shipping, notifications, analytics | Facts with several durable consumers | **Service Bus topic** `orders-events` (topic per bounded context), subscriptions per consumer |
| `ReserveStock`, `CapturePayment`, `CreateShipment` | Commands with one owner, retries, DLQ | **Service Bus queues** owned by receivers |
| Per-order processing in billing and shipping | Ordered per order | **Sessions** with `SessionId = orderId` on those subscriptions |
| Review photo uploaded | Azure resource event | **Event Grid** system topic (Blob) → **Service Bus queue** → moderation worker |
| Clickstream | High-volume stream, several consumers, replay | **Event Hubs** |
| Order state | State with change stream | **Cosmos DB** container `orders`, partition key `/customerId`, with outbox items |

**2. Data model and Cosmos sizing** (Concepts 34–38, Module 12):

- `orders` partitioned by `/customerId` (the dominant query is "my orders"); order + line items + outbox item written in one `TransactionalBatch`.
- Lookup by order number (support, webhooks) → **GSI** keyed by `/orderNumber` with a slim projection.
- Peak writes: 250 orders/s × (order + outbox + status updates ≈ 4 writes) × ~8 RU ≈ **8,000 RU/s**; reads ≈ 2,000/s × 1–2 RU and queries ≈ 4,000 RU/s → plan **autoscale Tmax ≈ 20,000 RU/s**, dynamic scaling on, burst capacity on.
- Consistency: **session**, with the session token flowed to the client after `PlaceOrder` (read-your-writes on the confirmation page).
- Backup: **continuous, 30 days**.

**3. Messaging sizing** (Concepts 8, 12, 25):

- Service Bus **Premium** (private endpoints required, predictable latency, Geo-Replication) — benchmark: ~2 MU at peak with headroom; autoscale 2–4 MU on CPU.
- Duplicate detection on `orders-events` with a **1-hour** window; outbox relay via the **change feed processor** (Concept 41) with `MessageId = outbox id`.
- Event Hubs **Standard**, 32 partitions (partition key = session id), ~10 TUs at peak with auto-inflate to 20 and a nightly scale-down; three consumer groups (personalization, fraud, analytics); Capture to ADLS for the cold path — or Premium if the cost check in Concept 64b says so.

**4. Failure handling and backpressure** (Concepts 10, 19, 54):

- DLQ alerts per subscription; a repair-and-resubmit tool.
- Event Grid → Service Bus means no push-handler fragility; moderation worker scales on queue length via KEDA, capped at the moderation API's rate limit.
- Consumers' `MaxConcurrentCalls × maxReplicas` sized against Cosmos and payment-provider limits; low priority for any backfill.

**5. Multi-region** (Concepts 14, 42, 55):

- Cosmos: West Europe (write) + North Europe (read), zone-redundant both, **Business Critical with PPAF** — automatic partition failover in about 3 minutes; session consistency gives RPO under ~15 minutes *in theory*, typically seconds; if the RPO ≤ 1 minute must be guaranteed for orders, consider strong consistency between the two close regions (and accept the write latency) — say this trade-off out loud.
- Service Bus: **Geo-Replication, synchronous** (the two regions are close, so per-operation latency is acceptable) for RPO 0 on in-flight messages; a promotion runbook with an owner.
- Compute: a stamp per region behind Front Door (Module 26); consumers in the secondary stay scaled to zero until promotion.
- Event Hubs clickstream: **not** geo-replicated — losing a few minutes of clickstream in a regional disaster is acceptable; documented.

**6. Security** (Concept 56): managed identities with entity-scoped roles, local auth off everywhere, private endpoints, topology and containers in Bicep.

**7. Close with what you're not doing**: no Event Hubs for orders (they're commands and facts with consumers that need DLQs), no multi-region writes (single write region with PPAF is simpler), no distributed transactions (the aggregate and outbox share a partition), no Kafka (no Kafka clients or skills in the estate).

---

## Worked example 2 — "Telemetry from 200,000 connected vehicles"

*Each vehicle sends a 1 KB status every 5 seconds over MQTT and must receive occasional commands; operations need a live map (latest position and state per vehicle), alerting within 10 seconds on fault codes, and 2 years of history for analytics.*

**1. Rates:** 200,000 ÷ 5 s = **40,000 messages/s**, 1 KB each → **~40 MB/s** ingress.

**2. Device connectivity — Event Grid namespace MQTT broker** (Concept 21):

- Sessions: 200,000 ÷ ~10,000 per TU → **~20 TUs**; inbound messages: 40,000 ÷ ~1,000 per TU → **~40 TUs** → messages, not sessions, drive capacity; check the maximum TUs per namespace and plan **two namespaces** (split fleets by region or vehicle ID range) if needed.
- X.509 per vehicle; topic spaces `vehicles/${client.authenticationName}/telemetry` and `.../commands`; permission bindings by client group.
- Route MQTT messages to a namespace topic with **push delivery to Event Hubs**.

**3. Stream — Event Hubs Premium** (Concepts 24, 25):

- 40 MB/s rules out comfortable Standard (40-TU ceiling with no headroom) → **Premium**, roughly 5–8 PUs after a load test; partitions: 40 MB/s ÷ ~1–2 MB/s → **64** (Premium allows 100 per hub), partition key = vehicle ID (per-vehicle order).
- Consumer groups: `latest-state`, `alerts`, `analytics`; **Capture** (included) to ADLS for history.

**4. Latest state — Cosmos DB**, but not 40,000 writes/s:

- Writing every message would be 40,000 × ~6 RU ≈ **240,000 RU/s** — expensive and pointless for a map refreshed every few seconds. Instead, the `latest-state` processor **coalesces per vehicle** and upserts (or patches) at most every 15–30 s, or immediately on a state change: ~200,000 ÷ 20 s ≈ **10,000 writes/s × ~6 RU ≈ 60,000 RU/s**, autoscale with dynamic scaling. Partition key `/vehicleId` (point reads by vehicle; map queries by region via a **GSI** on `/regionCell`).
- Or, if the live map is served from a cache, Managed Redis for the latest-state hot set with Cosmos as the durable record.

**5. Alerts** — **Stream Analytics** (or a processor) on the `alerts` consumer group filtering fault codes and windowed conditions, writing alerts to a Service Bus queue for the notification and work-order services (commands with owners, DLQ, retries).

**6. Analytics** — Capture files in ADLS feed **Fabric** (Eventstreams/Eventhouse for recent data with KQL; lakehouse for two years of history). Nothing analytical touches the Cosmos container.

**7. Commands to vehicles** — services publish to MQTT command topics through Event Grid (HTTP publish), per vehicle.

**8. Say the trade-offs:** coalescing loses intermediate states in Cosmos (they're in Capture), per-vehicle order holds per partition only, and MQTT capacity is message-rate-bound.

---

## Worked example 3 — "Our Cosmos bill doubled and we get 429s at 30% utilization"

*A multi-tenant SaaS container partitioned by `/tenantId`, autoscale Tmax 50,000 RU/s, created in 2023 (so dynamic scaling is off), with a nightly reporting job and a new "search everything" feature.*

Diagnose in layers (Concepts 34–36, 47):

1. **429s at 30%** — open **Normalized RU Consumption by PartitionKeyRangeId**: one physical partition sits at 100% while the average is 30%. The partition-key RU logs show **one tenant** responsible for 40% of RU — a hot logical partition, capped by its physical partition's share of RU (and ultimately 10,000 RU/s).
2. **The bill doubled** — three causes compound: classic autoscale scales **all** partitions to the level the hot partition needs (dynamic scaling is off on this pre-September-2024 account); the new search feature runs **cross-partition queries** with `CONTAINS` across every partition; and the nightly report scans the container at normal priority, holding autoscale at Tmax for hours.
3. **Fixes, in order:**
   - **Enable dynamic scaling** — the hot partition scales alone; the bill drops immediately for the rest.
   - **Re-key the hot tenant's data**: move to a **hierarchical partition key** `/tenantId/userId/id` (via the partition-key change feature, or a new container plus change-feed migration), so big tenants span many physical partitions while tenant-prefix queries stay routed.
   - **Search**: move it to a **GSI** with full-text indexing and its own throughput, or to Azure AI Search — off the transactional container.
   - **Reporting**: **Fabric mirroring** (enable continuous backup first) instead of scanning the container; any remaining batch work at **low priority**.
   - **Indexing**: exclude the large free-text `notes` and `history` paths that were indexed by default.
4. **Verify with numbers**: normalized RU per partition, 429 rate (substatus 3200), RU per operation for the top queries before and after, and the hourly autoscale RU billed.

---

## Worked example 4 — "Make this pipeline survive a region outage — what's the RPO?"

*A payments flow: API → Cosmos (payment + outbox in one batch) → change feed relay → Service Bus Premium topic → ledger and notification consumers → Cosmos ledger container. Two regions, 300 km apart.*

1. **Per-component RPO/RTO** (Concept 55):

| Component | Choice | RPO | RTO |
|---|---|---|---|
| Cosmos payments + ledger | Strong consistency across the two close regions, or session + PPAF | 0 (strong) / seconds–minutes (session) | PPAF ~3 min |
| Service Bus | Geo-Replication, **synchronous** | 0 | Promotion decision + reconnection |
| Relay (change feed processor) | Runs in the active stamp; leases in the same Cosmos account | Leases replicate with the account; replay after failover | Restart time |
| Compute | Stamps per region behind Front Door | — | Front Door health probe + scale-out |

2. **The composition question**: with **session** consistency, a forced or partition failover can lose recent payment writes *and* their outbox items; with **synchronous** Service Bus replication, nothing relayed is lost. The weakest link is Cosmos at session — so for payments, choose **strong** consistency between the two close regions (accepting write latency of one cross-region round trip, a few milliseconds at 300 km) and get **system RPO 0**.
3. **In-flight work** replays: relay restarts from leases and re-sends recent outbox items (duplicate detection drops those within the window); consumers' inbox (Concept 53) absorbs the rest.
4. **Promotion runbook**: PPAF handles Cosmos partitions automatically; Service Bus promotion is triggered by automation on namespace health and replication metrics, gated by a human for forced promotion; compute in the secondary scales up on Front Door failover; **no Cosmos control-plane changes during the outage**; failback planned after recovery.
5. **Drill quarterly**: Cosmos PPAF fault simulation and forced failover in staging, Service Bus planned promotion, and a measured end-to-end RTO.

---

## Worked example 5 — "We still have `Microsoft.Azure.ServiceBus` in production"

1. **State the fact**: as of September 30, 2026 it's retired — no support or fixes; anything on `WindowsAzure.ServiceBus` over SBMP has stopped working (Concept 7). It's a security and support finding now.
2. **Inventory** direct and transitive dependencies (old Functions Service Bus extension 4.x, old MassTransit/NServiceBus transports, internal libraries).
3. **Migrate** to `Azure.Messaging.ServiceBus`: one `ServiceBusClient` singleton, cached senders, `ServiceBusProcessor` with explicit settlement; Functions to extension 5.x (and to the isolated worker before November 10 — Module 26).
4. **Handle old message bodies**: messages already in queues may have DataContract-serialized bodies; deserialize explicitly during the overlap, and try both formats before dead-lettering.
5. **Switch to Entra identity and disable SAS** while you're touching every client (Concept 15).
6. **Deploy consumers first** (able to read both formats), then producers; drain, verify, remove the compatibility path.

---
## Common interview questions, with model answers

**"Service Bus, Event Grid or Event Hubs — how do you choose?"** By what's moving. Event Grid routes *facts* — especially Azure resource events — to many independent handlers, by push or, with namespaces, by pull. Service Bus carries *commands and workflow messages* that need per-message locks, retries, dead-lettering, sessions, scheduling or transactions. Event Hubs carries *streams* — high-volume records consumed by several independent readers from their own offsets, with replay. Then I check ordering scope, replay and failure-handling needs, and volume (Concepts 1, 2, 49).

**"Why does our Event Grid handler process the same event several times?"** Event Grid is at-least-once: if the handler doesn't return 2xx within about 30 seconds, or returns an error, Event Grid retries with exponential backoff for up to 30 attempts or 24 hours — even if the handler actually did the work. Fix: acknowledge fast, do the work asynchronously (or route events into Service Bus), and dedupe on the event's `id` and `source` (Concepts 19, 53).

**"Our Cosmos container throttles at 30% utilization. Why?"** Provisioned RU/s are divided evenly across physical partitions, and the dashboard shows the average. One hot partition — usually one hot logical key — is at 100%. I'd confirm with normalized RU consumption per partition key range and the partition-key RU logs, then fix the key (hierarchical keys or a more granular key), enable dynamic scaling, and move cross-partition and analytical work off the container (Concepts 35, 36).

**"How do you get events out of Cosmos reliably?"** A Cosmos-native outbox: write the aggregate and an outbox item in one transactional batch in the same logical partition, then relay the change feed with the change feed processor to Service Bus using the outbox id as `MessageId`, so duplicate detection absorbs relay replays; consumers stay idempotent. Park permanent failures so a poison item doesn't block its lease (Concepts 40, 41).

**"Would you use the new Cosmos distributed transactions or a saga?"** For production today, neither by default: I'd put the invariant inside one logical partition and use `TransactionalBatch`. Distributed transactions are in public preview since June 2026 — 100 operations, 2 MB, write region only, no SLA, and incompatible with continuous backup, PPAF, hierarchical keys and CMK. Across services or long-running steps it's a saga with compensation. I'd revisit distributed transactions at GA for small cross-partition invariants in one account (Concept 40).

**"How would you make this survive a region outage, and what's the RPO?"** I'd compose per-service features: Cosmos with PPAF or a forced-failover runbook, and consistency chosen for the RPO I need — strong for zero, session for minutes; Service Bus Premium Geo-Replication, synchronous for zero loss if the regions are close; Event Hubs Geo-Replication where the stream matters. The system RPO is the weakest link for the data that matters, in-flight work replays through idempotent consumers, and someone owns promotion and failback, which we drill (Concepts 14, 42, 55).

**"What's the difference between Service Bus Geo-DR and Geo-Replication?"** Geo-DR pairs two Premium namespaces behind an alias and replicates only metadata — entities, subscriptions, rules — so messages in the old primary are stranded on failover, and failover breaks the pairing. Geo-Replication, GA since December 2025, replicates metadata, messages and their state to one secondary, synchronously or asynchronously, with planned or forced promotion that keeps the pairing; the secondary costs the same MUs. Availability zones are a third, in-region feature on all tiers (Concept 14).

**"How many partitions does this event hub need?"** The maximum of ingress MB/s over about 1 per partition, egress MB/s per consumer group over about 2, and the number of parallel consumers I want per group — with growth headroom on Standard, where partitions are immutable — then I check the hottest key fits one partition. TUs are the max of ingress MB/s, events per second over 1,000 and total egress across consumer groups over 2 (Concept 25).

**"Our Service Bus consumers process some messages twice even though the code completes them. Why?"** Most likely lock expiry: processing (plus time sitting in the prefetch buffer) exceeded the lock duration, the broker redelivered to another instance, and both completed their work. Fix: prefetch at zero for slow handlers, auto lock renewal with a sensible maximum, a lock duration covering p99, and idempotent handlers regardless (Concepts 10, 12).

**"When would you use Service Bus sessions?"** When processing must be ordered and exclusive per key — per order, per account — and the volume per key is modest. Sessions give FIFO per `SessionId`, one active receiver per session and session state, at the cost of serial throughput per key and head-of-line blocking. If an idempotent, version-checked write already makes order irrelevant, I skip them (Concept 11).

**"Is Event Hubs a drop-in Kafka replacement?"** For producers and consumers, compaction, Kafka Connect and MirrorMaker 2, largely yes on Standard and above, with Entra via OAUTHBEARER. But transactions are in preview and, like compression, only on Premium and Dedicated; topics are managed through Azure with tier partition rules; and clients need tuned timeouts and idle settings. If we depend on deep Kafka ecosystem features, a managed Kafka service is the honest choice (Concept 28).

**"Autoscale, manual or serverless for this Cosmos container?"** Manual for steady load near its peak; autoscale when hourly peaks average under about two-thirds of the max, since it bills the hourly peak at 1.5× — with dynamic scaling so partitions and regions scale independently; serverless only below roughly 9% utilization of what I'd provision, because it costs about eleven times more per consumed RU, and it has no throughput guarantees (Concepts 35, 64).

**"How do you secure these services?"** Entra identities with data-plane roles at the narrowest scope — Service Bus and Event Hubs sender/receiver per entity, Cosmos data-plane RBAC per container — shared keys and SAS disabled, private endpoints, CMK where required, topology and containers managed by pipelines rather than applications, and identity-based connections in Functions (Concepts 15, 45, 56).

**"We have a 2 MB payload to send through Service Bus Standard."** It exceeds the 256 KB limit; even on Premium, large messages are slow and consume MU capacity. Claim check: upload to Blob named by message id, send a reference with a hash, read with managed identity, expire with lifecycle rules (Concept 51).

**"What changed recently in this space that matters?"** The legacy Service Bus libraries and SBMP were retired on September 30, 2026; Service Bus Geo-Replication went GA in December 2025 and Event Hubs Geo-Replication in July 2025; at Build 2026 Cosmos made global secondary indexes, per-partition automatic failover, the Linux emulator, partition-key changes and the all-versions-and-deletes change feed GA, and previewed distributed transactions and Azure Backup; Cosmos accounts are now described as General Purpose or Business Critical; Synapse Link is closed to new projects in favor of Fabric mirroring; and vCore Mongo is now Azure DocumentDB.

**"How do you stop consumers from overwhelming Cosmos when the queue backs up?"** Bound concurrency per instance, cap replicas so replicas × concurrency fits the container's RU budget, let the SDK handle 429s without Polly multiplying them, slow consumers adaptively on throttling, and put batch work at low priority. The backlog should sit in the broker, not turn into a retry storm (Concepts 54, 59).

**"Why not just use Cosmos for everything?"** Because it's optimized for partitioned, key-based access at scale and global distribution; relational queries with joins, ad-hoc reporting and multi-row constraints are better in Azure SQL or PostgreSQL, ephemeral state in Redis, Mongo-compatible workloads in Azure DocumentDB, and telemetry analytics in Fabric Eventhouse — while keeping the number of stores small (Concept 48).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Lists features of each service | Names the contract each flow needs — fact, command, stream or state |
| "It's ordered" | "Ordered per session / partition / partition key, and here's the key at every hop" |
| Assumes exactly-once delivery | Says at-least-once everywhere and shows the dedup key and store |
| Uses Event Grid Basic as a queue | Routes push events into Service Bus or Event Hubs, or uses namespace pull |
| Puts telemetry on Service Bus | Uses Event Hubs with partition and TU arithmetic |
| Sizes Event Hubs from ingress only | Includes egress per consumer group, consumer parallelism and immutable partitions |
| Lets processor exceptions escape | Parks failures, because logs have no dead-letter queue |
| Dual-writes database and broker | Uses an outbox — or the Cosmos-native outbox via the change feed |
| Raises RU/s to fix 429s | Checks per-partition utilization and fixes the hot key |
| Leaves default indexing on write-heavy containers | Tunes indexing and measures RequestCharge |
| Picks serverless "because it's cheaper" | Knows it's ~11× per consumed RU and wins only at very low utilization |
| Assumes autoscale is always cheaper | Computes the two-thirds break-even, and enables dynamic scaling |
| Wraps SDK calls in Polly retries for 429s | Gives each failure one retry owner |
| "We're multi-region" | States RPO/RTO per component, the failover type, who promotes, and the drill cadence |
| Confuses Geo-DR with Geo-Replication | Separates zones, metadata pairing and data replication |
| Plans on distributed transactions | Knows the preview limits and keeps invariants in one partition |
| Uses connection strings | Uses Entra data-plane roles, disables local auth, private endpoints |
| Creates topology from application startup | Provisions topology through pipelines; no Manage rights in apps |
| Unaware of the September 30 retirement | Knows the legacy SDKs and SBMP are gone and how to migrate bodies |
| Runs analytics on the OLTP container | Uses Fabric mirroring, GSIs or low priority |
| Adds Premium, Business Critical and geo-replication everywhere | Buys each for a named requirement and attaches its cost |

---

## Practice exercises

**Exercise 1 — Flow classification (45 min).** Take a system you know. List every asynchronous flow and every dataset. For each flow, classify it (fact, command, stream), name the ordering scope and the dedup key, and assign a service; for each dataset, decide whether it belongs in Cosmos or elsewhere (Concept 48). Draw the topology using the shapes in Concept 50.

**Exercise 2 — Event Hubs and Service Bus sizing (1 hour).** For a stream of 25 MB/s at peak with 4 consumer groups and up to 40 consumers in the largest group, compute partitions and TUs on Standard, then PUs on Premium after estimating per-PU throughput; price both with current list prices, including Capture (Concept 64b). Repeat for a Service Bus workload of 3,000 messages/s at 4 KB with sessions — design a benchmark plan for one MU.

**Exercise 3 — Cosmos RU lab (half a day).** In the Cosmos Linux emulator or a dev account, create a container with default indexing and one with a tuned policy. Insert 10,000 realistic 3 KB documents into each and record write RU; run your five most important queries with `PopulateIndexMetrics` and record RU before and after adding composite indexes. Then compute the monthly cost difference at your production write rate (Concept 34).

**Exercise 4 — Throughput mode break-even (1 hour).** For three real containers (or invented profiles: flat, diurnal, spiky), compute monthly cost under manual at peak, autoscale (with and without dynamic scaling, for a skewed partition profile), scheduled manual, and serverless (Concepts 35, 64a). Identify the break-even utilization for each.

**Exercise 5 — Cosmos outbox relay (half a day).** Implement an aggregate + outbox `TransactionalBatch`, a change feed processor relaying outbox items to a Service Bus emulator topic with `MessageId = outbox id` and duplicate detection, and an idempotent consumer with a Cosmos inbox. Kill the relay mid-batch and the consumer mid-message; verify exactly one effect (Concepts 40, 41, 53).

**Exercise 6 — Failure behavior lab (2 hours).** With emulators: make a Service Bus handler slower than the lock duration with prefetch on — observe duplicates; fix with renewal and prefetch 0. Make an Event Hubs handler throw on one event — observe skipped or stalled processing; fix with parking. Make a change feed delegate throw permanently — observe the blocked lease; fix (Concepts 10, 12, 27, 41).

**Exercise 7 — DR composition (1 hour).** For a pipeline of your choice, fill in Concept 55's table: per-component RPO/RTO, who triggers failover, what replays, how you fail back. Write the one-page runbook and the quarterly drill plan.

**Exercise 8 — Security review (45 min).** For one environment, list every client of Service Bus, Event Hubs, Event Grid and Cosmos; mark which use keys or connection strings; design the move to Entra data-plane roles at entity/container scope, local auth disabled and private endpoints, including Functions identity-based connections (Concept 56).

---

## Resources (free)

### Foundational papers and essays

| Resource | Why it's worth your time |
|---|---|
| [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) — Jay Kreps | The mental model behind Event Hubs and the Cosmos change feed |
| [Kafka: a Distributed Messaging System for Log Processing](https://notes.stephenholiday.com/Kafka.pdf) — Kreps, Narkhede, Rao (NetDB 2011) | The original partitioned-log design that Event Hubs mirrors |
| [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) (SOSP 2007) | Partitioning, replication and conflict resolution ideas behind globally distributed stores |
| [Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service](https://www.usenix.org/conference/atc22/presentation/elhemali) (USENIX ATC 2022) | The best public account of operating a partitioned, throughput-provisioned database — directly comparable to Cosmos RU/s |
| [Implementing Decentralized Per-Partition Automatic Failover in Azure Cosmos DB](https://arxiv.org/abs/2505.14900) | The design behind PPAF |
| [azure-cosmos-tla](https://github.com/Azure/azure-cosmos-tla) | TLA⁺ specifications of Cosmos DB's consistency levels |
| [Probabilistically Bounded Staleness](http://pbs.cs.berkeley.edu/) | The theory behind Cosmos's PBS metric |
| [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/) — Hohpe & Woolf (pattern catalog online) | Claim check, competing consumers, dead letter channel, idempotent receiver — the vocabulary |
| [CloudEvents specification](https://cloudevents.io/) | The event envelope used by Event Grid and recommended across transports |
| [OpenTelemetry messaging semantic conventions](https://opentelemetry.io/docs/specs/semconv/messaging/) | Naming and propagation for traces through brokers |

### Architecture guidance

| Resource | What it covers |
|---|---|
| [Asynchronous messaging options in Azure](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/messaging) | Microsoft's decision guide for Service Bus, Event Grid and Event Hubs |
| [Transactional outbox pattern with Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-outbox-cosmos) | The Cosmos-native outbox, with change feed relay |
| [Claim-Check pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check) · [Competing Consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers) · [Queue-Based Load Leveling](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) | Core patterns with Azure implementations |
| [Multitenancy and Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/cosmos-db) · [Multitenancy and Azure Service Bus](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/service-bus) | Isolation models per service |
| [Well-Architected: Azure Service Bus](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-service-bus) · [Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/cosmos-db) | Reliability, security, cost and operations checklists |
| [Mission-critical workloads on Azure](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-overview) and the [Azure Mission-Critical reference implementation](https://github.com/Azure/Mission-Critical) | Multi-region stamps with Cosmos and messaging, end to end |

### Azure Service Bus

| Resource | What it covers |
|---|---|
| [Service Bus messaging overview](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview) | The entity model and features |
| [Premium messaging tier](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-premium-messaging) · [Automatically update messaging units](https://learn.microsoft.com/en-us/azure/service-bus-messaging/automate-update-messaging-units) | MUs, isolation, large messages, partitioned namespaces, autoscale |
| [Quotas and limits](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas) | Entity counts, sizes, filters, connections |
| [Message transfers, locks and settlement](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement) | Peek-lock, settlement, lock renewal — read it properly |
| [Message sessions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions) · [Duplicate detection](https://learn.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection) · [Dead-letter queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues) | The three features interviews probe most |
| [Topic filters and actions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/topic-filters) · [Auto-forwarding](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-auto-forwarding) · [Transactions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-transactions) | Topology and send-via |
| [Performance best practices](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-performance-improvements) | Client reuse, prefetch, concurrency — and the retirement notice |
| [Geo-Replication](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-geo-replication) · [Geo-Disaster Recovery](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-geo-dr) · [Reliability in Service Bus](https://learn.microsoft.com/en-us/azure/reliability/reliability-service-bus) | The three resilience layers |
| [Authenticate with Microsoft Entra ID / RBAC](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-role-based-access-control) · [Private endpoints](https://learn.microsoft.com/en-us/azure/service-bus-messaging/private-link-service) | Security baseline |
| [Service Bus emulator](https://learn.microsoft.com/en-us/azure/service-bus-messaging/overview-emulator) · [What's new in the emulator](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-emulator-whats-new) | Local development and CI |
| [Migration guide from Microsoft.Azure.ServiceBus](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Azure.Messaging.ServiceBus/MigrationGuide.md) · [from WindowsAzure.ServiceBus](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Azure.Messaging.ServiceBus/MigrationGuide_WindowsAzureServiceBus.md) | The September 2026 retirement, practically |
| [Azure.Messaging.ServiceBus troubleshooting guide](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Azure.Messaging.ServiceBus/TROUBLESHOOTING.md) | Locks, connections, exceptions — very practical |
| [Service Bus Explorer (open source)](https://github.com/paolosalvatori/ServiceBusExplorer) | Inspecting entities and DLQs |

### Azure Event Grid

| Resource | What it covers |
|---|---|
| [What is Event Grid?](https://learn.microsoft.com/en-us/azure/event-grid/overview) · [Namespace concepts](https://learn.microsoft.com/en-us/azure/event-grid/concepts-event-grid-namespaces) | Basic vs namespaces, push vs pull |
| [Delivery and retry](https://learn.microsoft.com/en-us/azure/event-grid/delivery-and-retry) · [Dead-letter and retry policies](https://learn.microsoft.com/en-us/azure/event-grid/manage-event-delivery) | The retry schedule and dead-lettering |
| [Webhook event delivery](https://learn.microsoft.com/en-us/azure/event-grid/webhook-event-delivery) | Endpoint validation handshakes |
| [Pull delivery overview](https://learn.microsoft.com/en-us/azure/event-grid/pull-delivery-overview) | Receive, acknowledge, release, reject |
| [MQTT broker overview](https://learn.microsoft.com/en-us/azure/event-grid/mqtt-overview) · [Routing MQTT messages](https://learn.microsoft.com/en-us/azure/event-grid/mqtt-routing) | Topic spaces, clients, routing |
| [System topics](https://learn.microsoft.com/en-us/azure/event-grid/system-topics) · [Event filtering](https://learn.microsoft.com/en-us/azure/event-grid/event-filtering) · [Event domains](https://learn.microsoft.com/en-us/azure/event-grid/event-domains) | Resource events, subject and advanced filters, multitenant publishing |
| [Quotas and limits](https://learn.microsoft.com/en-us/azure/event-grid/quotas-limits) | Throughput units, subscription counts, sizes |

### Azure Event Hubs

| Resource | What it covers |
|---|---|
| [Features and terminology](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features) · [Compare tiers](https://learn.microsoft.com/en-us/azure/event-hubs/compare-tiers) · [Quotas](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas) | The model and per-tier limits |
| [Scalability](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability) · [Auto-inflate](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-auto-inflate) · [FAQ (partitions)](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-faq) | TUs, PUs, partition guidance |
| [Balance partition load across instances](https://learn.microsoft.com/en-us/azure/event-hubs/event-processor-balance-partition-load) | Ownership, checkpoints, the processor |
| [Event Hubs for Apache Kafka](https://learn.microsoft.com/en-us/azure/event-hubs/azure-event-hubs-apache-kafka-overview) · [Kafka troubleshooting guide](https://learn.microsoft.com/en-us/azure/event-hubs/apache-kafka-troubleshooting-guide) | What's supported, transactions preview, configuration gotchas |
| [Capture](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-overview) · [Schema Registry](https://learn.microsoft.com/en-us/azure/event-hubs/schema-registry-overview) · [Log compaction](https://learn.microsoft.com/en-us/azure/event-hubs/log-compaction) | Cold path, schemas, key-based retention |
| [Geo-Replication](https://learn.microsoft.com/en-us/azure/event-hubs/geo-replication) · [Geo-disaster recovery](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-geo-dr) · [Reliability in Event Hubs](https://learn.microsoft.com/en-us/azure/reliability/reliability-event-hubs) | Resilience layers |
| [Event Hubs emulator](https://learn.microsoft.com/en-us/azure/event-hubs/overview-emulator) | Local AMQP and Kafka |

### Azure Cosmos DB

| Resource | What it covers |
|---|---|
| [Request units](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units) · [Indexing policies](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy) · [Service quotas](https://learn.microsoft.com/en-us/azure/cosmos-db/concepts-limits) | RU economics, indexing, limits |
| [Partitioning and horizontal scaling](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning) · [Hierarchical partition keys](https://learn.microsoft.com/en-us/azure/cosmos-db/hierarchical-partition-keys) · [Change a partition key](https://learn.microsoft.com/en-us/azure/cosmos-db/change-partition-key) | Partition design and the 2026 change capability |
| [Global secondary indexes](https://learn.microsoft.com/en-us/azure/cosmos-db/global-secondary-indexes) | GA in 2026: definition, sync, costs, monitoring |
| [Autoscale throughput](https://learn.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autoscale) · [Dynamic scaling](https://learn.microsoft.com/en-us/azure/cosmos-db/autoscale-per-partition-region) · [Serverless](https://learn.microsoft.com/en-us/azure/cosmos-db/serverless) | Throughput modes |
| [Burst capacity](https://learn.microsoft.com/en-us/azure/cosmos-db/burst-capacity) · [Priority-based execution](https://learn.microsoft.com/en-us/azure/cosmos-db/priority-based-execution) · [Redistribute throughput across partitions](https://learn.microsoft.com/en-us/azure/cosmos-db/distribute-throughput-across-partitions) | Edge-case throughput tools |
| [Consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels) · [Manage consistency](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-manage-consistency) | Levels, per-request overrides, session tokens |
| [Transactional batch](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/transactional-batch) · [Optimistic concurrency](https://learn.microsoft.com/en-us/azure/cosmos-db/database-transactions-optimistic-concurrency) · [Partial document update (patch)](https://learn.microsoft.com/en-us/azure/cosmos-db/partial-document-update) | Single-partition transactional tools |
| [Distributed transactions (preview)](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-configure-and-use-distributed-transactions) | API, limits and incompatibilities |
| [Change feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed) · [Change feed modes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-modes) · [Change feed processor](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-processor) · [Design patterns](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-design-patterns) | The integration backbone |
| [Reliability in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/reliability/reliability-cosmos-db) | The full failover matrix, RPO table and outage guidance |
| [Per-partition automatic failover](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-configure-per-partition-automatic-failover) · [Multi-region writes](https://learn.microsoft.com/en-us/azure/cosmos-db/multi-region-writes) · [Conflict resolution policies](https://learn.microsoft.com/en-us/azure/cosmos-db/conflict-resolution-policies) | Failover and conflicts |
| [Online backup and restore](https://learn.microsoft.com/en-us/azure/cosmos-db/online-backup-and-restore) · [Continuous backup](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction) | Backup modes and PITR |
| [.NET SDK v3 performance tips](https://learn.microsoft.com/en-us/azure/cosmos-db/performance-tips-dotnet-sdk-v3) · [.NET best practices](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/best-practice-dotnet) · [SDK availability troubleshooting](https://learn.microsoft.com/en-us/azure/cosmos-db/troubleshoot-sdk-availability) | Client configuration, regions, hedging |
| [SDK observability (OpenTelemetry)](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/sdk-observability) | Tracing and diagnostics thresholds |
| [Role-based access control (data plane)](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-connect-role-based-access-control) | Entra identities and built-in data roles |
| [Mirroring Cosmos DB in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/mirroring/azure-cosmos-db) · [Vector search](https://learn.microsoft.com/en-us/azure/cosmos-db/vector-search) | Analytics and AI retrieval |
| [Cosmos DB emulator](https://learn.microsoft.com/en-us/azure/cosmos-db/emulator) | Local development (Linux emulator GA) |
| [.NET SDK v3 changelog](https://github.com/Azure/azure-cosmos-dotnet-v3/blob/master/changelog.md) | What each release adds — preview APIs land here first |
| [Build 2026 Cosmos DB announcements](https://devblogs.microsoft.com/cosmosdb/announced-at-ms-build-2026-azure-cosmos-db-mcp-toolkit-semantic-reranking-global-secondary-indexes-and-more/) | GSIs, PPAF, distributed transactions, Linux emulator, partition-key change |
| [Cosmos DB Agent Kit](https://github.com/AzureCosmosDB/cosmosdb-agent-kit) | 100+ best-practice rules on modeling, partitioning, queries and SDK usage — useful as a checklist |
| [Cosmos DB capacity calculator](https://cosmos.azure.com/capacitycalculator/) | RU and cost estimation |
| [Azure DocumentDB FAQ](https://learn.microsoft.com/en-us/azure/documentdb/faq) | The renamed vCore MongoDB service and how it relates to Cosmos |

### .NET, Aspire and tooling

| Resource | What it covers |
|---|---|
| [Dependency injection with the Azure SDK for .NET](https://learn.microsoft.com/en-us/dotnet/azure/sdk/dependency-injection) | `Microsoft.Extensions.Azure` registration and credentials |
| [Azure SDK for .NET repository](https://github.com/Azure/azure-sdk-for-net) | Samples for Service Bus, Event Hubs, Event Grid and Identity |
| [Aspire documentation](https://aspire.dev/) · [Aspire repository](https://github.com/microsoft/aspire) | Hosting integrations and emulators for Service Bus, Event Hubs and Cosmos |
| [Testcontainers for .NET](https://dotnet.testcontainers.org/) | Running emulators in integration tests |

### Free learning paths and channels

| Resource | What it covers |
|---|---|
| [AZ-204: Develop message-based solutions](https://learn.microsoft.com/en-us/training/paths/az-204-develop-message-based-solutions/) | Free Microsoft Learn path on Service Bus and queues |
| [AZ-204: Develop event-based solutions](https://learn.microsoft.com/en-us/training/paths/az-204-develop-event-based-solutions/) | Event Grid and Event Hubs |
| [Azure Cosmos DB Developer Specialty (DP-420) learning paths](https://learn.microsoft.com/en-us/credentials/certifications/azure-cosmos-db-developer-specialty/) | The deepest free Cosmos curriculum Microsoft publishes |
| [Azure Cosmos DB blog](https://devblogs.microsoft.com/cosmosdb/) · [Messaging on Azure blog](https://techcommunity.microsoft.com/category/azure/blog/messagingonazureblog) | Where GA and preview announcements land first |
| [Azure Updates](https://azure.microsoft.com/updates/) | Retirements and GA dates — the lifecycle feed for Concept 66 |
| [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/) | Current prices for the arithmetic in Concept 64 |

---

## Quick-recall sheet

**The contracts.** Event Grid = router for facts (push; pull with namespaces). Service Bus = broker for commands/workflows (locks, DLQ, sessions, transactions). Event Hubs = partitioned log for streams (offsets, consumer groups, replay). Cosmos = partitioned database + change feed. All at-least-once.

**Ordering.** Service Bus per session; Event Hubs per partition; change feed per logical partition key; Event Grid never. Carry one key through every hop, or use versions.

**Service Bus.** Basic/Standard/Premium; Standard 256 KB, Premium 1 MB → 100 MB; MUs 1/2/4/8/16, scale on CPU; 2,000 subscriptions per topic; correlation filters (100k) over SQL filters (2k); lock ≤ 5 min (default 60 s); max delivery count 10; duplicate detection 20 s–7 days (default 10 min); prefetch kills slow handlers; sessions = per-key FIFO + exclusivity; send-via transactions don't include your DB → outbox. Zones (all tiers) vs Geo-DR (metadata) vs Geo-Replication (data, GA Dec 2025, sync/async, planned/forced, one secondary, same MUs). **Legacy SDKs + SBMP retired Sept 30, 2026.** `Azure.Messaging.ServiceBus` 7.21.0.

**Event Grid.** Basic (custom/system/domain/partner topics, push) vs namespaces (pull + push, MQTT, TUs). 2xx within ~30 s or retry: exponential backoff, 30 attempts / 24 h; no retry on 400/413; dead-letter to Blob or lose it. Validation: code echo or CloudEvents OPTIONS. Pull: lock ≤ 5 min, ack/release (with delay)/reject/renew, 7-day retention. MQTT v3.1.1/v5, ~10k sessions and ~1k msg/s per TU. 1 MB events billed per 64 KB; fan-out multiplies operations.

**Event Hubs.** Standard: TU = 1 MB/s or 1,000 events/s in, 2 MB/s out; ≤ 40 TUs; 32 immutable partitions; 20 consumer groups; 7 days. Premium: PUs ≤ 16, 100 partitions/hub (addable), 100 consumer groups, 90 days, CMK, Geo-Replication (GA July 2025). Partitions = max(in ÷ 1, out per group ÷ 2, consumers). Auto-inflate never deflates. Processor: Blob checkpoints, ownership, catch and park, lag = last enqueued − processed. Kafka on Standard+: transactions preview + compression on Premium/Dedicated only; idle connections close in minutes.

**Cosmos.** 1 KB point read ≈ 1 RU (2 at strong/bounded); 1 KB write ≈ 5–6 RU; cross-partition +2–3 RU per partition. ~$5.84 per 100 RU/s-month manual; autoscale 1.5× on hourly peak (wins below ~⅔); serverless ~$0.25/M RU (~11× per RU; wins below ~9%); dynamic scaling default since Sept 25, 2024. Split at ~50 GB or RU > partitions × 10k; RU divides evenly → hot partition = "429s at 30%". HPK prefix routing; GSIs GA (autoscale, eventual, +50–100% RU on source replace/delete). Batch: 100 ops/2 MB in one logical partition; distributed transactions preview (100 ops/2 MB, write region only, no CMK/PPAF/continuous backup/HPK). Change feed: latest-version (no deletes) vs all-versions-and-deletes (GA, needs continuous backup); CFP retries the batch on throw → park poison. Tiers: General Purpose vs Business Critical (multi-write, PPAF ~3 min). Forced failover = yours; service-managed ≥ 1 h. RPO: strong 0, bounded K/T (≥100k ops/300 s), others < ~15 min. Continuous backup 7/30 days → new account. Data-plane RBAC; key auth off. Synapse Link → Fabric mirroring. vCore Mongo → Azure DocumentDB. SDK 3.63.2.

**Composition.** Grid → Bus to buffer; Cosmos batch(aggregate + outbox) → change feed → Bus with MessageId = outbox id; Hubs with a group per consumer + Capture; MQTT → Grid → Hubs. Claim check above size limits. Inbox with TTL for dedup. Cap replicas × concurrency against the dependency. One retry owner per failure.

---

## Threads this module closes, and threads it leaves open

Threads closed:

- **Module 11's platform depth** — Service Bus, Event Hubs and Event Grid as operated platforms, with capacity, topology, SDK tuning, multi-region and security — is now covered (Parts B–D).
- **Modules 7, 8 and 12's Cosmos threads** — RU economics, throughput modes, partition operations, GSIs, change-feed integration, multi-region failover and SDK configuration — are covered in Part E.
- **Module 21's Concepts 79 and 86** and **Module 25's "Service Bus, Event Grid/Event Hubs and Cosmos configuration in depth"** — closed here, including the retry-composition rule applied to SDK settings (Concept 59).
- **Module 26's "managed services are where portability is decided"** — this module is that layer, with lock-in made explicit through contracts and one-way doors.

Threads left open on purpose:

- **Observability in depth** — SLOs and burn-rate alerts built from the metrics in Concept 61, trace propagation through brokers, consumer-lag SLIs, and the cost of telemetry — is **Module 28**.
- **Security architecture** — threat-modeling a messaging pipeline with STRIDE, Entra application design for producers and consumers, Key Vault, network topologies with private endpoints — is **Module 29**.
- **Writing it down** — the platform choices here as ADRs with review dates, and C4 diagrams of the topologies in Concept 50 — is **Module 31**.
- **Brownfield change** — migrating from self-managed RabbitMQ or Kafka to Service Bus or Event Hubs, and moving data into Cosmos with the change feed — is **Module 32**.
- **Cost conversations with executives** — the numbers in Concept 64 as a build-vs-buy and TCO story — are **Module 33**.
- **Worked system designs** — notification systems, order/payment systems and rate limiters built on this platform — are in **Module 37**.

Next in the curriculum: **Module 28 — Observability: OpenTelemetry, distributed tracing, SLO/SLI/error budgets, structured logging**, which takes the signals this module and Module 26 produce and turns them into SLOs you can defend, alerts that page for the right reasons, and traces that follow a request through a broker and back.
