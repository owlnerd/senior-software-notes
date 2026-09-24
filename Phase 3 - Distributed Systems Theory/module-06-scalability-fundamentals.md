# Module 6 — Scalability Fundamentals
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

This module is the foundation for everything else in Phase 3. Replication, partitioning, caching, messaging, and reliability patterns (Modules 7–13) are all answers to one question this module asks: *what happens when one machine is no longer enough?* We build the answer one concept at a time, starting with a precise definition of scalability, then the math that explains why systems stop scaling, then the two fundamental strategies (up and out), then the three enabling mechanisms of scale-out: statelessness, load balancing, and autoscaling. Each concept has a .NET/ASP.NET Core angle, because interviewers for .NET-heavy roles routinely drop from "boxes and arrows" into "and how would you actually do that in ASP.NET Core?"

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What scalability means | Scalability is meaningless without a load number and a quality-of-service target. |
| 2 | Why systems stop scaling | Contention and coherency costs, not hardware, set the ceiling. Queueing explains the latency cliff. |
| 3 | Vertical scaling | Underrated. Big boxes are huge, simple, and fast — until the ceiling, the price curve, or the blast radius bites. |
| 4 | Horizontal scaling | Near-linear capacity and redundancy, paid for with distributed-systems complexity. |
| 5 | The Scale Cube | Three independent axes: clone, split by function, split by data. |
| 6 | Choosing | Scale the stateless tier out early; scale the stateful tier up first, then out. |
| 7 | Statelessness | State isn't removed; it's moved to a tier built to hold it. |
| 8 | Stateless ASP.NET Core | Data Protection keys, sessions, hosted services, SignalR, and rate limiters are the usual traps. |
| 9 | Sticky sessions | Affinity is an acceptable optimization and a dangerous correctness requirement. |
| 10 | What a load balancer does | Distribution is only one of its jobs; health, draining, TLS, and protection matter as much. |
| 11 | L4 vs L7 | Connection-level vs request-level balancing. HTTP/2 and gRPC expose the difference. |
| 12 | LB algorithms | Power of two choices is the default to beat; hashing when you need affinity. |
| 13 | Health checks & draining | A health check that can fail fleet-wide is an outage generator. |
| 14 | Global load balancing | DNS, anycast, and geo-routing — and why data, not traffic, is the hard part. |
| 15 | Client-side LB & mesh | In .NET: gRPC client LB, HttpClient DNS pinning, service discovery. |
| 16 | Autoscaling | Choose the right signal, respect warm-up time, and don't DDoS your own database. |
| 17 | Protecting the system | Scaling out has a lag; backpressure and shedding cover the gap. |
| 18 | The data tier | Once compute is stateless, every scaling problem lands on your data stores. |

---

## Concept 1 — What "scalability" actually means

Scalability is a system's ability to handle increased load by adding resources, such that throughput grows roughly in proportion to the resources added and quality of service — latency percentiles, error rate — stays within target. Two phrases in that definition do the heavy lifting. "Load" must be described with numbers, and "within target" means there's an SLO. A system isn't "scalable" in the abstract; it scales from one load to another while holding an objective.

It helps to separate scalability from three neighbors it's often confused with. A **performance** problem means the system is slow for a single user at low load; a **scalability** problem means it's fast for one user but degrades as load increases. The fixes differ: the first is profiling and algorithmic work, the second is architecture. **Elasticity** is the ability to add and remove capacity quickly and automatically in response to demand. A sharded database can be scalable without being elastic, because resharding may take days. **Availability** is about continuing to serve through failures. Horizontal scaling often delivers scalability and availability through the same mechanism (many replicas), which is why candidates conflate them, but they're different goals: you can have a highly available system that doesn't scale (an active-passive pair of large servers) and a scalable one that isn't highly available (a large fleet behind a single, non-redundant load balancer).

**Load parameters.** Before discussing scaling, describe the load in the dimensions that matter for *this* system: requests per second at average and peak, the read/write ratio, concurrent connections, payload sizes, data volume and growth rate, fan-out per request, and skew (are 1% of keys receiving 50% of traffic?). The classic illustration from Kleppmann's *Designing Data-Intensive Applications* is a social timeline: the hard load parameter isn't posts per second, it's fan-out — how many followers each post must reach — which drives the choice between computing timelines on read and precomputing them on write.

**Dimensions of scale.** Load scalability is what most people mean, but senior-level interviewers listen for awareness of the others:

| Dimension | The question | Typical answer |
|---|---|---|
| Load | Can we handle 10× the requests? | Replicas, load balancing, caching |
| Data | Can we handle 10× the stored data? | Partitioning, tiered storage, archiving |
| Geographic | Can we serve distant users with low latency? | CDNs, edge, multi-region |
| Organizational | Can 10× the engineers work on it without colliding? | Modular boundaries, service decomposition (Conway's Law) |
| Tenancy / operational | Can we onboard 10× the customers without 10× the ops toil? | Automation, cells/stamps, self-service |

**Measure the right thing.** Throughput and latency are the primary metrics, and latency must be discussed as percentiles, not averages. Averages hide the experience of your worst-served users, and at scale the tail dominates. The reason is fan-out: if serving one page requires calls to 100 backends and each is slow 1% of the time, the probability that at least one is slow is 1 − 0.99¹⁰⁰ ≈ 63%. A p99 problem at the leaves becomes a median problem at the root. This is the central argument of Dean and Barroso's *The Tail at Scale*, and it's why "we'll scale out to more backends" can make latency worse unless you also manage tail latency.

**How this is asked.** "Design a system that scales" is a trap if you don't pin it down. A senior answer converts it into a statement like: "We need 50k RPS at peak, 95% reads, p99 under 200 ms, with 2× headroom over forecast peak." Everything after that is justified against those numbers.

---

## Concept 2 — Why systems stop scaling: the math

You'll rarely be asked to derive these, but using them fluently is one of the clearest signals separating senior candidates from people who've memorized architecture diagrams. Four models explain almost every scaling ceiling you'll meet.

### 2a. Amdahl's Law — the serial fraction caps your speedup

If a fraction *p* of the work can be parallelized and (1 − *p*) is inherently serial, the maximum speedup on *N* workers is:

```
S(N) = 1 / ((1 − p) + p / N)
```

With 95% parallelizable work, 16 workers yield about 9.1×, and infinite workers yield at most 20×. The 5% serial part sets a hard ceiling. In web systems, the "serial fraction" is anything every request must pass through one at a time: a single database primary for writes, a global lock, a hot row, a single Redis key used as a counter, a sequence generator, or a leader node. When you hear yourself say "all requests go through X," you've found your Amdahl bottleneck.

### 2b. The Universal Scalability Law — why adding nodes can make things *worse*

Neil Gunther's Universal Scalability Law (USL) extends Amdahl with a second penalty:

```
C(N) = N / (1 + α(N − 1) + βN(N − 1))
```

*C(N)* is relative capacity with *N* nodes (or threads, or concurrent users). The α term is **contention**: queueing for shared resources — Amdahl's serial fraction. The β term is **coherency**: the cost of keeping nodes consistent with each other — cache-invalidation broadcasts, cross-node coordination, distributed locks, gossip, chatty replication. With β = 0 the USL reduces to Amdahl. Because the coherency term grows with *N²*, throughput doesn't just flatten; it peaks and then declines. Gunther calls this retrograde scaling, and it's the formal explanation for "we added servers and it got slower."

The peak is at *N\** = √((1 − α) / β). A worked example makes this concrete: with α = 0.03 and β = 0.0005, the peak is at about 44 nodes, where capacity is only about 13.6× a single node. Double the fleet to 88 nodes and capacity *falls* to about 11.8×. The practical use is empirical: run load tests at 1, 2, 4, 8, and 16 instances (or concurrency levels), fit α and β by regression, and you can predict where your system tops out before you pay to find out in production.

The .NET translation: a `lock` around a shared `Dictionary` on a hot path is an α problem inside one process. A cache-invalidation message fanned out to every instance on every write is a β problem across the fleet. The design lesson is that scaling out well requires minimizing both shared bottlenecks and cross-node agreement — which is exactly why the rest of Phase 3 obsesses over partitioning and consistency trade-offs.

### 2c. Little's Law — how latency turns into a capacity problem

For any stable system, the average number of requests in flight *L* equals the arrival rate *λ* times the average time each spends in the system *W*:

```
L = λ × W
```

At 2,000 RPS with 50 ms average latency, the system holds about 100 requests in flight at any moment. Now let a downstream database slow to 500 ms: in-flight requests jump to 1,000. Traffic didn't change, but you now need ten times the concurrency — threads, sockets, memory, and database connections. In .NET this is where things break in practice: `SqlClient` and Npgsql both default to a maximum pool size of 100 connections per connection string, so requests start queueing for connections, latency rises further, and the loop feeds itself. Little's Law is why slow dependencies cause outages in services with plenty of spare CPU, and it's the right tool for sizing thread pools, connection pools, and concurrency limits.

### 2d. Queueing theory — the utilization cliff

For a simple single-server queue (M/M/1), mean response time *R* relates to service time *S* and utilization *ρ* as:

```
R = S / (1 − ρ)
```

| Utilization ρ | Response time as a multiple of service time |
|---|---|
| 50% | 2× |
| 70% | 3.3× |
| 80% | 5× |
| 90% | 10× |
| 95% | 20× |
| 99% | 100× |

The curve is flat and then vertical. This is why latency-sensitive services are planned at 60–70% utilization rather than 90%, while batch systems (where throughput matters and latency doesn't) can run hot. It's also why "CPU is only at 75%, we're fine" can be wrong: at 75% you're on the steep part of the curve, and a small traffic increase produces a large latency increase.

One refinement connects directly to load balancing: *c* servers fed from one shared queue (M/M/c) behave much better than *c* servers each with its own queue. Pooling lets you run at higher utilization for the same latency. A load balancer that sends work to the least-busy server approximates a shared queue; one that assigns work blindly approximates separate queues. That's the theoretical reason good load-balancing algorithms matter (Concept 12).

---

## Concept 3 — Vertical scaling (scale up)

Vertical scaling means handling more load by giving a single machine more resources: more cores, more memory, faster disks, a faster network. It's the simplest scaling strategy and, in interviews, the most underrated.

**What it buys you.** No code changes. No distributed-systems problems: no network partitions between components, no distributed transactions, and strong consistency stays trivial because there's one copy of the data. In-process and in-memory operations are orders of magnitude faster than network calls (a main-memory reference is on the order of 100 ns; a round trip inside a datacenter is a few hundred microseconds — thousands of times slower). Operations stay simple: one thing to monitor, patch, and back up.

**Modern hardware is enormous.** Cloud VM families offer hundreds of vCPUs and terabytes of RAM, and local NVMe drives deliver hundreds of thousands to millions of IOPS. A surprising number of "we need to distribute this" problems fit on one large machine. The HotOS 2015 paper *Scalability! But at what COST?* (McSherry, Isard, Murray) made the point memorably: well-written single-threaded implementations on a laptop outperformed many published distributed graph-processing systems running on clusters. The lesson isn't "never distribute" — it's that distribution has overhead that must be paid back before it wins.

**The .NET case study everyone should know.** Stack Overflow, one of the highest-traffic sites on the internet and a .NET/SQL Server shop, famously ran on a small number of powerful servers rather than a large fleet. Nick Craver's 2016 architecture write-up lists 11 IIS web servers, 4 SQL Servers, 2 Redis servers, and 4 HAProxy load balancers. It's the canonical proof that aggressive performance engineering plus vertical scaling goes a very long way — and a great story to cite when an interviewer pushes you toward premature microservices.

**The limits.** Vertical scaling hits five walls. First, a hard ceiling: there's a largest SKU, and if you need more, you're re-architecting under pressure. Second, a superlinear price curve at the top end, made worse by per-core licensing (SQL Server Enterprise is licensed per core, so a bigger box multiplies license cost, not just hardware cost). Third, blast radius: one big box is one big failure domain, so you need a standby anyway — and now you're paying for two big boxes. Fourth, resizing usually means downtime, since resizing a VM typically requires a restart. Fifth, diminishing returns inside the box: Amdahl and the USL apply within a machine too. Lock contention, memory bandwidth, and NUMA effects (memory attached to another socket is slower to reach) mean 128 cores rarely deliver 8× the throughput of 16.

**The .NET angle on big boxes.** The runtime has to be configured for the hardware. Server GC creates a heap and a GC thread per logical core by default — great for throughput on a big dedicated server, wasteful for many small containers. DATAS (Dynamic Adaptation To Application Sizes) arrived as an opt-in in .NET 8 and became the default in .NET 9. It keeps heap size roughly proportional to the application's long-lived data instead of growing aggressively the way classic Server GC does, adjusting the number of heaps as demand changes. Microsoft's own benchmark on a 48-core Linux machine showed a working-set reduction of over 80% for a 2–3% drop in peak RPS. On very large machines you'll also look at NUMA-aware settings, heap affinitization, and heap hard limits. Module 14 goes deep; for now the interview point is that "scale up" isn't free of configuration work — the runtime must be tuned to use the machine well.

**When vertical is the right answer.** For the stateful tier early in a system's life (the primary database is the classic example), for workloads needing strong consistency or large in-memory working sets, for small teams that can't afford distributed-systems complexity, and as the first move while you design the scale-out path. A senior answer often sounds like: "I'd scale the database vertically and add read replicas; I wouldn't shard until writes exceed what the largest instance can handle with headroom, because sharding is a one-way door."

---

## Concept 4 — Horizontal scaling (scale out)

Horizontal scaling means handling more load by adding more machines and spreading work across them. It's the default for internet-scale systems because it's the only strategy without a hard ceiling.

**What it buys you.** Near-linear capacity growth for work that partitions cleanly. Fault tolerance through redundancy: with *N* instances, losing one costs 1/*N* of capacity instead of everything. Elasticity: commodity instances can be added and removed in minutes. Zero-downtime deployments via rolling updates, blue/green, and canaries, all of which require more than one instance. And cost efficiency at scale, since small instances are priced roughly linearly while the largest SKUs carry a premium.

**What it costs you.** Everything above requires machinery you didn't need before. You need a load balancer (which must itself be redundant). Instances must be stateless or state must be partitioned, which means a shared state tier (cache, database, blob storage) that becomes the new bottleneck. You inherit the fallacies of distributed computing — the network isn't reliable, latency isn't zero, bandwidth isn't infinite — and with them, partial failure: some instances healthy, some slow, some dead, some partitioned. Observability gets harder because one request now touches many processes. And coordination (locks, leader election, exactly-once processing) becomes a real design problem instead of a `lock` statement.

**Capacity planning becomes failure planning.** With horizontal scaling, you size for the failure case, not the happy case. If you run across three availability zones and want to survive losing one, the remaining two must carry full peak load at an acceptable utilization. That means provisioning 150% of the no-failure requirement (each zone carries one-third normally, one-half during a zone outage). This "N+1 at the zone level" reasoning is exactly the kind of detail that marks a senior answer; we'll do the arithmetic in the worked example.

---

## Concept 5 — The Scale Cube: three ways to scale out

"Scale out" isn't one technique. The AKF Scale Cube, from Abbott and Fisher's *The Art of Scalability*, names three independent axes, and using this vocabulary gives your interview answers structure.

The **X-axis is horizontal duplication**: run *N* identical copies of the application behind a load balancer. Each copy can serve any request. This is what most people mean by scaling out; it's easy for stateless services, and it doesn't help with data volume, because every copy still talks to the same database.

The **Y-axis is functional decomposition**: split the system by function or noun — orders, catalog, payments, identity — so each piece can scale, deploy, and fail independently. This is the microservices axis, and it's as much about organizational scalability as traffic (Module 21 covers when it's worth it).

The **Z-axis is data partitioning**: identical code, but each instance or group of instances is responsible for a subset of the data, chosen by a key like customer ID, tenant, or region. Sharding is the Z-axis applied to a database. Applied to the whole stack, it becomes a *cell-based* or *deployment stamp* architecture: complete, independent copies of the entire system, each serving a subset of tenants. Stamps cap the size any single deployment reaches, limit blast radius (a bad deploy or noisy tenant affects one stamp), and turn "scale" into "add another stamp."

Real systems use all three. A mature SaaS platform might run replicas of each service (X), split into bounded-context services (Y), deployed as regional stamps each serving a set of tenants (Z).

---

## Concept 6 — Choosing between vertical and horizontal

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| Ceiling | Hard limit: largest available machine | Effectively unbounded (limited by shared dependencies) |
| Code changes | None | Requires statelessness or partitioning |
| Consistency | Trivial (single copy) | Must be designed (Module 7) |
| Fault tolerance | Single failure domain; needs a standby | Built in via redundancy |
| Elasticity | Poor; resizing usually restarts | Good; add/remove instances in minutes |
| Cost curve | Superlinear at the top; per-core licenses multiply | Roughly linear, plus LB and coordination overhead |
| Latency | Excellent (in-process, in-memory) | Adds network hops |
| Operational complexity | Low | High (LB, discovery, observability, deploys) |

The pragmatic default most experienced architects converge on is to **scale the stateless tier out from day one and scale the stateful tier up first**. Stateless web and API tiers should run at least two instances anyway for availability, so they're already horizontal and cost nothing extra to scale further. Stateful stores go the other way: scale up, then add read replicas and caching to absorb reads, and only partition when write throughput or data volume genuinely exceeds a single large node with headroom — because partitioning is expensive to undo. Combining the two — sizing each instance up to a sweet spot, then scaling out with more of them — is sometimes called diagonal scaling. It suits .NET services well, since each process benefits from a warm JIT, larger in-memory caches, and fewer total connections to shared stores.

In an interview, the strongest move is to state the trade-off explicitly and anchor it to your Concept 1 numbers: "At 5k writes/sec the primary fits on a single large instance with 3× headroom, so I'd defer sharding and revisit at 15k."

---

## Concept 7 — Statelessness: the prerequisite for scaling out

A service is stateless when any instance can serve any request, because no instance holds data that must survive beyond a single request or that another instance would need to handle the next one. Statelessness is what makes X-axis scaling trivial: the load balancer can send any request anywhere, instances can be killed and replaced at will, and autoscaling can add or remove capacity without migrating anything.

The critical insight is that **state is never eliminated — it's moved** to a tier purpose-built to hold it: a database, a distributed cache, blob storage, or a message broker. Those tiers are designed for durability, replication, and concurrent access; your application servers are not. The Twelve-Factor App states this as "processes are stateless and share-nothing," paired with "disposability": processes should start fast and shut down gracefully, because the platform will move them around.

State hides in more places than session variables. Here's where it typically lives in an ASP.NET Core application and where it belongs:

| State type | Where it hides in an ASP.NET Core app | Where it should live |
|---|---|---|
| Session / user state | `ISession` backed by `AddDistributedMemoryCache` (per-process despite the name); static dictionaries keyed by user | Redis or SQL via `IDistributedCache`; or push it to the client (signed cookies/tokens) |
| Auth cookie & antiforgery encryption keys | Data Protection key ring on the local filesystem | Shared key ring (Blob Storage + Key Vault, Redis, database) |
| Cache | `IMemoryCache`, static caches | Per-instance L1 is fine if it's purely an optimization; add a shared L2 (Redis) via `HybridCache` |
| Files | Uploads and generated files on local disk | Blob/object storage |
| Background work & timers | `IHostedService` / `BackgroundService` running on every instance | Queue with competing consumers, a clustered scheduler, or a distributed lease |
| Real-time connections | SignalR hubs holding connections in memory | Redis backplane or Azure SignalR Service |
| Rate-limit counters | ASP.NET Core rate limiting middleware (in-process) | Per-instance limits scaled to the fleet, or a distributed limiter at the gateway |
| Locks and leadership | `lock`, `SemaphoreSlim`, "only instance 0 does X" | Distributed locks with fencing, leases (Module 9) |
| Idempotency / dedup records | In-memory "seen" sets | Database or cache with TTL |
| Workflow state | In-memory state machines | Database, durable orchestrations, sagas (Module 12) |

**A useful test**: an instance is stateless if you could kill it mid-traffic, replace it with a fresh one, and no user would notice anything beyond a retried request. If killing an instance logs users out, loses uploads, drops a scheduled job, or resets a rate limit, you have hidden state.

**Caches are the nuance.** An in-process cache doesn't break statelessness *if correctness doesn't depend on it* — if a request served by a cold instance gets the same answer, just slower. What breaks is when a cache becomes a source of truth, or when per-instance caches diverge long enough that users see contradictory data as the load balancer bounces them between instances.

**The counterpoint senior candidates should know.** Stateless-everything over a shared database isn't the only valid architecture. Some workloads scale better with deliberately **stateful, partitioned ownership**: each entity (a chat room, a game match, a co-edited document, a trading account) is owned by exactly one instance at a time, requests for it are routed to its owner by key, and the owner keeps the entity's state in memory. This eliminates round trips to a shared store and serializes access to each entity without distributed locks. In .NET, Microsoft Orleans is the flagship implementation: "virtual actors" (grains) are activated on demand on some silo, located via a distributed directory, and reactivated elsewhere on failure. The cost is that routing, rebalancing, and failover become your problem (or your framework's). Knowing when to reach for this — high-contention per-entity state with tight latency requirements — is a strong senior signal. It pairs with consistent hashing (Concept 12 and Module 8).

---

## Concept 8 — Making ASP.NET Core stateless in practice

This is where .NET interviews often go concrete: "You've got an ASP.NET Core app on one server. Make it run on ten." Here are the traps, roughly in the order they bite.

### 8a. Data Protection keys — the silent killer

ASP.NET Core uses the Data Protection system to encrypt authentication cookies, antiforgery tokens, TempData, and session cookies. By default the key ring lives on the local filesystem. In a container or on a fresh VM, each instance generates its own keys. The result: a user logs in on instance A, their next request lands on instance B, B can't decrypt the cookie, and the user is silently logged out — or form posts fail antiforgery validation intermittently. It's intermittent because it depends on load balancer placement, which makes it maddening to debug. (Azure App Service shares the key ring across instances of the same app automatically, which is why teams sometimes discover this only when moving to containers.)

The fix is a shared, protected key ring and a fixed application name:

```csharp
// Packages: Azure.Extensions.AspNetCore.DataProtection.Blobs,
//           Azure.Extensions.AspNetCore.DataProtection.Keys, Azure.Identity
var credential = new DefaultAzureCredential();

builder.Services.AddDataProtection()
    .SetApplicationName("orders-api") // identical on every instance
    .PersistKeysToAzureBlobStorage(
        new Uri(builder.Configuration["DataProtection:KeyBlobUri"]!), credential)
    .ProtectKeysWithAzureKeyVault(
        new Uri(builder.Configuration["DataProtection:KeyVaultKeyId"]!), credential);
```

Redis (`PersistKeysToStackExchangeRedis`) and EF Core (`PersistKeysToDbContext`) are alternatives. The principle: every instance shares the same keys, and the keys are protected at rest.

### 8b. Session and distributed cache

`AddSession` stores data in whatever `IDistributedCache` is registered. `AddDistributedMemoryCache` is an in-process implementation meant for development and single-server use — nothing about it is distributed. For multi-instance deployments, register a real distributed cache:

```csharp
builder.Services.AddStackExchangeRedisCache(o =>
{
    o.Configuration = builder.Configuration.GetConnectionString("redis");
    o.InstanceName = "orders-api:";
});

builder.Services.AddSession(o =>
{
    o.IdleTimeout = TimeSpan.FromMinutes(20);
    o.Cookie.HttpOnly = true;
    o.Cookie.IsEssential = true;
});
```

Better still, question whether you need server-side session at all. Many modern APIs carry identity in tokens and keep durable user data in the database, with no session store.

### 8c. Caching across instances: HybridCache and L1 staleness

`HybridCache` (Microsoft.Extensions.Caching.Hybrid, from the .NET 9 wave) gives you a two-level cache: an in-process L1 plus the registered `IDistributedCache` as L2, with built-in stampede protection so concurrent misses for the same key trigger a single factory call per instance. It's the right default for multi-instance caching in modern .NET. Understand its limitation: each instance's L1 is independent, so after an invalidation other instances may serve stale L1 data until it expires. Keep L1 expirations short where cross-instance staleness matters, or use a library with a backplane for L1 invalidation (FusionCache is a popular community option). Module 10 goes deep on invalidation.

### 8d. Background services run on every instance

A `BackgroundService` that generates a nightly report or polls an outbox runs on *every* replica. Scale to ten instances and the report runs ten times. It's one of the most common real-world bugs when scaling out. The options, roughly from most to least preferred: turn the work into messages on a queue so instances act as competing consumers (work divides naturally, and scaling consumers scales throughput); use a scheduler with clustering (Quartz.NET with a clustered job store, Hangfire) so each job fires once; or guard the job with a distributed lock or lease (a PostgreSQL advisory lock, an Azure Blob lease, a Redis lock with a fencing token). Running the job in a separate single-replica deployment works but reintroduces a single point of failure. Module 9 covers why distributed locks need fencing tokens to be safe.

### 8e. SignalR and WebSockets

SignalR holds each client's connection in the memory of the server that accepted it. Scaling out raises two problems. Messages sent from server A must reach clients connected to server B, which requires a backplane (Redis) or Azure SignalR Service, which offloads connections entirely. And the negotiate-then-connect handshake requires sticky sessions unless every client uses WebSockets with negotiation skipped. Long-lived connections also create a rebalancing problem: when you scale out, existing connections stay where they are, so new instances only receive new connections. Plan for connection recycling or accept slow rebalancing.

### 8f. Rate limiting is per instance

The built-in ASP.NET Core rate limiting middleware keeps its counters in process memory. A limit of 100 requests per minute per user across ten instances allows up to roughly 1,000 per minute, depending on how the balancer spreads that user. Either divide the budget by instance count (approximate, and it breaks as autoscaling changes the count), enforce quotas at a gateway that sees all traffic, or use a distributed limiter backed by Redis. The per-instance middleware remains excellent for protecting each instance from overload (its concurrency limiter), which is a different goal from enforcing per-customer quotas.

### 8g. Smaller traps

**Forwarded headers**: behind a load balancer or reverse proxy, the app sees the proxy's IP and scheme unless you enable the Forwarded Headers middleware and configure which proxies to trust; otherwise redirects go to `http://` and client-IP logic uses the proxy's address. **`string.GetHashCode()` is randomized per process** in .NET Core and later, so two instances compute different hashes for the same string. Never use it for anything that must agree across instances — sharding, routing, cache keys, idempotency keys. Use a stable hash like `XxHash64` from System.IO.Hashing. **Static mutable state** — static dictionaries, singletons holding per-user or per-tenant data — is per-process by definition. **Local file writes** vanish when a container is replaced. **Auth tokens**: JWT access tokens are self-contained, so any instance can validate them without a session lookup, which scales well; the cost is revocation, handled with short token lifetimes plus refresh tokens, or a denylist — which reintroduces shared state, but only for the rare revoked token.

---

## Concept 9 — Sticky sessions (session affinity)

Sticky sessions route all requests from a given client to the same backend instance. The load balancer implements it by inserting a cookie identifying the chosen backend (Azure App Service calls this "ARR affinity"; YARP supports it through its `SessionAffinity` configuration), by hashing the client's source IP, or by hashing a header.

Teams reach for affinity because it makes a stateful app "work" on multiple instances without refactoring — which is also why it's a design smell. The problems are well known. Load becomes uneven, because the balancer can no longer choose freely: a few heavy users pinned to one instance can overload it while others idle. Autoscaling doesn't rebalance existing sessions, so new instances only pick up new users. When an instance dies or is drained for a deployment, everyone pinned to it loses their state. Source-IP affinity is especially fragile: thousands of mobile users behind a carrier-grade NAT share one IP and all land on one server, while a user whose IP changes mid-session (Wi-Fi to cellular) gets moved.

There are legitimate uses. SignalR's negotiate handshake with non-WebSocket transports needs it. Soft affinity for cache locality is useful — routing a user to the same instance so their data is warm in L1, while still producing correct results elsewhere. Partitioned stateful ownership (Concept 7) is affinity by *entity key*, not by client, and it's a deliberate architecture rather than a crutch. And during a brownfield migration, affinity can be a temporary bridge while you externalize state.

The senior position in one sentence: **affinity may be an optimization, never a correctness requirement** — the system must still produce correct results when a request lands on a different instance.

---

## Concept 10 — What a load balancer actually does

A load balancer is often described as "the thing that spreads requests across servers," but distribution is only one of its jobs. It also performs **health checking** (probing backends and removing unhealthy ones), **failover** (routing around failed instances, zones, or regions), **TLS termination** (decrypting once at the edge so backends don't each pay the handshake cost, and centralizing certificates), **connection management** (holding pooled keep-alive connections to backends and buffering slow clients, which protects your app from slow-client attacks), **routing** (by host, path, header, or weight for canaries), **resilience** (timeouts, retries, outlier ejection), **security** (WAF, DDoS absorption), and **observability** (a single choke point where every request can be measured).

In a real system, load balancing happens at several layers, each with a different job:

```
Client
  │  DNS / GSLB (Azure Traffic Manager, Route 53)       → chooses a region
  ▼
Global L7 edge, anycast (Azure Front Door, Cloudflare)  → TLS, WAF, CDN, fast region failover
  ▼
Regional L7 (Application Gateway, K8s ingress, YARP)    → host/path routing, per-request balancing
  ▼
Regional L4 (Azure Load Balancer, kube-proxy)           → per-connection spreading
  ▼
Application instances
  │  client-side LB or service mesh                     → service-to-service balancing
  ▼
Internal services
```

A load balancer must itself scale and must not be a single point of failure. Cloud load balancers handle this for you — they're distributed systems behind a single address. Self-hosted ones need redundancy, typically an active-passive pair sharing a virtual IP, or multiple active instances behind DNS or an L4 tier.

---

## Concept 11 — Layer 4 vs Layer 7 load balancing

The OSI layer at which a load balancer makes decisions determines what it can see and what it can do.

A **Layer 4 (transport) load balancer** works with TCP and UDP. It sees IP addresses and ports — the connection's 5-tuple — and makes one decision per *connection*: once a TCP connection is assigned to a backend, every byte on it goes there. It doesn't parse HTTP, so it can't route by path or header, can't retry a failed HTTP request, and typically passes TLS through untouched. In exchange it's extremely fast, handles enormous connection rates, adds microseconds of latency, and works for any TCP/UDP protocol (databases, MQTT, game traffic). Many L4 balancers hash the 5-tuple to pick a backend, and some use direct server return, where responses bypass the balancer entirely.

A **Layer 7 (application) load balancer** terminates the client connection, parses the HTTP request, and makes a decision per *request*. It can route `/api/orders` to one pool and `/api/catalog` to another, split traffic by weight for canaries, inject headers, retry idempotent requests, compress, cache, and apply a WAF. The costs are CPU (TLS and HTTP parsing), a little more latency, and the fact that it's a full proxy that must scale with your traffic.

| | Layer 4 | Layer 7 |
|---|---|---|
| Decision unit | Per connection | Per request |
| Sees | IPs, ports, protocol | Full HTTP: method, path, headers, cookies, body |
| TLS | Usually passthrough | Usually terminates (can re-encrypt to backends) |
| Routing | By address/port/hash | By host, path, header, weight |
| Retries, header rewriting, WAF | No | Yes |
| Performance | Very high throughput, very low latency | Lower; CPU-bound on TLS and parsing |
| Protocols | Any TCP/UDP | HTTP/1.1, HTTP/2, HTTP/3, gRPC, WebSockets |
| Azure | Azure Load Balancer | Application Gateway (regional), Front Door (global) |
| AWS | Network Load Balancer | Application Load Balancer |
| Self-hosted | HAProxy (TCP mode), IPVS, Envoy | NGINX, HAProxy, Envoy, YARP |

Azure's own guidance classifies its options along two axes: global versus regional, and HTTP(S) versus non-HTTP(S). Azure Load Balancer is regional L4; Application Gateway is regional L7 with WAF; Front Door is global L7 with anycast, CDN, and WAF; Traffic Manager is global DNS-based routing.

**The HTTP/2 and gRPC trap.** This is a favorite deep-dive question because it tests whether you really understand the difference. HTTP/2 multiplexes many requests as streams over a single long-lived TCP connection, and gRPC runs on HTTP/2. An L4 balancer assigns that connection to one backend once, so *all* requests from that client go to one backend for the life of the connection. With a few clients and many servers, most servers sit idle while a few are overloaded — and scaling out doesn't help, because new servers get no connections. In Kubernetes, a standard ClusterIP Service is implemented by kube-proxy at L4, so gRPC services behind one are routinely unbalanced. The fixes: put an L7 proxy or service mesh in the path (Envoy, Linkerd, Istio) that balances individual streams; use client-side load balancing, where the client opens connections to every backend and spreads calls across them (in .NET, `Grpc.Net.Client` supports this with a DNS resolver pointed at a headless Service — see Concept 15); or periodically recycle connections so they redistribute. The same "long-lived connections don't rebalance" problem applies to WebSockets and database connection pools.

---

## Concept 12 — Load balancing algorithms

The algorithm decides which backend receives each request or connection. There are three families: **static** algorithms that ignore backend load, **dynamic** algorithms that use load signals, and **hash-based** algorithms that deliberately send related requests to the same place. We'll build up from the simplest.

### 12.1 Round robin

Cycle through backends in order. It's simple, stateless apart from a counter, and perfectly fair *in request count*. Its weakness is blindness: if requests vary in cost (a 5 ms lookup next to a 2-second report) or backends vary in capacity or health (one is in a GC pause, one has a noisy neighbor), round robin keeps sending each an equal share. It works well when requests are homogeneous and backends identical.

### 12.2 Weighted round robin

Assign each backend a weight proportional to its capacity, and send proportionally more traffic to heavier ones. This handles mixed hardware, and it's how canary releases get 5% of traffic. The naive implementation (send *w* consecutive requests to each backend) produces bursts. NGINX's **smooth weighted round robin** interleaves them instead, and it's a neat algorithm worth knowing:

```csharp
// Smooth weighted round-robin (NGINX's algorithm).
// Weights A=5, B=1, C=1 produce A A B A C A A — interleaved, no bursts.
public sealed class SmoothWeightedRoundRobin<T>
{
    private sealed class Node(T item, int weight)
    {
        public T Item { get; } = item;
        public int Weight { get; } = weight;
        public int Current;
    }

    private readonly Node[] _nodes;
    private readonly int _totalWeight;
    private readonly Lock _gate = new(); // System.Threading.Lock (.NET 9+)

    public SmoothWeightedRoundRobin(IEnumerable<(T Item, int Weight)> items)
    {
        _nodes = items.Select(i => new Node(i.Item, i.Weight)).ToArray();
        _totalWeight = _nodes.Sum(n => n.Weight);
    }

    public T Next()
    {
        lock (_gate)
        {
            Node best = _nodes[0];
            foreach (var n in _nodes)
            {
                n.Current += n.Weight;          // everyone earns their weight
                if (n.Current > best.Current) best = n;
            }
            best.Current -= _totalWeight;       // the winner pays the total
            return best.Item;
        }
    }
}
```

Each round, every node "earns" its weight, and the richest node is chosen and "pays" the total. Over a full cycle each node is chosen exactly in proportion to its weight, with choices spread as evenly as possible.

### 12.3 Random

Pick a backend uniformly at random. It needs no shared state at all, which makes it attractive when many independent balancers (or clients) decide in parallel. Over many requests it evens out, but at any moment the imbalance is significant: throw *n* requests at *n* servers at random, and the busiest server ends up with Θ(log *n* / log log *n*) of them with high probability.

### 12.4 Least connections and least outstanding requests

Send each request to the backend with the fewest active connections — or better, the fewest in-flight requests, since with HTTP keep-alive and HTTP/2, connection count is a poor proxy for load. This adapts automatically to uneven request costs and slow backends: a backend stuck on slow requests accumulates in-flight work and receives less.

It has three failure modes worth naming in an interview. **The new-instance flood**: a freshly started instance has zero connections, so least-connections sends it a burst while its JIT is cold and its caches empty; the mitigation is slow start, which ramps new backends up gradually. **The black hole**: a broken instance that fails every request *instantly* always has the fewest in-flight requests, so it attracts more traffic and fails more of it; the Amazon Builders' Library describes this and suggests slowing failed responses to match normal latency, and outlier detection ejects such instances. **Herding**: with several independent balancers each holding a slightly stale view of backend load, they all see the same "least loaded" server at the same moment and pile onto it, overload it, then all move to the next. This last problem is exactly what the next algorithm solves.

### 12.5 Power of two random choices (P2C)

Pick two backends at random and send the request to whichever has fewer in-flight requests. That's the whole algorithm, and it's one of the most useful results in distributed systems.

The math is striking. Where purely random placement leaves the busiest server with about log *n* / log log *n* requests, choosing the less loaded of two random candidates cuts that to about log log *n* — an exponential improvement from one extra random sample. A third or fourth choice helps only by a constant factor, so two captures almost all the benefit.

Why is it better than least-connections, which looks at *every* server? Because of herding. Mitzenmacher's work, explained accessibly in Marc Brooker's blog post, showed that when load information is stale, "always pick the least loaded" performs badly as everyone converges on the same target, while "best of two random" stays robust: the randomness breaks symmetry between independent balancers, and the comparison still steers traffic away from the worst servers. It's also cheaper: O(1) per decision instead of scanning every backend.

This isn't academic. P2C is YARP's default policy when none is configured. HAProxy's `random` algorithm defaults to two draws, and HAProxy's authors noted that its advantage over least-connections shows up specifically when many balancers decide independently — the service-mesh case, where every sidecar is its own balancer. Envoy's least-request algorithm also samples two hosts by default.

A minimal implementation shows how little it needs:

```csharp
public sealed class PowerOfTwoChoicesBalancer
{
    public sealed class Backend(Uri address)
    {
        public Uri Address { get; } = address;
        internal int InFlight;
    }

    // Disposing the lease decrements the backend's in-flight count.
    public readonly struct Lease : IDisposable
    {
        private readonly Backend _backend;
        internal Lease(Backend backend) => _backend = backend;
        public Uri Address => _backend.Address;
        public void Dispose() => Interlocked.Decrement(ref _backend.InFlight);
    }

    private readonly Backend[] _backends;

    public PowerOfTwoChoicesBalancer(IEnumerable<Uri> addresses) =>
        _backends = addresses.Select(a => new Backend(a)).ToArray();

    public Lease Acquire()
    {
        int n = _backends.Length;
        Backend chosen;
        if (n == 1)
        {
            chosen = _backends[0];
        }
        else
        {
            int i = Random.Shared.Next(n);
            int j = Random.Shared.Next(n - 1);
            if (j >= i) j++;                                  // two distinct candidates
            Backend a = _backends[i], b = _backends[j];
            chosen = Volatile.Read(ref a.InFlight) <= Volatile.Read(ref b.InFlight) ? a : b;
        }
        Interlocked.Increment(ref chosen.InFlight);
        return new Lease(chosen);
    }
}

// Usage:
// using var lease = balancer.Acquire();
// var response = await http.GetAsync(new Uri(lease.Address, "/orders/42"));
```

A production version would filter unhealthy backends, weight by capacity, and ramp new instances in slowly, but the core is exactly this.

### 12.6 Latency-aware and server-reported load

In-flight counts only capture load the balancer itself created. Latency-aware algorithms track an exponentially weighted moving average (EWMA) of each backend's response time and prefer faster ones; "peak EWMA," used by Finagle and Linkerd, reacts quickly to latency spikes and slowly to recoveries — the safe asymmetry. Other systems have backends report their own utilization in response headers, so the balancer sees load from *all* sources, including other balancers. Netflix's edge load balancing combined P2C with server-reported utilization and error-aware filtering for exactly this reason. These signals are usually layered on top of P2C rather than replacing it.

### 12.7 Hash-based algorithms: when you *want* affinity

Sometimes you want the same key to go to the same backend every time: a cache node holding that key's data, a partition owner in a stateful system, or a shard. Hashing the key does this, but *how* you hash matters enormously when the set of backends changes.

**Modulo hashing** (`hash(key) % N`) is simple and catastrophic under change: going from 9 to 10 nodes remaps about 90% of keys, which for a cache tier means a near-total miss storm. **IP hash** is modulo hashing on the client address and inherits both this problem and the NAT problem from Concept 9.

**Consistent hashing** places nodes and keys on a ring and assigns each key to the next node clockwise. Adding or removing a node moves only the keys in its segment, about 1/*N* of the total. Virtual nodes (placing each physical node at many ring positions) smooth the distribution. **Rendezvous hashing** (highest random weight) achieves the same minimal disruption more simply: for each key, score every node and pick the highest. It's O(*N*) per lookup, fine for tens or hundreds of nodes:

```csharp
using System.IO.Hashing;
using System.Text;

public static class Rendezvous
{
    // Stable across processes — never use string.GetHashCode() for this.
    public static T PickOwner<T>(string key, IReadOnlyList<T> nodes, Func<T, string> nodeId)
    {
        T best = nodes[0];
        ulong bestScore = 0;
        foreach (var node in nodes)
        {
            ulong score = XxHash64.HashToUInt64(Encoding.UTF8.GetBytes($"{nodeId(node)}|{key}"));
            if (score >= bestScore) { bestScore = score; best = node; }
        }
        return best;
    }
}
```

When a node is removed, only the keys it owned move, each to its next-highest scorer. **Maglev hashing**, from Google's software L4 load balancer, builds a lookup table that gives O(1) selection with very even distribution and minimal disruption.

Plain consistent hashing still has a load problem: its balance is no better than random assignment, and popular keys make their owners hot. **Consistent hashing with bounded loads** fixes this by capping each node at a fixed multiple of the average load and forwarding overflow to the next node on the ring. It's a practical success story: Vimeo implemented it in HAProxy for their video cache tier and cut cache bandwidth by a factor of almost eight.

We'll return to consistent hashing in depth in Module 8 when we partition data. For now: hash-based balancing trades load evenness for locality, and you reach for it only when locality is worth the trade.

### 12.8 Choosing an algorithm

| Algorithm | Uses load info | Handles uneven request cost | Safe with many balancers | Gives affinity | Good default for |
|---|---|---|---|---|---|
| Round robin | No | No | Yes | No | Homogeneous requests, identical backends |
| Weighted round robin | Static weights | No | Yes | No | Mixed hardware, canaries |
| Random | No | No | Yes | No | Very large fleets with many clients |
| Least connections / requests | Local | Yes | Poor (herding) | No | A single balancer, variable request cost |
| Power of two choices | Local | Yes | Yes | No | The general default; meshes; YARP's default |
| EWMA / server-reported | Latency / utilization | Yes | Yes (with P2C) | No | Heterogeneous backends, tail-sensitive services |
| Consistent / rendezvous / Maglev | No | No | Yes | Yes | Caches, stateful partitions, L4 balancing |
| Bounded-load consistent hashing | Yes | Partially | Yes | Mostly | Caches with hot keys |

---

## Concept 13 — Health checks, draining, and slow start

A load balancer is only as good as its knowledge of which backends can serve traffic. Health checking supplies that knowledge — and poorly designed health checks cause more outages than they prevent.

**Active vs passive.** Active health checks probe each backend on an interval (for example, `GET /healthz/ready` every 10 seconds, unhealthy after three consecutive failures). Passive health checks, also called outlier detection, watch real traffic and eject backends whose error rate or latency is anomalous. Active checks catch dead instances before users hit them; passive checks catch instances that pass the probe but fail real requests. Use both.

**Liveness vs readiness vs startup.** Three different questions, and conflating them is a classic mistake. **Liveness** asks "is this process fundamentally broken and in need of a restart?" It should check only the process itself, never dependencies, because restarting your app won't fix a database outage. **Readiness** asks "should this instance receive traffic right now?" — false during warm-up, during graceful shutdown, or when the instance is saturated. **Startup** asks "has initialization finished?" and keeps liveness checks from killing slow-starting apps.

**The fleet-wide failure trap.** Suppose readiness checks the database. The database has a brief hiccup, every instance fails readiness simultaneously, the balancer removes the entire fleet, and a partial degradation becomes a total outage. The Amazon Builders' Library article on health checks describes exactly this: a check failing for a non-critical reason, correlated across servers, can take down a whole fleet. The recommended behavior is to **fail open**: automation should remove a single bad server, but if the whole fleet looks unhealthy at once, keep serving, because the check is probably wrong or the problem is shared. Many load balancers implement this (Envoy's panic threshold; AWS load balancers routing to all targets when all are unhealthy). For your own checks, keep dependencies out of liveness, be deliberate about what's in readiness, and surface dependency health through metrics and alerts instead.

In ASP.NET Core, separate the endpoints with tags:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddCheck<WarmupHealthCheck>("warmup", tags: ["ready"])
    .AddCheck<DrainingHealthCheck>("draining", tags: ["ready"]);

var app = builder.Build();

app.MapHealthChecks("/healthz/live",  new HealthCheckOptions { Predicate = r => r.Tags.Contains("live") });
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions { Predicate = r => r.Tags.Contains("ready") });

// Report not-ready as soon as shutdown begins, so pollers stop routing here.
public sealed class DrainingHealthCheck(IHostApplicationLifetime lifetime) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default) =>
        Task.FromResult(lifetime.ApplicationStopping.IsCancellationRequested
            ? HealthCheckResult.Unhealthy("draining")
            : HealthCheckResult.Healthy());
}
```

(`WarmupHealthCheck` would return healthy once caches are primed and connection pools opened.)

**Graceful shutdown and connection draining.** When an instance is removed — by a deployment, a scale-in, or node maintenance — in-flight requests should complete. The sequence: stop receiving new traffic, finish what's in flight, then exit. ASP.NET Core does its part on SIGTERM: Kestrel stops accepting new connections and waits for in-flight requests up to `HostOptions.ShutdownTimeout`, which you should set explicitly. In Kubernetes there's a race: the pod receives SIGTERM at roughly the same time its endpoint is being removed from load balancers, so a few requests may still arrive after shutdown begins. The common fix is a short `preStop` sleep (a few seconds) so routing updates propagate before the app stops listening, with `terminationGracePeriodSeconds` longer than the sleep plus the shutdown timeout.

**Slow start.** New instances are cold: the JIT hasn't optimized hot paths yet (tiered compilation and Dynamic PGO keep optimizing through the first seconds to minutes), caches are empty, connection pools unopened. Sending full traffic immediately causes latency spikes — and, with least-connections, a flood. Slow start ramps a new backend's share over a configurable window. Pair it with readiness that stays false until warm-up completes, and with startup optimizations: ReadyToRun compilation, or Native AOT for services that support it.

A YARP cluster with active and passive health checks (and affinity configured but deliberately disabled) looks like this:

```json
"Clusters": {
  "orders": {
    "LoadBalancingPolicy": "PowerOfTwoChoices",
    "SessionAffinity": {
      "Enabled": false,
      "Policy": "HashCookie",
      "AffinityKeyName": ".Orders.Affinity"
    },
    "HealthCheck": {
      "Active": {
        "Enabled": true,
        "Interval": "00:00:10",
        "Timeout": "00:00:05",
        "Policy": "ConsecutiveFailures",
        "Path": "/healthz/ready"
      },
      "Passive": {
        "Enabled": true,
        "Policy": "TransportFailureRate",
        "ReactivationPeriod": "00:02:00"
      }
    },
    "Destinations": {
      "orders-1": { "Address": "http://orders-1:8080/" },
      "orders-2": { "Address": "http://orders-2:8080/" },
      "orders-3": { "Address": "http://orders-3:8080/" }
    }
  }
}
```

Enable affinity only for the legitimate cases from Concept 9.

---

## Concept 14 — Global load balancing: DNS, anycast, and geo-routing

Within a region, load balancers route to instances. Across regions, the problem is choosing which region serves a user, and failing over when a region is unhealthy.

**DNS-based load balancing** returns different IP addresses for the same hostname. Plain DNS round robin rotates through a list, with no awareness of health or load, and it's undermined by caching: resolvers and clients cache records for the TTL, and some ignore short TTLs, so failover takes as long as the slowest cache. Smarter DNS services (Azure Traffic Manager, AWS Route 53) add health checks and routing policies — geographic (European users go to the EU region), latency-based, weighted (for migrations), and priority (active-passive failover). They're simple and protocol-agnostic, but failover speed is bounded by DNS caching.

**Anycast** advertises the same IP address from many locations via BGP, and internet routing delivers each user's packets to the topologically nearest one. There's no DNS caching delay because the IP never changes; if a location fails, its routes are withdrawn and traffic flows to the next nearest. Global edge networks such as Azure Front Door and Cloudflare use anycast, typically terminating TLS at the edge and forwarding over the provider's backbone to your origin. This is why Microsoft's guidance notes that Traffic Manager, being DNS-based, can't fail over as quickly as Front Door.

**The hard part is data, not traffic.** Routing users to the nearest region is easy. Serving them from that region requires their data to be there, which is a replication and consistency problem (Modules 7 and 8). Active-passive multi-region is simpler but wastes capacity and has recovery-time implications; active-active serves from all regions but forces you to confront write conflicts or partition users by home region. Module 13 covers the trade-off. In an interview, if you propose multi-region, the follow-up is always "and where do the writes go?"

---

## Concept 15 — Client-side load balancing and service meshes

For internal service-to-service traffic, there are three options: a dedicated load balancer in the middle (a proxy hop), a load balancer inside the client (client-side LB), or a sidecar or node proxy that does it transparently (service mesh).

**Client-side load balancing** has the client discover all backend addresses and choose one per request. It removes a network hop and a component to operate, balances per request even over HTTP/2, and lets the client use its own latency and error observations. The costs: every client, in every language, must implement it correctly, and clients need a service-discovery mechanism.

**In .NET, three things matter.** First, gRPC client-side load balancing is built into `Grpc.Net.Client`. You configure a resolver (DNS, or a static list) and a balancing policy, and the channel maintains connections to every backend. In Kubernetes, point the DNS resolver at a *headless* Service so DNS returns individual pod IPs rather than one virtual IP. Because the channel holds the addresses, connections, and balancing state, it must be created once and reused — a new channel per call defeats balancing entirely.

```csharp
var channel = GrpcChannel.ForAddress(
    "dns:///orders-grpc-headless.default.svc.cluster.local:5000",
    new GrpcChannelOptions
    {
        Credentials = ChannelCredentials.Insecure, // use TLS in production
        ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() } }
    });
// Register the channel (or a typed client over it) as a singleton.
```

Second, **`HttpClient` connection pooling can defeat DNS-based balancing**. A long-lived `HttpClient` (which you should use, to avoid socket exhaustion) keeps pooled connections open indefinitely by default, so it never re-resolves DNS and never notices added or removed backends. Set `PooledConnectionLifetime` so connections are periodically recycled, or use `IHttpClientFactory`, which rotates handlers (every two minutes by default):

```csharp
var handler = new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),   // re-resolve DNS, rebalance
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1)
};
var http = new HttpClient(handler); // long-lived and shared
```

Third, **service discovery**: `Microsoft.Extensions.ServiceDiscovery` (used by .NET Aspire) lets `HttpClient` resolve logical names such as `https+http://orders` to endpoints from configuration or DNS SRV records, with simple client-side selection.

**Service meshes** (Linkerd, Istio, and their newer sidecar-less modes) move all of this into infrastructure: a proxy near each workload handles discovery, P2C or EWMA balancing, retries, timeouts, mTLS, and telemetry with no code changes. The costs are operational complexity, resource overhead, and some added latency per hop. A mesh is usually justified with many services across multiple languages, and usually overkill for a handful of .NET services, where `IHttpClientFactory`, gRPC client LB, and Polly (Module 25) cover the needs.

---

## Concept 16 — Autoscaling

Autoscaling adjusts the number of instances to match demand, turning a scalable system into an elastic one. It sounds like a checkbox; getting it right is subtle.

**Three modes.** **Reactive** scaling responds to a metric crossing a threshold — the common default. **Scheduled** scaling adds capacity before known peaks (business hours, a launch, a ticket on-sale). **Predictive** scaling forecasts load from history and scales ahead of it. Mature systems combine them: scheduled or predictive for the baseline shape, reactive for surprises.

**Choosing the signal is the key decision.** CPU utilization is the default everywhere and often the wrong signal for .NET web services, which are typically I/O-bound: they spend their time awaiting databases and HTTP calls, so CPU stays modest while latency degrades and pools saturate. Better signals, depending on the workload: requests per second per instance, in-flight requests (concurrency), p95/p99 latency, and — for queue consumers — **queue depth or oldest-message age**, which directly measures work not yet done. Scaling a consumer fleet on backlog is the canonical case: KEDA (Kubernetes Event-Driven Autoscaling) scales on Service Bus, Kafka, RabbitMQ, and dozens of other sources, including down to zero, and Azure Container Apps' scale rules are built on it.

**How the Kubernetes HPA decides.** The Horizontal Pod Autoscaler computes `desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`. At 10 pods averaging 90% CPU against a 60% target, it asks for 15. It applies a scale-down stabilization window (five minutes by default) so it doesn't remove pods on a momentary dip.

**Scale out fast, scale in slow.** Under-provisioning costs users; over-provisioning costs money. So scale-out should be aggressive and scale-in conservative, with cooldowns to avoid flapping (oscillating as each scaling action changes the very metric it responds to).

**Provisioning time is the hidden variable.** Autoscaling isn't instantaneous. Adding a pod to an existing node might take 30–90 seconds including image pull, startup, and warm-up; if the cluster autoscaler must add a node first, several minutes. During that window the existing fleet absorbs the growth, so headroom must satisfy roughly *headroom ≥ traffic ramp rate × time to ready capacity*. If traffic can grow 20% per minute and new capacity takes three minutes, reactive scaling alone can't save you unless you run with about 60% spare capacity — which is why spiky workloads use scheduled pre-scaling, higher minimum replica counts, and faster startup. On the .NET side, faster startup means ReadyToRun, trimming, Native AOT where compatible, and deferring non-critical initialization.

**Don't DDoS your own dependencies.** Scaling the stateless tier moves pressure onto shared resources, and connection pools are the most common casualty. Each instance opens up to its pool's maximum (100 by default in SqlClient and Npgsql). Scale from 10 to 126 instances and the database may face up to 12,600 connections, while PostgreSQL's default `max_connections` is 100 and each Postgres connection is a full backend process. The fixes: lower per-instance pool sizes, put a connection pooler such as PgBouncer (transaction mode) in front of Postgres, cap the autoscaler's maximum replicas at what downstream can sustain, and cache reads. The general rule: **every autoscaling policy needs a maximum derived from the capacity of what it depends on.**

---

## Concept 17 — When scaling out isn't enough: protecting the system

Scaling out has lag, and some load exceeds what any budget should absorb. A scalable system also needs mechanisms to degrade gracefully under overload instead of collapsing. These get full treatment in Modules 13 and 25, but they belong in any scalability answer.

**Bounded queues and backpressure.** Unbounded queues turn overload into unbounded latency and memory growth; a request that waited 30 seconds in a queue is usually worthless by the time it's served. Bound every queue and signal "slow down" upstream when it fills. In .NET, a bounded `System.Threading.Channels` channel is the in-process tool (Module 15).

**Load shedding.** When an instance is at capacity, rejecting excess requests quickly (HTTP 503 or 429) keeps latency low for the requests it accepts, so throughput under overload stays near maximum instead of collapsing as everything times out. ASP.NET Core's rate limiting middleware includes a concurrency limiter for exactly this, per instance, and Kestrel has connection limits. Shed low-priority work (analytics, prefetching) before user-facing requests.

**Queue-based load leveling.** Put a durable queue between a spiky producer and a steady consumer, so the consumer processes at its sustainable rate and the queue absorbs bursts. This converts a latency problem into a delay the business can often accept ("your order is being processed").

**Retry discipline.** Retries multiply load exactly when a system is struggling: three layers each retrying three times turn one failed request into 27 attempts at the bottom. Retry budgets, exponential backoff with jitter, and circuit breakers keep resilience from becoming the cause of the outage (Module 13).

---

## Concept 18 — The data tier: where scaling problems move

Once the application tier is stateless, the scaling story is really about data stores, because that's where all the state went. This concept bridges to the rest of Phase 3.

The sequence most systems follow: scale the primary database vertically; add caching to absorb repeated reads (Module 10); add read replicas and route reads to them, accepting replication lag for queries that tolerate it (Modules 7 and 8); separate read and write models where their scaling needs diverge (CQRS, Module 23); move asynchronous work to queues and streams (Module 11); and only then partition the data (sharding, Module 8), because it's the most expensive step to undo and it complicates transactions, joins, and operations (Module 12).

An interviewer who asks "how would you scale this?" usually wants the stateless tier handled in a sentence or two and the conversation moved to the data tier, where the real trade-offs live.

---

## Putting it together: how this shows up in interviews

Scalability fundamentals rarely appear as a standalone question at the senior level. They appear inside every system design round — as the deep-dive follow-up ("you have one API server; what happens at 10× traffic?") and in the wrap-up ("where's your bottleneck?"). For .NET roles they also appear in the technical round as concrete questions about ASP.NET Core in multi-instance deployments.

### Worked example: scaling a .NET API to 100k RPS

**Requirements.** An ASP.NET Core API serving 100k RPS at peak, 95% reads, p99 under 200 ms, deployed across three availability zones, and it must survive a full zone outage.

**Per-instance capacity comes from measurement, not assumption.** A load test on a 4-vCPU pod shows p99 holding until about 2,000 RPS, then climbing steeply (the utilization knee from Concept 2). Plan at 60% of that: 1,200 RPS per pod.

**Instance count.** 100,000 / 1,200 ≈ 84 pods with no failures. To survive losing a zone, the remaining two zones must carry the full 100k: 84 pods across two zones means 42 per zone, so 126 pods across three. Alternatively, accept higher utilization during a zone failure — say 85% of the knee: 100,000 / 1,700 ≈ 59 pods across two zones, about 30 per zone and 90 total — trading latency during a rare event for roughly 30% lower cost. Stating that trade-off explicitly is the senior move.

**Load balancing.** Front Door (global L7, anycast, WAF) routes to the regional AKS ingress (L7), which spreads requests with P2C. Internal gRPC calls use client-side load balancing over a headless Service, because L4 kube-proxy would pin HTTP/2 connections.

**Statelessness.** Data Protection keys in Blob Storage protected by Key Vault; JWT bearer auth with no server session; `HybridCache` with Redis as L2; background jobs converted to Service Bus consumers.

**Autoscaling.** HPA on in-flight requests per pod rather than CPU (the service is I/O-bound); a scheduled minimum of 126 during peak hours; fast scale-out with a slow scale-in window; and a maximum derived from database capacity.

**The data tier, where the real problem is.** Reads: 95,000 RPS; with a 90% cache hit rate the database sees about 9,500 read QPS, spread across read replicas. Writes: 5,000 per second on the primary — feasible on a large instance, so no sharding yet (scale the stateful tier up first). Connections: 126 pods × a default pool of 100 is 12,600 potential connections, which would overwhelm Postgres, so per-pod pools drop to about 20 and PgBouncer sits in front of the database.

**Bottlenecks to name in the wrap-up.** Write throughput on the primary (the Amdahl serial fraction), cache stampedes on hot keys after deployments, Redis as a single hot dependency, and pod startup time versus traffic ramp rate.

### Common questions and what a strong answer contains

**"Would you scale this vertically or horizontally?"** Don't pick one reflexively. Separate the tiers: stateless compute out (you need at least two instances for availability anyway), stateful storage up first, with a stated threshold at which you'd partition. Mention cost curves, licensing, and blast radius.

**"This ASP.NET Core app runs on one server. Make it run on ten."** Walk the state inventory: Data Protection key ring, session store, in-memory caches, local files, hosted services running ten times, SignalR backplane, per-instance rate limits, forwarded headers, and anything using `string.GetHashCode()` across processes. Then mention sticky sessions as a last-resort bridge, not the solution.

**"L4 or L7 here?"** L7 for HTTP APIs needing path routing, retries, WAF, and per-request balancing; L4 for raw throughput, non-HTTP protocols, or TLS passthrough. Note that they're usually layered, and raise the HTTP/2 pinning problem unprompted if gRPC is involved.

**"Why is our gRPC service unevenly loaded after scaling out?"** Long-lived HTTP/2 connections balanced at L4. Fix with client-side balancing (headless Service plus `Grpc.Net.Client` load balancing), an L7 proxy or mesh, or connection recycling.

**"Least connections or power of two choices?"** Least connections is fine with a single balancer and fresh information. With many balancers or stale information it herds; P2C avoids herding with a small dose of randomness, costs O(1), and is the default in YARP and common in Envoy and HAProxy.

**"What metric would you autoscale this queue consumer on?"** Queue depth or oldest-message age, not CPU. Mention KEDA, the cold-start trade-off of scale-to-zero, and a maximum set by downstream capacity.

**"Our readiness probe checks the database. Good idea?"** Usually not. A database blip fails every instance simultaneously and removes the whole fleet. Keep liveness local, make readiness about the instance's own ability to serve, and fail open when everything looks unhealthy.

### Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "We'll just add more servers." | Identifies the shared bottleneck more servers won't fix, and quantifies it. |
| Detailing the stateless tier, ignoring the database. | Handles compute in two sentences and spends the time on data. |
| Sizing for average load. | Sizes for peak, plus headroom, plus a zone failure. |
| Autoscaling on CPU by default. | Chooses a signal that matches the workload (concurrency, latency, queue depth). |
| Sticky sessions as the answer to state. | Externalizes state; uses affinity only as an optimization. |
| "Round robin," with no reasoning. | Explains P2C and when hashing is worth losing evenness. |
| Deep health checks everywhere. | Separates liveness and readiness; designs checks that fail open. |
| Microservices as the scaling plan. | Uses the Scale Cube to say which axis solves which problem. |
| No numbers. | Uses Little's Law and utilization targets to justify capacity. |

---

## Practice exercises

**Exercise 1 — Balls into bins, in C#.** Write a console app that assigns 1,000,000 requests to 100 simulated servers under five policies: random, round robin, least-loaded with perfect information, least-loaded with information 50 requests stale (simulating many independent balancers), and power of two choices. Give each request a random service time and model each server as a queue. Report the maximum and p99 queue length per policy. You should see P2C nearly match perfect least-loaded, and stale least-loaded do far worse than random. This is the single best way to internalize Concept 12.

**Exercise 2 — Break and fix a web farm.** Using .NET Aspire (or Docker Compose), run three replicas of a small ASP.NET Core app with cookie authentication and an antiforgery-protected form, behind YARP. Without shared Data Protection keys, log in and click around; watch intermittent logouts and antiforgery failures. Fix it with a shared key ring. Then add a `BackgroundService` that logs once a minute and watch it run three times; fix it with a queue or a lease. Finally, kill a replica mid-load and watch YARP's passive health checks eject it.

**Exercise 3 — Fit the USL to your own service.** Load test an endpoint from one of your projects at several concurrency levels (k6 and NBomber both work well for .NET services), record throughput, and fit α and β with the `usl` R package or a few lines of Python using `scipy.optimize.curve_fit`. Predict the concurrency at which throughput peaks, then verify it.

**Exercise 4 — A one-page capacity plan.** A ticketing platform expects a concert on-sale: 5k RPS baseline rising to 60k RPS within 90 seconds of opening, lasting about 20 minutes. Pod startup is 45 seconds; node provisioning is 4 minutes. Write a one-page plan covering pre-scaling, load-balancing tiers, queue-based admission (a virtual waiting room), a shedding policy, and database protection. This is exactly the shape of an architect-round exercise.

**Exercise 5 — Five-minute verbal drill.** Answer out loud, timed: "Walk me through what happens, layer by layer, when a user on another continent opens our app, and how each layer scales." Record yourself and check that you named the balancer at each layer, the algorithm, where state lives, and the first bottleneck.

---

## Free resources

### Foundations and overviews

| Resource | What it covers | Why read it |
|---|---|---|
| [The System Design Primer](https://github.com/donnemartin/system-design-primer) (GitHub) | Scalability, load balancers, horizontal scaling, caching, with links | The most-referenced free system design overview; good for vocabulary |
| [Introduction to modern network load balancing and proxying](https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236) — Matt Klein | L4 vs L7, topologies, health checking, service mesh | Written by Envoy's creator; the best single free article on load balancing |
| [What is load balancing?](https://www.cloudflare.com/learning/performance/what-is-load-balancing/) — Cloudflare Learning Center | Plain-language overview | Quick primer before the deeper material |
| [Twelve-Factor App: Processes](https://12factor.net/processes) and [Disposability](https://12factor.net/disposability) | Stateless processes, fast startup, graceful shutdown | The canonical short statement of statelessness |
| [The Scale Cube](https://microservices.io/articles/scalecube.html) — microservices.io | X, Y, Z axes | Vocabulary for structuring scale-out answers |

### Google SRE book (free online)

| Resource | What it covers |
|---|---|
| [Ch. 19 — Load Balancing at the Frontend](https://sre.google/sre-book/load-balancing-frontend/) | DNS and virtual-IP balancing, global traffic |
| [Ch. 20 — Load Balancing in the Datacenter](https://sre.google/sre-book/load-balancing-datacenter/) | Subsetting, weighted round robin, handling unhealthy tasks |
| [Ch. 21 — Handling Overload](https://sre.google/sre-book/handling-overload/) | Client-side throttling, criticality, load shedding |
| [Ch. 22 — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) | How overload spreads and how to stop it |

### Amazon Builders' Library (free)

| Resource | What it covers |
|---|---|
| [Implementing health checks](https://aws.amazon.com/builders-library/implementing-health-checks/) | Liveness vs dependency checks, fail-open, the black-hole problem |
| [Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) | Why and how to reject work under overload |
| [Workload isolation using shuffle-sharding](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/) | Limiting blast radius by how customers are assigned to workers |

### Algorithms and theory

| Resource | What it covers |
|---|---|
| [The power of two random choices](https://brooker.co.za/blog/2012/01/17/two-random/) — Marc Brooker | P2C, and why stale information breaks least-loaded |
| [The power of two choices in randomized load balancing](https://www.eecs.harvard.edu/~michaelm/abstracts/tpds2001.html) — Michael Mitzenmacher | The original supermarket-model analysis |
| [Test driving "Power of Two Random Choices"](https://www.haproxy.com/blog/power-of-two-load-balancing) — HAProxy | Empirical comparison against other algorithms |
| [Consistent hashing with bounded loads](https://research.google/blog/consistent-hashing-with-bounded-loads/) — Google Research | The algorithm plus the Vimeo production story |
| [Maglev: a fast and reliable software network load balancer](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/) — Google | How a production L4 balancer and Maglev hashing work |
| [The Tail at Scale](https://research.google/pubs/the-tail-at-scale/) — Dean & Barroso | Why fan-out makes tail latency dominate |
| [Visualizing scalability](https://calendar.perfplanet.com/2017/visualizing-scalability/) — Neil Gunther | The USL explained by its creator |
| [Scalability! But at what COST?](https://www.usenix.org/conference/hotos15/workshop-program/presentation/mcsherry) — McSherry, Isard, Murray | Measure against a single machine before distributing |
| [Netflix edge load balancing](https://netflixtechblog.com/netflix-edge-load-balancing-695308b5548c) — Netflix Tech Blog | P2C combined with server-reported utilization in production |

### .NET and Azure documentation

| Resource | What it covers |
|---|---|
| [YARP load balancing](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/load-balancing) | Built-in policies (session affinity and health checks are documented alongside) |
| [Host ASP.NET Core in a web farm](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/web-farm) | The checklist for multi-instance ASP.NET Core |
| [Data Protection configuration](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview) | Key storage, encryption at rest, application name |
| [Distributed caching in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed) | `IDistributedCache` with Redis, SQL, and others |
| [HybridCache](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) | L1/L2 caching with stampede protection |
| [SignalR scale-out](https://learn.microsoft.com/en-us/aspnet/core/signalr/scale) | Sticky sessions, Redis backplane, Azure SignalR Service |
| [Proxy servers and load balancers](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer) | Forwarded headers configuration |
| [Health checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) | Liveness/readiness separation, tags, publishers |
| [Rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) | Fixed, sliding, token bucket, and concurrency limiters |
| [gRPC client-side load balancing](https://learn.microsoft.com/en-us/aspnet/core/grpc/loadbalancing) | Resolvers, policies, Kubernetes setup |
| [HttpClient guidelines](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines) | Pooling, DNS changes, `PooledConnectionLifetime` |
| [DATAS GC](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/datas) | How .NET adapts heap size to the workload |
| [Azure load-balancing options](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview) | Decision tree across Front Door, App Gateway, Load Balancer, Traffic Manager |
| [Design to scale out](https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/scale-out) | Azure's scale-out design principles |
| [Autoscaling best practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/auto-scaling) | Metrics, schedules, pitfalls |
| [Deployment Stamps pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) | Z-axis scaling of the whole stack |
| [Queue-Based Load Leveling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) | Absorbing bursts with a queue |
| [Performance Efficiency pillar](https://learn.microsoft.com/en-us/azure/well-architected/performance-efficiency/) | Well-Architected guidance on capacity and scaling |
| [Architecting Cloud Native .NET Applications for Azure](https://learn.microsoft.com/en-us/dotnet/architecture/cloud-native/) (free e-book) | Scaling, resilience, and service communication in .NET |

### Kubernetes, proxies, and autoscaling

| Resource | What it covers |
|---|---|
| [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | The HPA algorithm, stabilization, behavior tuning |
| [gRPC load balancing on Kubernetes without tears](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/) | The HTTP/2 pinning problem and its fixes |
| [KEDA](https://keda.sh/) | Event-driven autoscaling, including queue-based and scale-to-zero |
| [Envoy load balancers](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers) | Precise descriptions of production algorithms |
| [NGINX HTTP load balancing](https://nginx.org/en/docs/http/load_balancing.html) | Round robin, least connections, IP hash in practice |

### Case studies

| Resource | What it covers |
|---|---|
| [Stack Overflow: The Architecture — 2016 Edition](https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/) | A .NET/SQL Server site at massive scale on few, powerful servers |
| [Stack Overflow: The Hardware — 2016 Edition](https://nickcraver.com/blog/2016/03/29/stack-overflow-the-hardware-2016-edition/) | The machines behind it — vertical scaling made concrete |

### Video (optional)

| Resource | What it covers |
|---|---|
| [Hussein Nasser's channel](https://www.youtube.com/@hnasr) | Backend engineering deep dives, including L4 vs L7, proxies, and HTTP/2 |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| The definition of scalability | Load numbers plus an SLO; performance vs scalability vs elasticity vs availability |
| Why more servers didn't help | Amdahl's serial fraction and USL coherency; find the shared bottleneck |
| Latency rising under load | Little's Law and the utilization knee; plan at 60–70% |
| Vertical vs horizontal | Stateless tier out, stateful tier up first; name the partition threshold |
| Making an app stateless | State moves to a purpose-built tier; walk the inventory |
| Sticky sessions | An optimization, never a correctness requirement |
| L4 vs L7 | Per-connection vs per-request; the HTTP/2/gRPC pinning trap |
| The best algorithm | P2C by default; hashing only when locality beats evenness |
| Health checks | Liveness local, readiness deliberate, fail open fleet-wide |
| Autoscaling | The right signal, provisioning lag, a maximum set by downstream capacity |
| Multi-region | Routing is easy; the data is the hard part |

---

## Progress

Module 6 complete. Next: **Module 7 — CAP theorem, PACELC, and consistency models**, which picks up where Concepts 14 and 18 leave off: once state is replicated across machines and regions, what guarantees can you actually offer?
