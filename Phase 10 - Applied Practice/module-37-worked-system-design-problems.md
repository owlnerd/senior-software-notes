# Module 37 — Worked System Design Problems with .NET-Specific Implementation Notes
*Phase 10: Applied Practice · Senior/Architect Interview Prep for .NET & C#*

> **Platform state verified on October 8, 2026.** The design reasoning in this module is stable — the five problems have been interview staples for a decade. What moves is the platform underneath the ".NET implementation notes". The facts that calibrate the module:
>
> - **.NET 10 (LTS, November 2025) with C# 14** is the current production release. **.NET 11 (STS) with C# 15** is in preview; Microsoft will launch it at **.NET Conf, November 10–12, 2026**. Design answers should assume .NET 10.
> - **Azure Cache for Redis is being retired.** Per Microsoft's retirement FAQ (updated August 2026): **Basic, Standard and Premium** tiers retire on **September 30, 2028** (instances disabled from October 1, 2028); **Enterprise and Enterprise Flash** retire on **March 31, 2027**. The successor is **Azure Managed Redis** — a first-party service built on **Redis Enterprise** software, **clustered by default** (OSS cluster policy by default; a non-clustered option exists up to 25 GB), zone-redundant by default, and built for **Microsoft Entra ID** authentication. Redis's own announcement also states that new Azure Cache for Redis instances can't be created from October 1, 2026. **In a 2026 design, say "Azure Managed Redis"**, and know why the name changed.
> - **Redis licensing:** Redis 8 (May 2025) is **tri-licensed — RSALv2, SSPLv1 or AGPLv3**. **Valkey**, the Linux Foundation fork of Redis 7.2.4, stays **BSD-3-Clause** (9.x line in 2026) and is the default on AWS and Google Cloud managed caches. **Garnet** — Microsoft Research's RESP-compatible cache server written in .NET — has a 2.x line on NuGet (`Microsoft.Garnet`).
> - **HybridCache** (.NET 9+, `Microsoft.Extensions.Caching.Hybrid`) is the recommended in-process + distributed cache abstraction: L1 + L2, stampede protection, tag-based invalidation. It still has **no built-in backplane** for invalidating other instances' L1 entries; **FusionCache** (which can serve as a `HybridCache` implementation) provides one.
> - **Rate limiting in .NET:** `System.Threading.RateLimiting` (since .NET 7) ships token-bucket, fixed-window, sliding-window and concurrency limiters, and ASP.NET Core's rate-limiting middleware partitions them per key — **in-process only**. Distributed limits need Redis (the community **RedisRateLimiting** package, 1.2.x, implements the same abstractions) or the gateway: **Azure API Management** `rate-limit-by-key` / `quota-by-key` (documented as approximate in distributed deployments) and **Azure Front Door WAF** rate-limit rules.
> - **Messaging libraries:** **MassTransit v9 is commercial** (released January 2026); v8 remains open source, with security patches promised through 2026. **Wolverine** (MIT) and the plain Azure SDKs are the main free alternatives; **NServiceBus** is commercial. Expect the licensing question in any design that names one (Modules 11, 33).
> - **Durable workflows:** the **Durable Task Scheduler** — the managed backend for Durable Functions and the standalone **Durable Task SDKs** (which run on Container Apps, AKS or App Service) — is **GA with its Dedicated SKU** (November 2025); the Consumption SKU is in preview.
> - **Notifications:** Google's legacy FCM APIs are gone; **FCM HTTP v1** is required, and **Azure Notification Hubs** treats FCM v1 as a separate platform (registrations must be migrated). **Azure Communication Services** covers email and SMS; **Azure SignalR Service** and **Azure Web PubSub** cover real-time in-app delivery.
> - **Aspire** — formerly ".NET Aspire" — dropped the ".NET" with **Aspire 13** (November 2025) and is now a polyglot app platform. It's the natural answer to "how would you run this locally?"
> - **Payments:** Stripe's idempotency semantics — the reference most interviewers have in mind — are: the **first result for a key is saved, including 500 errors**; reusing a key with **different parameters is an error**; keys **may be pruned once at least 24 hours old**; idempotency keys apply to **POST** requests.
>
> Product names, retirement dates and licences change; the design reasoning doesn't.

## Orientation

Here is the sentence to carry through the whole module: **a worked system-design problem is not a recall test — the interviewer already knows the "standard answer", so what they are scoring is whether you can derive it from the requirements, defend each choice with numbers, go deep on the one or two components where the real difficulty lives, and say concretely how you'd build it on the stack you claim to know.**

The curriculum entry reads: *Worked system design problems with .NET-specific implementation notes (URL shortener, rate limiter, notification system, distributed cache, order/payment system).* Module 36 ended by promising that several coding-round problems — the rate limiter, the LRU cache, the ledger — reappear here at system scale.

**Why these five problems.** They are not a random sample. Each one isolates a different core of distributed-systems design, so together they cover most of what Phases 3–6 taught:

| Problem | The core it isolates | Modules it exercises most |
|---|---|---|
| **URL shortener** | Read-heavy key–value lookup; unique ID generation; caching under skew | 5 (estimation), 8 (partitioning), 10 (caching) |
| **Rate limiter** | Distributed counting on the hot path, under a sub-millisecond budget; failure policy | 6 (overload), 9 (coordination), 13 (reliability), 25 (Polly) |
| **Notification system** | Asynchronous fan-out; at-least-once delivery to unreliable third parties; priority isolation | 11 (messaging), 13 (bulkheads, retries), 27 (Service Bus) |
| **Distributed cache** | Partitioning, replication and memory management *as the product*; hot keys; stampedes | 7 (consistency), 8 (consistent hashing), 10 (caching) |
| **Order/payment system** | Correctness over throughput; idempotency end to end; multi-step workflows with compensation | 11 (outbox), 12 (transactions, sagas), 22 (aggregates), 24 (event sourcing) |

How this connects to earlier modules:

- **Modules 3–5** (the 7-step framework, requirements, estimation) — this module *applies* them five times. Each design follows the same skeleton, so you can see what changes and what doesn't.
- **Modules 6–13** (distributed-systems theory) — every deep dive here is one of those modules under load. When a concept was derived there, it's referenced rather than re-derived.
- **Modules 14–19 and 25** (.NET mastery, Polly) — the source of the implementation notes: `HybridCache`, `System.Threading.RateLimiting`, EF Core concurrency tokens, `BackgroundService`, resilience pipelines.
- **Modules 26–29** (Azure compute, messaging/data, observability, security) — the source of the platform choices.
- **Modules 30–33** (design documents, ADRs, brownfield, cost) — the architect's version of each problem: "how would you get there from what we have, and what does it cost?"
- **Module 36** (coding rounds) — the in-process algorithms (token bucket, LRU, ledger) whose distributed versions appear here.
- **Module 38** (mock interviews) — uses these five problems as its default practice set.

Why it matters in interviews:

1. **These are the questions you will actually be asked.** Rate limiter, notification system, URL shortener and payment system are among the most frequently reported system-design prompts across big tech and product companies. Practising them is not "gaming" the interview — the interviewer chose them precisely because there's a well-understood space of good answers to calibrate against.
2. **Familiarity is a trap.** Because the "standard answer" is everywhere, a recited answer is easy to spot and scores low: it skips requirements, uses borrowed numbers, and collapses under the first "why?" Senior signal comes from derivation and depth.
3. **The .NET layer is where you differentiate.** Most candidates can draw the boxes. Far fewer can say *which* .NET primitive implements each box, what its exact semantics are, and where it breaks — `HybridCache` stampede protection is per process, the built-in rate limiters are in-process, `ExecuteUpdateAsync` bypasses the change tracker, a Durable Functions orchestrator must be deterministic. That's what a .NET-focused interviewer probes.
4. **Architect loops reuse them differently.** The same prompts appear as "here's our existing system — how would you evolve it?" or "write the ADR for the payment workflow engine". The design is the same; the deliverable is a decision and a migration path.

This module has six jobs:

1. **Teach the method for running a worked problem** — the time budget, the reusable toolbox, and how to talk about .NET and Azure without product-pitching (Part A).
2. **Work the five problems end to end** — requirements, estimation, API, data model, high-level design, two or three deep dives, failure modes, and .NET implementation notes for each (Parts B–F).
3. **Show the code that matters** — not whole systems, but the 20–40 lines an interviewer might ask you to sketch: the redirect path, the Lua token bucket, the dedupe step, the hash ring, the idempotent payment call and the saga orchestrator.
4. **Compare across problems** — what each one tests, how follow-ups pivot, and which mistakes recur (Part G).
5. **Give a full worked transcript** — one 45-minute round with timestamps and what the interviewer writes down.
6. **Give you reference material** — interview Q&A, mistakes vs senior signals, exercises, resources, a quick-recall sheet and one-page cards per problem.

Seven framings to carry through:

1. **Derive, don't recite.** Every box on your diagram should be traceable to a requirement or a number. "We need a cache because reads are 100× writes and p99 must be under 50 ms" beats "add Redis".
2. **Find where the difficulty lives.** Each problem has one or two genuinely hard parts — ID generation and hot keys in the shortener; atomic counting and failure policy in the rate limiter; dedupe and priority isolation in notifications; rebalancing and stampedes in the cache; idempotency and compensation in payments. Spend your deep-dive time there.
3. **State the consistency stance per component.** A redirect can be stale for a minute; a rate-limit count can be approximate; a notification can arrive twice; a cache can lose data; money can do none of these. Say which, and why.
4. **Name the failure mode before the interviewer does.** For every external dependency — Redis, the payment provider, APNs — say what happens when it's slow, down, or returns an ambiguous result.
5. **Implementation notes are claims; make them precise.** "Use `HybridCache`" is a slogan. "`HybridCache.GetOrCreateAsync` coalesces concurrent misses within one process, so 50 instances still make up to 50 origin calls on a cold key" is a claim an interviewer can respect.
6. **Prefer the managed default, and know when to leave it.** Say what you'd use on Azure by default, and the specific condition that would make you build or choose something else. That's the build-vs-buy reasoning from Module 33, compressed.
7. **Leave seams for the follow-up you can see coming.** Multi-region, 10× scale, strict ordering, a new channel, a second payment provider — most rounds add one. Designs that absorb it are the clearest evidence of judgment.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What worked problems test | The interviewer knows the answer; they score derivation and depth |
| 2 | The 45-minute run | Requirements and numbers in 10, high-level by 20, deep dives to 40 |
| 3 | The reusable toolbox | IDs, partition keys, cache tiers, queues + outbox, idempotency, back-pressure |
| 4 | Talking about .NET and Azure | Default + why + when not; exact semantics, not product names |
| 5 | Shortener: requirements and estimation | 100:1 reads; billions of links; skewed popularity |
| 6 | Shortener: API and data model | One access pattern: code → URL; partition by code |
| 7 | Shortener: short-code generation | Random + conditional insert, or leased counter + permutation |
| 8 | Shortener: the redirect path | 302 vs 301; L1 + L2 + edge cache; negative caching |
| 9 | Shortener: analytics, expiry and abuse | Clicks async; TTL capped in caches; creation is the abuse surface |
| 10 | Shortener: .NET implementation notes | Minimal API, `HybridCache`, Cosmos point reads, conditional create |
| 11 | Rate limiter: requirements and placement | What, per whom, where — edge, gateway, service, client |
| 12 | Rate limiter: algorithms | Fixed window → sliding log → sliding counter → token bucket → GCRA |
| 13 | Rate limiter: distributing the count | Atomic scripts in Redis; local vs global; clocks |
| 14 | Rate limiter: failure, hot keys and multi-region | Fail open or closed by purpose; token leasing for hot tenants |
| 15 | Rate limiter: .NET implementation notes | Built-in limiters, partitions, Lua via StackExchange.Redis, APIM |
| 16 | Rate limiter: client experience and rollout | 429 + Retry-After; shadow mode; per-policy metrics |
| 17 | Notifications: requirements and estimation | Channels, priorities, preferences; campaigns dominate throughput |
| 18 | Notifications: architecture | Accept → plan → per-channel lanes → providers → receipts |
| 19 | Notifications: fan-out and in-app feeds | Expand segments in a job; topics for broadcast; inbox per user |
| 20 | Notifications: delivery guarantees | At-least-once + dedupe = effectively once; know the crash windows |
| 21 | Notifications: real-time delivery | SignalR Service / Web PubSub; fall back to push |
| 22 | Notifications: .NET implementation notes | Service Bus scheduling, dedupe, DLQ; KEDA; Notification Hubs; ACS |
| 23 | Cache: which question, and the numbers | "Design Redis" vs "add caching"; memory drives node count |
| 24 | Cache: partitioning and routing | Modulo breaks; consistent hashing / hash slots; smart clients |
| 25 | Cache: replication and failover | Async replicas; the cache may lose data — say so |
| 26 | Cache: memory and eviction | Approximated LRU, LFU with decay, lazy + active expiry |
| 27 | Cache: hot keys, stampedes and invalidation | Single-flight, early refresh, jitter, leases, delete-on-write |
| 28 | Cache: .NET implementation notes | `HybridCache`, multiplexer, Azure Managed Redis, Garnet, a hash ring |
| 29 | Orders: requirements and estimation | Low QPS, high stakes; flash-sale contention is the scale problem |
| 30 | Orders: data model | Order state machine; money as minor units; double-entry ledger |
| 31 | Orders: idempotency end to end | Client key → API → PSP → webhooks → consumers |
| 32 | Orders: the checkout workflow | Reserve → authorize → confirm → capture; compensations |
| 33 | Orders: concurrency and inventory | Optimistic concurrency; conditional decrements; no overselling |
| 34 | Orders: reconciliation, audit and failure | Unknown outcomes, sweepers, reconciliation, PCI scope |
| 35 | Orders: .NET implementation notes | EF Core tokens, outbox, Durable Task orchestrator, Stripe.net keys |
| 36 | Comparing the five | Dominant requirement, core structure, consistency stance |
| 37 | Pivots and follow-ups | Multi-region, 10×, ordering, new channel, brownfield |
| 38 | The architect's version | Decisions, ADRs, migration path, cost |

---
# Part A — How to run a worked problem

## Concept 1 — What worked problems actually test

Start from the interviewer's position. They have asked "design a rate limiter" perhaps fifty times. They know the algorithms, the Redis Lua trick, the fail-open debate. **You cannot surprise them with the answer, so the answer is not what's being scored.** What they are calibrating is:

| What they look for | What it sounds like | What it rules out |
|---|---|---|
| **Derivation** | "Since a single tenant can send 50k requests a second, a central counter would put all of it on one Redis shard — so…" | Reciting a design you read |
| **Quantified trade-offs** | "Fixed windows allow up to 2× the limit at a boundary; for a 100/min login limit that's acceptable, for API billing it isn't" | Adjective-only reasoning ("faster", "more scalable") |
| **Depth where it matters** | Ten minutes on the atomic check-and-decrement, three on the dashboard | Even, shallow coverage of every box |
| **Failure awareness** | "If Redis times out, we fail open for abuse limits but closed for paid quotas" | Happy-path designs |
| **Implementation credibility** | "`TokenBucketRateLimiter` is per process, so on 40 pods a 100 rps limit becomes 4,000" | Hand-waving the stack |
| **Judgment about scope** | "I'd use APIM's policy first and only build this if we need per-tenant dynamic limits" | Building everything from scratch by reflex |

### The familiarity trap

Candidates who have read the popular breakdowns often score *lower* on these problems than on unfamiliar ones, for three reasons:

1. **They skip requirements** because they "know" the problem — and miss the twist the interviewer planned ("the shortener is for an internal enterprise tool, 10k links total", "the rate limiter protects a downstream that allows 20 concurrent calls").
2. **They borrow numbers** ("100 million URLs a day") instead of deriving them, then can't defend them.
3. **They present a finished design**, which leaves nothing for the interviewer to probe except the parts they didn't prepare.

The fix is procedural: **run the same method every time** (Concept 2), even when you know where it ends — and say out loud where your design differs from the textbook version and why.

### What "senior" adds over "mid-level" here

| Step | Mid-level | Senior | Staff / architect |
|---|---|---|---|
| Requirements | Lists features | Separates functional from non-functional; picks numbers that shape the design | Asks who the users are, what failure costs, what exists today |
| Estimation | Computes QPS and storage | Uses the numbers to make decisions ("fits in one node", "needs partitioning") | Adds cost and growth; knows which number is uncertain |
| High-level | Draws the standard boxes | Every box justified; data flow and consistency per arrow | Shows what's bought vs built; team and operational ownership |
| Deep dive | Explains one component | Two deep dives with alternatives and failure modes | Chooses deep dives by risk; connects them to rollout and migration |
| Wrap-up | "We could add monitoring" | Names bottlenecks and the next scaling step | Names the decision record, the migration, the risk register |

**Interview-grade sentence:** *"I treat a well-known design problem as a derivation exercise rather than a recall one — I let the requirements and numbers pick each component, spend my deep-dive time where the real difficulty is, say how each piece fails, and say precisely which .NET or Azure primitive I'd use and where its guarantees stop."*

---

## Concept 2 — The 45-minute run

Module 3 introduced the 7-step framework. Applied to these problems, a 45-minute round (about 40 working minutes) budgets like this:

```text
00–05  Requirements: functional (3–5 core operations) + non-functional (scale, latency,
       availability, consistency, durability) — and what's out of scope
05–09  Estimation: only the numbers that change the design (QPS peak, storage, fan-out,
       hot-key concentration); state the conclusion of each number
09–12  API: 3–5 endpoints or operations with their key parameters and idempotency
12–15  Data model: entities, the dominant access pattern, the partition key
15–22  High-level design: request path(s) end to end; one sentence per arrow
22–38  Deep dives: 2 (sometimes 3) components, chosen by where the difficulty lives —
       alternatives, the choice, failure modes, the .NET implementation
38–42  Wrap-up: bottlenecks, failure scenarios, what changes at 10× / multi-region
42–45  Your questions
```

For a 60-minute round, extend the deep dives (a third one) and the wrap-up, not the requirements.

### Three rules for the clock

1. **High-level design by minute 22, at the latest.** The most common time failure is spending 15 minutes on requirements and estimation for a problem the interviewer considers simple. The deep dives carry the most weight (Module 3).
2. **Offer the deep-dive choice, but have a default.** *"The two hard parts here are the atomic counting and what happens when the store is unavailable. I'd like to go deep on those — unless you'd rather look at multi-region?"* This shows you know where the difficulty is, and gives the interviewer control without losing yours.
3. **Every number must produce a decision.** "11,600 QPS" alone is wasted time. "11,600 QPS at peak — that's well within a few app instances, but 40k reads/s on one Cosmos container is a few thousand RU/s after caching, so cost is fine" is a number doing work.

### The deep-dive menu

For each problem, know three or four candidate deep dives in advance and their order of importance. Appendix D lists them; for example, the URL shortener's menu is: (1) short-code generation, (2) the redirect path and caching under skew, (3) analytics at scale, (4) abuse and expiry. Interviewers rarely want all four.

**Interview-grade sentence:** *"I budget the round so the high-level design is done by about minute twenty — requirements and only the numbers that change the design first — then I propose the two deep dives where the real difficulty is, with a default choice, and keep the last few minutes for bottlenecks and what changes at ten times the scale."*

---

## Concept 3 — The reusable toolbox

The five problems look different, but they're built from the same dozen parts. Naming the reuse is itself a senior signal — it shows you see the design space, not five memorised answers.

| Tool | What it solves | Where it appears in this module | Derived in |
|---|---|---|---|
| **Unique ID generation** | Identifiers without coordination: random, counter blocks, Snowflake-style, UUIDv7 | Short codes; order IDs; notification delivery IDs | Concept 7; Module 8 |
| **Partition key choice** | Spreading load and data; keeping related data together | Links by code; limits by key; inboxes by user; cache by key hash; orders by order ID | Module 8 |
| **Cache tiers** | Latency and origin protection: CDN/edge → L1 in-process → L2 distributed | Redirects; rate-limit token leasing; preferences; product catalogue | Module 10; Part E |
| **Queues and logs** | Decoupling, load levelling, fan-out, retries | Click events; notification lanes; order events | Modules 11, 27 |
| **Transactional outbox** | Publishing events atomically with a state change | Order confirmed → notify, ship; notification accepted → dispatch | Module 11 |
| **Idempotency keys + dedupe stores** | Turning at-least-once into effectively-once | Link creation; notification sends; payment calls; webhooks | Concept 31; Module 11 |
| **Atomic conditional writes** | Correctness under concurrency without locks | Code collision insert; inventory decrement; rate-limit scripts | Modules 9, 12 |
| **Back-pressure and rate limits** | Protecting yourself and your providers | API ingress; provider egress (APNs, SMS, PSP) | Part C; Module 6 |
| **State machines** | Making multi-step processes explicit, resumable and auditable | Notification delivery status; order lifecycle | Concept 30; Module 22 |
| **Sweepers and reconcilers** | Repairing what the happy path missed | Expired links; stuck notifications; unknown payment outcomes | Concept 34 |
| **Bulkheads / priority lanes** | Isolating important work from bulk work | OTPs vs campaigns; checkout vs reporting | Concept 18; Module 13 |
| **Observability spine** | Knowing it works: traces across async hops, per-policy metrics, SLOs | Every problem | Module 28 |

### Three cross-cutting patterns worth saying by name

**1. "At-least-once plus idempotency equals effectively-once."** No distributed component in these designs delivers exactly once — not Service Bus, not Event Hubs, not webhooks, not HTTP retries. The design goal is always: tolerate duplicates at every hop, and make the *effect* happen once using a key the operation carries with it. You'll apply this in Parts B, D and F.

**2. "Hot path stays synchronous and small; everything else goes async."** The redirect, the rate-limit check and the checkout confirmation have latency budgets; analytics, notifications and reconciliation don't. Each design draws that line explicitly.

**3. "The cache is an asynchronous, weakly consistent replica."** Module 10's framing. It explains every cache decision here — TTLs, invalidation, what's allowed to be stale — and why the payment system caches almost nothing on its write path.

**Interview-grade sentence:** *"Most of these designs are built from the same dozen tools — coordination-free IDs, a deliberate partition key, cache tiers, queues with an outbox, idempotency keys, atomic conditional writes, back-pressure, explicit state machines and reconcilers — and I try to name the reuse, because the decisions about each tool are what actually differ between problems."*

---

## Concept 4 — Talking about .NET and Azure in a design round

Interviewers on .NET teams want to hear the stack, but there's a right and a wrong register. The wrong one is **product soup**: "Front Door to APIM to Container Apps with Dapr calling Cosmos and Service Bus and Event Grid…" — names without reasons, which reads as marketing. The right one has three parts per choice.

### The sentence shape: default → why → when not

> *"For the link store I'd default to **Cosmos DB for NoSQL** with the short code as the partition key, **because** the only hot access pattern is a point read by key — about 1 RU for a small item — and it scales horizontally without me managing shards. **I'd choose Azure SQL instead** if we needed relational reporting on links or the team had no Cosmos experience; at a few hundred writes a second either works."*

That's one choice, fully defended, in about twenty seconds.

### The five places .NET interviewers probe

| Area | What they want to hear | Typical probe |
|---|---|---|
| **Compute and hosting** | Which compute, why, and how it scales (Module 26) | "Functions or Container Apps for the notification workers?" |
| **Data access** | The exact client semantics: singleton clients, point reads vs queries, concurrency control, transactions (Modules 12, 19, 27) | "How do you prevent two requests from claiming the same code?" |
| **Messaging** | Broker choice, delivery semantics, sessions, DLQ, outbox, library licensing (Modules 11, 27) | "What happens if the worker crashes after sending the SMS but before completing the message?" |
| **Resilience** | Timeouts, retries only where idempotent, circuit breakers, hedging (Module 25) | "How do you call the payment provider safely?" |
| **Observability** | Traces across async boundaries, per-component SLIs (Module 28) | "How do you know notifications are late?" |

### Precision beats breadth

One precise implementation note is worth more than five product names. Examples you'll meet in this module:

- "`HybridCache` coalesces concurrent misses **per process**, not across the fleet."
- "The built-in `TokenBucketRateLimiter` is **in-memory per instance**; a fleet-wide limit needs a shared store."
- "`ExecuteUpdateAsync` runs immediately and **bypasses change tracking**, so I wrap it in the same transaction as `SaveChanges` if both must commit together."
- "A Durable Functions orchestrator **replays**, so it must be deterministic: `context.CurrentUtcDateTime`, never `DateTime.UtcNow`."
- "Service Bus duplicate detection keys on **`MessageId` within a configurable window**; it protects against producer retries, not consumer redelivery."

### Name the local-dev and test story once

A short sentence at the end of the high-level design earns credibility: *"Locally I'd wire this up with **Aspire** — the API, workers, Redis and the Cosmos and Service Bus emulators — and I'd integration-test the data paths with **Testcontainers**."* Then move on.

### Cost, briefly

Architect interviewers increasingly ask "what does this cost?" You don't need prices; you need the **cost driver** per component: Cosmos by RU/s and storage, Redis by memory, Event Hubs by throughput or processing units, SMS by message, Front Door by requests and egress. Naming the driver shows you'd know where to look (Module 33).

**Interview-grade sentence:** *"When I name a .NET or Azure component I give the default, the reason tied to a requirement, and the condition under which I'd choose differently — and I prefer one precise claim about its semantics, like `HybridCache` only coalescing misses within a process, over a list of product names."*

---
# Part B — URL shortener

> *"Design a URL shortener like bit.ly."*

It's the "easy" problem — which is exactly why it's used to calibrate. A mid-level answer draws a load balancer, a service, a database and a cache. A senior answer derives the code length from the key space, chooses a generation strategy from three real alternatives with their failure modes, explains why the redirect is a 302, and handles skewed popularity and abuse.

## Concept 5 — Requirements and estimation

### Functional requirements (confirm 3–5)

1. **Create** a short link for a long URL, optionally with a **custom alias** and an **expiry**.
2. **Redirect** from the short link to the long URL.
3. **Owner management**: list, disable, delete own links.
4. **Basic analytics**: click counts per link (per day, referrer, country) — *confirm whether real-time*.

Out of scope unless asked: link editing (changing the target), QR codes, branded domains, user billing.

### Non-functional requirements — the questions that shape the design

| Question | Default I'd state | Why it matters |
|---|---|---|
| Read/write ratio? | ~100:1 | Drives caching and the read path's dominance |
| Redirect latency? | p99 < 50 ms at the service (excluding the client's network) | Sets the cache tiers |
| Availability? | Redirects 99.99%; creation 99.9% | Redirects are the product; creation can degrade |
| Can codes be guessable? | No — must not be enumerable | Rules out plain sequential codes |
| Code length? | As short as possible; 7 characters is the usual target | Derived below |
| Consistency? | A new link must redirect immediately for its creator; a disabled link may keep working for up to ~1 minute | Allows caching with short TTLs |
| Retention? | Links live 5 years by default unless an expiry is set | Drives storage |

### Estimation — only the numbers that decide something

Assume **200 million new links per month** (roughly bit.ly scale) and **100:1** reads to writes.

```text
Writes:   200M / (30 × 86,400 s) ≈ 77/s average;  peak ×5 ≈ 400/s
Reads:    20B redirects/month ≈ 7,700/s average;  peak (viral skew) ×5 ≈ 40,000/s
Links:    200M × 12 × 5 years = 12 billion links
Storage:  12B × ~500 bytes (URL ~100–200 B + metadata + index overhead) ≈ 6 TB, before replicas
```

What each number decides:

- **400 writes/s and 40k reads/s** — a handful of stateless instances serve this; the database is the question, not the compute.
- **12 billion links** — too many for one relational node to be comfortable with as a single table on commodity sizing, and 6 TB means **partitioning is needed** (or a horizontally-scaled store from day one).
- **Code length.** With an alphabet of 62 characters (`0–9A–Za–z`):

```text
62^6 ≈ 5.7 × 10^10  (57 billion)      → 12B would fill 21% of the space
62^7 ≈ 3.5 × 10^12  (3.5 trillion)    → 12B fills 0.34% of the space
```

  Six characters technically fit, but at 21% occupancy random generation collides on one insert in five by year 5. **Seven characters** keep occupancy under 0.5% — so collisions are rare, and there's room for growth. That's a derived decision, not a convention.

- **Popularity is Zipf-like.** A tiny fraction of links receives most traffic, and a viral link can take thousands of requests a second on its own. That's the case for caching (and the risk of one hot key).

**Interview-grade sentence:** *"At about 200 million links a month and 100 reads per write, I get roughly 400 writes and 40,000 reads a second at peak and 12 billion links over five years — so compute is easy, storage needs to be horizontally partitioned, and seven base-62 characters keep the key space under half a percent full, which keeps random code collisions rare."*

---

## Concept 6 — API and data model

### API

```http
POST /api/links
Idempotency-Key: 7f3c…            (optional; makes client retries safe)
{ "longUrl": "https://example.com/very/long/path?x=1",
  "customAlias": null,
  "expiresAt": "2027-01-01T00:00:00Z" }

→ 201 Created
{ "code": "aZ3kQ9x", "shortUrl": "https://sho.rt/aZ3kQ9x", "expiresAt": "2027-01-01T00:00:00Z" }

GET /{code}                        → 302 Found, Location: <longUrl>   (or 404 / 410 Gone)
DELETE /api/links/{code}           → 204   (owner only)
GET /api/links/{code}/stats?from=&to=  → aggregated counts
```

Points worth saying:

- **Creation is not naturally idempotent** — two identical POSTs make two codes. An optional `Idempotency-Key` header (or deduplicating per owner by a hash of the long URL) makes client retries safe (Concept 31 develops the pattern fully).
- **Validate the long URL** at creation: scheme allow-list (`http`, `https`), length cap (2,048 is a practical limit), and a reputation check (Concept 9).
- **410 Gone** for expired or disabled links distinguishes "existed" from "never existed" — useful for clients, but leaks existence; pick one deliberately.

### Data model

The dominant access pattern is a **single-key lookup: code → long URL**. Everything else is secondary.

```text
Link
  code         string   (partition key and id)
  longUrl      string
  ownerId      string?
  createdAt    timestamp
  expiresAt    timestamp?
  status       Active | Disabled
  ttl          int?     (store-native expiry, seconds — Cosmos)

OwnerLinks  (secondary access: "list my links")
  ownerId (partition key), createdAt, code
```

**Partition key = `code`.** Codes are uniformly distributed (random or permuted — Concept 7), so load spreads evenly, and every redirect is a **single-partition point read**. "List my links" is the secondary pattern; serving it from a second container (or a materialised view fed by the change feed) keeps it from forcing cross-partition queries on the hot path.

### Which store?

| Option | Fit | Trade-off |
|---|---|---|
| **Cosmos DB for NoSQL**, partition key `/code` | Point read ≈ 1 RU for a ≤1 KB item; conditional create gives collision detection for free; native TTL; global distribution if needed | RU cost model to learn; secondary queries need care |
| **Azure SQL / PostgreSQL** with a unique index on `code` | Familiar; strong uniqueness; easy reporting | 12B rows and 6 TB need partitioning or Hyperscale; more operational work at scale |
| **Cassandra / ScyllaDB / DynamoDB-style KV** | Built for this access pattern | Outside the Azure-native default; still a fine answer |

At this scale I'd default to Cosmos DB; I'd say Azure SQL Hyperscale is a credible alternative if the organisation is SQL-first. Either way the hot path is a key lookup — the store matters less than the partition key and the cache.

**Interview-grade sentence:** *"The data model is dominated by one access pattern — code to URL — so the code is both the identifier and the partition key, which makes every redirect a single-partition point read; secondary patterns like 'list my links' get their own container or view so they never touch the hot path."*

---

## Concept 7 — Short-code generation

This is the first and most important deep dive. There are five real options; a senior answer compares at least three and picks one with its failure mode named.

### Option 1 — Hash the long URL and truncate

`base62(SHA-256(longUrl))[..7]`.

- *Pro:* deterministic — the same URL gets the same code (free deduplication).
- *Con:* **collisions are not rare at all.** Seven base-62 characters is ~41.7 bits. By the birthday bound, the expected number of colliding pairs among *n* random values in a space of size *N* is about *n²/2N*:

```text
n = 1.2 × 10^10 links,  N = 3.5 × 10^12
n² / 2N = 1.44 × 10^20 / 7.0 × 10^12 ≈ 2 × 10^7   → ~20 million collisions over five years
```

  So you need collision detection anyway (append a salt and rehash), and you lose determinism the moment you do. Deduplication by URL is also often *unwanted*: two users shortening the same URL usually want separate links and separate analytics.

### Option 2 — Global counter + base-62

A single sequence: 1, 2, 3… encoded as base-62.

- *Pro:* no collisions, ever; shortest possible codes.
- *Con:* **enumerable** — anyone can walk the code space and harvest every link (a real privacy problem: shortened links often point at unlisted documents). And a single counter is a coordination point.
- *Fix for coordination:* **range leasing** — each instance leases a block of, say, 10,000 IDs from a counter store (an atomic increment in SQL, Cosmos or Redis) and hands them out locally. A crash wastes at most one block; nobody notices gaps.
- *Fix for enumeration:* **permute the counter** with a keyed bijection before encoding (below).

### Option 3 — Snowflake-style 64-bit IDs

Twitter's Snowflake layout: 41 bits of millisecond timestamp (~69 years), 10 bits of worker ID, 12 bits of sequence (4,096 IDs per millisecond per worker). Coordination-free and time-ordered.

- *Con for this problem:* 64 bits encodes to **11 base-62 characters** — too long for a *short* URL. Snowflake is the right answer for order IDs and event IDs, not short codes. Saying so is a good signal: you know the tool and why it doesn't fit.

### Option 4 — Random code + conditional insert *(my default)*

Generate 7 random characters with a cryptographic RNG and insert **only if absent** (Cosmos `CreateItem` returns 409 Conflict on an existing id; SQL raises a unique-constraint violation). On conflict, generate another and retry.

- *Pro:* unguessable (≈41.7 bits of entropy), no coordination, trivially horizontal.
- *Collision cost:* the probability that one new insert collides equals the current occupancy — at worst **0.34%** in year 5. Expected attempts per insert ≈ 1.003. A retry cap of 5 makes failure astronomically unlikely (0.0034⁵ ≈ 5 × 10⁻¹³).
- *Con:* the store must support an atomic insert-if-absent — but every serious store does.

### Option 5 — Pre-generated key service (KGS)

A background job generates unused codes into a pool; instances claim batches.

- *Pro:* no collision check on the write path.
- *Con:* a stateful service to operate, and claiming must itself be atomic — you've moved the problem, not removed it. Worth mentioning; rarely worth building when Option 4's retry rate is 0.3%.

### The permutation trick (for Option 2)

If the organisation wants counter-based codes (no collision retries, predictable capacity planning) but not enumerability, run the counter through a **Feistel network** — a construction that turns any round function into a **bijection** on a fixed-size domain. Over a 40-bit domain (2⁴⁰ ≈ 1.1 trillion codes, which still fits in 7 base-62 characters because 62⁷ > 2⁴⁰), every counter value maps to a unique, scrambled code:

```csharp
// Bijective scrambling of a 40-bit counter: distinct inputs -> distinct outputs.
// Obscures the sequence; it is NOT a security boundary. Keep the round keys secret.
public sealed class CodePermutation
{
    private const int HalfBits = 20;
    private const ulong HalfMask = (1UL << HalfBits) - 1;
    private readonly ulong[] _roundKeys;

    public CodePermutation(ulong[] roundKeys)
    {
        ArgumentNullException.ThrowIfNull(roundKeys);
        ArgumentOutOfRangeException.ThrowIfLessThan(roundKeys.Length, 3);
        _roundKeys = (ulong[])roundKeys.Clone();
    }

    public ulong Permute(ulong counter)
    {
        ArgumentOutOfRangeException.ThrowIfGreaterThan(counter, (1UL << (2 * HalfBits)) - 1);

        ulong left = counter >> HalfBits, right = counter & HalfMask;
        foreach (var key in _roundKeys)
            (left, right) = (right, (left ^ Round(right, key)) & HalfMask);
        return (left << HalfBits) | right;
    }

    private static ulong Round(ulong half, ulong key) =>
        (((half ^ key) * 0x9E3779B97F4A7C15UL) >> 29) & HalfMask;
}
```

Why it's a bijection regardless of the round function: each round maps `(L, R)` to `(R, L ⊕ F(R))`, which is invertible — given the output you recover `R`, recompute `F(R)` and XOR it back out. Composing invertible rounds gives an invertible whole.

### The decision

> *"I'd generate seven random base-62 characters with a cryptographic RNG and insert them conditionally — that's unguessable, needs no coordination, and at under half a percent occupancy the retry rate is negligible. If the business preferred counter-based codes, I'd lease ID ranges per instance and pass them through a keyed Feistel permutation so they aren't enumerable. Snowflake IDs are too long here at eleven characters, and hashing collides about twenty million times over five years, so it doesn't save the collision check."*

### Custom aliases

Same conditional insert, same container — a custom alias is just a user-chosen code. Add a **reserved-word list** (`api`, `admin`, `login`, brand names), a profanity filter, and a length/charset rule. Custom aliases are where the namespace becomes contested, so rate-limit their creation (Part C).

**Interview-grade sentence:** *"For code generation I compare hashing, which collides about twenty million times at our scale so it doesn't avoid the check; a counter, which is collision-free but enumerable unless I lease ranges and permute them through a keyed Feistel network; Snowflake IDs, which are eleven characters and too long; and random codes with a conditional insert — my default, because they're unguessable and need no coordination, and at under half a percent occupancy retries are negligible."*

---

## Concept 8 — The redirect path

### 301 or 302?

| Status | Behaviour | Consequence |
|---|---|---|
| **301 Moved Permanently** | Browsers and intermediaries may cache it indefinitely | Fastest for repeat visitors — but **you lose the click** (no request reaches you), and you can never disable or retarget the link for that browser |
| **302 Found** / **307 Temporary Redirect** | Not cached by default (unless `Cache-Control` says so) | Every click reaches your edge: analytics work, disabling works |
| **308 Permanent Redirect** | Like 301 but preserves the method | Same caching downsides as 301 |

**Default: 302** (or 307 — for `GET` they behave the same), with an explicit, short `Cache-Control: private, max-age=…` if you want *some* browser caching. Choose 301 only if the product explicitly doesn't need analytics or revocation. Saying *why* is the signal; the code itself is trivial.

### The read path, layer by layer

```text
Client ──► Edge (Front Door / CDN): optional caching of hot redirects (short TTL)
        ──► App instance: L1 in-process cache (top N codes, seconds–minutes)
        ──► L2 distributed cache (Redis): hot set, minutes–hours
        ──► Link store (Cosmos point read)
        ──► async: click event to the analytics pipeline (never blocks the redirect)
```

What each layer buys, with numbers:

- **L1 in-process** (e.g. 100,000 entries × ~0.5 KB ≈ 50 MB per instance): absorbs the hottest links with zero network hops — and protects Redis from a single viral key (a hot key in L2 is one shard's problem; in L1 it's spread across every instance).
- **L2 Redis**: if ~50M distinct links are clicked per day and the top 10M take most of the traffic, 10M × 0.5 KB ≈ **5 GB** — one modest Azure Managed Redis instance.
- **Store**: with a 90%+ combined hit ratio, 40k reads/s becomes ≤4k point reads/s ≈ 4,000 RU/s on Cosmos — cheap.
- **Edge**: caching redirects at Front Door is the strongest protection against a viral link, at the cost of analytics fidelity (edge hits don't reach you — you'd use edge logs instead) and revocation delay. Use it with a short TTL, or only when a link is detected as hot.

### Negative caching

Codes that **don't exist** are the attack surface: a scanner walking random codes would miss every cache and hammer the store. Cache "not found" results too (shorter TTL — say 1–5 minutes), and rate-limit 404-heavy clients (Part C). A Bloom filter of existing codes is an optional extra: a "definitely not present" answer in memory, with a tunable false-positive rate. At 12B codes and a 1% false-positive rate a Bloom filter needs about 9.6 bits per element ≈ 14 GB — usually not worth it versus negative caching.

### Consistency on the read path

- **Read-your-writes for the creator:** after creation, write-through to L2 (or simply rely on the store, since nobody has cached a brand-new code — there's nothing stale to serve).
- **Disable/delete:** delete from L2, and accept that L1 entries live until their (short) TTL — this is why L1 TTLs are seconds-to-minutes, and why "disabled links may work for up to a minute" was stated as a requirement. If immediate revocation matters (abuse takedowns), broadcast invalidations (Redis pub/sub or a FusionCache backplane — Concept 27) and purge the edge.
- **Expiry:** **cap every cache entry's lifetime at the link's remaining lifetime**, or an expired link keeps redirecting from cache.

**Interview-grade sentence:** *"I redirect with a 302 so every click reaches us for analytics and revocation; the read path goes edge, in-process L1, Redis L2, then a single-partition point read, which turns 40,000 peak reads a second into a few thousand store reads; I negatively cache unknown codes so scanners can't bypass the cache, and I cap every cache TTL at the link's own expiry."*

---

## Concept 9 — Analytics, expiry and abuse

### Analytics without slowing redirects

Writing a counter to the database on every redirect would turn a read-heavy system into a write-heavy one and put a write on the latency path. Instead:

1. The redirect handler emits a **click event** (code, timestamp, referrer, user-agent class, country from the edge header) into an in-memory buffer and returns immediately.
2. A background sender batches events to **Event Hubs** (or Kafka). At peak that's ~40,000 events/s — size the namespace on *events per second*, not bytes, and check the tier's per-unit ingress limits; at this volume Premium or Dedicated tiers, or **pre-aggregation in the service** (count per code per minute in memory, emit every few seconds), are the realistic options.
3. A stream processor aggregates into per-link, per-day counters (Azure Stream Analytics, Fabric Real-Time Intelligence / Azure Data Explorer, or a consumer service), stored for the stats API.

**Loss tolerance:** analytics can tolerate losing a few events on a crash — say so. That's what justifies in-memory buffering instead of a durable write per click. If the product sells analytics (billing per click), the answer changes: you'd need at-least-once capture and deduplication.

### Expiry

- Store `expiresAt`; use the store's native TTL (Cosmos per-item `ttl`) for physical deletion.
- **Check expiry on read** anyway, and cap cache TTLs (Concept 8) — background TTL deletion is not instantaneous, and caches don't know about it.
- Expired codes are **not reused** by default: someone may have printed the old link. Reuse is a product decision with abuse implications.

### Abuse — the creation endpoint is the attack surface

| Threat | Mitigation |
|---|---|
| Phishing/malware links (shorteners hide destinations) | Reputation check at creation (e.g. a Safe Browsing-style lookup) and periodic re-scans; a takedown path with immediate invalidation; an interstitial warning page for flagged links |
| Mass creation by bots | Rate limits per account and per IP (Part C); CAPTCHA or proof-of-work for anonymous creation; require accounts above a threshold |
| Code-space scanning | Unguessable codes (Concept 7); negative caching; rate limit 404s per client |
| Open-redirect abuse of *your* domain's reputation | Separate short domain from the main brand domain |
| Custom-alias squatting | Reserved words, per-account alias quotas |

**Interview-grade sentence:** *"Analytics must never slow a redirect, so the handler buffers a click event in memory and a background sender batches it to Event Hubs for stream aggregation — accepting a small loss window unless clicks are billable; expiry is enforced on read and in cache TTLs, not only by background TTL deletion; and the creation endpoint gets the abuse controls: reputation checks, rate limits and a takedown path that invalidates caches immediately."*

---

## Concept 10 — .NET implementation notes

### The redirect endpoint (minimal API)

```csharp
app.MapGet("/{code:length(7)}", async (string code, LinkResolver resolver, ClickBuffer clicks,
                                       HttpContext http, CancellationToken ct) =>
{
    var link = await resolver.ResolveAsync(code, ct);
    if (link is null)
        return Results.NotFound();

    clicks.TryRecord(code, http);                         // non-blocking; drops if the buffer is full
    http.Response.Headers.CacheControl = "private, max-age=60";
    return Results.Redirect(link.LongUrl, permanent: false);   // 302
});
```

Points to make: the route constraint rejects malformed codes before any lookup; `Results.Redirect(..., permanent: false)` is a 302 (`preserveMethod: true` would give 307); click recording **never awaits I/O on the request path**.

### Resolution: `HybridCache` over a Cosmos point read

```csharp
public sealed record CachedLink(string? LongUrl, DateTimeOffset? ExpiresAt);   // LongUrl null = "not found"
public sealed record ResolvedLink(string LongUrl);

public sealed class LinkResolver
{
    private readonly HybridCache _cache;
    private readonly Container _links;          // from a singleton CosmosClient
    private readonly TimeProvider _time;

    public LinkResolver(HybridCache cache, Container links, TimeProvider time)
    {
        _cache = cache;
        _links = links;
        _time = time;
    }

    public async Task<ResolvedLink?> ResolveAsync(string code, CancellationToken ct)
    {
        var cached = await _cache.GetOrCreateAsync(
            $"link:{code}",
            (code, self: this),
            static async (state, token) => await state.self.LoadAsync(state.code, token),
            new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromHours(1),              // L2 (Redis)
                LocalCacheExpiration = TimeSpan.FromMinutes(1)   // L1 (in-process) — bounds revocation delay
            },
            cancellationToken: ct);

        if (cached.LongUrl is null) return null;
        if (cached.ExpiresAt is { } expires && expires <= _time.GetUtcNow()) return null;
        return new ResolvedLink(cached.LongUrl);
    }

    private async Task<CachedLink> LoadAsync(string code, CancellationToken ct)
    {
        using var response = await _links.ReadItemStreamAsync(code, new PartitionKey(code), cancellationToken: ct);
        if (response.StatusCode == HttpStatusCode.NotFound)
            return new CachedLink(null, null);                   // negative cache entry
        response.EnsureSuccessStatusCode();

        var doc = await JsonSerializer.DeserializeAsync<LinkDocument>(response.Content, JsonSerializerOptions.Web, ct)
                  ?? throw new InvalidOperationException($"Empty link document for {code}.");
        return doc.Status == LinkStatus.Active
            ? new CachedLink(doc.LongUrl, doc.ExpiresAt)
            : new CachedLink(null, null);
    }
}
```

What to say about it:

- **The state-passing overload** of `GetOrCreateAsync` with a `static` lambda avoids a closure allocation per call — a small performance signal on the hottest path (Module 17).
- **A sentinel record for "not found"** gives negative caching explicitly, rather than relying on how the cache treats `null`.
- **`ReadItemStreamAsync`** avoids the exception the typed `ReadItemAsync` throws on 404 — exceptions on a path that scanners hit deliberately are expensive.
- **Expiry is checked after the cache**, so cached entries can't outlive the link. (In production, also cap `Expiration` at the remaining lifetime when the link has one.)
- **`HybridCache` coalesces concurrent misses per process** — on a cold viral key, *N* instances still make up to *N* store reads. That's fine here; it would matter for an expensive origin (Concept 27).
- **One `CosmosClient` per application** (singleton) — it owns connection pools and partition routing (Module 27).
- Negative entries should get a *shorter* lifetime than positive ones; that needs a second options instance chosen after the load (or a separate key prefix) — worth a sentence, not code.

### Creation: random code + conditional insert

```csharp
public static class ShortCode
{
    private const string Alphabet = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    public const int Length = 7;

    public static string NewRandom() => RandomNumberGenerator.GetString(Alphabet, Length);   // .NET 8+

    public static string Encode(ulong value, int minLength = 0)       // for counter/Feistel codes
    {
        Span<char> buffer = stackalloc char[11];                       // 62^11 > 2^64
        int pos = buffer.Length;
        do { buffer[--pos] = Alphabet[(int)(value % 62)]; value /= 62; } while (value > 0);
        while (buffer.Length - pos < minLength) buffer[--pos] = Alphabet[0];
        return new string(buffer[pos..]);
    }
}

public async Task<string> CreateAsync(CreateLink request, CancellationToken ct)
{
    const int MaxAttempts = 5;
    for (int attempt = 1; attempt <= MaxAttempts; attempt++)
    {
        var code = request.CustomAlias ?? ShortCode.NewRandom();
        var doc = LinkDocument.From(code, request, _time.GetUtcNow());

        using var response = await _links.CreateItemStreamAsync(Serialize(doc), new PartitionKey(code), cancellationToken: ct);
        if (response.IsSuccessStatusCode)
            return code;
        if (response.StatusCode != HttpStatusCode.Conflict)
            response.EnsureSuccessStatusCode();                       // throws for other failures
        if (request.CustomAlias is not null)
            throw new AliasTakenException(request.CustomAlias);      // don't retry a user-chosen alias
    }
    throw new InvalidOperationException("Could not allocate a unique code.");   // ~5 × 10^-13 at year-5 occupancy
}
```

- `RandomNumberGenerator.GetString` (.NET 8) uses a cryptographic RNG and draws characters uniformly from the alphabet — no modulo bias, unlike `bytes[i] % 62`.
- **Conflict is the collision signal** — the store's atomic insert-if-absent does the work; no read-then-write race.
- On Azure SQL the equivalent is `INSERT` with a unique index and catching the duplicate-key error (number 2627 or 2601), or EF Core's `DbUpdateException` around it (Module 19).

### Click buffering

A bounded `Channel<ClickEvent>` with `BoundedChannelFullMode.DropWrite` gives a non-blocking `TryWrite` on the request path and back-pressure by dropping under overload — the explicit loss policy from Concept 9. A `BackgroundService` drains it into an `EventHubBufferedProducerClient` (which batches and publishes in the background), or pre-aggregates counts per code per minute first.

### Hosting and the rest

- **Compute:** Azure Container Apps (scale on HTTP concurrency) or App Service; the redirect service is stateless and tiny — a good Native AOT candidate if cold start or memory density matters (Module 17).
- **Edge:** Azure Front Door for TLS, global anycast entry, WAF (bot rules, rate limits on creation), and optional caching of hot redirects.
- **Observability:** redirect latency p99 and cache hit ratio per tier as the SLIs; trace sampling low on redirects (high volume), full on creation.

**Interview-grade sentence:** *"In .NET the redirect is a minimal-API endpoint returning a 302, resolving through `HybridCache` — L1 bounded to a minute for revocation, L2 in Azure Managed Redis — over a Cosmos stream point read so a 404 costs no exception; creation draws seven characters from `RandomNumberGenerator.GetString` and relies on the store's 409 on conflict; and clicks go through a bounded channel with drop-on-full into a buffered Event Hubs producer, so analytics can never slow a redirect."*

---
# Part C — Rate limiter

> *"Design a rate limiter for our public API."* — or its variants: *"…that works across all our instances"*, *"…per tenant, with different plans"*, *"…to protect a fragile downstream"*.

Module 36 treated the rate limiter as an in-process coding problem. At system scale the algorithm is the easy part; the difficulty is **counting correctly across many instances inside a sub-millisecond budget, and deciding what happens when the counter itself is unavailable.**

## Concept 11 — Requirements and placement

### First, what is it for?

"Rate limiter" covers four different goals, and they lead to different designs. Ask which one — or say which you're assuming:

| Goal | Example | Accuracy needed | When the limiter fails… |
|---|---|---|---|
| **Abuse protection** | 5 login attempts/min per account; 100 req/s per IP | Approximate is fine | Fail **open** — don't take the API down with it |
| **Fair use between tenants** | 1,000 req/min per API key | Within ~10% | Fail open, with a local fallback |
| **Commercial quotas / billing** | 1M calls/month on the Pro plan | Exact (or reconciled exactly later) | Fail **closed** or count asynchronously and reconcile |
| **Protecting a downstream** | The legacy service handles 20 concurrent calls | Strict | A **concurrency** limit, not a rate limit; fail closed |

Distinguish neighbouring concepts while you're at it — interviewers like it:

- **Rate limit** — requests per unit time, per key.
- **Quota** — requests per long period (day, month), usually commercial.
- **Concurrency limit** — requests *in flight*; the right tool for protecting a downstream with fixed capacity (Little's Law, Module 6).
- **Load shedding** — rejecting work when *you* are overloaded, regardless of who sent it (Module 6).
- **Throttling** — often used loosely for any of these; Azure's pattern catalogue uses it for "degrade or reject under load".

### Functional requirements (for the public-API version)

1. Limit requests **per key** — API key, authenticated user, tenant, IP — with **multiple rules** per request (e.g. per-key 100/s *and* per-endpoint 10/s for `POST /exports`).
2. Rules configurable **per plan**, changeable without redeploying.
3. Rejected requests get **429 Too Many Requests** with `Retry-After`.
4. Optional: **burst** allowance above the sustained rate.

### Non-functional

- **Latency:** the check adds < 1 ms p99 at the service (it's on every request).
- **Scale:** e.g. **500,000 requests/s** at peak across ~1 million active keys and ~50 API instances.
- **Accuracy:** for fair-use limits, over-admission under ~10% is acceptable.
- **Availability:** the limiter must not reduce the API's availability (fail-open by default for fair-use).

### Where to put it

| Layer | Azure / .NET option | Good for | Limits |
|---|---|---|---|
| **Edge** | Azure Front Door WAF rate-limit rules | Volumetric abuse by IP or geography, before traffic reaches you | Coarse keys; per-edge counting is approximate; no business identity |
| **API gateway** | Azure API Management `rate-limit-by-key`, `quota-by-key` | Per subscription/key, per plan, without code | Counts are documented as approximate across a distributed gateway; less flexible logic; cost |
| **Service** | ASP.NET Core rate-limiting middleware (+ a shared store for fleet-wide limits) | Business-aware keys (tenant, user, endpoint cost), custom rules | You own the code and the store |
| **Client / caller side** | Polly rate limiter, `System.Threading.RateLimiting` in `HttpClient` pipelines | Respecting *someone else's* limits (APNs, SMS, PSP) | Only protects them from you |

**Senior default:** layer them. Edge for volumetric abuse, the gateway for plan-based limits if you already run APIM, and service-level limits only where you need business-aware keys the gateway can't see. *"I wouldn't build a distributed limiter if APIM's `rate-limit-by-key` meets the accuracy requirement — I'd build one when limits depend on request cost or tenant state the gateway doesn't have."* That sentence is build-vs-buy judgment (Module 33).

**Interview-grade sentence:** *"First I ask what the limiter is for — abuse protection, tenant fairness, commercial quotas or protecting a downstream — because that sets the accuracy and the failure policy; then I layer it: the edge WAF for volumetric abuse, the gateway for plan limits, and a service-level limiter only where the key or cost depends on business data, with a concurrency limit rather than a rate limit for fragile downstreams."*

---

## Concept 12 — The algorithms

Derive them in order; each fixes a flaw in the previous one.

### 1. Fixed window counter

One counter per key per window (e.g. `rl:{key}:{minute}`); increment; reject above the limit.

- *Cost:* one integer per key.
- *Flaw:* **boundary bursts.** With a limit of 100/min, a client can send 100 at 12:00:59 and 100 at 12:01:00 — **200 requests in one second**, i.e. up to **2× the limit** in any window-length interval.

### 2. Sliding window log

Store a timestamp per accepted request; on each request drop timestamps older than the window and count the rest (a Redis sorted set: `ZREMRANGEBYSCORE`, `ZCARD`, `ZADD`).

- *Accuracy:* exact.
- *Cost:* **O(limit) memory per key** — 100 timestamps × ~8–16 bytes ≈ 1–2 KB per key; for 1M keys, 1–2 GB, and much more for high limits. Also O(log n) operations per request.

### 3. Sliding window counter (weighted)

Keep only the current and previous fixed-window counts and estimate the sliding count by weighting the previous window by how much of it still overlaps:

```text
estimate = previous × (window − elapsed_in_current) / window + current
```

Cloudflare's published example: previous minute 42, current minute 18, 15 s into the current minute → 42 × 45/60 + 18 = **49.5**. Cloudflare reported that across real traffic this approximation wrongly allowed or limited only about **0.003%** of requests.

- *Cost:* two integers per key.
- *Assumption:* requests in the previous window were evenly spread. Usually good enough — this is the best default for "N per window" semantics.

### 4. Token bucket

A bucket of capacity *b* refills at rate *r* tokens/second; each request takes a token; empty bucket → reject. Store only `(tokens, lastRefill)` and refill lazily on each request: `tokens = min(b, tokens + (now − last) × r)`.

- *Semantics:* sustained rate *r* **with bursts up to *b*** — matches how most APIs want to behave ("10/s, bursts of 50").
- *Cost:* two numbers per key.
- *Variable cost:* a request can take more than one token (an export costs 10) — easy to express.

### 5. Leaky bucket

Requests enter a FIFO queue drained at a constant rate; full queue → reject.

- *Semantics:* **smooths output** — useful in front of something that needs a steady rate (a provider, a batch job). As a pure admission check it's equivalent to a token bucket; the queue is what differs.
- *Cost:* the queue adds latency and memory.

### 6. GCRA (generic cell rate algorithm)

Store a single timestamp per key — the **theoretical arrival time** (TAT) of the next request if traffic were perfectly paced. With emission interval *T* = period / limit and burst tolerance τ = (burst − 1) × *T*:

```text
on request at time t:
    tat = max(stored_tat, t)
    if tat − t > τ:  reject  (retry after tat − τ − t)
    else:            stored_tat = tat + T;  accept
```

Check the burst: starting idle, after *k* accepted requests at the same instant, TAT − t = *kT*; the next is accepted while *kT* ≤ (burst − 1)*T*, so exactly *burst* requests pass at once — then one per *T*. Token-bucket semantics with **one number of state** and a precise `Retry-After`. It's what several production limiters (e.g. `redis-cell`, and Stripe-style designs described publicly) use.

### Comparison

| Algorithm | State per key | Boundary accuracy | Bursts | Best for |
|---|---|---|---|---|
| Fixed window | 1 int | Up to 2× at boundaries | Uncontrolled | Coarse abuse limits; simplest Redis `INCR` |
| Sliding log | O(limit) timestamps | Exact | None beyond limit | Low limits where exactness matters (login attempts) |
| Sliding counter | 2 ints | ~Approximate (very good) | Smoothed | Default "N per window" |
| Token bucket | 2 numbers | Exact rate + burst | Explicit *b* | Default "rate with bursts"; cost-weighted requests |
| Leaky bucket | Queue | Exact output rate | Absorbed by queue | Shaping outbound traffic |
| GCRA | 1 timestamp | Exact rate + burst | Explicit | Token bucket semantics with minimal state |

**Interview-grade sentence:** *"I'd derive the algorithm from the semantics: fixed windows allow twice the limit at a boundary, a sliding log is exact but costs memory per request, a weighted sliding-window counter fixes the boundary problem with two integers per key, and a token bucket — or GCRA, which is the same semantics in one timestamp — gives a sustained rate with an explicit burst and supports weighted requests; for a public API I'd use a token bucket."*

---

## Concept 13 — Distributing the count

### The core problem

With 50 API instances, a per-instance limiter of 100 req/s admits up to **5,000 req/s** per key. Three ways out:

| Approach | How | Accuracy | Latency | Failure exposure |
|---|---|---|---|---|
| **Local only, divided** | Each instance enforces limit / N | Poor if load is uneven (sticky clients, few connections) or N changes | Zero | None |
| **Sticky routing** | Route each key to one instance (consistent hashing at the gateway) | Good | Zero extra | Rebalancing on scale events; hot keys pin one instance |
| **Central store** | Every check is an atomic operation in Redis | Good | One round trip (~0.3–1 ms in-region) | Redis availability is now on the hot path |
| **Hybrid: token leasing** | Instances lease batches of tokens from the store and spend them locally | Bounded error (≤ batch × instances) | Mostly zero | Degrades gracefully |

The central store is the standard answer; token leasing is the senior refinement for hot keys (Concept 14).

### Atomicity: the race to avoid

The naïve sequence — `GET` the count, compare in the app, `SET` the new value — loses updates under concurrency: two instances read 99, both admit, both write 100. The fix is to make **read-decide-write one atomic operation in the store**:

- **Fixed window:** `INCR` is atomic; then set the expiry. But `INCR` followed by a separate `EXPIRE` has a crash window where the key never expires — use a script, or `EXPIRE key ttl NX` (Redis 7+) on every call.
- **Token bucket / GCRA / sliding counter:** a **Lua script** (or Redis function), which Redis executes atomically:

```lua
-- KEYS[1] = bucket key;  ARGV[1] = refill rate (tokens/s);  ARGV[2] = capacity;  ARGV[3] = cost
local rate      = tonumber(ARGV[1])
local capacity  = tonumber(ARGV[2])
local cost      = tonumber(ARGV[3])

local t   = redis.call('TIME')                          -- one clock for every caller
local now = tonumber(t[1]) + tonumber(t[2]) / 1000000

local state  = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(state[1]) or capacity
local ts     = tonumber(state[2]) or now

tokens = math.min(capacity, tokens + math.max(0, now - ts) * rate)

local allowed, retry_after = 0, 0
if tokens >= cost then
    tokens  = tokens - cost
    allowed = 1
else
    retry_after = (cost - tokens) / rate
end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], math.ceil(capacity / rate) + 1)   -- idle buckets disappear once full again
return { allowed, tostring(retry_after), tostring(tokens) }      -- strings: Lua numbers become integers in replies
```

Points to say:

- **The script runs atomically** — no other command interleaves — so concurrent checks for the same key serialise inside Redis.
- **`redis.call('TIME')` gives one clock** for all instances, so clock skew between app servers doesn't distort refills. (Since Redis 5, scripts replicate their *effects*, so a non-deterministic `TIME` call is allowed.) The trade-off: tests can't control time; either accept real-time tests or pass `now` as an argument in a test-only mode.
- **Returning strings** — Redis truncates Lua numbers to integers in replies, so fractional `retry_after` must be returned as a string.
- **The TTL** lets idle keys expire: once a bucket would be full again, its state is indistinguishable from "no key".
- **Redis Cluster:** a script must only touch keys in one hash slot. One key per call is fine; if a rule needs two keys (per-key and per-endpoint), co-locate them with a hash tag — `rl:{tenant42}:api`, `rl:{tenant42}:exports` — or run two scripts.

### Sizing the store

```text
1M active keys × ~150 bytes (hash with two fields + key + overhead) ≈ 150 MB   → memory is trivial
500k checks/s; a Lua script costs more than a plain GET — budget roughly tens of thousands to
~100k script executions/s per shard, depending on hardware            → plan on several shards
```

So the rate limiter's store is **throughput-bound, not memory-bound** — the opposite of most caches. Shard by key (Redis Cluster does it by slot), and watch for hot keys.

**Interview-grade sentence:** *"Across fifty instances a local limiter admits fifty times the limit, so the count has to be shared; I'd keep it in Redis and make the check a single Lua script — read state, refill using Redis's own clock, decide and write back — so it's atomic without locks, one key per call so it works in a cluster; at a million keys that's only about 150 MB, so the store is sized for script throughput, not memory."*

---

## Concept 14 — Failure, hot keys and multi-region

### When Redis is slow or down

Every API request now depends on Redis. Decide the policy explicitly — **per rule**, by purpose (Concept 11):

| Rule type | On store timeout/unavailability | Why |
|---|---|---|
| Abuse / fair use | **Fail open**, but fall back to a **local in-process limiter** at limit ÷ instance count | The limiter must not take the API down; local limits still cap a single bad actor per instance |
| Commercial quota | **Fail closed**, or admit and **reconcile from logs** later | Revenue and contractual accuracy |
| Downstream protection | **Fail closed** (concurrency limit is local anyway) | The downstream will fall over otherwise |

Implementation details that make this real:

- A **tight timeout** on the check (e.g. 5–20 ms) — a rate limiter that waits a second for Redis has already failed. In .NET, StackExchange.Redis commands don't take a `CancellationToken`; use the multiplexer's timeouts plus `Task.WaitAsync(budget)` for a per-call budget (the command still completes in the background).
- A **circuit breaker** around the store (Module 25) so that during an outage you stop paying the timeout on every request and go straight to the local fallback.
- **Metrics on fallback mode** — "fail open" without an alert is "no rate limiting" without anyone knowing.

### Hot keys

One tenant at 50,000 req/s puts 50,000 script executions/s on **one shard** (all its operations hash to one slot). Options, cheapest first:

1. **Local pre-check:** if the local in-process limiter for that key already rejects, don't call Redis at all — cuts traffic from over-limit clients.
2. **Token leasing:** each instance takes a batch of tokens (`cost = 100`) from the shared bucket and spends it locally until exhausted. Redis load drops by the batch size; worst-case over-admission is bounded by *batch × instances* (and unspent tokens expire with the lease). Make the batch adaptive: large for hot keys, 1 for quiet ones.
3. **Split the key:** `rl:{tenant42}:{shard 0..7}` with limit/8 each, chosen by instance — spreads load at the cost of accuracy under uneven routing.

### Multi-region

A global limit across regions forces a choice:

- **Per-region budgets** (limit × region's traffic share) — simple, no cross-region calls, inaccurate when traffic shifts.
- **Global with asynchronous replication** — e.g. active geo-replication in Azure Managed Redis, which uses conflict-free replicated counters; regions converge, and over-admission is bounded by replication lag × rate.
- **A single home region for the counter** — accurate, but adds a cross-region round trip (tens of ms) to every request. Almost never acceptable on the hot path.

The senior answer is to ask whether a *global* limit is actually required — for fair use it rarely is.

**Interview-grade sentence:** *"Because Redis is now on every request, I set a tight timeout and a circuit breaker on the check and choose the failure policy per rule — fail open with a local per-instance fallback for fair-use limits, fail closed or reconcile later for paid quotas — and I handle hot tenants with local pre-checks and adaptive token leasing, which bounds over-admission to the batch size times the number of instances."*

---

## Concept 15 — .NET implementation notes

### The built-in building blocks

`System.Threading.RateLimiting` (since .NET 7) provides:

| Type | Algorithm |
|---|---|
| `FixedWindowRateLimiter` | Fixed window |
| `SlidingWindowRateLimiter` | Sliding window divided into segments (a segmented approximation of a sliding log) |
| `TokenBucketRateLimiter` | Token bucket |
| `ConcurrencyLimiter` | In-flight concurrency limit |
| `PartitionedRateLimiter<TResource>` | One limiter per partition key, created on demand; `CreateChained` combines several |

All are **in-memory, per process**. That's their contract — it's the single most important thing to say about them in a distributed design.

### ASP.NET Core middleware — per-key, with a correct `Retry-After`

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    options.OnRejected = (context, _) =>
    {
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out TimeSpan retryAfter))
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)Math.Ceiling(retryAfter.TotalSeconds)).ToString(CultureInfo.InvariantCulture);
        return ValueTask.CompletedTask;
    };

    options.AddPolicy("per-tenant", http =>
    {
        // Partition by the *authenticated* identity, not a raw header an attacker can rotate.
        string tenant = http.User.FindFirst("tenant_id")?.Value ?? "anonymous";
        return RateLimitPartition.GetTokenBucketLimiter(tenant, _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,                              // burst
            TokensPerPeriod = 50,                          // sustained 50/s
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            QueueLimit = 0,                                // reject rather than queue on an API
            AutoReplenishment = true,
        });
    });
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();                                      // after auth, so the identity exists

app.MapGet("/api/orders", ListOrders).RequireRateLimiting("per-tenant");
```

Points to make:

- **Middleware order matters:** after authentication if you partition by identity; a limiter that partitions by an unauthenticated header lets an attacker mint fresh partitions per request.
- **`"anonymous"` is one shared partition** — that's deliberate (it caps all anonymous traffic together); IP-based partitions for anonymous traffic are the usual addition, ideally at the edge.
- **Partitions are created on demand and kept in memory** — with unbounded key cardinality (IPs), that's memory you need to think about; idle partitions are cleaned up, but cardinality is still a design input.
- **`QueueLimit = 0`** for APIs: queuing holds connections and hides overload; 429 lets clients back off.
- **Metrics:** ASP.NET Core emits rate-limiting metrics through `System.Diagnostics.Metrics` (meter `Microsoft.AspNetCore.RateLimiting`, .NET 8+), which OpenTelemetry can export — rejected requests, queued requests, lease durations (Module 28).

### Fleet-wide limits: the Redis-backed check

Two routes:

1. **The community `RedisRateLimiting` package** implements the same `RateLimiter` abstractions over Redis (fixed window, sliding window, token bucket, concurrency), so it plugs into `AddPolicy` the same way. A fair answer — check its maintenance and semantics before adopting (Module 33).
2. **Your own check** over the Lua script from Concept 13, used from middleware or an endpoint filter:

```csharp
public sealed record LimitDecision(bool Allowed, TimeSpan RetryAfter);

public sealed class RedisTokenBucket
{
    private static readonly string Script = TokenBucketLua.Source;   // the Lua script from Concept 13
    private readonly IDatabase _db;                       // from the singleton ConnectionMultiplexer
    private readonly TimeSpan _budget = TimeSpan.FromMilliseconds(15);

    public RedisTokenBucket(IConnectionMultiplexer redis) => _db = redis.GetDatabase();

    public async Task<LimitDecision> TryAcquireAsync(string key, double ratePerSecond, int capacity, int cost = 1)
    {
        var result = await _db.ScriptEvaluateAsync(
                Script,
                [(RedisKey)$"rl:{{{key}}}"],                // hash tag {key}: one slot per partition
                [ratePerSecond, capacity, cost])
            .WaitAsync(_budget);                           // throws TimeoutException past the budget

        var values = (RedisResult[])result!;
        bool allowed = (long)values[0] == 1;
        double retry = double.Parse((string)values[1]!, CultureInfo.InvariantCulture);
        return new LimitDecision(allowed, TimeSpan.FromSeconds(retry));
    }
}
```

- `ScriptEvaluateAsync` with a script string lets StackExchange.Redis cache the script's SHA and use `EVALSHA` after the first call — no need to manage `SCRIPT LOAD` yourself.
- The caller catches `TimeoutException` and `RedisConnectionException` and applies the rule's failure policy (local fallback or reject), behind a circuit breaker.
- `{{{key}}}` in the interpolated string produces a literal `{key}` hash tag.

### Outbound limits with Polly

When *you* are the client — APNs, an SMS provider, a payment API with a documented limit — put a limiter in the outbound pipeline so you never hit theirs:

```csharp
services.AddHttpClient<SmsProviderClient>()
    .AddResilienceHandler("sms", pipeline =>
    {
        pipeline.AddRateLimiter(new SlidingWindowRateLimiter(new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 100, Window = TimeSpan.FromSeconds(1), SegmentsPerWindow = 10,
            QueueLimit = 500,                               // outbound work can wait briefly
        }));
        pipeline.AddTimeout(TimeSpan.FromSeconds(5));
    });
```

(Per instance, again — divide the provider's limit by the worker count, or centralise the token bucket.)

### API Management, when it's enough

```xml
<rate-limit-by-key calls="100" renewal-period="1"
                   counter-key="@(context.Subscription?.Key ?? context.Request.IpAddress)"
                   increment-condition="@(context.Response.StatusCode < 500)" />
<quota-by-key calls="1000000" renewal-period="2592000"
              counter-key="@(context.Subscription.Id)" />
```

Know its caveats: counters are approximate across gateway instances and regions, the policy language is limited, and changes are deployments of APIM configuration rather than data.

**Interview-grade sentence:** *"In .NET, the built-in token-bucket, sliding-window and concurrency limiters partitioned per key in the ASP.NET Core middleware are the right tool per instance — after authentication, with queueing off and `Retry-After` from the lease metadata — but they're in-memory by contract, so for a fleet-wide limit I'd either use APIM's `rate-limit-by-key` if its approximation is acceptable or call an atomic Lua token bucket in Redis with a tight time budget and a local fallback; and I'd put a Polly rate limiter on outbound clients so we respect providers' limits."*

---

## Concept 16 — Client experience, rollout and testing

### What the client sees

- **429 Too Many Requests** (RFC 6585) with **`Retry-After`** in seconds.
- Optionally the IETF **RateLimit header fields** (an HTTP API working-group draft: `RateLimit-Policy` describing the quota and `RateLimit` describing remaining capacity) so well-behaved clients can pace themselves before they're rejected.
- Documented limits per plan, and a recommendation that clients retry with **exponential backoff and jitter** — otherwise a fleet of clients that hit a limit at the same moment retries at the same moment.
- **Don't** return 503 for rate limiting (that's "we're overloaded") or 403 (that's "you're not allowed").

### Rolling out limits safely

1. **Shadow mode:** compute decisions and emit metrics, but don't reject. Look at which tenants *would* be limited.
2. **Enforce per rule**, starting with the most permissive limits, with an override list for key customers.
3. **Dynamic configuration:** rules in a config store (Azure App Configuration, or a database table cached locally) so limits change without redeploys.

### Observability

- Decisions per rule and outcome (`allowed`, `rejected`, `fallback`), top limited keys, and store latency p99.
- An alert on **fallback mode** and on **rejection rate spikes** (which may be an attack — or a bug in your own client SDK).

### Testing

- **Built-in limiters:** set `AutoReplenishment = false` and call `TryReplenish()` to step time deterministically.
- **Your own in-process algorithms:** inject `TimeProvider` and test with `FakeTimeProvider` (Module 36).
- **The Redis script:** integration-test against a real Redis in a container (Testcontainers), including concurrent callers and the over-admission bound.

**Interview-grade sentence:** *"Clients get a 429 with `Retry-After` — optionally the IETF RateLimit headers so they can pace themselves — and are told to back off with jitter; I'd roll limits out in shadow mode first with per-rule metrics, make rules dynamic data rather than code, alert on fallback mode, and test the built-in limiters with manual replenishment and the Redis script against a real container."*

---
# Part D — Notification system

> *"Design a notification system that sends push, SMS, email and in-app notifications."*

This problem tests asynchronous design: fan-out, queues, retries, third-party providers that fail in creative ways, and — the part most candidates miss — **isolating urgent notifications from bulk ones** and being precise about **duplicates**.

## Concept 17 — Requirements and estimation

### Functional requirements

1. Internal services **request** a notification: recipient(s), type (e.g. `order.shipped`, `auth.otp`, `marketing.weekly`), template data, optional channel preference and schedule.
2. Deliver over **push (iOS/Android/web), SMS, email and in-app** (a feed plus real-time delivery when the user is online).
3. Respect **user preferences and consent** — per-type opt-in/out, per-channel, quiet hours, unsubscribe.
4. **Templates and localisation** per type and locale.
5. **Delivery tracking**: accepted → sent → delivered/failed, with provider receipts where available.
6. **Campaigns**: send one notification to a segment (e.g. 50M users).

### The non-functional questions that shape the design

| Question | Default | Why it matters |
|---|---|---|
| Latency for transactional? | OTP p99 < 10 s end to end; order updates < 1 min | Needs a fast lane that bulk traffic can't block |
| Campaign throughput? | 50M recipients within 1 hour | Dominates peak throughput |
| Delivery guarantee? | At-least-once internally; **no duplicates visible to users for most types** | Dedupe design |
| Ordering? | Not required in general; per-user ordering for some flows (e.g. chat-like updates) | Sessions only where needed |
| Compliance? | Consent records, unsubscribe in every marketing email (CAN-SPAM/GDPR), SMS consent (TCPA in the US) | Preference checks are mandatory, not best-effort |
| Retention of delivery logs? | 30–90 days | Storage |

### Estimation

```text
Users: 50M;  devices: ~1.5 per user → 75M push tokens
Transactional: ~20M/day ≈ 230/s average; peaks ×10 ≈ 2,300/s
Campaigns: 50M recipients / 3,600 s ≈ 14,000/s for the campaign hour
Token store: 75M × ~300 B ≈ 22 GB
Delivery log: ~100M deliveries/day × ~1 KB ≈ 100 GB/day → 3–9 TB at 30–90 days retention
In-app: if 5% of users are online at peak → ~2.5M concurrent real-time connections
```

What the numbers decide:

- **Campaigns are ~6× the transactional peak** — if both share a queue, an OTP waits behind millions of marketing messages. That single observation drives the architecture (priority lanes).
- **Provider limits, not your compute, are the bottleneck.** APNs and FCM accept high rates; SMS senders and email providers enforce per-sender and per-account throughput and quotas. Your system must **shape egress** per provider (Part C applied outbound).
- **2.5M concurrent connections** for real-time in-app — a managed connection service, not your API pods (Concept 21).

**Interview-grade sentence:** *"With 50 million users the transactional load is only a couple of thousand a second at peak, but a 50-million-recipient campaign in an hour is around 14,000 a second — six times higher — so the first design decision is separating urgent and bulk traffic, and the real throughput limits are the providers', which means egress has to be rate-shaped per provider."*

---

## Concept 18 — The architecture

### The pipeline, stage by stage

```text
 Producers (order, auth, marketing services)
        │  POST /notifications  (idempotency key = producer's event id)
        ▼
 1. Notification API ── validate, authorise producer, persist request, enqueue    [fast ACK: 202]
        ▼
 2. Planner ── resolve recipients → for each: load preferences & consent, pick channels,
              apply quiet hours / frequency caps, render template per locale,
              create one Delivery per (notification, recipient, channel)
        ▼
 3. Per-channel, per-priority queues
        push-high │ push-bulk │ sms-high │ sms-bulk │ email-high │ email-bulk │ inapp
        ▼
 4. Channel workers ── dedupe claim → provider call (rate-shaped, circuit-broken) → record result
        ▼
 5. Providers ── APNs, FCM (or Notification Hubs), Azure Communication Services / SendGrid / Twilio
        ▼
 6. Receipts ── provider webhooks & feedback → Delivery status; invalid tokens → token cleanup
```

Why each boundary exists:

- **API → queue** (stage 1): producers get a fast, durable acknowledgement; downstream slowness never back-pressures the order service. The API stores the request (or relies on the queue's durability) and returns **202 Accepted** with a notification ID.
- **Planner separate from senders** (stage 2): preference lookups, template rendering and fan-out are CPU- and data-heavy and have nothing to do with provider latency. Separating them lets each scale on its own signal.
- **One delivery per (notification, recipient, channel)**: the unit of retry, dedupe, status and analytics. A notification to one user over push and email is two deliveries.
- **Per-channel queues**: a slow SMS provider must not stall push. These are **bulkheads** (Module 13).
- **Per-priority queues**: the OTP lane has its own queue and its own workers (or reserved capacity), so a campaign can't delay it. Within a lane, work is FIFO-ish; across lanes, isolation is physical, not a priority flag on a shared queue.

### The data you need

| Data | Store | Access pattern |
|---|---|---|
| Preferences and consent | Cosmos DB, partition key `userId` (or SQL) | Point read per recipient during planning; cached briefly |
| Device tokens / endpoints | Same, per `userId`; or delegated to Notification Hubs installations | Read per recipient; update on registration; delete on invalid-token feedback |
| Templates | SQL or blob, versioned; cached in memory | Read-mostly |
| Deliveries (status log) | Cosmos (partition by `deliveryId` or `userId`) or a time-partitioned table; TTL 30–90 days | Write-heavy; read by support and analytics |
| In-app inbox | Cosmos, partition key `userId`, items sorted by time | "Latest N for user" |

### Preferences are a correctness requirement

An unsubscribed user receiving marketing is a legal problem, not a UX bug. So: **preferences are read at planning time**, caches holding them use short TTLs or are invalidated on change, and the planner fails *closed* (doesn't send marketing) if the preference store is unavailable — while transactional types (OTP, security alerts) bypass marketing opt-outs by design. Say that distinction out loud.

**Interview-grade sentence:** *"I'd split the pipeline into an API that durably accepts and returns 202, a planner that resolves recipients, checks preferences and consent, renders templates and creates one delivery per recipient and channel, and per-channel, per-priority queues with their own workers — bulkheads, so a slow SMS provider can't stall push and a campaign can't delay an OTP — with provider receipts flowing back into delivery status."*

---

## Concept 19 — Fan-out and in-app feeds

### Campaign fan-out: never in the request

A campaign to a 50M-user segment must not expand inside the API call. Instead:

1. The API accepts a **campaign** (segment definition, template, schedule) and returns.
2. A **segment expander** job pages through the segment (a query over the user store or a precomputed segment table) in **batches** of, say, 1,000 users, emitting one planning message per batch.
3. Planners process batches in parallel, check preferences in bulk, and emit deliveries into the **bulk** lanes.
4. A **campaign-level rate** caps total egress (Concept 14's outbound limiter), so a campaign takes its hour rather than its first five minutes — and leaves provider headroom for transactional traffic.

Checkpoint the expander's position, so a crash resumes rather than restarts (and dedupe handles the overlap).

### Broadcast shortcut: topics and tags

For "everyone subscribed to topic X" push, the provider can fan out: **FCM topics**, or **Notification Hubs tags** and tag expressions. You send one request; the provider sends to millions of devices. Trade-off: you lose per-recipient preference checks and per-delivery tracking at your layer, so it suits broadcast content (breaking news, a sports score), not personalised messages.

### In-app feed: fan-out on write vs on read

The same trade-off as a social feed:

| Strategy | How | Fits |
|---|---|---|
| **Fan-out on write** | Write each notification into each recipient's inbox partition | Personal notifications (most of them); reads are a single-partition "latest N" query |
| **Fan-out on read** | Store the broadcast once; merge it into the feed when the user reads | Broadcasts to millions; avoids writing 50M copies |
| **Hybrid** | Personal → on write; broadcasts → on read, merged in the read API | The usual answer |

Unread counts: a counter per user updated on write and reset on read; tolerate small inaccuracy (it's a badge, not money).

**Interview-grade sentence:** *"Campaign fan-out happens in a checkpointed segment-expansion job that emits batches into the bulk lanes under a campaign-level egress rate, never inside the API call; true broadcasts can use FCM topics or Notification Hubs tags at the cost of per-recipient control; and the in-app feed is fan-out-on-write for personal notifications into an inbox partitioned by user, with broadcasts merged on read."*

---

## Concept 20 — Delivery guarantees: effectively once

### Where duplicates come from

| Source | Example |
|---|---|
| Producer retries | The order service times out on `POST /notifications` and retries |
| Queue redelivery | A worker's lock expires mid-send; the message is delivered again |
| Expander overlap | A crashed segment job resumes from its last checkpoint |
| Ambiguous provider result | The SMS API times out — did it send? |

So "exactly once" is impossible end to end, and the design target is: **at-least-once delivery inside the system + deduplication keyed by a stable identity = effectively once — with the remaining crash window chosen deliberately.**

### The keys

- **Notification ID** — derived from the producer's event ID (`order-123:shipped`), so producer retries collapse at the API (idempotency key, or Service Bus duplicate detection on `MessageId`).
- **Delivery ID** — deterministic: `hash(notificationId, recipientId, channel)`. Any replay of planning produces the same delivery IDs, so later stages deduplicate naturally.

### The claim–send–record protocol in the worker

```text
1. Claim:   conditional update on Delivery(deliveryId):
            status ∈ {Pending, Retrying} or lease expired  →  status = Sending, leaseUntil = now + 2 min
            already Sent / PermanentlyFailed               →  complete the message, stop
            a live lease held by another worker             →  complete this copy (the other worker's own
                                                               message will be redelivered if it crashes)
2. Send:    call the provider (with the provider's idempotency/reference key if it has one)
3. Record:  status = Sent (+ provider message id) → complete the queue message
```

**The window that remains:** a crash *after* the provider accepted but *before* step 3. On redelivery the lease has expired, so the delivery is sent again — **a duplicate**. The alternative (record "Sent" before sending) turns the same crash into **a lost message**. There's no third option without provider-side idempotency, so choose per type:

| Type | Prefer | Mitigation |
|---|---|---|
| OTP, security alert, order update | Duplicate over loss (send-then-record) | Push **collapse keys** (`apns-collapse-id`, FCM `collapse_key`) so a duplicate replaces rather than stacks on the device; idempotent content ("Your code is 123456" twice is harmless) |
| Marketing | Loss over duplicate (record-then-send is acceptable) | Duplicate marketing is a complaint and an unsubscribe |
| Provider supports idempotency keys | Neither — pass the delivery ID | Best case; check each provider's API |

Saying this table out loud — that the crash window is a *choice*, made per notification type — is one of the strongest signals in this problem.

### Retries, failures and the dead-letter queue

| Outcome | Action |
|---|---|
| Success | Record `Sent`; complete |
| Transient failure (timeout, 5xx, 429) | Retry with backoff + jitter, bounded; honour provider `Retry-After`; circuit-break per provider |
| Permanent failure (invalid token, unsubscribed number, hard bounce) | Record `Failed`; **clean up the token/address**; don't retry |
| Retries exhausted | Dead-letter; alert on DLQ depth; a tool to inspect and replay |
| Provider down (circuit open) | Fail over to a secondary provider for SMS/email if configured; push has no secondary |

**Token hygiene matters at scale:** APNs returns *410 Unregistered* and FCM returns *UNREGISTERED* for dead tokens. Not deleting them means a growing share of every campaign is wasted calls — and some providers penalise senders with high invalid rates.

**Interview-grade sentence:** *"Nothing in the pipeline is exactly-once, so I make it effectively-once with stable keys — notification IDs from the producer's event ID and deterministic delivery IDs — and a claim-send-record step in the worker; that leaves one crash window, between the provider accepting and our record, and I choose per type whether it produces a duplicate or a loss: duplicates with collapse keys for OTPs and order updates, losses for marketing, and no window at all where the provider takes an idempotency key."*

---

## Concept 21 — Real-time in-app delivery

When the user is online, in-app notifications should appear instantly; when offline, they're in the inbox for next time, and urgent ones also go out as push.

### Connection handling: don't hold millions of sockets in your API

At ~2.5M concurrent connections, running WebSockets in your own API pods means sticky sessions, a scale-out backplane and connection draining on every deploy. The managed answer:

- **Azure SignalR Service** — your ASP.NET Core hubs keep their programming model; the service holds the client connections and your servers keep only a few connections to the service. Capacity is bought in **units of about 1,000 concurrent connections**, so millions of connections means many units and, beyond a single resource's maximum, **several resources sharded by user**.
- **Azure Web PubSub** — the same idea with plain WebSocket / pub-sub semantics and no SignalR protocol requirement; better for non-.NET clients or simple fan-out.

### The delivery decision

```text
In-app delivery for user U:
   write to U's inbox (always)                         → source of truth
   if U has live connections → push over SignalR/Web PubSub to group "user:{U}"
   if the type is urgent and U is offline (or doesn't ack within N s) → also send mobile push
```

Presence — knowing whether a user is online — can be approximate: the service tells you about connections, and a "send to user" to an offline user is simply a no-op. The inbox write is the durable part; the real-time push is best-effort.

**Interview-grade sentence:** *"For real-time in-app delivery I'd keep the inbox as the durable source of truth and use Azure SignalR Service or Web PubSub to hold the millions of client connections — capacity in roughly thousand-connection units, sharded across resources beyond one resource's limit — sending to a per-user group when they're online and falling back to mobile push for urgent types when they're not."*

---

## Concept 22 — .NET implementation notes

### Queues: Service Bus, with the features this design actually uses

| Need | Service Bus feature | Note |
|---|---|---|
| Producer retries don't duplicate | **Duplicate detection** on `MessageId` within a configurable window | Protects against *producer* resends, not consumer redelivery — the worker still needs its claim step |
| Quiet hours / scheduled sends | **Scheduled messages** (`ScheduleMessageAsync`) | For very large campaigns, a scheduler table + sweeper avoids millions of scheduled messages |
| Poison messages | **Dead-letter queue**, `MaxDeliveryCount` | Alert on DLQ depth; build a replay tool |
| Per-user ordering where required | **Sessions** (`SessionId = userId`) | Only for the flows that need it — sessions cap parallelism per key |
| Priority isolation | **Separate queues** per lane | Not a priority property on one queue |

```csharp
// Scheduling a delivery for the end of the user's quiet hours, with a dedupe-friendly MessageId:
var message = new ServiceBusMessage(BinaryData.FromObjectAsJson(job))
{
    MessageId = job.DeliveryId,          // deterministic — duplicate detection collapses producer retries
    Subject = job.Type,
    ApplicationProperties = { ["priority"] = job.Priority.ToString() },
};
await sender.ScheduleMessageAsync(message, quietHoursEnd, ct);
```

**Retry delays:** abandoning a Service Bus message makes it available again immediately — there's no built-in per-message backoff. For transient provider failures, either retry briefly in-process (a Polly pipeline), or **schedule a retry copy** with a delay and complete the original (tracking the attempt count in the message), so the queue isn't hot-looping on a failing provider.

### The worker

```csharp
public sealed class PushWorker : BackgroundService
{
    private readonly ServiceBusProcessor _processor;     // AutoCompleteMessages = false, MaxConcurrentCalls tuned
    private readonly IDeliveryStore _deliveries;
    private readonly IPushSender _push;
    private readonly ILogger<PushWorker> _log;

    public PushWorker(ServiceBusProcessor processor, IDeliveryStore deliveries, IPushSender push, ILogger<PushWorker> log)
        => (_processor, _deliveries, _push, _log) = (processor, deliveries, push, log);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _processor.ProcessMessageAsync += HandleAsync;
        _processor.ProcessErrorAsync += args =>
        {
            _log.LogError(args.Exception, "Service Bus error on {Entity}", args.EntityPath);
            return Task.CompletedTask;
        };

        await _processor.StartProcessingAsync(stoppingToken);
        try { await Task.Delay(Timeout.Infinite, stoppingToken); }
        catch (OperationCanceledException) { /* shutting down */ }
        await _processor.StopProcessingAsync(CancellationToken.None);
    }

    private async Task HandleAsync(ProcessMessageEventArgs args)
    {
        var job = args.Message.Body.ToObjectFromJson<DeliveryJob>()
                  ?? throw new InvalidOperationException("Empty delivery job.");   // → retried, then dead-lettered

        var claim = await _deliveries.TryClaimAsync(job.DeliveryId, TimeSpan.FromMinutes(2), args.CancellationToken);
        if (claim is not ClaimResult.Claimed)
        {
            await args.CompleteMessageAsync(args.Message, args.CancellationToken);  // already sent, or owned elsewhere
            return;
        }

        var result = await _push.SendAsync(job, args.CancellationToken);           // collapse id = job.CollapseKey
        switch (result)
        {
            case SendResult.Sent sent:
                await _deliveries.MarkSentAsync(job.DeliveryId, sent.ProviderMessageId, args.CancellationToken);
                await args.CompleteMessageAsync(args.Message, args.CancellationToken);
                break;
            case SendResult.InvalidToken:
                await _deliveries.MarkFailedAsync(job.DeliveryId, "invalid-token", args.CancellationToken);
                await _deliveries.RemoveTokenAsync(job.RecipientId, job.Token, args.CancellationToken);
                await args.CompleteMessageAsync(args.Message, args.CancellationToken);
                break;
            case SendResult.Transient transient:
                await _deliveries.ReleaseForRetryAsync(job.DeliveryId, args.CancellationToken);
                await ScheduleRetryOrDeadLetterAsync(args, job, transient.RetryAfter);   // delayed copy, attempt + 1
                break;
        }
    }
}
```

Points to say: completion is **manual** so the message is only removed after the record; the claim makes redelivery and producer duplicates harmless; `InvalidToken` cleans up; the transient path schedules a delayed retry rather than hot-looping. The pattern-match over a closed set of result records is clearer than exception-driven flow (Module 36). On .NET 11, C# 15 union types would model `SendResult` natively — in .NET 10 a sealed record hierarchy does the job.

### Compute and scaling

- **Azure Container Apps** with **KEDA** scaling each worker type on its **queue length** (the Service Bus scaler), with different minimum replicas per lane — the OTP lane never scales to zero.
- **Azure Functions** with Service Bus triggers is a fine alternative for the senders; mind the concurrency settings so a burst doesn't exceed provider limits.

### Providers

- **Push:** call APNs (HTTP/2, token-based `.p8` auth) and FCM HTTP v1 (OAuth2 service-account tokens) directly, or delegate to **Azure Notification Hubs** — which manages platform credentials, installations/registrations, tags and templates. Note the 2024–26 change: FCM v1 is a **separate platform** in Notification Hubs, so registrations created for legacy FCM must be migrated.
- **Email and SMS:** **Azure Communication Services** (email domains, SMS numbers) or SendGrid/Twilio; plan for **sending quotas** that need raising before campaign volumes, and **sender reputation** (SPF, DKIM, DMARC for email).
- **Abstraction:** one `IChannelSender` per channel with provider adapters behind it, so a secondary SMS provider is a configuration change; resilience per adapter with `Microsoft.Extensions.Http.Resilience` — timeout, retry on transient only, circuit breaker, outbound rate limiter (Concept 15).

### Templates and preferences

- **Liquid-style templates** (Scriban or Fluid) rendered per locale, versioned; validate templates at publish time, not send time.
- **Preferences** read through `HybridCache` with a short local expiry, invalidated on change — and the planner fails closed for marketing if the store is unavailable.

### Messaging libraries

You can build this on the plain `Azure.Messaging.ServiceBus` SDK (as above), or on a messaging framework for outbox, retries and sagas. In 2026 the licensing matters: **MassTransit v9 is commercial**, **NServiceBus** is commercial, **Wolverine** is MIT. Name the trade-off rather than assuming the free MassTransit of 2024.

### Observability

End-to-end latency per type and lane (accepted → sent), queue age per lane (the earliest warning that a lane is starved), provider error rates, DLQ depth, invalid-token rate. Propagate trace context through `ServiceBusMessage` application properties so one notification is one distributed trace across API, planner and sender (Module 28).

**Interview-grade sentence:** *"In .NET I'd build the senders as `BackgroundService` workers on `ServiceBusProcessor` with manual completion, a claim step against a delivery store and a closed set of result records, scaled by KEDA on queue length per lane; I'd use Service Bus duplicate detection for producer retries, scheduled messages for quiet hours and delayed retry copies because abandon has no backoff; and I'd reach providers through per-channel adapters with resilience pipelines — Notification Hubs or APNs and FCM v1 directly for push, Azure Communication Services for email and SMS."*

---
# Part E — Distributed cache

> *"Design a distributed cache."* — or *"Design Redis"*, *"Design Memcached"*, *"Design a distributed key-value store with TTLs."*

## Concept 23 — Which question is it, and the numbers

### Two different questions share one prompt

| Question | What they want | Where Module 10 helps |
|---|---|---|
| **A. "Design a distributed cache *service*"** (build Redis/Memcached) | Partitioning, replication, failover, eviction, memory management, the client protocol | The internals |
| **B. "Add caching to this system"** (use one) | Cache-aside vs read-through, TTLs, invalidation, stampedes, consistency | The usage patterns |

Ask. If they mean A, most of the time still goes to A's internals, but B's concerns — hot keys, stampedes, invalidation — come up as the follow-up ("now the clients…"). This part covers both, in that order.

### Requirements for version A

- **Operations:** `GET`, `SET` (with optional TTL), `DELETE`; optionally atomic `INCR`, compare-and-set.
- **Scale:** **1 TB** of hot data; **1M reads/s and 100k writes/s**; values ~1 KB average, up to 1 MB.
- **Latency:** p99 < 1 ms within a datacentre (single-digit ms across zones).
- **Availability:** survive the loss of a node or a zone without a full outage.
- **Durability:** **not required** — it's a cache; losing data costs a reload from the source of truth. (Say it explicitly: it's the most important non-functional requirement here, because it licenses asynchronous replication and memory-only storage.)
- **Consistency:** a read may return a slightly stale value after a failover; no cross-key transactions.

### Estimation

```text
Memory per node: 64 GB RAM; usable for data ≈ 70% → ~45 GB (headroom for fragmentation, replication buffers, forks)
Primaries:       1 TB / 45 GB ≈ 23 → round to 24 (or 32 for growth)
Replicas:        one per primary → 48 nodes total
Throughput:      1.1M ops/s / 24 primaries ≈ 46k ops/s per primary   (an in-memory server does 100k+ simple ops/s)
Network:         1.1M × ~1 KB ≈ 1.1 GB/s ≈ 9 Gbps aggregate → ~370 Mbps per node (fine)
Per-key overhead: tens of bytes per key → at ~1 billion keys, tens of GB of metadata alone
```

What the numbers decide: **memory, not CPU, sets the node count** (it usually does for caches — unlike the rate limiter in Part C); throughput per node is comfortable; and **per-key metadata overhead** matters when values are small — at 1 KB values it's a few percent, at 50-byte values it's most of your memory.

**Interview-grade sentence:** *"I'd first confirm whether they want me to build the cache or use one; for building it, the key requirement is that durability isn't needed — losing data only costs a reload — which licenses memory-only storage and asynchronous replication; at a terabyte with 45 GB usable per node that's about 24 primaries plus replicas, so memory sets the node count and per-node throughput is comfortable."*

---

## Concept 24 — Partitioning and routing

### Why modulo hashing fails

`node = hash(key) mod N` distributes evenly — until N changes. Going from N to N+1 nodes, a key stays put only if `hash mod N = hash mod (N+1)`, which is true for about 1/(N+1) of keys. **Adding the 25th node moves ~96% of keys** — for a cache, a near-total miss storm on the origin.

### Consistent hashing

Map both nodes and keys onto a ring (the hash space); a key belongs to the first node clockwise. Adding a node takes over only the arc before it: about **1/N of keys move**. Two refinements:

- **Virtual nodes:** each physical node appears at 100–200 points on the ring. Without them, arcs are uneven (some nodes get 2–3× the average load); with them, load evens out, and a failed node's keys spread across *all* survivors instead of landing on one neighbour.
- **Weighted nodes:** more virtual nodes for bigger machines.

### Hash slots (Redis Cluster's approach)

A fixed number of slots — **16,384** in Redis Cluster — with `slot = CRC16(key) mod 16384`, and a slot → node table. Rebalancing moves **whole slots** between nodes; the table is small enough to gossip to every node and cache in every client. It's consistent hashing with an explicit, operable indirection layer — easier to reason about and to migrate incrementally.

**Hash tags:** only the part of the key inside `{…}` is hashed, so `{user42}:cart` and `{user42}:profile` land in the same slot and can be used together in a multi-key command or script.

### Other options worth one sentence each

- **Rendezvous (highest-random-weight) hashing:** for each key, pick the node with the highest `hash(key, node)`; minimal movement, no ring, O(N) per lookup — fine for small N.
- **Jump consistent hash:** O(1) memory, very fast, but nodes can only be added/removed at the end — good for numbered shards, not arbitrary membership.

### Routing: who knows where the key lives?

| Model | How | Trade-off |
|---|---|---|
| **Smart client** | Clients hold the slot map, connect directly to the owning node; on `MOVED`/`ASK` redirects they update the map | Lowest latency; clients must be cluster-aware (StackExchange.Redis is) |
| **Proxy** | Clients talk to a proxy tier that routes (Twemproxy, Envoy, a managed endpoint) | Simple clients; an extra hop and a tier to scale |
| **Any-node forwarding** | Any node accepts and forwards | Simple; extra hop for most requests |

### Rebalancing without downtime

Move one slot at a time: mark it *migrating* on the source and *importing* on the target, copy keys, and during the move answer requests for keys already moved with an **`ASK` redirect** (one-off, "try over there") versus **`MOVED`** once the slot has fully moved ("update your map"). Clients that understand both keep working throughout.

**Interview-grade sentence:** *"Modulo hashing moves almost every key when a node is added, which for a cache is a miss storm, so I'd partition with consistent hashing — in practice a fixed slot space like Redis Cluster's 16,384 slots mapped to nodes — so rebalancing moves whole slots incrementally with ASK and MOVED redirects, smart clients route directly to the owner, and hash tags co-locate keys that must be used together."*

---

## Concept 25 — Replication and failover

### Replication model

Each primary has one or more replicas, replicated **asynchronously**: the primary acknowledges a write, then streams it to replicas. Synchronous replication would add a network round trip to every write and couple availability to the slowest replica — a bad trade for data you're allowed to lose.

Consequences you should state:

- **Failover loses the replication lag** — writes acknowledged by the primary but not yet replicated are gone. For a cache, acceptable.
- **Reads from replicas are stale by the lag** — fine for many read-heavy keys, wrong for read-your-writes flows. Default to primary reads; opt into replica reads per use case.
- Optional **`WAIT n timeout`**-style semantics (block until n replicas acknowledge) give bounded durability for specific writes — rarely used in a cache.

### Failure detection and promotion

- Nodes **gossip** heartbeats; a primary is marked failed when a **majority of primaries** agree it's unreachable (prevents one partitioned node from declaring others dead).
- A replica of the failed primary is **promoted**, takes over its slots, and the new map propagates to clients.
- **Split brain:** a primary isolated in a minority partition keeps accepting writes until it notices it has lost the majority — those writes are lost when it rejoins as a replica. Mitigation: the primary stops accepting writes when it can't reach enough peers for a configured time (a node timeout), trading availability on the minority side for consistency.

This is Module 9's quorum reasoning applied to a system that's explicitly allowed to lose data — so the design optimises for **fast, safe promotion**, not for zero loss.

### Zones and regions

- Place each primary and its replica in **different availability zones**; a zone loss promotes replicas elsewhere. (Azure Managed Redis is zone-redundant by default.)
- **Multi-region:** either independent caches per region (each warms from its local source of truth — simplest), or **active geo-replication** where writes in any region converge using conflict-free replicated data types (available in Azure Managed Redis). Choose independent caches unless the cached data is itself expensive to recompute in every region.

**Interview-grade sentence:** *"Because the cache is allowed to lose data, I'd replicate asynchronously — accepting that a failover loses the replication lag and that replica reads are stale — detect failures by majority agreement among primaries to avoid one partitioned node causing promotions, have an isolated primary stop accepting writes after a timeout to limit split-brain loss, and spread each primary and replica across zones."*

---

## Concept 26 — Memory management and eviction

### Eviction policies

When memory is full, something must go. The choice depends on access patterns:

| Policy | Idea | Cost / caveat |
|---|---|---|
| **LRU** | Evict least recently used | Exact LRU needs a linked list + map (Module 36) — two pointers per key and a list update on *every read* |
| **Approximated LRU** | Sample a few keys at random, evict the least recently used of the sample (Redis samples 5 by default, with a pool of good candidates) | Close to exact LRU in practice, almost no memory per key |
| **LFU** | Evict least frequently used | Needs frequency counters that **decay**, or yesterday's popular key lives forever; Redis uses a probabilistic logarithmic counter (8 bits) plus decay |
| **TinyLFU / W-TinyLFU** | Admission filter: only admit a new key if it's estimated (via a count-min sketch) to be more frequent than the victim | Excellent hit ratios under skew; what the Caffeine (Java) cache uses |
| **TTL-based (volatile-*)** | Only evict keys with an expiry | Lets you mix cache keys and must-keep keys — a smell in a pure cache |
| **No eviction** | Reject writes when full | For when the "cache" is really a datastore — then it isn't a cache |

**Scan resistance:** a one-off batch job reading millions of keys once will flush an LRU cache. LFU-style or admission-filtered policies resist this; LRU doesn't. Say which workload you expect.

### Expiration (TTL)

Two complementary mechanisms:

- **Lazy expiry:** on access, if the key's TTL has passed, delete it and report a miss. Free, but expired keys that are never read again stay in memory.
- **Active expiry:** a background loop samples keys that have TTLs and deletes the expired ones, repeating while a high fraction of the sample is expired. Bounded CPU, bounded leftover garbage.

### Memory itself

- **Fragmentation:** variable-size values fragment the allocator; Memcached uses **slab classes** (fixed-size chunks per size class) to avoid it, at the cost of internal waste; Redis uses a general allocator and can defragment actively.
- **Big keys:** a 50 MB value blocks a single-threaded event loop while it's serialised, and is a network hot spot. Cap value size; split big values.
- **Fork for snapshots:** if you snapshot to disk (for warm restarts), copy-on-write during the fork can nearly double memory under heavy writes — hence the ~70% usable figure.

### Threading model

Single-threaded event loops (classic Redis) make every command atomic and simple, and scale by **more shards**; multi-threaded designs (Garnet, Dragonfly, Redis's I/O threads) get more throughput per node on many-core machines. In an interview, single-threaded-per-shard is the easier design to defend; mention the alternative.

**Interview-grade sentence:** *"For eviction I'd use approximated LRU by random sampling — near-exact hit ratios without per-key list pointers — or a decaying LFU if scans are a risk; TTLs expire lazily on access plus an active sampling loop so dead keys don't accumulate; and I'd keep about thirty percent of memory as headroom for fragmentation and snapshot forks, cap value sizes, and scale a single-threaded shard design by adding shards."*

---

## Concept 27 — Hot keys, stampedes and invalidation

These are the client-side problems — the half of the question that applies even when you "just use Redis".

### Hot keys

A single key at 200,000 reads/s saturates the one shard that owns it, no matter how many shards you have.

| Mitigation | How | Trade-off |
|---|---|---|
| **Client-side L1 cache** | Each app instance caches the hot key in memory for seconds | Spreads load across every instance; staleness bounded by L1 TTL |
| **Key replication** | Write `key#0…key#7`; readers pick one at random | 8× write cost; invalidation must hit all copies |
| **Read from replicas** | Route reads for that key to replicas | Replication lag; limited by replica count |
| **Detect and adapt** | Track top keys (e.g. a count-min sketch), promote detected hot keys to L1 automatically | More machinery; the most general answer |

### Stampedes (the thundering herd)

A popular key expires; 5,000 requests miss at once and all hit the database. Mitigations, in order of how often you need them:

1. **Request coalescing (single-flight):** concurrent misses for the same key share one origin call. Within a process this is cheap — `HybridCache` does it for you. **Across instances**, each instance still makes one call; for an expensive origin, add a **distributed lease**: the first miss takes a short lock key (`SET lock:{k} NX PX 5000`), others wait briefly or serve stale.
2. **Serve stale while revalidating:** keep the old value past its soft expiry, refresh in the background, serve the stale copy meanwhile.
3. **Probabilistic early expiration (XFetch):** each reader recomputes slightly *before* expiry with a probability that rises as expiry approaches: recompute if `now − δ·β·ln(rand()) ≥ expiry`, where δ is the recompute time and β ≥ 1 tunes eagerness. Because `ln(rand()) ≤ 0`, the term pushes "now" forward by a random amount — one reader usually refreshes early, the rest never miss.
4. **TTL jitter:** add ±10–20% random spread to TTLs so keys written together don't expire together (cache avalanche).

### Penetration and negative caching

Requests for keys that **don't exist** in the source (scanners, bugs) always miss. Cache the absence with a short TTL (Part B did exactly this), and/or use a Bloom filter in front of the origin.

### Invalidation: the actual hard part

| Strategy | How | Consistency |
|---|---|---|
| **TTL only** | Let entries expire | Stale for up to the TTL; simplest; fine for many reads |
| **Delete on write** (cache-aside) | Write the database, then **delete** the cache key | Short race windows (below); the standard default |
| **Write-through / update on write** | Write the database, then **set** the new value | Concurrent writers can leave the *older* value in cache — prefer delete |
| **Versioned keys** | Key includes a version (`product:42:v17`); bump the version on change | No invalidation race; old versions age out; needs a version lookup |
| **Change-feed driven** | A consumer of the database's change feed / CDC deletes keys | Decoupled from the writer; small lag |
| **Broadcast for L1** | Publish invalidations (Redis pub/sub or a backplane) so every instance drops its L1 copy | Needed whenever there's an in-process tier |

**The classic race in delete-on-write:** reader A misses and reads the *old* value from the database; writer B updates the database and deletes the key; reader A then writes its old value into the cache — stale until the TTL. Mitigations: short TTLs as a backstop; **leases** (the Facebook memcache paper's approach — a miss hands out a lease token, and a delete invalidates outstanding leases so the late `SET` is rejected); or versioned keys.

**Interview-grade sentence:** *"On the client side I'd handle hot keys with an in-process L1 tier and replica reads, stampedes with request coalescing — which `HybridCache` gives per process, plus a short distributed lease for expensive origins — stale-while-revalidate and TTL jitter; I'd negatively cache misses; and I'd invalidate by deleting on write with a TTL backstop, broadcast invalidations to every L1, and use leases or versioned keys where the read-after-delete race actually matters."*

---

## Concept 28 — .NET implementation notes

### Using a cache from .NET: `HybridCache` first

```csharp
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = redisConnectionString);   // registers the L2
builder.Services.AddHybridCache(o =>
{
    o.MaximumPayloadBytes = 1024 * 1024;                     // reject oversized values
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),                // L2
        LocalCacheExpiration = TimeSpan.FromMinutes(1),       // L1 — the staleness bound across instances
    };
});

// Usage: coalesced per process, L1 → L2 → factory
var product = await cache.GetOrCreateAsync(
    $"product:{id}",
    async ct => await db.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id, ct),
    tags: [$"category:{categoryId}"],
    cancellationToken: ct);

// Invalidation after a write:
await cache.RemoveAsync($"product:{id}", ct);
await cache.RemoveByTagAsync($"category:{categoryId}", ct);
```

Precise claims to make about it:

- **Stampede protection is per process.** Concurrent `GetOrCreateAsync` calls for one key in one instance share one factory call; 40 instances can still make 40 calls on a cold key.
- **`RemoveAsync` clears L2 and this instance's L1 — not other instances' L1.** There's no built-in backplane, so other instances serve their L1 copy until `LocalCacheExpiration`. Bound it with a short L1 lifetime, or use **FusionCache** (which can act as the `HybridCache` implementation and adds a Redis backplane for cross-instance invalidation and fail-safe stale serving).
- **Serialization:** values go through a serializer for L2 (System.Text.Json by default); keep cached types simple DTOs, not EF entities with navigation graphs.
- **`IDistributedCache`** is the older, lower-level abstraction (bytes in, bytes out, no coalescing); `HybridCache` sits on top of it.

### StackExchange.Redis essentials

- **One `ConnectionMultiplexer` per application** (singleton): it multiplexes all commands over a few connections and is thread-safe. Creating one per request is the Redis equivalent of `new HttpClient()` per call.
- **Async and pipelined by default:** concurrent `await`s are pipelined over the shared connection; avoid synchronous calls on hot paths (ThreadPool starvation, Module 15).
- **Cluster-aware:** follows `MOVED`/`ASK`; works with Azure Managed Redis's clustered endpoints without special configuration. Multi-key commands need keys in one slot — use hash tags.
- **Timeouts:** commands don't take `CancellationToken`; configure `SyncTimeout`/`AsyncTimeout` and use `WaitAsync` for per-call budgets (Concept 14).
- **Big values and slow commands** (`KEYS *`, large `SMEMBERS`) block a shard — use `SCAN`, cap sizes.

### Choosing the server in 2026

| Option | When | Note |
|---|---|---|
| **Azure Managed Redis** | Default managed cache on Azure | Redis Enterprise software, clustered by default, zone-redundant by default, Entra ID auth, active geo-replication available. Successor to Azure Cache for Redis (Basic/Standard/Premium retire September 30, 2028; Enterprise tiers March 31, 2027) |
| **Valkey** (self-hosted, or managed on other clouds) | Permissive licence (BSD-3) matters; multi-cloud | Wire-compatible fork of Redis 7.2.4 |
| **Redis 8 Open Source** (self-hosted) | You want upstream Redis features | Tri-licensed RSALv2 / SSPLv1 / AGPLv3 — check the licence fits how you distribute it |
| **Garnet** | Very high throughput per node on many-core VMs; a .NET-native shop willing to run it | Microsoft Research, MIT licence, RESP-compatible; can also be embedded in a .NET process |
| **In-process only** (`MemoryCache` via `HybridCache` L1) | Single instance, or data that's cheap to have per instance | No network hop; no shared invalidation |

### If asked to sketch the partitioning in C#

A consistent-hash ring with virtual nodes — the kind of 30 lines an interviewer may ask for:

```csharp
using System.IO.Hashing;     // System.IO.Hashing package: XxHash64
using System.Text;

public sealed class HashRing<TNode> where TNode : notnull
{
    private readonly ulong[] _points;          // sorted ring positions
    private readonly TNode[] _owners;          // _owners[i] owns (_points[i-1], _points[i]]

    public HashRing(IReadOnlyCollection<TNode> nodes, Func<TNode, string> nodeId, int virtualNodes = 160)
    {
        ArgumentOutOfRangeException.ThrowIfZero(nodes.Count);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(virtualNodes);

        var entries = new List<(ulong Point, TNode Node)>(nodes.Count * virtualNodes);
        foreach (var node in nodes)
            for (int v = 0; v < virtualNodes; v++)
                entries.Add((Hash($"{nodeId(node)}#{v}"), node));

        entries.Sort((a, b) => a.Point.CompareTo(b.Point));
        _points = entries.ConvertAll(e => e.Point).ToArray();
        _owners = entries.ConvertAll(e => e.Node).ToArray();
    }

    public TNode Locate(string key)
    {
        int i = Array.BinarySearch(_points, Hash(key));
        if (i < 0) i = ~i;                        // index of the first point >= hash
        return _owners[i == _points.Length ? 0 : i];   // wrap around the ring
    }

    private static ulong Hash(string value) => XxHash64.HashToUInt64(Encoding.UTF8.GetBytes(value));
}
```

What to say: the ring is **immutable** — on membership change, build a new ring and swap the reference (`Volatile.Write` or an `ImmutableInterlocked`-style swap), so readers never see a half-built ring; `Array.BinarySearch`'s complement gives the ceiling in O(log n); 160 virtual nodes per node keeps load within a few percent of even; xxHash is fast and well distributed (don't use `string.GetHashCode()` — it's randomised per process in .NET, so different instances would disagree about ownership).

**Interview-grade sentence:** *"From .NET I'd use `HybridCache` over Azure Managed Redis — knowing its stampede protection and `RemoveAsync` are per process, so cross-instance staleness is bounded by the L1 lifetime unless I add a backplane like FusionCache's — with one singleton `ConnectionMultiplexer`, async calls, hash tags for multi-key operations and per-call time budgets; and if I had to build the partitioning myself, an immutable consistent-hash ring with virtual nodes and a stable hash like xxHash, never `string.GetHashCode`, which is randomised per process."*

---
# Part F — Order and payment system

> *"Design the checkout and payment flow for an e-commerce site."* — or *"Design a payment system"*, *"Design an order management system"*.

The other four problems are about throughput; this one is about **correctness**. The QPS is modest, the stakes are high, and the interviewer is listening for three things: **no double charges, no lost orders, and a precise answer to "the payment provider timed out — did the customer pay?"**

## Concept 29 — Requirements and estimation

### Clarify which system

"Payment system" can mean a **merchant's** checkout that calls a payment service provider (PSP) like Stripe or Adyen — the common case — or **the PSP itself** (card networks, acquirers, settlement). Assume the merchant side unless told otherwise; say so.

### Functional requirements

1. **Place an order** from a cart: price it, reserve inventory, take payment, confirm.
2. **Payment** via a PSP: authorise at checkout, **capture** at shipment (or immediately for digital goods); support asynchronous flows (3-D Secure / strong customer authentication that needs a customer action).
3. **Cancel** before shipment (release inventory, void the authorisation) and **refund** after capture (full or partial).
4. **Order history and status** for customers and support.
5. **Events** to downstream systems: fulfilment, notifications (Part D), analytics.

### Non-functional requirements

| Requirement | Target | Consequence |
|---|---|---|
| **No double charge** | Zero tolerance | Idempotency at every hop (Concept 31) |
| **No lost or orphaned orders** | Every paid order is fulfilled or refunded | Durable workflow + reconciliation (Concepts 32, 34) |
| **No overselling** | Inventory never below zero (or bounded oversell with a policy) | Atomic conditional decrements (Concept 33) |
| **Auditability** | Every money movement traceable, immutable history | Ledger + event history (Concept 30) |
| **Availability of checkout** | 99.95%+ — downtime is lost revenue | Degrade non-essential steps; queue what can wait |
| **Latency** | Checkout response in 2–3 s, dominated by the PSP call (often 1–2 s) | Don't add avoidable synchronous hops |
| **Compliance** | Minimise PCI DSS scope | Never touch card numbers (Concept 34) |

### Estimation — small numbers, sharp peaks

```text
Orders: 1M/day ≈ 12/s average; Black Friday peak ×20 ≈ 230/s
Payment calls: ~2 per order (authorise + capture) + refunds ≈ 25/s average, ~500/s peak
Ledger postings: ~4–6 per order → 5M/day
Flash sale: one SKU, 100k buyers in 60 s ≈ 1,700 reservation attempts/s on ONE row
```

What the numbers decide: **throughput is not the problem** — a single well-indexed relational database handles this with headroom, so you can choose **ACID transactions** where they simplify correctness. The scale problem is **contention on hot inventory rows** during flash sales (Concept 33). Say that, and resist designing sharded microservices for 12 orders a second.

**Interview-grade sentence:** *"I'd assume the merchant side calling a PSP; at a million orders a day it's only about 12 a second, peaking in the low hundreds, so a relational database with real transactions is the right default — the hard requirements are no double charges, no lost orders and no overselling, and the only real scale problem is contention on hot inventory rows during a flash sale."*

---

## Concept 30 — The data model

### The order as an aggregate with a state machine

```text
                ┌──────────────► Cancelled ◄───────────┐
                │                    ▲                 │
 Created ──► InventoryReserved ──► PaymentPending ──► PaymentAuthorized ──► Confirmed ──► Shipped (captured) ──► Completed
                │                    │                                                       │
                ▼                    ▼                                                       ▼
          ReservationFailed    PaymentFailed / RequiresAction(3DS)                    Refunded / PartiallyRefunded
```

Say three things about it:

- **Transitions are explicit methods** on the aggregate (`MarkAuthorized(paymentId)`, `Cancel(reason)`), which **reject invalid transitions** — a webhook that arrives late or twice can't move a cancelled order to "paid" (Module 22).
- **Every transition is persisted with the order's version** (optimistic concurrency, Concept 33) and emits a domain event through the **outbox** (Module 11).
- **Status history is kept** (an append-only table or event-sourced stream — Module 24): support needs to see *how* an order got where it is.

### Money

- Store amounts as **integer minor units** (`long` cents) **plus an ISO 4217 currency code** — or `decimal` with an explicit currency; **never `double`**. Mind currencies with zero or three decimal places (JPY has none; some have three).
- **Never mix currencies** in arithmetic; a `Money` value type that throws on mismatched currencies makes this structural (Module 36's ledger example).
- Prices on an order are **copied at order time** (unit price, tax, discount lines) — never recomputed from the current catalogue.

### The ledger: double-entry, append-only

For anything beyond the simplest shop — marketplaces, wallets, refunds, partial captures — record money movement in a **double-entry ledger**:

```text
JournalEntry(id, occurredAt, type, orderId, idempotencyKey)
Posting(entryId, account, amountMinor (signed), currency)

Invariant: for every entry, the sum of postings per currency = 0
e.g. capture of €50.00:   customer_receivable −5000   |   merchant_revenue +4850   |   psp_fees +150
```

- **Immutable:** corrections are new entries (a reversal), never updates — the audit trail is the data.
- **Balances are derived** (sum of postings), optionally cached in a balance table updated in the same transaction.
- The ledger is what **reconciliation** compares against the PSP's settlement reports (Concept 34).

### Supporting tables

| Table | Purpose |
|---|---|
| `IdempotencyKeys(key, requestHash, status, responseCode, responseBody, createdAt)` | Replay-safe API (Concept 31) |
| `OutboxMessages(id, type, payload, occurredAt, dispatchedAt)` | Atomic event publication (Module 11) |
| `InboxMessages(messageId, processedAt)` | Consumer-side dedupe for events and webhooks |
| `Payments(id, orderId, pspPaymentId, status, amount, currency, idempotencyKey)` | One row per payment attempt, mirroring the PSP's object |
| `InventoryItems(sku, available, reserved, version)` and `Reservations(id, orderId, sku, qty, expiresAt)` | Concept 33 |

**Interview-grade sentence:** *"I model the order as an aggregate with an explicit state machine whose transition methods reject invalid moves — so late or duplicate webhooks can't corrupt it — persisted with a version for optimistic concurrency and publishing events through an outbox; money is integer minor units plus a currency, prices are copied at order time, and money movement goes into an append-only double-entry ledger whose postings sum to zero per entry, which is what we reconcile against the PSP."*

---

## Concept 31 — Idempotency end to end

The single most important deep dive. Duplicates can enter at four hops; each needs its own key.

```text
Customer double-clicks / app retries ──► [1] Checkout API ──► [2] PSP call ──► PSP
                                                   ▲                              │
                                                   │                       [3] webhooks (redelivered)
                                          [4] internal events (at-least-once) ◄──┘
```

### Hop 1 — the client → your API

The client generates an **idempotency key per checkout attempt** (a UUID created when the checkout page loads, reused on every retry of *that* attempt) and sends it as a header. The server:

1. **Inserts** `(key, hash(request body), status = InProgress)` — atomically; a unique constraint is the lock.
2. On conflict, reads the existing row:
   - **different request hash** → `422`: the key is being reused for a different request (a client bug);
   - **InProgress** → `409 Conflict` (or wait briefly and re-check): the first attempt is still running;
   - **Completed** → **replay the stored status code and body** — the client sees exactly what the first attempt returned.
3. Executes, then stores the response with `status = Completed` — **ideally in the same database transaction as the business change**, so "the order exists" and "the key says completed" can't disagree.
4. Expires keys after a retention period (24 hours to a few days).

This is the model Stripe documents for its own API: the first result for a key is saved — **including 500 errors** — so retries return the same outcome; reusing a key with different parameters is an error; and keys may be pruned once they're at least 24 hours old.

### Hop 2 — your service → the PSP

Every mutating PSP call carries **your** idempotency key, derived deterministically from your own identifiers: `order-{orderId}-authorize`, `order-{orderId}-capture-1`, `refund-{refundId}`. If your call times out, you **retry with the same key**, and the PSP returns the original result instead of charging again.

### The unknown-outcome problem

> *"The PSP call timed out. Did the customer pay?"*

**You don't know** — and the senior answer says exactly that, then resolves it:

1. **Never treat a timeout as a failure** — that's how you get "payment failed" screens for customers who were charged.
2. Record the payment as **Unknown/Pending**, and **retry with the same idempotency key** (safe by construction), or **query the PSP** for the payment by your reference.
3. If still unresolved, a **sweeper** (Concept 34) retries the lookup on a schedule, and the webhook (hop 3) usually arrives with the answer.
4. The customer sees "processing" — not "failed" — and gets the final status by notification.

### Hop 3 — PSP webhooks → your service

Webhooks are at-least-once and **unordered**: `payment_intent.succeeded` can arrive twice, or before your own synchronous response. So:

- **Verify the signature** (it's an unauthenticated public endpoint otherwise).
- **Dedupe by the PSP's event ID** in the inbox table.
- **Return 2xx quickly** and process asynchronously (persist the event, enqueue).
- **Apply through the state machine** — an event that doesn't match a valid transition (e.g. "succeeded" for an order already cancelled) becomes a **case to handle** (refund or void), not a silent overwrite.
- Prefer **fetching the current object** from the PSP when processing, rather than trusting the ordering of events.

### Hop 4 — internal events → consumers

Outbox dispatch and broker delivery are at-least-once, so every consumer is idempotent: an **inbox table** keyed by message ID inside the consumer's own transaction, or naturally idempotent operations ("set status to Shipped if Confirmed").

**Interview-grade sentence:** *"I make every hop idempotent with its own key: a client-generated key per checkout attempt whose stored response is replayed on retries and committed in the same transaction as the order; deterministic keys from our own IDs on every PSP call, so a timeout is retried with the same key rather than treated as a failure; webhooks verified, deduplicated by event ID and applied through the state machine; and inbox tables in every event consumer."*

---

## Concept 32 — The checkout workflow

### The steps, and why not one transaction

```text
1. Create order (priced, status Created)
2. Reserve inventory         ── compensate: release reservation
3. Authorise payment (PSP)   ── compensate: void authorisation
4. Confirm order             ── emit OrderConfirmed → fulfilment, notification
   … later …
5. Capture at shipment       ── compensate: refund
```

Steps 2–5 touch **different systems** (inventory, the PSP, fulfilment) — and the PSP can't participate in your database transaction. **Two-phase commit** isn't available across a third-party HTTP API, and even internally it couples availability and holds locks across slow calls (Module 12). So it's a **saga**: a sequence of local transactions, each with a **compensating action** that semantically undoes it.

### Why authorise-then-capture

Authorising holds the funds without moving them; capturing moves them. Splitting them means a cancellation before shipment is a **void** (no money moved, no fees, nothing on the customer's statement after the hold expires) instead of a **refund**. Authorisations **expire** (commonly around 7 days for cards, varying by network and method), so long fulfilment delays need re-authorisation — a nice detail to mention.

### Orchestration or choreography?

| | **Orchestration** — one coordinator drives the steps | **Choreography** — each service reacts to the previous event |
|---|---|---|
| Visibility | The whole flow is in one place; easy to answer "where is order 123?" | Spread across services; needs tracing to reconstruct |
| Coupling | Coordinator knows every step | Services know only events |
| Change | Change the orchestrator | Change several services, carefully |
| Failure handling | Compensations explicit, in order | Each service must know when and how to compensate |
| Fits | **Checkout** — a short, business-critical, ordered flow with compensations | Loose, fan-out reactions: "order confirmed" → loyalty points, analytics, email |

**Default: orchestrate checkout; choreograph the reactions to `OrderConfirmed`.** That's the combination most production systems converge on, and it's defensible in one sentence each.

### How to implement the orchestrator

| Option | What you get | Cost |
|---|---|---|
| **Hand-rolled state machine** on the order aggregate + outbox + Service Bus | Full control; no new infrastructure; state lives with the order | You build timers, retries, timeouts and the "stuck" sweeper yourself |
| **Durable execution engine** — Durable Functions or the Durable Task SDKs on the Durable Task Scheduler; Temporal; Dapr Workflow | Workflow-as-code with durable timers, retries and history; replay after crashes | A new runtime to learn; determinism rules; workflow versioning on change |
| **Messaging framework sagas** — MassTransit state machines, NServiceBus sagas, Wolverine | Sagas integrated with the messaging layer and outbox | Library licensing (MassTransit v9 and NServiceBus are commercial in 2026) |

At 12 orders a second with a short flow, a **hand-rolled state machine with an outbox** is perfectly defensible and keeps state where support can see it; a **durable execution engine** wins when the flow has long waits (3-D Secure, manual review, "capture when shipped" days later) and many timeouts. Say which you'd pick and why — either is senior if justified.

### Timeouts are part of the workflow

- **Reservation TTL:** a reservation expires (say 10–15 minutes) if payment doesn't complete — otherwise abandoned checkouts lock stock forever.
- **Payment action timeout:** a 3-D Secure challenge that isn't completed in N minutes cancels the order and releases stock.
- **Compensation can fail too:** a void call that fails is retried (idempotently) until it succeeds, and alerts if it can't — a compensation is a must-eventually-succeed step, not best-effort.

**Interview-grade sentence:** *"Checkout spans inventory, a third-party PSP and fulfilment, so it can't be one transaction — it's a saga of local steps with compensations: reserve inventory, authorise, confirm, and capture at shipment, so a cancellation is a void rather than a refund; I'd orchestrate checkout — either a state machine on the order with an outbox, or a durable workflow engine when there are long waits like 3-D Secure — and choreograph the downstream reactions to `OrderConfirmed`, with reservation TTLs and compensations that retry until they succeed."*

---

## Concept 33 — Concurrency and inventory

### Optimistic concurrency on the order

Two things can touch an order at once — the orchestrator and a webhook, or a customer cancel and a payment success. Each update reads the order with its **version** and writes conditionally on it (`WHERE id = @id AND version = @version`); the loser gets a concurrency conflict, **reloads, and re-applies through the state machine** — which may now reject the transition (you can't cancel a shipped order). That's how the state machine and optimistic concurrency together make concurrent events safe.

### Preventing overselling

**Wrong:** read `available`, check in application code, write `available − qty` — two buyers both read 1 and both succeed.

**Right:** a **single conditional update**, so the check and the decrement are atomic in the database:

```sql
UPDATE InventoryItems
SET    Available = Available - @qty, Reserved = Reserved + @qty
WHERE  Sku = @sku AND Available >= @qty;
-- 1 row affected → reserved;  0 rows → out of stock
```

Plus a `Reservations` row with an `ExpiresAt`, and a sweeper that releases expired reservations (moves `Reserved` back to `Available`).

### Flash-sale contention

1,700 attempts per second on **one row** serialise on that row's lock; latency climbs and timeouts cascade. Options, in order of escalation:

1. **Keep the transaction tiny** — just the conditional update; no PSP call or other I/O inside it.
2. **Split the stock into buckets** — `(sku, bucket 0..15)` rows each holding a share of stock; a request picks a random bucket and falls back to others. Spreads lock contention 16 ways; the total is exact.
3. **Admission control in front** — a queue or token gate (Part C) admitting buyers at the rate inventory can process, with a "you're in the queue" experience; or **pre-allocate tokens** equal to the stock count in Redis (`DECR` with a floor) as a fast gate, with the database still the source of truth.
4. **Accept bounded oversell** with a business policy (backorder or apologise and refund) — sometimes the right answer for low-cost items, and worth saying as an option.

### Isolation levels, briefly

The conditional update is correct at **Read Committed** — the `WHERE` re-evaluates under the row lock. For multi-row invariants (a bundle of three SKUs reserved together), either do it in one transaction with all conditional updates and roll back on any zero-row result, or raise isolation for that transaction (Module 12's anomaly catalogue).

**Interview-grade sentence:** *"Concurrent events on an order are handled by optimistic concurrency plus the state machine — the loser reloads and re-applies, and the state machine may reject the move; inventory is reserved with one conditional update, `available >= qty` in the WHERE clause, so check and decrement are atomic, with expiring reservations; and for flash sales I'd keep that transaction tiny, split hot stock into buckets, and put admission control in front before considering a policy of bounded oversell."*

---

## Concept 34 — Reconciliation, audit and failure handling

### Things will get stuck — design the repair path

| Stuck state | Detected by | Repair |
|---|---|---|
| Payment `Unknown` after a timeout | Sweeper: payments Unknown > 2 min | Query the PSP by your reference; apply the result through the state machine |
| Order `PaymentPending` > N min (abandoned 3DS) | Sweeper | Cancel, release inventory, void if authorised |
| Reservation expired | Sweeper | Release stock |
| Compensation failing repeatedly | Retry count / DLQ | Alert; manual runbook |
| Webhook for an unknown or cancelled order | State machine rejection | Investigate; usually void/refund |

### Reconciliation: the safety net that catches what idempotency missed

Daily (and intraday for large merchants), compare **three views of money**:

```text
Your ledger  ⇄  Your payment records  ⇄  PSP settlement / payout reports
```

Every mismatch is a case: a capture you recorded but the PSP didn't settle, a refund the PSP processed that you didn't record, fees that differ. Reconciliation is how payment teams find the bugs their idempotency missed — mentioning it unprompted is a strong signal, because it shows you assume your own system will be wrong sometimes.

### Audit

- The ledger is append-only; order status history is retained; every money-moving action records **who/what** initiated it (customer, support agent, system job) and **why**.
- Support actions (manual refunds) go through the same API and idempotency path as everything else — no direct database edits.

### Security and compliance

- **PCI DSS scope:** never let card numbers touch your servers. Use the PSP's **hosted fields / payment elements / redirect**, which tokenise the card in the browser; your backend handles only tokens and payment IDs. This collapses your PCI scope to the simplest self-assessment level and removes a whole class of breach.
- **Strong customer authentication** (3-D Secure, required in the EU/UK): the authorisation can return "requires action"; the flow becomes asynchronous — a timeout and a webhook are part of the normal path.
- **Fraud checks** before authorisation (the PSP's risk scoring, plus your own rules), with a manual-review state in the order machine for high-risk orders.
- Webhook signature verification, secrets in Key Vault, managed identity for Azure resources (Module 29).

### Availability under PSP failure

If the PSP is down, you can't take payment — but you can **degrade**: queue orders as "payment pending" for a short window (for low-risk customers, authorise when the PSP recovers), fail over to a **secondary PSP** for new checkouts (a big project — tokens aren't portable between PSPs unless you use network tokens or a vault), or show a clear "try again shortly". Say which the business would choose; don't promise a seamless failover you haven't designed.

**Interview-grade sentence:** *"I design the repair path as carefully as the happy path — sweepers that resolve unknown payments by querying the PSP, cancel abandoned 3-D Secure checkouts and release expired reservations, and a daily reconciliation of our ledger and payment records against the PSP's settlement reports, which is how payment teams catch what idempotency missed; and I keep card data out of our systems entirely with the PSP's hosted fields, so PCI scope stays minimal."*

---

## Concept 35 — .NET implementation notes

### The order aggregate and optimistic concurrency in EF Core

```csharp
public sealed class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public long TotalMinor { get; private set; }
    public string Currency { get; private set; } = "";
    public string? PspPaymentId { get; private set; }
    public byte[] Version { get; private set; } = [];      // SQL Server rowversion

    private readonly List<object> _events = [];
    public IReadOnlyList<object> DomainEvents => _events;

    public void MarkAuthorized(string pspPaymentId)
    {
        if (Status is not OrderStatus.PaymentPending)
            throw new InvalidOrderTransitionException(Id, Status, OrderStatus.PaymentAuthorized);
        Status = OrderStatus.PaymentAuthorized;
        PspPaymentId = pspPaymentId;
        _events.Add(new PaymentAuthorized(Id, pspPaymentId, TotalMinor, Currency));
    }
    // Cancel, Confirm, MarkShipped… each validates the current state the same way
}

// Configuration
modelBuilder.Entity<Order>().Property(o => o.Version).IsRowVersion();
```

- `IsRowVersion()` (or `[Timestamp]`) makes EF Core add `WHERE Version = @original` to updates and throw **`DbUpdateConcurrencyException`** when zero rows match — the handler reloads and re-applies (Module 19). On PostgreSQL, use the `xmin` system column; on Cosmos DB, the item **ETag** with `IfMatchEtag`.
- Domain events are turned into **outbox rows in the same `SaveChanges`** — typically in a `SaveChangesInterceptor` or a unit-of-work wrapper — so state and events commit atomically.

### The inventory reservation with `ExecuteUpdateAsync`

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);

int reserved = await db.InventoryItems
    .Where(i => i.Sku == sku && i.Available >= qty)
    .ExecuteUpdateAsync(s => s
        .SetProperty(i => i.Available, i => i.Available - qty)
        .SetProperty(i => i.Reserved, i => i.Reserved + qty), ct);

if (reserved == 0)
{
    await tx.RollbackAsync(ct);
    return ReservationResult.OutOfStock;
}

db.Reservations.Add(new Reservation(orderId, sku, qty, expiresAt: time.GetUtcNow().AddMinutes(15)));
order.MarkInventoryReserved();
await db.SaveChangesAsync(ct);          // reservation row + order transition + outbox rows
await tx.CommitAsync(ct);
return ReservationResult.Reserved;
```

The point to make: **`ExecuteUpdateAsync` runs immediately as its own SQL statement and bypasses change tracking** — it is *not* deferred to `SaveChanges`. Wrapping both in one explicit transaction makes the conditional decrement, the reservation row and the order transition commit or roll back together.

### The idempotent checkout endpoint (outline)

```csharp
app.MapPost("/api/checkout", async (CheckoutRequest request, [FromHeader(Name = "Idempotency-Key")] string? key,
                                   CheckoutService checkout, CancellationToken ct) =>
{
    if (string.IsNullOrWhiteSpace(key) || key.Length > 255)
        return Results.BadRequest("An Idempotency-Key header is required.");

    var outcome = await checkout.ExecuteAsync(key, request, ct);
    return outcome switch
    {
        IdempotentOutcome.Replayed r     => Results.Content(r.Body, "application/json", statusCode: r.StatusCode),
        IdempotentOutcome.InProgress     => Results.Conflict(new { error = "A request with this key is in progress." }),
        IdempotentOutcome.KeyMismatch    => Results.UnprocessableEntity(new { error = "Key reused with a different request." }),
        IdempotentOutcome.Completed c    => Results.Json(c.Response, statusCode: c.StatusCode),
        _ => throw new UnreachableException(),
    };
});
```

Inside `ExecuteAsync`: insert the key row (unique constraint), branch on conflict as in Concept 31, run the business operation, and write the stored response **in the same transaction** as the order. For cross-cutting reuse, the same logic fits an **endpoint filter** applied to every mutating endpoint.

### Calling the PSP safely (Stripe.net shown)

```csharp
var intent = await _paymentIntents.CreateAsync(
    new PaymentIntentCreateOptions
    {
        Amount = order.TotalMinor,
        Currency = order.Currency.ToLowerInvariant(),
        CaptureMethod = "manual",                         // authorise now, capture at shipment
        Metadata = new Dictionary<string, string> { ["orderId"] = order.Id.ToString() },
    },
    new RequestOptions { IdempotencyKey = $"order-{order.Id}-authorize" },
    ct);
```

- The idempotency key is **deterministic from your order ID**, so every retry — by Polly, by the orchestrator, by a sweeper — is the same request to the PSP.
- The HTTP client behind it gets a resilience pipeline with a **timeout and retries only on transient failures** (Module 25); the retry is safe *because* of the key. Without the key, retrying a payment call is the classic double-charge bug.
- A timeout leaves the payment `Unknown` — the orchestrator or sweeper resolves it (Concept 31).

### The orchestrator as a durable workflow (Durable Functions, isolated worker)

```csharp
[Function(nameof(CheckoutOrchestrator))]
public static async Task<CheckoutResult> CheckoutOrchestrator([OrchestrationTrigger] TaskOrchestrationContext context)
{
    var request = context.GetInput<CheckoutInput>()!;
    var retry = TaskOptions.FromRetryPolicy(new RetryPolicy(
        maxNumberOfAttempts: 5, firstRetryInterval: TimeSpan.FromSeconds(2), backoffCoefficient: 2));

    var reservation = await context.CallActivityAsync<ReservationResult>(nameof(ReserveInventory), request, retry);
    if (reservation is not ReservationResult.Reserved)
        return CheckoutResult.OutOfStock(request.OrderId);

    try
    {
        var auth = await context.CallActivityAsync<AuthorizationResult>(nameof(AuthorizePayment), request, retry);
        if (!auth.Approved)
        {
            await context.CallActivityAsync(nameof(ReleaseInventory), request.OrderId, retry);
            return CheckoutResult.PaymentDeclined(request.OrderId);
        }

        await context.CallActivityAsync(nameof(ConfirmOrder), new ConfirmInput(request.OrderId, auth.PspPaymentId), retry);
        return CheckoutResult.Confirmed(request.OrderId);
    }
    catch (TaskFailedException)
    {
        // Compensate in reverse order; each compensation is idempotent and retried.
        await context.CallActivityAsync(nameof(VoidAuthorizationIfAny), request.OrderId, retry);
        await context.CallActivityAsync(nameof(ReleaseInventory), request.OrderId, retry);
        await context.CallActivityAsync(nameof(MarkOrderFailed), request.OrderId, retry);
        return CheckoutResult.Failed(request.OrderId);
    }
}
```

What to say:

- **Orchestrators replay** from their history after every await, so the code must be **deterministic**: no `DateTime.UtcNow` (use `context.CurrentUtcDateTime`), no `Guid.NewGuid()` (use `context.NewGuid()`), no direct I/O — all I/O happens in activities.
- **Activities are at-least-once**, so every activity is idempotent — `AuthorizePayment` uses the deterministic PSP key, `ReleaseInventory` is a no-op if already released.
- **Durable timers** (`context.CreateTimer`) model the reservation TTL and the 3-D Secure wait; **external events** (`context.WaitForExternalEvent`) model the webhook arriving.
- On Azure, back it with the **Durable Task Scheduler** (GA, Dedicated SKU); outside Functions, the **Durable Task SDKs** run the same model on Container Apps or AKS.
- **Versioning:** changing a workflow while instances are in flight needs care (new orchestration name or version-aware code) — a classic follow-up question.

### The outbox dispatcher

A `BackgroundService` that polls unsent outbox rows in order, publishes to Service Bus with `MessageId = outbox row id` (so Service Bus duplicate detection absorbs re-publishes after a crash), and marks them dispatched. Consumers still keep an inbox table, because duplicate detection only covers its window and only the producer side (Modules 11, 27). Libraries that provide this — Wolverine (MIT), NServiceBus and MassTransit v9 (commercial) — are a build-vs-buy decision (Module 33).

### Webhook endpoint (outline)

```csharp
app.MapPost("/webhooks/psp", async (HttpRequest http, WebhookInbox inbox, CancellationToken ct) =>
{
    string payload = await new StreamReader(http.Body).ReadToEndAsync(ct);
    Event evt;
    try
    {
        evt = EventUtility.ConstructEvent(payload, http.Headers["Stripe-Signature"], webhookSecret);
    }
    catch (StripeException)
    {
        return Results.BadRequest();                                      // bad signature: reject, don't store
    }

    await inbox.TryStoreAsync(evt.Id, evt.Type, payload, ct);   // unique on event id; a duplicate is a no-op
    return Results.Ok();                                        // acknowledge fast; process from the inbox
}).AllowAnonymous();
```

(Processing happens asynchronously from the inbox table; the endpoint only verifies, stores and acknowledges.)

**Interview-grade sentence:** *"In .NET I'd model the order as an EF Core aggregate with a rowversion concurrency token and domain events written to an outbox in the same `SaveChanges`; reserve stock with a conditional `ExecuteUpdateAsync` inside the same explicit transaction, since it bypasses change tracking; call the PSP with deterministic idempotency keys so Polly retries are safe; and run the saga either as a state machine with an outbox dispatcher or as a Durable Functions orchestrator on the Durable Task Scheduler — deterministic orchestrator code, idempotent activities, durable timers for the reservation TTL and external events for webhooks."*

---
# Part G — Across the five problems

## Concept 36 — Comparing the five

Laid side by side, the problems differ along a few axes — and those axes are what you should be able to state in the first two minutes of any of them.

| | URL shortener | Rate limiter | Notification system | Distributed cache | Order/payment |
|---|---|---|---|---|---|
| **Dominant requirement** | Read latency under skew | Hot-path latency + correct shared counting | Throughput isolation + delivery guarantees | Memory, partitioning, availability | Correctness, auditability |
| **Read/write shape** | 100:1 reads | ~1:1 (every check writes) | Write-heavy pipeline | Read-heavy, some writes | Low volume, high stakes |
| **Core data structure** | KV by code | Counter / bucket per key | Queue per lane + delivery state machine | Hash ring / slots + eviction structure | Aggregate state machine + ledger |
| **Consistency stance** | Stale ≤ L1 TTL is fine | Approximate is fine (except quotas) | Duplicates possible; choose per type | May lose data; stale after failover | Strong where money moves |
| **Hardest deep dive** | Code generation; hot links | Atomic distributed check; failure policy | Dedupe windows; campaign vs OTP isolation | Rebalancing; stampedes; invalidation | Idempotency; unknown outcomes; compensation |
| **Typical follow-up** | "Analytics in real time?" / "custom domains?" | "Multi-region?" / "Redis is down?" | "Exactly once?" / "50M in an hour?" | "Add a node live?" / "hot key?" | "PSP timed out?" / "flash sale?" |
| **.NET probe** | `HybridCache` semantics; Cosmos point reads; 302 | Built-in limiters are per process; Lua via SE.Redis | `ServiceBusProcessor`, dedupe, KEDA | Multiplexer, `HybridCache` invalidation, hashing | EF Core concurrency, outbox, durable orchestrators |
| **Biggest trap** | Hash-and-truncate without a collision plan | Distributed limit built from in-process limiters | One queue for everything | Modulo hashing; "the cache is consistent" | Treating a timeout as a failure; retries without keys |

### What they share

- **An idempotency story** (shortener creation, notification deliveries, payments) — or a deliberate decision that one isn't needed (rate-limit checks, cache sets).
- **A hot-key story** (viral link, noisy tenant, celebrity notification, hot cache key, flash-sale SKU) — the same phenomenon in five costumes; the mitigations rhyme: spread it (L1, buckets, replicas), gate it (admission control), or batch it (leasing).
- **A failure-policy story** for each external dependency: Redis, providers, the PSP.

**Interview-grade sentence:** *"Across these problems the axes are the same — dominant requirement, read/write shape, consistency stance per component, where the hot key is, and what each dependency's failure does — and stating them in the first two minutes is how I pick the deep dives rather than following a memorised script."*

---

## Concept 37 — Pivots and follow-ups

Most rounds add a requirement. The pivot tells the interviewer whether your design absorbs change. Here are the common ones and the shape of a good answer.

| Pivot | Shortener | Rate limiter | Notifications | Cache | Orders/payments |
|---|---|---|---|---|---|
| **"Make it multi-region"** | Cosmos multi-region reads; codes globally unique already (random); writes in one region or multi-write with conflict-free creation (code is the key — conflicts are just collisions) | Per-region budgets, or geo-replicated counters with bounded over-admission | Region-local pipelines; preferences replicated; route by user's home region | Independent caches per region, or active geo-replication | Usually **single write region** for money, with failover; or per-region partitioning by merchant/customer with no cross-region transactions |
| **"10× traffic"** | Edge caching for hot links; more L1 | More shards; token leasing | More workers per lane; provider limits become the wall | More shards; watch per-key overhead | Contention, not throughput — bucketed stock, admission queues |
| **"Strict ordering"** | n/a | n/a | Service Bus sessions by user — at the cost of per-user parallelism | n/a | Per-order ordering via the orchestrator or sessions keyed by order ID |
| **"Add a new channel / provider / method"** | Custom domains: a domain → tenant map at the edge | New rule types as data | A new channel adapter + lane; the planner's channel selection is data | New eviction policy behind a seam | A new payment method behind a PSP adapter; the order state machine unchanged |
| **"Cut cost by half"** | Lower Cosmos RU via higher cache hit; cheaper analytics tier | Move fair-use limits to the gateway | Push over SMS where allowed; batch email | Right-size memory; LFU for better hit ratio | Fewer synchronous calls; reserved capacity |
| **"Data residency (EU only)"** | Regional deployment; codes route by domain | Per-region limits are natural | EU providers / regions; no cross-region PII | Region-local caches | Region-pinned data; PSP region selection |
| **"We already have a monolith doing this"** | Strangle the redirect path first (read-only, easy to verify) | Start at the gateway, no code | Extract senders behind the existing API, then the planner | Introduce the cache-aside layer behind repositories | Extract payments behind an anti-corruption layer; keep the ledger as the source of truth (Module 32) |

### How to deliver a pivot

1. **Restate what changes and what doesn't:** *"Multi-region affects the counter store and the failure policy; the algorithm and the API are unchanged."*
2. **Point to the seam** your design already has — or admit there isn't one and say where you'd add it.
3. **Give the trade-off** of the new choice with a number if possible ("over-admission bounded by replication lag × rate — at 200 ms lag and 100 rps, about 20 extra requests").
4. **Ask whether to go deeper** or move on.

**Interview-grade sentence:** *"When a follow-up changes the requirements, I say which parts of the design it touches and which it doesn't, point to the seam that absorbs it — or say where I'd add one — and give the trade-off of the new choice with a number before asking whether to go deeper."*

---

## Concept 38 — The architect's version

In architect loops the same prompts come with different deliverables. The design is the same; what's scored is **the decision, its record, and the path to it**.

### The questions that change

| IC-loop question | Architect-loop version |
|---|---|
| "Design a rate limiter" | "Our API has no limits and a tenant took us down last week. What do you do in the next month, and what's the long-term design?" |
| "Design a notification system" | "Five teams each send their own emails and SMS. Should we centralise, and how?" |
| "Design a distributed cache" | "We're on Azure Cache for Redis Premium. What's our plan for the retirement, and does anything change in the architecture?" |
| "Design a payment system" | "We want to add a second PSP. Write the ADR." |
| "Design a URL shortener" | "Should we build one or buy one?" |

### What a strong architect answer contains

1. **The decision framed as options with consequences** (Module 30): at least two credible options, their costs and risks, and a recommendation.
2. **The ADR** (Module 31) — one decision, context, options, consequences, review trigger. For example: *"ADR-014: Orchestrate checkout with a durable workflow engine"*, with the alternative (state machine + outbox) and the condition that would reverse it.
3. **The migration path** (Module 32): what ships first and what gets strangled — e.g. rate limits first at APIM with no code change (immediate risk reduction), then service-level limits for cost-weighted endpoints.
4. **Cost and ownership** (Module 33): cost drivers per component, who runs it, what the on-call burden is, build vs buy (the notification platform is a classic "buy the providers, build the thin orchestration" case).
5. **Risk and rollout:** shadow mode for limits, dual-run for notifications, canary for the new PSP, reconciliation before cut-over for payments.

### A worked architect vignette: the Azure Cache for Redis retirement

> *"We run Azure Cache for Redis Premium for sessions, product caching and a rate limiter. What's the plan?"*

A strong answer, compressed:

> *"Three facts frame it: Basic, Standard and Premium retire on September 30, 2028; the successor is Azure Managed Redis; and it's clustered by default. So the work is mostly verification, not redesign. First, inventory the clients and usages — anything using multi-key commands or Lua across keys needs hash tags in a cluster; StackExchange.Redis itself handles clustering. Second, move authentication to Entra ID while we're at it rather than carrying access keys forward. Third, migrate per workload by risk: product caching first — it can start cold — then the rate limiter, which needs its Lua keys checked for single-slot use, and sessions last, with a dual-write or a planned re-login window. Decision record: whether we stay on Redis-compatible managed services at all — I'd say yes; the alternatives, Valkey or Garnet self-hosted, add operational ownership for a licence benefit we don't need on Azure. Timeline: well ahead of 2028, because the Enterprise-tier deadline of March 31, 2027 tells us how Microsoft runs these retirements."*

**Interview-grade sentence:** *"In an architect loop the same design becomes a decision: I'd present credible options with consequences, record the choice as an ADR with the condition that would reverse it, sequence a migration that reduces risk first, name cost drivers and ownership, and plan the rollout with shadow modes, dual runs or reconciliation before any cut-over."*

---
# Worked example — A 45-minute rate limiter round, end to end

**Prompt:** *"We run a public REST API on about 40 instances of an ASP.NET Core service in Azure. Some customers are hammering it. Design rate limiting."*

### 00:00–05:00 — Requirements

> **Candidate:** "Let me check what it's for. Is the goal protecting us from abuse, enforcing plan limits, or protecting something downstream?"
> **Interviewer:** "Mostly fairness between customers — one tenant's batch job shouldn't hurt others. Plans have different limits."
> **Candidate:** "Then I'll assume: limits per tenant, per plan — say Free 10 req/s, Pro 100 req/s, with bursts of about 2× — plus a stricter per-endpoint limit on expensive endpoints like exports. 429 with Retry-After on rejection. The check should add under a millisecond at p99. Accuracy within about 10% is fine for fairness. And the limiter must never take the API down — so fail open. Quotas for billing are out of scope unless you want them. Traffic?"
> **Interviewer:** "Peak around 200k requests a second, maybe 300k active tenants."

### 05:00–08:00 — Numbers, each with a conclusion

> "200k checks a second across 40 instances is 5k per instance — fine for in-process work. A per-instance limiter alone would let a tenant get 40× its limit, so the count must be shared. 300k tenants × ~150 bytes of bucket state is about 45 MB, so memory is trivial; the store is sized on operations: 200k atomic scripts a second needs several Redis shards. And the biggest tenants may be thousands of requests a second on one key — one shard — so I'll plan for hot keys."

### 08:00–12:00 — API and placement

> "The 'API' is the decision: `TryAcquire(tenant, rule, cost) → allowed, retryAfter`. Placement: Front Door WAF for IP-level abuse; then — do we run API Management?"
> **Interviewer:** "No, clients hit Front Door then the service."
> "Then the per-tenant limiter lives in the service, as middleware after authentication, because the tenant and plan come from the token. Rules are data: plan → rate, burst, per-endpoint costs, in App Configuration, cached locally."

### 12:00–20:00 — High-level design

> "Request → auth → limiter middleware: resolve the tenant's rules, call one atomic check in Redis per rule, allow or 429. The check is a token bucket — sustained rate with a burst, and exports can cost 10 tokens. One Lua script per check: read tokens and timestamp, refill using Redis's TIME so instance clocks don't matter, decide, write back, set a TTL. Keys like `rl:{tenant}:api` with the tenant as a hash tag so a rule's keys stay in one slot. Behind that: Azure Managed Redis, clustered. Locally, an in-process token bucket per tenant as the fallback, at limit ÷ 40."

### 20:00–32:00 — Deep dive 1: atomicity and the hot path

> "Why a script: get-compare-set from the app loses updates under concurrency — two instances both see 1 token. The script is atomic in Redis, so checks for one tenant serialise there. Latency is one round trip, roughly half a millisecond in-region; StackExchange.Redis pipelines it over the shared multiplexer, and caches the script's SHA after the first EVAL.
> Hot tenants: the biggest tenant at, say, 5,000 req/s means 5,000 scripts a second on one shard. Two mitigations. A local pre-check — if the local bucket already says no, don't call Redis. And token leasing for large tenants: each instance takes 50 tokens per call and spends them locally. That cuts Redis calls 50-fold; worst-case over-admission is 50 × 40 = 2,000 tokens — for a tenant at 5k req/s that's under half a second of their rate, within our 10% budget."
> **Interviewer:** "Why not just sticky-route tenants to instances?"
> "It works until the instance count changes or one tenant exceeds one instance's capacity — then the hottest tenants are exactly the ones you can't pin. Leasing degrades more gracefully."

### 32:00–39:00 — Deep dive 2: failure policy

> "Redis is on every request now. The check gets a 15 ms budget — SE.Redis doesn't take cancellation tokens, so I'd use `WaitAsync` — and a circuit breaker. On timeout or an open circuit, we fail open to the local per-instance bucket, emit a 'fallback' metric and alert on it. For fairness limits that's right: an approximate local limit is better than either no limit or rejecting everyone. If you later add billing quotas, those fail closed or are reconciled from request logs — a different rule type with a different policy.
> Rollout: shadow mode first — compute decisions and log would-be rejections per tenant for a week; then enforce for Free, then Pro, with an override list."

### 39:00–42:00 — Wrap-up

> "Bottlenecks: Redis shard throughput under hot tenants — leasing; memory isn't one. Multi-region later: per-region budgets proportional to traffic, unless a global limit is contractually required. Testing: the built-in limiter with manual replenishment for the fallback, the Lua script against a Redis container with concurrent callers to check the over-admission bound. Observability: decisions per rule and outcome, top limited tenants, Redis p99, fallback mode."

### What the interviewer can write down

> *"Asked what the limiter was for and set accuracy and failure policy from the answer. Derived that per-instance limiting admits 40× and that the store is throughput-bound, not memory-bound. Token bucket with cost-weighted requests via an atomic Lua script using Redis server time, hash-tagged keys; knew StackExchange.Redis specifics (pipelining, script caching, no cancellation). Strong hot-key analysis: local pre-check plus token leasing with a computed over-admission bound, and a reasoned rejection of sticky routing. Fail-open-to-local with a circuit breaker, alerting, and a different policy for quotas. Shadow-mode rollout. **Strong hire, senior.**"*

**What made it senior:** not the algorithm — the purpose question that set the failure policy, the 40× derivation, the computed over-admission bound, and the rollout plan.

---
# Common interview questions with model answers

These are the probes that decide level inside these five problems — plus the cross-cutting questions .NET and architect interviewers add.

**Q1. "Why not just hash the URL to make the short code?"**
> "Seven base-62 characters is about 41.7 bits; with twelve billion links the birthday bound gives roughly twenty million colliding pairs, so I'd need collision detection anyway — and once I re-salt on collision I've lost the determinism that was the point. Dedup by URL is also usually unwanted, since different users want separate links and analytics. So I'd use random codes with a conditional insert."
*Key signal:* the birthday arithmetic, and seeing that hashing doesn't remove the check.

**Q2. "301 or 302 for the redirect?"**
> "302 by default. A 301 can be cached by browsers indefinitely, which is fastest but means the clicks never reach us — no analytics — and we can't disable or retarget the link for that browser. If the product doesn't need either, a 301 saves load; I'd still put a short `Cache-Control` on the 302 if we want some browser caching."
*Key signal:* tying an HTTP detail to product requirements.

**Q3. "Your service runs on 50 instances. How does the rate limit stay correct?"**
> "It doesn't, with in-process limiters — each admits the full limit, so a tenant gets 50×. I'd keep the bucket in Redis and make the check a single Lua script — refill using Redis's clock, decide, write back — so concurrent checks serialise atomically; one key per call so it works in a cluster. For hot tenants I'd add local pre-checks and token leasing."
*Key signal:* knowing the built-in limiters' contract is per process, and how to fix it atomically.

**Q4. "Redis is down. What does your rate limiter do?"**
> "It depends on the rule's purpose. For abuse and fairness limits: fail open to a local per-instance bucket at limit divided by instance count, behind a circuit breaker with a short time budget, and alert on fallback mode. For paid quotas: fail closed, or admit and reconcile from logs. For a fragile downstream: that's a local concurrency limit anyway, so it's unaffected."
*Key signal:* a failure policy per rule, chosen by purpose, with operability.

**Q5. "Token bucket or sliding window?"**
> "For an API, a token bucket: it expresses a sustained rate plus an explicit burst, and requests can cost more than one token. For 'N per window' semantics like login attempts, a weighted sliding-window counter — two integers per key, no boundary doubling. A fixed window allows up to twice the limit across a boundary, which matters for billing but often not for abuse."
*Key signal:* choosing by semantics, with the 2× boundary fact.

**Q6. "Can you guarantee each notification is sent exactly once?"**
> "Not end to end — queues redeliver, producers retry, and a provider call can time out after succeeding. I make it effectively once: stable notification and delivery IDs, a claim-send-record step in the worker, dedupe at every stage. That leaves one window — the provider accepted but we crashed before recording — and I choose per type: resend for OTPs, with collapse keys so the device shows one; don't resend marketing. Where a provider accepts an idempotency key, the window disappears."
*Key signal:* naming the residual crash window and choosing its outcome deliberately.

**Q7. "Marketing wants 50 million pushes in an hour. How do OTPs still arrive in seconds?"**
> "Physical isolation: separate queues and workers per priority lane, with the transactional lane never scaled to zero, and a campaign-level egress rate so the campaign uses its hour and leaves provider headroom. The campaign expands in a checkpointed background job in batches, never in the API call. A priority flag on a shared queue doesn't work — a million queued bulk messages still sit in front of the OTP."
*Key signal:* bulkheads, not priorities; expansion outside the request path.

**Q8. "How do you add a node to your cache cluster without downtime?"**
> "With a fixed slot space mapped to nodes, I move slots one at a time: the source marks a slot migrating, the target importing; keys are copied; requests for already-moved keys get an ASK redirect during the move and MOVED once the slot is fully transferred, so smart clients update their maps. Only the moved slots' keys change owner — about 1/N of the data — unlike modulo hashing, which would move almost everything."
*Key signal:* the mechanics, and why modulo fails.

**Q9. "A hot product page's cache entry expires and the database falls over. What happened, and how do you prevent it?"**
> "A stampede: thousands of concurrent misses all recompute. Within a process, request coalescing — `HybridCache` does it — means one factory call per instance; across instances, for an expensive origin, a short distributed lease so one instance recomputes while others serve stale. Plus stale-while-revalidate or probabilistic early refresh, and TTL jitter so related keys don't expire together."
*Key signal:* knowing coalescing is per process, and the layered mitigations.

**Q10. "You use `HybridCache` across 30 instances. A product's price changes. When do all instances show the new price?"**
> "`RemoveAsync` clears the shared L2 and the local L1 on the instance that called it — other instances keep their L1 copy until its local expiration. So the bound is `LocalCacheExpiration`. If that's too long, shorten it, or add a backplane — FusionCache can act as the `HybridCache` implementation with a Redis backplane — and for prices specifically I might not cache them in L1 at all at checkout."
*Key signal:* precise semantics, and a business-aware exception.

**Q11. "The payment provider timed out. Did the customer pay?"**
> "We don't know — so we must not treat it as a failure. The payment goes to an Unknown state; we retry with the same idempotency key, which is safe by construction, or query the PSP by our reference; a sweeper keeps resolving unknowns and the webhook usually brings the answer. The customer sees 'processing', not 'failed'."
*Key signal:* the unknown-outcome state, and why the deterministic key makes retries safe.

**Q12. "Orchestration or choreography for checkout?"**
> "Orchestration for the checkout itself: it's a short, ordered, business-critical flow with compensations, and support needs to answer 'where is order 123?' in one place. Choreography for the reactions to `OrderConfirmed` — loyalty, analytics, email — which are independent and shouldn't be coupled to the orchestrator."
*Key signal:* using both, each where it fits.

**Q13. "How do you stop overselling in a flash sale?"**
> "Atomically: one conditional update — decrement where `available >= qty` — so the check and the write can't be split by a race, with expiring reservations. Under flash-sale contention on one row, keep that transaction tiny, split the stock across bucket rows, and put admission control in front. If the business accepts it, a bounded oversell policy is cheaper still."
*Key signal:* atomicity first, contention second, business policy third.

**Q14. "Why not a distributed transaction across order, inventory and payment?"**
> "The PSP is a third-party HTTP API — it can't join our transaction. Even internally, two-phase commit couples availability across services and holds locks across slow calls. So it's a saga: local transactions with compensations, an outbox for events, and idempotent steps."
*Key signal:* the concrete reason (the PSP), not just "2PC is bad".

**Q15. "Which .NET and Azure services would you use, and what would change your mind?"**
> Answer per component in the default → why → when-not shape (Concept 4). For example: *"Cosmos DB for the link store because the hot path is a point read by code; Azure SQL if we needed relational reporting. Azure Managed Redis for L2 — the successor to Azure Cache for Redis; Garnet only if we needed extreme per-node throughput and were willing to run it. Service Bus for notification lanes because we need sessions, scheduled messages and dead-lettering; Event Hubs for click streams because it's a high-volume log."*
*Key signal:* reasons and exit conditions, not a list.

**Q16. "How would you know this design works in production?"**
> "SLIs per path — redirect p99 and cache hit ratio; limiter decision latency and fallback rate; notification end-to-end latency and queue age per lane; checkout success rate and the count of payments in Unknown. Traces that cross the async hops, by propagating context through message properties. And for payments, reconciliation is the ultimate test — it tells us when the system is wrong."
*Key signal:* per-problem SLIs and reconciliation as verification.

**Q17. "Would you build the notification platform or buy one?"**
> "Buy the delivery — APNs and FCM directly or via Notification Hubs, Azure Communication Services or a vendor for SMS and email — and decide on the orchestration layer by need: if we have a few notification types, a vendor's workflow product is cheaper than a team; if notifications are core to the product, with complex preferences and many producers, a thin internal orchestration layer over bought providers pays off. I'd frame it as total cost including the on-call."
*Key signal:* splitting build-vs-buy by layer (Module 33).

**Q18. "We already have checkout in a monolith. How do we get to this design?"**
> "Incrementally. First make the existing flow safe — idempotency keys on the checkout endpoint and on PSP calls, which is the highest-value change. Then add the outbox and publish `OrderConfirmed` so new consumers attach without touching the monolith. Then extract payments behind an anti-corruption layer, routing a percentage of traffic through the new service with reconciliation comparing both paths, and strangle the old code once the numbers match."
*Key signal:* risk-first sequencing, with reconciliation as the cut-over gate (Module 32).

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Requirements | Skips them because the problem is "known" | Confirms purpose and the numbers that shape the design; finds the twist |
| Estimation | Numbers without conclusions | Every number produces a decision ("memory sets the node count") |
| Time | Deep dives start at minute 30 | High-level by ~22; two deep dives chosen by difficulty |
| Products | Product soup | Default → why → when not, per component |
| .NET claims | "Use `HybridCache`" | "Coalescing and `RemoveAsync` are per process; L1 lifetime bounds staleness" |
| Shortener IDs | Hash-and-truncate, no collision plan | Birthday math; random + conditional insert, or leased counter + permutation |
| Shortener redirect | 301 by default | 302 for analytics and revocation; L1/L2/edge; negative caching |
| Shortener analytics | DB write per click | Async events with an explicit loss policy |
| Rate limiter algorithm | Fixed window without caveats | Chosen by semantics; token bucket / GCRA; 2× boundary fact |
| Rate limiter distribution | In-process limiters on 50 pods | Shared atomic script; one slot per call; token leasing for hot keys |
| Rate limiter failure | Unconsidered | Policy per rule purpose; time budget; circuit breaker; alert on fallback |
| Notifications priority | One queue with a priority field | Separate lanes and workers; campaign egress rate |
| Notifications fan-out | Expand 50M recipients in the request | Checkpointed batch expansion; topics for broadcast |
| Notifications guarantee | "Exactly once" | Effectively once; named crash window; per-type choice; collapse keys |
| Notifications providers | Happy path only | Provider limits, invalid-token cleanup, failover, DLQ and replay |
| Cache partitioning | `hash mod N` | Slots or consistent hashing with virtual nodes; ASK/MOVED |
| Cache replication | "Synchronous for safety" | Async, because loss is acceptable; split-brain bound |
| Cache usage | TTL only | Coalescing, early refresh, jitter, negative caching, invalidation with leases or versions |
| Hashing in .NET | `string.GetHashCode()` for placement | A stable hash (xxHash) — `GetHashCode` is randomised per process |
| Orders scale | Sharded microservices for 12 orders/s | Relational + ACID; contention is the real scale problem |
| Orders idempotency | Retry payment calls without keys | Deterministic keys at every hop; replayed responses |
| Orders timeouts | Timeout = failure | Unknown state; retry with same key or query; sweeper |
| Orders workflow | 2PC or "call them in sequence" | Saga with compensations; authorise/capture split |
| Inventory | Read-check-write in code | Conditional update; expiring reservations; buckets for hot SKUs |
| EF Core | `ExecuteUpdateAsync` assumed part of `SaveChanges` | Wrapped in an explicit transaction with `SaveChanges` |
| Durable workflows | `DateTime.UtcNow` in orchestrators | Deterministic replay rules; idempotent activities |
| Payments compliance | Card numbers through the backend | Hosted fields; tokens only; minimal PCI scope |
| Verification | "Add monitoring" | Per-path SLIs; reconciliation; shadow modes |
| Architect loop | Re-designs from scratch | Options, ADR, migration path, cost and ownership |
| Library choices | Assumes MassTransit is free | Knows 2026 licensing; build-vs-buy |

---
# Practice exercises

1. **Derive, don't recite.** For each of the five problems, write only the requirements and the estimation in 8 minutes, then list the *decision* each number produced. Discard any number that produced none.
2. **Code length.** Recompute Concept 5's code-length decision for 2 billion links per month over 10 years. Does 7 still hold? At what volume would you move to 8?
3. **Collision maths.** Compute the expected collisions for hash-and-truncate at 6, 7 and 8 characters for 12 billion links, and the per-insert retry probability for random codes at each length.
4. **Feistel.** Implement `CodePermutation` and its inverse; property-test that `Inverse(Permute(x)) == x` for random 40-bit inputs and that 1 million sequential inputs produce 1 million distinct outputs.
5. **Rate limiter algorithms.** Implement fixed window, weighted sliding window, token bucket and GCRA in-process with `TimeProvider`; drive each with the same synthetic trace (including a boundary burst) using `FakeTimeProvider`, and tabulate admitted counts.
6. **Redis token bucket.** Run the Lua script against Redis in a container (Testcontainers); hammer one key from 50 concurrent tasks and verify admissions never exceed `capacity + rate × elapsed`. Then add token leasing and measure the over-admission against the bound `batch × instances`.
7. **Failure policy.** Add a 15 ms budget, a circuit breaker and a local fallback around exercise 6; kill the Redis container mid-test and show the fallback metric.
8. **Notification lanes.** Build a minimal pipeline with two Service Bus queues (high, bulk) or the emulator, a planner and a sender `BackgroundService`; flood the bulk queue with 1M messages and measure OTP latency with and without lane separation.
9. **Crash windows.** In exercise 8's sender, inject a crash after the provider call and before recording; show the duplicate, then add a collapse key and explain what the user would see.
10. **Hash ring.** Implement `HashRing<TNode>`; measure load imbalance with 1, 10, 100 and 160 virtual nodes for 10 nodes and 1M keys; measure the fraction of keys moved when adding an 11th node, versus modulo hashing.
11. **Stampede.** Write a load test against an endpoint backed by a slow origin (200 ms). Compare `IMemoryCache` naïve get-or-set, `HybridCache`, and `HybridCache` + a Redis lease across 5 instances. Count origin calls on a cold key.
12. **Checkout saga.** Implement the checkout as (a) an EF Core state machine with an outbox dispatcher and (b) a Durable Functions orchestrator with a fake PSP that times out 10% of the time *after* succeeding. Verify no double charges and no lost orders across 10,000 simulated checkouts.
13. **Flash sale.** Load-test the conditional decrement on one SKU row at 2,000 attempts/s; then split into 16 buckets and compare p99 latency and failure rate.
14. **Pivots.** For each problem, answer the "multi-region" and "10×" pivots from Concept 37 aloud in 3 minutes each, with one number per answer.
15. **Architect version.** Write the ADR for "Orchestrate checkout with a durable workflow engine vs a state machine with an outbox" (Module 31 template), including the condition that would reverse it.
16. **Full mock.** Run each problem as a 45-minute mock with a partner (or an AI interviewer instructed to probe the deep dives); score yourself with Appendix E and compare with the worked rate-limiter transcript.

---
# Free resources and learning material

All free to read online unless marked *(book)* or *(partly paid)*. Start with the ★ items. Platform facts were checked on October 8, 2026.

### Worked breakdowns of these five problems
- ★ [Bitly / URL shortener — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly) — a staff-engineer breakdown with deep dives on code generation and scaling reads.
- ★ [Distributed rate limiter — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/distributed-rate-limiter).
- [Notification system — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/notification-system) *(partly paid)*.
- [Distributed cache — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/distributed-cache) *(partly paid)*.
- ★ [Payment system — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/payment-system) *(partly paid)*.
- [Flash sale — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/flash-sale) and [Ticketmaster](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ticketmaster) — inventory contention.
- [Patterns: dealing with contention](https://www.hellointerview.com/learn/system-design/patterns/dealing-with-contention), [multi-step processes](https://www.hellointerview.com/learn/system-design/patterns/multi-step-processes), [scaling reads](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads), [real-time updates](https://www.hellointerview.com/learn/system-design/patterns/realtime-updates) — Hello Interview.
- ★ [The System Design Primer — GitHub](https://github.com/donnemartin/system-design-primer) — includes a [Pastebin/URL-shortener-style solution](https://github.com/donnemartin/system-design-primer/blob/master/solutions/system_design/pastebin/README.md).
- [ByteByteGo blog](https://blog.bytebytego.com/) *(partly paid)* — illustrated summaries of the same problems.

### Method and numbers
- [Latency numbers every programmer should know](https://gist.github.com/jboner/2841832).
- [Numbers to know — Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/numbers-to-know).
- [Delivery framework — Hello Interview](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery).
- ★ [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/) and the [cloud design patterns catalogue](https://learn.microsoft.com/azure/architecture/patterns/).
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/).

### URL shortener: IDs, redirects, storage
- [Birthday problem — Wikipedia](https://en.wikipedia.org/wiki/Birthday_problem) — the collision maths.
- [Feistel cipher — Wikipedia](https://en.wikipedia.org/wiki/Feistel_cipher) — why the permutation is a bijection.
- ★ [Announcing Snowflake — X/Twitter engineering](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake) and [Sharding & IDs at Instagram](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c).
- [IdGen — Snowflake-style IDs for .NET (GitHub)](https://github.com/RobThree/IdGen).
- [RFC 9562 — UUIDs, including UUIDv7](https://www.rfc-editor.org/rfc/rfc9562) and [`Guid.CreateVersion7` (.NET 9+)](https://learn.microsoft.com/dotnet/api/system.guid.createversion7).
- [RFC 9110 — HTTP redirection status codes](https://www.rfc-editor.org/rfc/rfc9110#name-redirection-3xx).
- [`RandomNumberGenerator.GetString` — API reference](https://learn.microsoft.com/dotnet/api/system.security.cryptography.randomnumbergenerator.getstring).
- ★ [Partitioning and horizontal scaling in Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/partitioning-overview) and [Cosmos DB .NET SDK best practices](https://learn.microsoft.com/azure/cosmos-db/nosql/best-practice-dotnet).
- [Time to live in Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/nosql/time-to-live).
- [Caching with Azure Front Door](https://learn.microsoft.com/azure/frontdoor/front-door-caching).
- [Output caching middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/output).
- [Azure Event Hubs overview](https://learn.microsoft.com/azure/event-hubs/event-hubs-about).
- [Bloom filter — Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter).
- [Google Safe Browsing](https://developers.google.com/safe-browsing) — URL reputation checks.

### Rate limiting
- ★ [How we built rate limiting capable of scaling to millions of domains — Cloudflare](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) — the weighted sliding-window counter and its measured error.
- ★ [Scaling your API with rate limiters — Stripe](https://stripe.com/blog/rate-limiters) — four limiter types in production, including load shedders.
- ★ [Rate limiting, cells, and GCRA — Brandur Leach](https://brandur.org/rate-limiting).
- [An alternative approach to rate limiting — Figma](https://www.figma.com/blog/an-alternative-approach-to-rate-limiting/).
- [Generic cell rate algorithm — Wikipedia](https://en.wikipedia.org/wiki/Generic_cell_rate_algorithm) and [Token bucket — Wikipedia](https://en.wikipedia.org/wiki/Token_bucket).
- ★ [Rate limiting middleware in ASP.NET Core — Microsoft Learn](https://learn.microsoft.com/aspnet/core/performance/rate-limit).
- [Announcing rate limiting for .NET — .NET Blog](https://devblogs.microsoft.com/dotnet/announcing-rate-limiting-for-dotnet/) — the design of `System.Threading.RateLimiting`.
- [aspnetcore-redis-rate-limiting (RedisRateLimiting) — GitHub](https://github.com/cristipufu/aspnetcore-redis-rate-limiting).
- [Rate-limit-by-key policy — Azure API Management](https://learn.microsoft.com/azure/api-management/rate-limit-by-key-policy) and [advanced request throttling](https://learn.microsoft.com/azure/api-management/api-management-sample-flexible-throttling).
- [WAF rate limiting for Azure Front Door](https://learn.microsoft.com/azure/web-application-firewall/afds/waf-front-door-rate-limit).
- [Rate Limiting pattern](https://learn.microsoft.com/azure/architecture/patterns/rate-limiting-pattern) and [Throttling pattern — Azure Architecture Center](https://learn.microsoft.com/azure/architecture/patterns/throttling).
- [Redis scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/).
- [RFC 6585 — 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585) and the [IETF RateLimit header fields draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/).
- [Rate limiter strategy in Polly](https://www.pollydocs.org/strategies/rate-limiter.html).

### Notification systems
- ★ [Azure Notification Hubs overview](https://learn.microsoft.com/azure/notification-hubs/notification-hubs-push-notification-overview) and [Notification Hubs and FCM v1 migration](https://learn.microsoft.com/azure/notification-hubs/notification-hubs-gcm-to-fcm).
- [Migrate from legacy FCM APIs to HTTP v1 — Firebase](https://firebase.google.com/docs/cloud-messaging/migrate-v1).
- [Sending notification requests to APNs — Apple](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns).
- [Azure Communication Services email overview](https://learn.microsoft.com/azure/communication-services/concepts/email/email-overview) and [SMS concepts](https://learn.microsoft.com/azure/communication-services/concepts/sms/concepts).
- ★ [Service Bus duplicate detection](https://learn.microsoft.com/azure/service-bus-messaging/duplicate-detection), [message sessions](https://learn.microsoft.com/azure/service-bus-messaging/message-sessions), [message sequencing and scheduled messages](https://learn.microsoft.com/azure/service-bus-messaging/message-sequencing) and [dead-letter queues](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-dead-letter-queues).
- [Priority Queue pattern](https://learn.microsoft.com/azure/architecture/patterns/priority-queue), [Queue-Based Load Leveling](https://learn.microsoft.com/azure/architecture/patterns/queue-based-load-leveling), [Competing Consumers](https://learn.microsoft.com/azure/architecture/patterns/competing-consumers) and [Bulkhead](https://learn.microsoft.com/azure/architecture/patterns/bulkhead) — Azure Architecture Center.
- [KEDA Azure Service Bus scaler](https://keda.sh/docs/latest/scalers/azure-service-bus/).
- [Azure SignalR Service overview](https://learn.microsoft.com/azure/azure-signalr/signalr-overview) and [Azure Web PubSub overview](https://learn.microsoft.com/azure/azure-web-pubsub/overview).
- [Evolving Netflix's Pushy WebSocket server — Netflix Tech Blog](https://netflixtechblog.com/pushy-to-the-limit-evolving-netflixs-websocket-proxy-for-the-future-b468bc0ff658).
- [Slack's job queue — Hello Interview "in the wild"](https://www.hellointerview.com/learn/system-design/in-the-wild/slack-job-queue).
- [Scriban](https://github.com/scriban/scriban) and [Fluid](https://github.com/sebastienros/fluid) — Liquid-style templating for .NET.

### Distributed caches
- ★ [Dynamo: Amazon's highly available key-value store (2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — consistent hashing with virtual nodes in production.
- ★ [Scaling Memcache at Facebook (NSDI 2013)](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf) — leases, stampedes, regional invalidation.
- [Consistent hashing and random trees — Karger et al. (1997)](https://dl.acm.org/doi/10.1145/258533.258660), [Jump consistent hash (2014)](https://arxiv.org/abs/1406.2294), [Rendezvous hashing — Wikipedia](https://en.wikipedia.org/wiki/Rendezvous_hashing).
- [Consistent hashing — Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing).
- ★ [Optimal probabilistic cache stampede prevention (XFetch) — Vattani et al.](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf).
- [TinyLFU: a highly efficient cache admission policy](https://arxiv.org/abs/1512.00727).
- ★ [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) and [key eviction](https://redis.io/docs/latest/develop/reference/eviction/).
- [Random notes on improving the Redis LRU algorithm — antirez](http://antirez.com/news/109).
- [Valkey](https://valkey.io/) and [Garnet — Microsoft Research](https://microsoft.github.io/garnet/).
- ★ [What is Azure Managed Redis?](https://learn.microsoft.com/azure/redis/overview), [Azure Cache for Redis retirement FAQ](https://learn.microsoft.com/azure/azure-cache-for-redis/retirement-faq) and [Azure Managed Redis client-library best practices](https://learn.microsoft.com/azure/redis/best-practices-client-libraries).
- ★ [HybridCache in ASP.NET Core — Microsoft Learn](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid).
- [FusionCache — GitHub](https://github.com/ZiggyCreatures/FusionCache) — backplane, fail-safe, `HybridCache` implementation.
- [StackExchange.Redis documentation](https://stackexchange.github.io/StackExchange.Redis/) — especially *Basic usage*, *Pipelines and multiplexers* and *Timeouts*.
- [Cache-Aside pattern — Azure Architecture Center](https://learn.microsoft.com/azure/architecture/patterns/cache-aside).
- [System.IO.Hashing (XxHash64) — API reference](https://learn.microsoft.com/dotnet/api/system.io.hashing.xxhash64).

### Orders, payments and workflows
- ★ [Designing robust and predictable APIs with idempotency — Stripe](https://stripe.com/blog/idempotency) and [Idempotent requests — Stripe API reference](https://docs.stripe.com/api/idempotent_requests).
- ★ [Implementing Stripe-like idempotency keys in Postgres — Brandur Leach](https://brandur.org/idempotency-keys).
- [The Idempotency-Key HTTP header field — IETF draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) — expired as a draft, but the convention is widely used.
- ★ [Avoiding double payments in a distributed payments system — Airbnb Engineering](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb).
- [PaymentIntent lifecycle — Stripe](https://docs.stripe.com/payments/paymentintents/lifecycle), [place a hold on a payment method (authorise/capture)](https://docs.stripe.com/payments/place-a-hold-on-a-payment-method) and [webhooks](https://docs.stripe.com/webhooks).
- ★ [Saga pattern](https://learn.microsoft.com/azure/architecture/patterns/saga), [Compensating Transaction](https://learn.microsoft.com/azure/architecture/patterns/compensating-transaction), [Scheduler Agent Supervisor](https://learn.microsoft.com/azure/architecture/patterns/scheduler-agent-supervisor) — Azure Architecture Center.
- [Transactional outbox with Cosmos DB — Azure Architecture Center](https://learn.microsoft.com/azure/architecture/databases/guide/transactional-outbox-cosmos).
- [Saga](https://microservices.io/patterns/data/saga.html) and [Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html) — microservices.io.
- [Sagas — Garcia-Molina & Salem (1987)](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) and [Life beyond distributed transactions — Pat Helland](https://queue.acm.org/detail.cfm?id=3025012).
- [Books: an immutable double-entry accounting database — Square](https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/) and [Accounting for developers — Modern Treasury](https://www.moderntreasury.com/journal/accounting-for-developers-part-i).
- [Shopify inventory reservations — Hello Interview "in the wild"](https://www.hellointerview.com/learn/system-design/in-the-wild/shopify-inventory-reservations).
- ★ [Handling concurrency conflicts — EF Core](https://learn.microsoft.com/ef/core/saving/concurrency) and [ExecuteUpdate and ExecuteDelete](https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete).
- ★ [Durable Functions overview](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview), [orchestrator code constraints](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-code-constraints) and [the Durable Task Scheduler](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler).
- [Dapr Workflow overview](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/) and [Temporal .NET SDK](https://github.com/temporalio/sdk-dotnet).
- [Wolverine](https://wolverinefx.net/), [MassTransit](https://masstransit.io/), [NServiceBus outbox](https://docs.particular.net/nservicebus/outbox/).
- [PCI Security Standards Council](https://www.pcisecuritystandards.org/).

### .NET platform references used across the module
- [What's new in .NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview) and [.NET Conf 2026 announcement](https://devblogs.microsoft.com/dotnet/dotnet-conf-2026/).
- [What's new in Aspire 13](https://aspire.dev/whats-new/aspire-13/) and [Aspire](https://aspire.dev/).
- [Build resilient HTTP apps (Microsoft.Extensions.Http.Resilience)](https://learn.microsoft.com/dotnet/core/resilience/http-resilience) and [Polly documentation](https://www.pollydocs.org/).
- [Background tasks with hosted services in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services).
- [Azure Service Bus client library for .NET](https://learn.microsoft.com/dotnet/api/overview/azure/messaging.servicebus-readme).
- [Channels in .NET](https://learn.microsoft.com/dotnet/core/extensions/channels).
- [OpenTelemetry for .NET](https://opentelemetry.io/docs/languages/dotnet/) and [ASP.NET Core built-in metrics](https://learn.microsoft.com/aspnet/core/log-mon/metrics/built-in).
- [Testcontainers for .NET](https://dotnet.testcontainers.org/).
- [dotnet/eShop — reference application](https://github.com/dotnet/eShop) — ordering, payments-style flows and an outbox in a realistic .NET codebase.

### Videos
- [Hello Interview — YouTube](https://www.youtube.com/@hello_interview) — mock walkthroughs of Bitly, rate limiter and payment system.
- [.NET Conf sessions (dotnetconf.net)](https://dotnetconf.net/) — Aspire, ASP.NET Core and caching talks.

### Books
- *(book)* Alex Xu — *System Design Interview*, Vol. 1 (rate limiter, URL shortener, notification system, key-value store) and Vol. 2 (payment system, digital wallet).
- *(book)* Martin Kleppmann — *Designing Data-Intensive Applications* — the theory under every deep dive.
- *(book)* Sam Newman — *Building Microservices* (2nd ed.) — sagas, orchestration vs choreography.
- *(book)* Chris Richardson — *Microservices Patterns* — sagas and the outbox, with worked order flows.
- *(book)* Michael Nygard — *Release It!* (2nd ed.) — stability patterns behind the failure policies.

### Previous modules to revisit
- Modules 3–5 (method, requirements, estimation) — the skeleton used five times here.
- Module 8 (partitioning, consistent hashing) and Module 10 (caching) — Parts B and E.
- Modules 11–13 (messaging, outbox, sagas, reliability) — Parts D and F.
- Module 19 (EF Core) and Module 25 (Polly) — the implementation notes.
- Modules 30–33 (design docs, ADRs, brownfield, cost) — Concept 38.

---
# Quick-recall sheet

**One sentence.** The interviewer knows the standard answer; you score by deriving it from requirements and numbers, going deep where the difficulty lives, stating each component's consistency stance and failure policy, and naming the exact .NET/Azure primitive with its limits.

**The run.** 0–5 requirements · 5–9 numbers-with-conclusions · 9–12 API · 12–15 data model · 15–22 high level · 22–38 two deep dives · 38–42 wrap-up · questions.

**Toolbox.** Coordination-free IDs · partition key · cache tiers · queues + outbox · idempotency + dedupe · conditional writes · back-pressure · state machines · sweepers/reconcilers · bulkheads · observability.

**Talking .NET.** Default → why → when not. One precise semantic claim beats five product names.

**Shortener.**
- 200M/month → ~400 w/s, ~40k r/s peak, 12B links, ~6 TB. 62⁷ ≈ 3.5 × 10¹² → 0.34% full → 7 chars.
- Hash-and-truncate: ~2 × 10⁷ collisions (n²/2N) → no. Counter: enumerable → lease ranges + Feistel. Snowflake: 11 chars → too long. **Random + conditional insert** → default.
- 302 (analytics, revocation). Edge → L1 → L2 → Cosmos point read. Negative caching. Cap cache TTL at link expiry.
- Clicks: bounded channel, drop on full → Event Hubs → stream aggregation.
- .NET: `Results.Redirect(permanent: false)`, `HybridCache` (per-process coalescing), `ReadItemStreamAsync`, 409 on conflict, `RandomNumberGenerator.GetString`.

**Rate limiter.**
- Purpose sets accuracy and failure policy: abuse/fairness (fail open), quotas (fail closed/reconcile), downstream (concurrency limit).
- Layers: Front Door WAF → APIM `rate-limit-by-key` → service middleware → outbound Polly limiter.
- Fixed window 2× at boundary · sliding log exact, O(limit) memory · sliding counter `prev×(W−e)/W + curr` · token bucket rate + burst + cost · GCRA one TAT, τ = (burst−1)T.
- Distributed: atomic Lua; Redis `TIME`; one slot per call (hash tags); store is throughput-bound.
- Hot keys: local pre-check, token leasing (over-admission ≤ batch × instances), split keys.
- Failure: 5–20 ms budget (`WaitAsync`), circuit breaker, local fallback, alert.
- .NET built-ins are **per process**; middleware after auth; `QueueLimit = 0`; Retry-After from lease metadata; test with `TryReplenish()`.

**Notifications.**
- Campaign 50M/h ≈ 14k/s vs transactional ~2.3k/s peak → priority lanes. Provider limits are the wall.
- API (202) → planner (preferences, consent, templates, one delivery per recipient×channel) → per-channel × per-priority queues → workers → providers → receipts.
- Fan-out in a checkpointed job; topics/tags for broadcast; inbox fan-out-on-write + broadcasts on read.
- Effectively once: notification ID from producer event; deterministic delivery ID; claim → send → record. Residual window: dup (OTP, with collapse keys) vs loss (marketing).
- Invalid tokens (410/UNREGISTERED) → delete. DLQ + replay. Secondary SMS/email provider.
- Real-time: SignalR Service / Web PubSub (~1k connections per unit); inbox is the truth.
- .NET: `ServiceBusProcessor` manual complete; duplicate detection (producer side only); scheduled messages; abandon has no backoff → schedule retry copy; KEDA per lane; Notification Hubs (FCM v1 separate platform); ACS.

**Distributed cache.**
- Build vs use. Durability not required → async replication, memory-only.
- 1 TB / ~45 GB usable → ~24 primaries + replicas; memory sets node count.
- Modulo moves ~N/(N+1) of keys; consistent hashing ~1/N; virtual nodes; 16,384 hash slots; hash tags; ASK vs MOVED.
- Failover by majority; isolated primary stops writes after timeout; zones.
- Eviction: approximated LRU (sampling), decaying LFU, TinyLFU; lazy + active expiry; 30% headroom.
- Hot keys: L1, replicas, key copies. Stampede: coalescing, distributed lease, stale-while-revalidate, XFetch (`now − δβ ln(rand) ≥ expiry`), TTL jitter. Delete-on-write + TTL backstop; leases or versioned keys for the race.
- .NET: `HybridCache` coalescing and `RemoveAsync` are per process → L1 lifetime bounds staleness, or FusionCache backplane; singleton multiplexer; never `string.GetHashCode()` for placement.
- Azure: Managed Redis (clustered by default; Entra ID). Azure Cache for Redis B/S/P retire 30 Sep 2028; Enterprise 31 Mar 2027. Redis 8 tri-licence; Valkey BSD; Garnet MIT.

**Orders and payments.**
- 1M/day ≈ 12/s → relational + ACID; contention (flash sale) is the scale problem.
- Order aggregate state machine; rowversion; outbox in the same `SaveChanges`; minor units + currency; double-entry ledger (postings sum to zero).
- Idempotency at 4 hops: client key (replay stored response; mismatch → 422; in progress → 409); deterministic PSP keys; webhooks (verify, dedupe by event ID, state machine); consumer inbox.
- Timeout ≠ failure → Unknown → retry same key / query / sweeper.
- Saga: reserve → authorise → confirm → capture at ship (void vs refund). Orchestrate checkout; choreograph reactions.
- Inventory: one conditional `UPDATE … WHERE available >= qty`; reservation TTL; buckets; admission control.
- Reconcile ledger ⇄ payments ⇄ PSP settlement. Hosted fields → minimal PCI scope.
- .NET: `ExecuteUpdateAsync` bypasses change tracking → explicit transaction; Stripe.net `RequestOptions.IdempotencyKey`; Durable orchestrators deterministic (`context.CurrentUtcDateTime`, `context.NewGuid()`), activities idempotent, timers and external events; Durable Task Scheduler GA.

**Pivots.** Say what changes and what doesn't · point to the seam · give the trade-off with a number · ask to go deeper.

**Architect version.** Options with consequences · ADR with a reversal condition · risk-first migration · cost drivers and ownership · shadow/dual-run/reconcile before cut-over.

---
# Appendix A — The numbers sheet

```text
SHORTENER     200M links/month → 77 w/s avg, ~400 peak · 20B redirects/month → 7.7k r/s avg, ~40k peak
              12B links / 5 years · ~500 B/link → ~6 TB · 62^6 ≈ 5.7e10 · 62^7 ≈ 3.5e12 · 62^11 > 2^64
              hash collisions ≈ n²/2N = 1.44e20 / 7.0e12 ≈ 2e7 · random-insert retry p = occupancy ≈ 0.34%
              L2 hot set 10M × 0.5 KB ≈ 5 GB · L1 100k × 0.5 KB ≈ 50 MB/instance · Cosmos point read ≈ 1 RU (≤1 KB)
              Bloom filter 1% FP ≈ 9.6 bits/element → 12B ≈ 14 GB

RATE LIMITER  50 instances × local limit = 50× admission · 1M keys × ~150 B ≈ 150 MB (memory trivial)
              500k checks/s → several shards (script throughput-bound) · Redis RTT in-region ≈ 0.3–1 ms
              fixed window worst case 2× limit · GCRA: T = period/limit, τ = (burst−1)·T
              leasing over-admission ≤ batch × instances (e.g. 50 × 40 = 2,000)

NOTIFICATIONS 50M users · 75M tokens × 300 B ≈ 22 GB · transactional 20M/day ≈ 230/s, peak ≈ 2.3k/s
              campaign 50M/h ≈ 14k/s · delivery log ≈ 100 GB/day · 5% online ≈ 2.5M connections
              SignalR Service ≈ 1,000 connections per unit

CACHE         1 TB / ~45 GB usable per 64 GB node ≈ 24 primaries (+ replicas = 48)
              1.1M ops/s / 24 ≈ 46k ops/s per primary · 1.1 GB/s ≈ 9 Gbps aggregate
              mod-N → N+1 moves ≈ N/(N+1) of keys · consistent hashing moves ≈ 1/(N+1) · 16,384 slots

ORDERS        1M/day ≈ 12/s avg, ×20 peak ≈ 230/s · PSP call 1–2 s · ~5 ledger postings/order
              flash sale 100k buyers / 60 s ≈ 1.7k attempts/s on one row · card auth holds expire ~7 days (varies)
```

---
# Appendix B — One-page cards

### URL shortener
- **Requirements:** create (alias, expiry), redirect, manage, analytics. Reads 100:1; redirect p99 < 50 ms; non-enumerable codes.
- **Numbers:** ~400 w/s · ~40k r/s · 12B links · ~6 TB · 7 chars.
- **Design:** Front Door → stateless redirect service (L1) → Redis L2 → Cosmos (`/code`). Creation: random code + conditional insert. Clicks → channel → Event Hubs → aggregation.
- **Deep dives:** code generation (5 options) · redirect path under skew · analytics · abuse.
- **.NET:** minimal API 302 · `HybridCache` · `ReadItemStreamAsync` · 409 on conflict · `RandomNumberGenerator.GetString` · bounded `Channel<T>` + `EventHubBufferedProducerClient`.
- **Traps:** 301 by default · hash without collision plan · DB write per click · cache outliving expiry.

### Rate limiter
- **Requirements:** per-tenant/plan/endpoint limits; 429 + Retry-After; < 1 ms; fail open for fairness.
- **Numbers:** N instances ⇒ N× · ~150 MB for 1M keys · throughput-bound store.
- **Design:** edge WAF → (APIM) → middleware after auth → atomic Lua token bucket in Redis → local fallback.
- **Deep dives:** algorithm choice · atomic distributed check · hot keys/leasing · failure policy · multi-region.
- **.NET:** `System.Threading.RateLimiting` partitions (per process!) · `ScriptEvaluateAsync` + `WaitAsync` budget · Polly outbound limiter · APIM policies.
- **Traps:** in-process limiters for a fleet limit · GET-then-SET · partitioning by unauthenticated header · no fallback alert.

### Notification system
- **Requirements:** 4 channels; preferences/consent; templates; tracking; campaigns; OTP < 10 s.
- **Numbers:** campaign ≈ 14k/s ≫ transactional ≈ 2.3k/s · providers are the limit.
- **Design:** API 202 → planner → per-channel × per-priority Service Bus queues → workers → providers → receipts; inbox + SignalR/Web PubSub.
- **Deep dives:** lane isolation · fan-out · effectively-once · provider failure and tokens.
- **.NET:** `ServiceBusProcessor` manual complete · duplicate detection · scheduled messages · KEDA · Notification Hubs/FCM v1 · ACS · Scriban/Fluid.
- **Traps:** one queue · "exactly once" · fan-out in the request · ignoring invalid tokens · marketing to unsubscribed users.

### Distributed cache
- **Requirements:** get/set/delete/TTL; 1 TB; 1M r/s; p99 < 1 ms; durability not required.
- **Numbers:** ~24 primaries + replicas · memory-bound.
- **Design:** hash slots → nodes; smart clients; async replicas across zones; majority failover; approximated LRU/LFU; lazy + active expiry.
- **Deep dives:** partitioning/rebalancing · failover/split brain · eviction · hot keys/stampedes/invalidation.
- **.NET:** `HybridCache` (per-process semantics) · FusionCache backplane · singleton multiplexer · Azure Managed Redis · Garnet · hash ring with xxHash.
- **Traps:** modulo hashing · synchronous replication by reflex · "cache is consistent" · `string.GetHashCode()`.

### Order / payment system
- **Requirements:** no double charge, no lost orders, no oversell, auditability, minimal PCI scope.
- **Numbers:** ~12/s · ~230/s peak · flash-sale row contention.
- **Design:** order aggregate + state machine + outbox; idempotent checkout API; PSP adapter with deterministic keys; saga (reserve → authorise → confirm → capture); webhooks → inbox; ledger; sweepers; reconciliation.
- **Deep dives:** idempotency at 4 hops · unknown outcomes · saga and compensation · inventory contention · reconciliation.
- **.NET:** EF Core rowversion + outbox in `SaveChanges` · `ExecuteUpdateAsync` in an explicit transaction · Stripe.net `IdempotencyKey` · Durable Functions/Durable Task Scheduler · Service Bus.
- **Traps:** timeout = failure · retries without keys · 2PC across the PSP · read-check-write inventory · card data on your servers.

---
# Appendix C — .NET and Azure choice map

| Need | Default | Choose instead when… |
|---|---|---|
| Stateless HTTP service | Azure Container Apps | App Service for simplest PaaS; AKS when you need cluster-level control (Module 26) |
| Background workers on queues | Container Apps + KEDA | Azure Functions for event-driven, spiky, short work |
| Key–value at scale, point reads | Cosmos DB for NoSQL | Azure SQL / PostgreSQL when relational queries or team skills dominate |
| Relational, transactional core (orders) | Azure SQL Database | PostgreSQL (Flexible Server) by team preference; Hyperscale for very large data |
| Distributed cache | Azure Managed Redis | Valkey/Redis self-hosted for portability; Garnet for extreme per-node throughput |
| In-process + distributed cache API | `HybridCache` | FusionCache when you need a backplane, fail-safe and richer controls |
| Commands, workflows, ordered per key | Azure Service Bus (queues, topics, sessions) | Event Grid for lightweight reactive events |
| High-volume event streams (clicks) | Event Hubs | Kafka (e.g. Confluent Cloud) when it is already the organisation's standard — Event Hubs also speaks the Kafka protocol |
| Durable multi-step workflows | Durable Functions / Durable Task SDKs + Durable Task Scheduler | State machine + outbox when the flow is short; Temporal/Dapr when portability matters |
| Messaging framework | Plain Azure SDK + your own outbox | Wolverine (MIT); NServiceBus or MassTransit v9 (commercial) when their features justify the licence |
| Rate limiting | APIM policies / ASP.NET Core middleware | Redis-backed limiter for fleet-wide, business-aware limits |
| Push | Notification Hubs | APNs + FCM v1 direct for full control |
| Email / SMS | Azure Communication Services | SendGrid / Twilio by features, deliverability or existing contracts |
| Real-time in-app | Azure SignalR Service | Azure Web PubSub for non-SignalR clients and simple pub/sub |
| Edge, TLS, WAF, global entry | Azure Front Door | Application Gateway for regional-only |
| Resilience | `Microsoft.Extensions.Http.Resilience` (Polly v8) | Custom pipelines for non-HTTP calls |
| Observability | OpenTelemetry → Azure Monitor / Application Insights | Any OTLP backend |
| Local orchestration | Aspire | Docker Compose |
| Integration tests | Testcontainers | Emulators (Cosmos, Service Bus, Azurite) where containers don't exist |

---
# Appendix D — Deep-dive menus (in order of usual importance)

```text
URL SHORTENER    1 short-code generation  2 redirect path & caching under skew  3 analytics pipeline
                 4 abuse, expiry, custom aliases  5 multi-region
RATE LIMITER     1 atomic distributed counting  2 failure policy  3 algorithm choice & semantics
                 4 hot keys / token leasing  5 multi-region  6 configuration & rollout
NOTIFICATIONS    1 priority isolation & campaign fan-out  2 effectively-once & crash windows
                 3 provider failure, limits, token hygiene  4 preferences/consent  5 real-time in-app
DISTRIBUTED      1 partitioning & rebalancing  2 replication, failover, split brain
CACHE            3 eviction & memory  4 hot keys & stampedes  5 invalidation & consistency  6 client design
ORDER/PAYMENT    1 idempotency end to end  2 unknown outcomes & reconciliation  3 saga & compensation
                 4 inventory contention  5 ledger design  6 PCI scope & 3-D Secure
```

---
# Appendix E — Self-scoring rubric for a worked system-design round

Score each 1–4 from **observable evidence** (a recording or a partner's notes). Senior target: 3+ everywhere, 4 on at least three rows. Module 38 extends this into a full mock protocol.

| Dimension | 4 — strong hire | 3 — lean hire | 2 — lean no hire | 1 — strong no hire |
|---|---|---|---|---|
| **Requirements** | Asked purpose and the design-shaping NFRs; found the twist; stated scope | Covered functional + some NFRs | Features only | Jumped to boxes |
| **Estimation** | Few numbers, each producing a decision | Correct numbers, some unused | Numbers wrong or irrelevant | None |
| **High-level design** | Every component justified; data flow and consistency per arrow; done by ~22 min | Standard design, mostly justified | Boxes without reasons; late | Incoherent |
| **Deep dives** | Two deep dives where the difficulty is; alternatives, choice, failure modes | One solid deep dive | Shallow everywhere | None |
| **Failure handling** | Each dependency's failure and policy stated unprompted | When asked | Vague ("retry") | Ignored |
| **.NET/Azure implementation** | Precise semantics and limits of the chosen primitives; default → why → when not | Correct products with reasons | Product soup | None or wrong |
| **Pivots / wrap-up** | Seams named; trade-off with a number; next scaling step | Reasonable pivot | Rewrites the design | Unable |
| **Communication** | Drove the round; offered deep-dive choices; checked in | Clear, mostly | Hard to follow | Silent or lost |

```text
Problem: ______  Date: ______  Duration: ____
Req __  Est __  HLD __  Deep __  Fail __  .NET __  Pivot __  Comm __
Time high-level finished: ____   Deep dives chosen: ____________________
Numbers that produced no decision: ____________________
Imprecise .NET claims: ____________________
One fix for next time: ____________________
```

---

*Next: **Module 38 — Mock interview structure and a self-scoring rubric**: how to run realistic mocks for every round type in this curriculum — system design (using this module's five problems), coding, behavioral and architect rounds — how to brief a partner or an AI interviewer, how to score evidence rather than impressions, and how to turn each mock into one targeted fix.*
