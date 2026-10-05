# Module 10 — Caching Strategy
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence that should reframe this whole module: **a cache is a replica with a weak consistency model, a bounded lifetime, and no coordination.** Every tool you built in Modules 7–9 applies directly. A cache is asynchronous replication (Module 8) with an eviction policy bolted on, whose staleness bound you choose (Module 7), and whose invalidation is a coordination problem (Module 9) that you deliberately refuse to solve properly because solving it properly would cost you the latency you added the cache to save.

That framing is what separates a senior answer from a mid-level one. Mid-level candidates treat caching as a performance tactic: "add Redis in front of the database." Senior candidates treat it as a **consistency and availability decision with a performance payoff**, and can state precisely what correctness they gave up, how stale a read can be, what happens when the cache dies, and why the hit ratio is not the metric they'd optimize.

This module has five jobs:

1. **Make the arithmetic reflexive.** Origin load is `(1 − h) × λ`. That one expression explains why 90% → 95% hit ratio *halves* your database load, why cache loss is a 10–20× origin load spike, and why "we'll add a cache" is a capacity plan, not a hope.
2. **Install the pattern vocabulary properly** — cache-aside, read-through, write-through, write-behind, write-around, refresh-ahead, negative caching, materialization — so you name patterns instead of describing them.
3. **Make you genuinely dangerous on invalidation.** Specifically: the stale-set race in cache-aside, why you delete rather than update, and the ladder from TTL → invalidate-on-write → versioned keys → leases. Most candidates cannot draw the race. Drawing it on a whiteboard is a staff-level signal.
4. **Give you the .NET surface cold** — `IMemoryCache`, `IDistributedCache`, **`HybridCache`**, FusionCache, output caching, StackExchange.Redis multiplexer behaviour, GC consequences of a large in-process cache, and the current Azure Redis story (which changed significantly in 2025–26).
5. **Teach the failure mode nobody prepares for**: what happens when the cache is empty or gone. Systems that are fast only with a warm cache are systems with a hidden single point of failure, and this is the question that separates people who have operated caches from people who have configured them.

Five framings to carry through:

1. **Caching is the act of choosing staleness in exchange for latency and load.** If you can't state the staleness bound, you haven't designed the cache.
2. **Invalidation is a coordination problem you're declining to solve.** TTL is "I accept bounded staleness". Explicit invalidation is "best-effort, racy, and usually fine". Versioned keys are "I made staleness structurally impossible instead of racing".
3. **Hit ratio is a proxy, not a goal.** The real objectives are *origin load reduction* and *tail latency*. A cache with 99% hit ratio and a 2-second miss path has a terrible p99. (This is the RobinHood paper's entire argument.)
4. **Never design a system that only works warm.** Cold-start, flush, failover, and deploy all produce cold caches. Design the origin, or the degradation, to survive it.
5. **The cheapest cache is the one you didn't need.** A better index, a smaller payload, a read replica, or a denormalized read model often beats a cache and adds no consistency debt. Reach for that first — saying so unprompted is an architect-level move.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Why caching works | Skew: Zipf-like popularity means a tiny working set serves most traffic; hit ratio grows ~logarithmically with cache size |
| 2 | The arithmetic | `origin load = (1−h)·λ`; `E[latency] = T_cache + (1−h)·T_origin`; 90→95% halves backend load |
| 3 | The cache hierarchy | Client → CDN → gateway → in-process L1 → distributed L2 → DB buffer pool; each layer is a different consistency contract |
| 4 | What's worth caching | High read:write, tolerable staleness, expensive to compute, small enough, and a working set that fits |
| 5 | A cache is a replica | Staleness = TTL + replication lag; breaks read-your-writes and monotonic reads; name the model |
| 6 | Latency numbers | L1 ~50–100 ns, Redis in-region ~0.3–1 ms p50 / 2–5 ms p99, SQL 1–10 ms, cross-region 70–150 ms, CDN hit 10–50 ms |
| 7 | Cache-aside | The default. App owns the logic. Lazy fill, miss penalty, stale-set race, stampede exposure |
| 8 | Read-through | Same behaviour, different ownership: the cache library loads. `HybridCache.GetOrCreateAsync` is this |
| 9 | Write-through | Synchronous write to cache + store. Fresher cache, higher write latency, still racy |
| 10 | Write-behind | Cache is authoritative briefly; batch/coalesce to the store. Fast writes, real data-loss window |
| 11 | Write-around / invalidate | Write the store, **delete** the key. Delete beats update — ordering races and wasted work |
| 12 | Refresh-ahead & SWR | Serve stale while refreshing in background; probabilistic early refresh (XFetch) beats a fixed window |
| 13 | Materialization & negative caching | Fan-out on write vs on read; cache the miss to blunt key-enumeration load |
| 14 | The stale-set race | Reader's late `SET` of a pre-write value poisons the cache until TTL. Draw the timeline |
| 15 | The invalidation ladder | TTL → invalidate-on-write → versioned keys → leases → don't cache |
| 16 | Versioned & namespaced keys | Put a version in the key; invalidation becomes a write of a version, not a distributed delete |
| 17 | Event-driven invalidation | CDC / outbox / change feed as the invalidation transport; monotonic version checks on set |
| 18 | Key design | `app:v3:tenant:entity:id:shape`; cardinality, collisions, tenant isolation, memory per key |
| 19 | Read-your-writes | Sticky-write patterns: write-through on the writer's path, per-user version bump, or skip-cache window |
| 20 | Eviction policies | LRU/LFU/FIFO/ARC/**W-TinyLFU**; Belady as the upper bound; Redis's sampled approximations |
| 21 | Expiry mechanics | Absolute vs sliding (and the sliding-forever trap); lazy + active expiry; jitter |
| 22 | Stampede / dogpile | Three distinct phenomena; single-flight, XFetch, SWR, jitter, negative caching, origin concurrency caps |
| 23 | Hot keys | Detect with Count-Min/Top-K; fix with L1 tier, key splitting, or client-side tracking |
| 24 | Cold cache & death spiral | The mode change nobody tests: full QPS at the origin plus retry amplification; load shed and coalesce |
| 25 | Redis execution model | Single-threaded command execution; O(N) commands are latency bombs; pipeline, `SCAN`, `UNLINK` |
| 26 | Redis data structures | Strings, hashes (+field TTL 7.4), sorted sets, bitmaps, HLL, streams; Bloom/Cuckoo/Top-K |
| 27 | Redis consistency | Async replication, failover loses acked writes, `WAIT` isn't consensus — never a source of truth |
| 28 | Redis Cluster | 16384 slots, CRC16 mod, `{hash tags}`, CROSSSLOT, resharding; Sentinel vs Cluster |
| 29 | Redis ops | `maxmemory-policy` (and the `volatile-*` OOM trap), persistence for a cache, fragmentation, memory as the cost driver |
| 30 | Azure Redis in 2026 | Azure Cache for Redis is being retired; Azure Managed Redis is the target — know the dates |
| 31 | `IMemoryCache` | Size limits, expiration, eviction callbacks, change-token invalidation, compaction, GC consequences |
| 32 | `IDistributedCache` | `byte[]` contract, no stampede protection, no tags, no L1; serializer choice dominates cost |
| 33 | `HybridCache` | The current default: L1+L2, stampede protection, tags, `GetOrCreateAsync` with state |
| 34 | FusionCache | Fail-safe, soft/hard timeouts, eager refresh, backplane, auto-recovery — the resilience-first option |
| 35 | Output vs response caching | Server-driven policy + tags vs client-header-driven; `MapStaticAssets` and fingerprinted immutables |
| 36 | EF Core & data-layer caching | No L2 cache by design; `AsNoTracking`, compiled queries, and why read models beat interceptors |
| 37 | HTTP caching semantics | `max-age`/`s-maxage`, `no-cache` vs `no-store`, `stale-while-revalidate`, `immutable`, `Vary`, ETags, 304 |
| 38 | CDN mechanics | PoPs, anycast, cache key composition, tiered caching / origin shield, midgress |
| 39 | Edge invalidation | Purge by URL/tag/wildcard, soft purge, and fingerprinted URLs as the strategy that avoids purging |
| 40 | Security & compliance | Cache poisoning via unkeyed input, tenant key collisions, PII and right-to-erasure, encryption, Entra auth |
| 41 | Cost | Cache memory vs read replicas vs recomputation; the GB-month comparison that wins architecture arguments |
| 42 | The decision framework | The ladder: fix the query → HTTP/CDN → read replica → read model → L2 → L1+L2 → precompute |
| 43 | Observability | Hit ratio by key class, miss latency, eviction rate, fragmentation, origin load reduction as the SLI |
| 44 | Anti-patterns | Cache as a band-aid, unbounded caches, cache as source of truth, double caching, no cold-start plan |

---

# Part A — Fundamentals

## Concept 1 — Why caching works at all: skew, not magic

Caching works because request popularity is **extremely skewed**. Web and API access patterns are consistently well-modelled by a Zipf-like distribution: the frequency of the *k*-th most popular item is proportional to `1/k^α`, with α typically between 0.6 and 1.0 for web workloads (Breslau et al., 1999 — still the canonical measurement).

Two consequences you should be able to state:

1. **A tiny cache captures most of the traffic.** With α ≈ 1, caching the top *N* items out of *M* yields a hit ratio roughly proportional to `ln(N)/ln(M)`. Caching 1% of your catalogue can plausibly serve 50–70% of reads. This is why "we only have 2 GB of Redis for a 2 TB database" is not absurd.
2. **Hit ratio has diminishing returns in cache size, but *increasing* returns in value.** Because the relationship is logarithmic, doubling memory adds a few points of hit ratio — but as Concept 2 shows, those last few points are worth the most in load terms. The curve of *cost* is linear; the curve of *benefit* is convex at the top. Knowing both halves is the interview signal.

The other structural reason caching works is **fan-out amplification**: one user request often triggers dozens of downstream reads (a product page needs the product, price, inventory, reviews, recommendations, and the user's cart). A 90% hit ratio on each of 20 dependencies changes the expected number of origin calls per page from 20 to 2.

**Where caching does *not* work** — say this before the interviewer asks:
- **Uniform access distributions** (random key lookups over a huge keyspace). No skew, no locality, no hit ratio. The cache becomes pure overhead plus a consistency problem.
- **Write-dominant data.** Each write invalidates; the entry never gets read enough to amortize the fill.
- **Working set ≫ cache.** You get thrashing: constant eviction, near-zero hit ratio, and full memory cost. This is why you measure the working set, not the dataset.
- **Data whose correctness requirement is strictly linearizable.** Then it isn't a cache question, it's a Module 7/9 question.

---

## Concept 2 — The arithmetic you should have reflexive

Three expressions. Learn them well enough to do them out loud.

**1. Origin load.** With request rate λ and hit ratio *h*:

```
origin_load = (1 − h) · λ
```

This is the most useful formula in the module because it is *not* linear in intuition:

| Hit ratio | Fraction reaching origin | Origin load at 100k RPS | Relative to h=0 |
|---|---|---|---|
| 0% | 100% | 100,000 | 1× |
| 50% | 50% | 50,000 | 2× reduction |
| 90% | 10% | 10,000 | 10× |
| 95% | 5% | 5,000 | 20× |
| 99% | 1% | 1,000 | 100× |
| 99.9% | 0.1% | 100 | 1000× |

**The senior observation:** going from 90% → 95% *halves* origin load; 95% → 99% cuts it by another 5×. Conversely — and this is the one people miss — **losing the cache multiplies origin load by `1/(1−h)`**. At 95%, a cache flush is a **20× instantaneous load spike**. That single number justifies everything in Concept 24.

**2. Expected latency.** If you always check the cache first:

```
E[latency] = T_cache + (1 − h) · (T_origin + T_fill)
```

At `T_cache = 0.5 ms`, `T_origin = 40 ms`, h = 0.95: `0.5 + 0.05 × 40 = 2.5 ms` mean. But the **p99 is dominated by the miss path**, not the mean — with a 5% miss rate, roughly 1-in-20 requests takes 40 ms, so your p95 *is* the origin latency. This is the number one reason to distrust hit ratio as your metric: *the tail is the miss path*. If the p99 matters, you must either bound the miss path (timeouts, fallbacks, stale-serving) or push the hit ratio high enough that misses fall below your percentile of interest.

**3. Memory sizing.**

```
memory ≈ keys_in_working_set × (avg_value_bytes + key_bytes + per_entry_overhead) × (1 + fragmentation)
```

Per-entry overhead in Redis is on the order of ~50–100 bytes for a simple string key (robj, dict entry, expires entry, jemalloc rounding). For a million 500-byte values: ~0.5 GB of data, ~0.6 GB with overhead, ~0.7 GB with 1.2 fragmentation, and you should provision ~1.5× that so eviction isn't constant and `COPY`-on-fork for persistence doesn't OOM. **Quoting the overhead and fragmentation factor is a distinguishing detail** — most candidates give you the raw payload size.

**Worked example to have ready.** Product catalogue: 5M SKUs, 2 KB JSON each, 60k RPS read, Zipf α≈0.9.
- Full dataset: 10 GB. Not needed.
- Measured (or estimated) working set for 95%: top ~2% ≈ 100k SKUs ≈ 200 MB data ≈ 300 MB provisioned. **Cheap.**
- Origin load: 60k × 0.05 = 3,000 RPS. A single well-indexed Postgres/SQL primary handles that; without the cache, 60k RPS needs a fleet.
- p99: dominated by the 5% miss path, so cap origin calls with a 100 ms timeout and serve stale-on-error.
- Bandwidth: 60k × 2 KB = 120 MB/s ≈ 1 Gbps out of the cache. **This is the constraint people forget** — you can be network-bound before you're CPU- or memory-bound, which is an argument for compression or for an in-process L1.

---

## Concept 3 — The hierarchy: seven places a value can live

Each layer is a *different* consistency contract, a different invalidation mechanism, and a different blast radius. Naming the layer you mean is half of sounding senior.

| Layer | Typical hit latency | Invalidation mechanism | Staleness risk |
|---|---|---|---|
| **Browser / client** | ~0 | `Cache-Control` max-age, fingerprinted URLs | Worst — you cannot purge a client. Only expiry and URL change |
| **CDN / edge PoP** | 10–50 ms | Purge by URL/tag, TTL, `s-maxage` | Seconds to minutes; purge propagation is not instant |
| **Reverse proxy / API gateway** | 1–5 ms | Output-cache tags, TTL | Same as CDN, smaller blast radius |
| **In-process L1** (`IMemoryCache`) | 50–100 ns | Per-instance only → needs a backplane or short TTL | **Highest risk of divergence between instances** |
| **Distributed L2** (Redis) | 0.3–1 ms p50 | Explicit `DEL`, TTL, versioned keys | Shared, so consistent across instances |
| **Read replica** | 1–10 ms | None — it's replication, not caching | Bounded by replication lag |
| **DB buffer pool / plan cache** | µs–ms | Automatic, transactional | None (it's inside the transactional boundary) |

**The two structural insights:**

1. **The further from the origin, the cheaper the hit and the harder the invalidation.** Client caches are free and unpurgeable. That asymmetry is *the* reason fingerprinted URLs exist (Concept 39).
2. **In-process L1 is the fastest and most dangerous layer.** It is ~1000× faster than Redis, and it is per-instance, which means N instances hold N independently stale copies. Two legitimate strategies: keep L1 TTLs very short (seconds) and accept divergence, or add a **backplane** (pub/sub invalidation message) — which is what FusionCache does and what plain `IMemoryCache` gives you nothing for.

**Double caching is a real anti-pattern.** The same value cached at the CDN for 5 minutes, in L1 for 5 minutes, and in L2 for 5 minutes has a worst-case staleness of up to 15 minutes, not 5. **Staleness composes additively down the chain.** If you take one sentence from this concept into an interview, take that one.

---

## Concept 4 — What's actually worth caching

A checklist you can run out loud. Cache when **most** of these hold:

| Criterion | Why it matters |
|---|---|
| **Read:write ratio ≫ 1** | Below ~10:1 the invalidation churn eats the benefit |
| **Staleness is tolerable** | If not, you need a read replica or a transactional read model, not a cache |
| **Expensive to produce** | Multi-join query, fan-out aggregation, external API call, or a render |
| **Small relative to value** | A 5 MB blob to save a 3 ms query is a bad trade; bandwidth and memory dominate |
| **Working set fits** | Otherwise thrashing: full cost, no benefit |
| **Popularity is skewed** | Concept 1 |
| **Idempotent / deterministic** | The value must be a pure function of the key, including the *shape* of the response |

Prioritize by **value density**: `λ_key × (T_origin − T_cache) × cost_per_origin_call`. A 5 RPS key that takes 2 s to compute and costs an external API call is often worth more cache memory than a 5,000 RPS key that takes 1 ms.

**The two cases that should make you say "don't cache":**
- **Per-user data with low per-user request rate.** A user hitting their dashboard twice a day has no locality; you'll pay memory for near-zero hit ratio. (Session state is different — high-frequency, per-request.)
- **Data behind a strict invariant** — balances, inventory counts, permission checks on sensitive operations. Use the escrow/versioning techniques from Module 9, or read the source of truth. "We cached authorization decisions for 5 minutes" is how you get a security incident.

---

## Concept 5 — A cache is a replica: name the consistency model

This is the Module 7 connection, and it's where you earn the most credit.

An entry written to a cache and given a 60-second TTL is an **asynchronous replica with a 60-second staleness bound and no read repair**. Therefore:

- **Consistency model:** at best **bounded staleness** (staleness ≤ TTL, assuming invalidation is best-effort); realistically **eventual consistency** with a bound. If invalidation can be lost (pub/sub at-most-once, network blip, instance restart mid-message), the bound is the TTL and *nothing else*. **Your invalidation mechanism does not define your staleness bound. Your TTL does.** Invalidation only improves the *average*.
- **Read-your-writes is broken by default.** User updates their profile → write goes to the DB → their next read hits a cache node that still holds the old value. This is the single most common user-visible caching bug. Fixes in Concept 19.
- **Monotonic reads are broken across cache nodes.** Two requests, two instances, two different L1 copies → the value appears to go backwards. Fixes: sticky routing (ugly), shared L2 only (slower), or short L1 TTL plus accepting it.
- **No causal consistency across keys.** Invalidate `order:123` and `customer:456` and a concurrent reader can see the new order with the old customer. If two cached values must be mutually consistent, **cache them as one entry** — that's the practical trick, and it's a direct application of Module 7's reasoning.

**The interview sentence:** *"I'm treating this cache as an asynchronous replica with a staleness bound equal to the TTL. That means reads can be up to 60 seconds stale, read-your-writes is broken unless I handle the writer's path explicitly, and two cached values that must agree with each other need to live in the same cache entry."* Delivering that unprompted, before being asked about consistency, is a strong senior/staff signal.

---

## Concept 6 — The latency table (Module 5, caching edition)

Have these cold. They're what you reason with when someone says "would a cache help here?"

| Operation | Order of magnitude |
|---|---|
| L1/L2 CPU cache hit | 1–10 ns |
| `Dictionary<K,V>` / `IMemoryCache` hit | 20–100 ns |
| Main memory read | ~100 ns |
| Lock-free concurrent read (contended) | 100 ns – 1 µs |
| JSON deserialize 2 KB | 1–10 µs |
| Same-AZ network round trip | 0.2–0.5 ms |
| **Redis GET, in-region, p50** | **0.3–1 ms** |
| **Redis GET, in-region, p99** | **2–5 ms** (worse with big payloads or O(N) commands) |
| Cross-AZ round trip | 0.5–2 ms |
| Indexed single-row SQL query | 1–10 ms |
| Multi-join analytical query | 50 ms – several s |
| NVMe random read | ~100 µs |
| CDN edge hit | 10–50 ms (dominated by client RTT) |
| CDN miss → origin fetch | +100–500 ms |
| Cross-region round trip | 70–150 ms |

**The three inferences that matter:**

1. **In-process L1 is ~1000× faster than Redis.** Not 2×. That gap is why a two-tier cache exists at all: for a very hot key at high RPS, the network hop to Redis *is* your latency budget. It's also why `HybridCache` was designed L1-first.
2. **Redis is not free.** At 0.5 ms per round trip, 30 sequential cache gets to render a page is 15 ms of pure network. **The fix is batching/pipelining, not a faster cache** — this is an extremely common real bug and a great thing to raise in a design round.
3. **Caching to avoid a 2 ms indexed query is usually the wrong optimization.** You've swapped a 2 ms transactional read for a 0.5 ms stale read and taken on an invalidation problem. Cache the 200 ms aggregation instead. Saying "I wouldn't cache that — the query is already fast; I'd cache the expensive aggregate above it" is exactly the judgment being scored.

---

# Part B — The patterns

## Concept 7 — Cache-aside (lazy loading): the default

The application owns the cache logic. On read: check cache → on miss, load from store → populate cache → return.

```csharp
public async Task<Product?> GetProductAsync(int id, CancellationToken ct)
{
    var key = $"catalog:v2:product:{id}";

    if (await _cache.TryGetAsync<Product>(key, ct) is { } cached)
        return cached;                                 // hit

    var product = await _repository.GetAsync(id, ct);  // miss → origin
    if (product is not null)
        await _cache.SetAsync(key, product, TimeSpan.FromMinutes(5), ct);

    return product;
}
```

**Why it's the default:** only requested data is cached (memory follows the working set automatically), the store stays the source of truth, and a cache outage degrades to slow rather than broken — *if* you wrote it that way.

**The four failure modes, all worth naming:**

| Failure mode | Mechanism | Mitigation |
|---|---|---|
| **Miss penalty** | Every miss pays cache RTT + origin latency + fill | Bound with timeouts; refresh-ahead (Concept 12) |
| **Cold start** | New instance / flush / deploy → hit ratio 0 | Warm-up, staged rollout, L2 shared across instances (Concept 24) |
| **Stale-set race** | A reader writes a pre-update value *after* the writer invalidated | Concept 14 — the important one |
| **Stampede** | N concurrent misses on the same key → N origin calls | Single-flight (Concept 22) |

**Cache-aside with no stampede protection is the single most common production caching bug in .NET.** `IMemoryCache.GetOrCreate` does **not** deduplicate concurrent factory invocations — 500 simultaneous requests for a just-expired key produce 500 database queries. That fact, stated plainly, is worth real credit, and it's the reason `HybridCache` exists (Concept 33).

---

## Concept 8 — Read-through: same behaviour, different ownership

The caller asks the *cache* for the value; the cache calls a loader function on miss. Behaviourally identical to cache-aside — **the difference is where the logic lives**, and therefore who is responsible for stampede protection, serialization, and instrumentation.

```csharp
// HybridCache: read-through, with stampede protection and tags built in.
var product = await _hybrid.GetOrCreateAsync(
    $"catalog:v2:product:{id}",
    id,                                                    // state, avoids closure allocation
    static async (id, ct) => await _repo.GetAsync(id, ct),
    new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(5) },
    tags: [$"product:{id}", "catalog"],
    cancellationToken: ct);
```

**Why prefer it:** centralizing the load path is the only practical way to get consistent single-flight, metrics, and serialization policy across a codebase. Hand-rolled cache-aside scattered across 40 repositories will have 40 subtly different TTL and error-handling behaviours.

**Interview framing:** *"Cache-aside and read-through are the same data flow; read-through just moves ownership into the caching layer, which is how you get stampede protection and observability without repeating yourself. In .NET that's `HybridCache.GetOrCreateAsync`."*

---

## Concept 9 — Write-through

Write to the cache **and** the store synchronously, in the same logical operation, before acknowledging.

- **Buys:** the cache is populated with fresh data immediately; subsequent reads hit; read-your-writes works on the writer's path.
- **Costs:** every write pays both latencies; you cache data that may never be read (bad for write-heavy, low-read-locality data); and **you have a two-phase write problem** — if the store commit succeeds and the cache write fails (or vice versa) you have divergence, with no transaction spanning them.
- **Still racy.** Two concurrent writers can commit to the store in order W1, W2 but reach the cache in order W2, W1, leaving the cache holding the *older* value indefinitely. This is exactly why Concept 11 says to delete rather than update.

**Where it's genuinely right:** session state, shopping carts, user preferences, and anything where the writer is overwhelmingly likely to read it back immediately. Combine with versioned writes (only set if the incoming version is newer) to fix the ordering race.

---

## Concept 10 — Write-behind (write-back)

Write to the cache; acknowledge; flush to the store asynchronously, usually batched and coalesced.

- **Buys:** very low write latency, and **write coalescing** — 1,000 increments of a view counter become one store write. That coalescing is the real reason to use it.
- **Costs:** a **data-loss window** equal to your flush interval if the cache node dies, and the cache is temporarily the source of truth for unflushed data. Read paths must go through the cache or see stale data. Ordering and idempotency of the flush are on you.

**Where it's right:** view counts, like counts, metrics, last-seen timestamps, rate-limit counters, telemetry — high-frequency, low-value-per-write, aggregate-tolerant data. `INCR` in Redis with a periodic flush is the canonical shape.

**Where it's wrong:** anything financial, anything a user is told "saved", anything auditable.

**The .NET shape:** Redis `HINCRBY`/`INCR` for the counter, a `PeriodicTimer` background service flushing on an interval, and `Channel<T>` as the in-process buffer if you're coalescing before you even reach Redis. Say explicitly: *"the loss window is the flush interval; I'd size it to the business tolerance and make the flush idempotent with an absolute value rather than a delta where possible."*

---

## Concept 11 — Write-around and invalidate-on-write: **delete, don't update**

The most common production pattern: **write to the store, then delete the cache key.** The next reader re-populates.

**Why delete beats update — three independent reasons. Have all three ready:**

1. **It eliminates the write-ordering race.** With `SET`, concurrent writers can apply to the store in one order and to the cache in another, and the cache can hold the older value *forever*. With `DEL`, the worst case is redundant deletes; the value is then re-read from the store, which has already serialized the writes. **Delete is idempotent and commutative; set is neither.**
2. **It avoids wasted work.** Updating requires materializing the cached shape (often a DTO assembled from several tables) on the write path, for a value that may never be read.
3. **It keeps the cached shape in one place.** If the cached value is a projection, only the read path should know how to build it. Duplicating that logic into every writer is a maintenance trap and a divergence source.

**The residual hazard:** delete-on-write is **best-effort**. If the delete fails (Redis unreachable, process crashed between commit and delete), the stale value survives until TTL. Hence: **always set a TTL, even when you invalidate explicitly.** The TTL is your correctness backstop; the delete is an optimization on top. This is one of those rules that sounds trivial and that a surprising number of production systems violate.

**Ordering: commit first, then delete.** If you delete first, a concurrent reader can miss, read the *pre-commit* value from the store, and re-populate the cache with stale data before your commit lands — turning a transient staleness into one that outlives the TTL reset. Commit-then-delete narrows the window but doesn't close it (Concept 14).

---

## Concept 12 — Refresh-ahead, stale-while-revalidate, and probabilistic early refresh

The problem with pure TTL expiry: the *unluckiest* request pays the full origin latency, and on a hot key **many** requests pay it simultaneously.

Three mitigations, in increasing sophistication:

**1. Background refresh (refresh-ahead).** A background job or timer re-computes hot entries before expiry. Simple; requires knowing which keys are hot; wastes work on keys that go cold.

**2. Stale-while-revalidate (SWR).** Keep two timestamps: a *soft* expiry and a *hard* expiry. Past soft expiry, serve the stale value immediately **and** kick off a background refresh. Past hard expiry, block and refresh. **No user ever waits for a refresh** unless the entry is truly ancient. This is the single most valuable caching pattern for tail latency, it maps directly to the HTTP `stale-while-revalidate` directive (Concept 37), and in .NET it's FusionCache's *eager refresh* + *fail-safe* combination (Concept 34).

**3. Probabilistic early expiration (XFetch).** From *Optimal Probabilistic Cache Stampede Prevention* (Vattani, Chierichetti & Lowenstein, VLDB 2015). Store the recomputation cost `delta` with the value; on read, refresh early if:

```
now − delta · β · ln(rand()) ≥ expiry          // β ≥ 1, typically 1.0
```

The elegance: refresh probability rises smoothly as expiry approaches, scaled by how *expensive* the recomputation is, so expensive entries get refreshed earlier and the herd is spread out rather than synchronized. It needs no locks and no coordination. Citing this paper by name when asked about stampedes is a genuine differentiator.

```csharp
// XFetch, ~10 lines. delta = measured recompute duration, stored alongside the value.
static bool ShouldRefreshEarly(DateTimeOffset expiry, TimeSpan delta, double beta = 1.0)
{
    var u = Random.Shared.NextDouble();          // (0,1)
    var earliness = delta.TotalSeconds * beta * -Math.Log(u);
    return DateTimeOffset.UtcNow.AddSeconds(earliness) >= expiry;
}
```

---

## Concept 13 — Materialization, fan-out direction, and negative caching

**Fan-out on read vs fan-out on write** is the classic caching-as-architecture decision, and it's really a question of *where you pay the amplification*.

| | Fan-out on read | Fan-out on write |
|---|---|---|
| How | Assemble the view at request time from N sources (cached individually) | Precompute and store the assembled view on every write |
| Read cost | N lookups + assembly | 1 lookup |
| Write cost | 1 write | N writes (one per consumer/view) |
| Wins when | Writes are frequent, reads are rare, or fan-out is huge | Reads dominate and fan-out is bounded |

The canonical example is a social timeline: fan-out on write (push each post into every follower's precomputed timeline) gives O(1) reads but a celebrity with 50M followers produces 50M writes per post. The universal answer is the **hybrid**: fan-out on write for normal accounts, fan-out on read for high-follower accounts, merged at read time. Knowing that the hybrid is the answer — and that the threshold is an operational tuning knob, not a constant — is the expected senior response.

**Negative caching.** Cache the *absence* of a value. Without it, a request for a non-existent key misses every time and goes to the origin every time — which makes key enumeration a free denial-of-service against your database. Rules:
- Use a **shorter TTL** than for positive entries (typically 10–60 s), because a negative entry becomes wrong the instant the entity is created.
- Use a **distinguishable sentinel**, not `null` — you must be able to tell "cached miss" from "not in cache". (`HybridCache` and FusionCache both handle `null` as a cacheable value; hand-rolled `IDistributedCache` code usually gets this wrong.)
- For very large keyspaces, a **Bloom filter** in front is cheaper than negative entries: "definitely not present" answers with ~1% false positives in a few bits per key. RedisBloom gives you this server-side.

---

# Part C — Invalidation and consistency

## Concept 14 — The stale-set race: draw this timeline

This is the concept most candidates cannot articulate, and being able to draw it is disproportionately valuable.

**Setup:** cache-aside reads, commit-then-delete writes. Key `product:42` is not cached. Current DB value is `v1`.

```
Time   Reader R                          Writer W                       Cache state
 t0    GET product:42 → MISS                                            (empty)
 t1    SELECT ... → reads v1
 t2                                       UPDATE ... → commits v2       (empty)
 t3                                       DEL product:42                (empty)
 t4    SET product:42 = v1                                              v1   ← STALE
 ...   every subsequent read returns v1 until the TTL expires
```

The reader's `SET` lands *after* the writer's `DEL`, so the delete has nothing to delete and the stale value is installed afterwards. **The invalidation was correct and still lost the race.** Staleness now persists for the full TTL, not for the width of the race window — that's what makes it nasty. It is rare (it needs a read that straddles a write) but at high RPS, rare means "several times an hour".

**The mitigations, and what each actually guarantees:**

| Mitigation | Mechanism | Guarantee |
|---|---|---|
| **Short TTL** | Bound the damage | Staleness ≤ TTL. Always do this; never the only defence |
| **Delayed double delete** | Delete again ~0.5–1 s after commit | Probabilistic; widely used, admits it's a hack |
| **Versioned value + monotonic set** | Store `(value, version)`; `SET` only if incoming version ≥ stored | **Eliminates it.** Needs a version (rowversion/xmin/ETag) and a Lua script or `HybridCache` wrapper |
| **Versioned keys** | The version is *in the key*; a stale write targets a key nobody reads | **Eliminates it structurally.** Concept 16 — the cleanest answer |
| **Lease on miss** (Facebook memcache) | Miss returns a lease token; `SET` is rejected if the key was invalidated since the lease was issued | **Eliminates it.** The canonical solution, from *Scaling Memcache at Facebook* |
| **Write-through under a lock** | Serialize the read-fill and the write | Correct, and slow — you've added coordination (Module 9) to the write path |
| **CDC-driven invalidation** | Invalidate from the DB's change log after commit, comparing versions | Eliminates it, at the cost of an invalidation pipeline |

**How to answer when asked "how do you keep the cache consistent with the database?"** — *"Strictly, you can't without coordination, so I pick a bound. My default is commit-then-delete plus a TTL as a backstop, which leaves a narrow stale-set race I can draw for you. If the data can't tolerate that, I move up the ladder: versioned keys, or a lease-on-miss like Facebook's memcache, or I stop caching it."* That answer scores because it names the limit rather than pretending to defeat it.

---

## Concept 15 — The invalidation ladder

Exactly like Module 9's lock ladder: **start at the top and only descend when the requirement forces you.**

| Rung | Strategy | Staleness | Cost/complexity | Use when |
|---|---|---|---|---|
| 0 | **Don't cache** | None | None | Strict invariants, low read rate, already-fast reads |
| 1 | **TTL only** | ≤ TTL, always | Trivial | Tolerable staleness; reference data; the default |
| 2 | **TTL + delete on write** | Usually ms, worst case TTL | Low | The workhorse for entity caches |
| 3 | **Versioned keys** | None for the *shape* you version | Low–medium | Deploys, schema changes, tenant-wide flushes, dependent projections |
| 4 | **Event-driven (CDC/outbox)** | Bounded by pipeline lag | Medium–high | Many consumers, cross-service caches, need for audit |
| 5 | **Lease / fence on fill** | None (no stale sets possible) | High | Very hot keys where the race actually bites |
| 6 | **Coordinated (lock per key)** | None | Highest; you re-added coordination | Almost never — at this point ask why you're caching |

**Rungs 1 and 2 cover the overwhelming majority of real systems.** The interview value is in knowing rungs 3–5 exist and being able to say which rung a given requirement demands. Jumping to rung 6 ("I'd take a distributed lock on every write") is the classic over-engineering tell, and after Module 9 you know exactly why it's expensive.

---

## Concept 16 — Versioned and namespaced keys: invalidation without deletes

The trick: **make the version part of the key.** Then invalidation is a single small write, and stale writers target keys nobody will ever read.

```
catalog:v3:tenant:88:product:42:full          // v3 = cached-shape schema version
user:88:prefs:gen:17                          // gen = per-user generation counter
```

Three uses, in increasing power:

1. **Schema/shape version (`v3`)** — bump it in configuration when you change the DTO. The old namespace ages out by TTL; no purge, no deploy-time cache stampede on a `FLUSHDB`, no risk of deserializing v2 bytes into a v3 type. **This alone prevents a whole class of deployment incident** and costs one constant.
2. **Entity generation counter** — store `gen:user:88 → 17` in the cache. Reads fetch the generation (cheap, itself cacheable in L1 for a second) and build the data key from it. A write does `INCR gen:user:88`, which atomically orphans every derived entry for that user — *one operation to invalidate an unbounded set of dependent keys*, including keys whose names you don't know. This is how you invalidate "everything derived from this entity" without maintaining a dependency index.
3. **Tenant / group namespace** — the same technique at tenant granularity, which makes "flush tenant 88" one `INCR` instead of a `SCAN`-and-delete over a shared keyspace.

**Cost:** an extra round trip for the generation (mitigate with a short-TTL L1), and orphaned entries occupying memory until they expire (mitigate with TTLs and eviction — which is exactly what eviction is for).

**Tag-based invalidation is the managed version of this idea.** `HybridCache.RemoveByTagAsync` and ASP.NET Core output caching's tags let you attach logical tags to entries and evict a whole tag at once. Under the hood these are generation/timestamp records; understanding the primitive means you can explain what the abstraction is buying you and where it can't help (a tag can't express "everything except X").

---

## Concept 17 — Event-driven invalidation: CDC, outbox, and change feeds

Once more than one service or more than one cache consumes the same data, invalidation should be a **published event**, not a line of code in every writer.

**Transports, from most to least coupled to the database:**

| Transport | Mechanism | Notes |
|---|---|---|
| **CDC from the log** | Debezium / SQL Server CDC / Postgres logical decoding | Cannot be forgotten by a developer, because it reads the commit log. Facebook's `mcsqueal` does exactly this from MySQL binlogs |
| **Cosmos DB change feed** | Ordered per-partition feed of changes | The Azure-native version; drive an Azure Function that invalidates |
| **Transactional outbox** | Write the event in the same transaction, relay after commit | Module 11's pattern; application-level, explicit, testable |
| **Redis pub/sub or Streams** | Fan-out invalidation messages to instances | The **backplane** for L1 invalidation. Pub/sub is **at-most-once** — a disconnected subscriber misses messages permanently, which is why TTL remains the backstop. Streams give you replay from an offset |
| **Domain events in-process** | `MediatR` notification → cache invalidator | Fine for a modular monolith; invisible to other services |

**Two rules that make event-driven invalidation actually safe:**
1. **Carry the version in the event** and make the cache set conditional on it (`only overwrite if incoming version > stored version`). Without this, an out-of-order invalidation-then-refill can reinstall old data.
2. **Make invalidation idempotent and cheap** so duplicates (at-least-once delivery, Module 9's Concept 28) are harmless. `DEL` is naturally idempotent; `INCR gen:` is not — use `SET gen: = max(...)` semantics or accept over-invalidation.

---

## Concept 18 — Key design (the unglamorous part that causes real incidents)

**A convention worth stating in an interview:**

```
{app}:{cache-schema-version}:{tenant}:{entity}:{id}:{shape}
orders:v4:t88:order:10023:summary
```

| Consideration | Guidance |
|---|---|
| **Tenant isolation** | Tenant ID in the key, always, in a multi-tenant system. A missing tenant segment is a cross-tenant data leak, and it's a *caching* bug that reads as a *security* incident |
| **Shape** | `:summary` vs `:full` vs `:v2` — two different projections of one entity must not share a key. Include culture/locale and any `Accept`-dependent variation |
| **Version prefix** | Concept 16. Cheap insurance against deploy-time deserialization failures |
| **Cardinality** | Keys must be bounded. Including a free-text search string or a timestamp in the key gives unbounded cardinality, ~0% hit ratio, and a memory leak. **Bucket or round anything continuous** |
| **Length** | Redis keys are binary-safe up to 512 MB but every byte is resident memory; at 100M keys a 40-byte saving is 4 GB. Hash long composite keys (`SHA-256`, truncated) and keep a readable prefix for debugging |
| **Authorization in the key** | If the response varies by permission, the permission set must be in the key — or you will serve one user's view to another. This is the single most dangerous key-design mistake |
| **No PII in keys** | Keys show up in slow logs, `SCAN` output, monitoring, and traces. Hash identifiers where regulation applies |

---

## Concept 19 — Read-your-writes: the bug your users will report

Symptom: "I saved it and it didn't change." Four fixes, in order of preference:

1. **Write-through on the writer's path only.** After committing, update (or delete and re-fill) the cache synchronously before returning the response. Cheap, effective for the common case, doesn't help other cache layers or other instances.
2. **Per-user generation bump** (Concept 16). `INCR gen:user:88` on commit; the user's subsequent reads build keys from the new generation and cannot hit stale entries, on any instance, at any layer that respects the key. **The most robust application-level fix.**
3. **Skip-cache window.** After a write, set a short marker (`user:88:nocache` with a 5-second TTL); reads for that user bypass the cache while it exists. Simple and very effective; costs a marker lookup per read.
4. **Route the writer to the source of truth.** Read-after-write goes to the primary DB, not the cache or replica. Correct, and it exposes the origin to whatever traffic follows writes.

At the HTTP layer the equivalent problem appears at the CDN, and the fixes are `Cache-Control: private` for personalized responses, or a cache-busting query/fingerprint after a mutation. Mentioning that the same bug exists at *three* layers (browser, CDN, app cache), with the same shape and different fixes, is a strong architectural signal.

---

# Part D — Eviction, stampedes, hot keys, and cold caches

## Concept 20 — Eviction policies, and why W-TinyLFU is the modern default

Eviction answers: *when memory is full, which entry dies?* Expiry answers: *when does an entry become invalid?* They are different mechanisms and mixing them up is a common error.

| Policy | Rule | Strength / weakness |
|---|---|---|
| **FIFO** | Oldest inserted | Trivial, ignores popularity. Rarely right |
| **LRU** | Least recently used | Good with temporal locality; **destroyed by scans** — one sequential sweep evicts your whole working set |
| **LFU** | Least frequently used | Robust to scans; **slow to adapt** — yesterday's hot key keeps its count (needs aging) |
| **ARC** | Adaptive balance of recency and frequency | Excellent hit ratios; patent history pushed most OSS away from it |
| **W-TinyLFU** | Small LRU admission window + frequency-based *admission filter* (Count-Min Sketch with periodic halving) guarding an SLRU main region | **The modern default.** Near-ARC hit ratios, scan-resistant, tiny metadata (~4 bits/entry). Caffeine (Java), `BitFaster.Caching` `ConcurrentLfu` (.NET) |
| **Belady's MIN** | Evict the entry used furthest in the future | Not implementable (needs the future) — but it's the **upper bound** you measure your policy against in offline simulation |
| **TTL-only / random** | No recency or frequency | Redis's `allkeys-random` — surprisingly acceptable when access is near-uniform, and cheaper |

**The insight to voice:** the admission decision matters more than the eviction decision. W-TinyLFU's contribution is asking "is this new item more valuable than the victim it would displace?" *before* admitting it — which is why a scan can't flush a warm cache. That framing ("admission, not just eviction") is a strong signal in a deep-dive round.

**Redis's actual implementation** is approximate, deliberately: it samples `maxmemory-samples` keys (default 5) and evicts the best candidate from the sample rather than maintaining a true global LRU list, because true LRU costs memory and pointer churn. Its LFU mode uses an 8-bit logarithmic counter with time-based decay. **"Redis LRU is sampled and approximate"** is a nice piece of precision to have.

---

## Concept 21 — Expiry mechanics, and the sliding-expiration trap

- **Absolute expiration** — dies at a fixed time. Predictable, bounds staleness. **The default you should reach for.**
- **Sliding expiration** — TTL resets on each access. **The trap:** a continuously-accessed entry *never expires*, so its staleness is unbounded. Sliding expiration is correct for *session-like* data (where "idle timeout" is the actual requirement) and wrong for *cached projections of mutable data*. The fix is what `IMemoryCache` supports directly: sliding window **plus** an absolute expiration ceiling.
- **Jitter.** Entries created together expire together. Seed a cache during startup or a deploy and you've built a synchronized expiry storm. **Always randomize: `ttl × (1 + rand(±10–20%))`.** This one line prevents a whole category of incident and costs nothing.
- **Redis expiry is lazy + active:** a key is removed when touched after expiry, and a background cycle samples volatile keys to reclaim them. Consequence: **expired-but-unreclaimed keys still consume memory**, so `used_memory` can exceed live data, and `DBSIZE` can overstate. It also means expiry is not instant.
- **Replica expiry:** replicas don't expire keys independently — they wait for the primary's `DEL`. A read from a replica can return a logically-expired key (Redis filters this on read for correctness, but the memory is held). Worth knowing if you're reading from replicas.

---

## Concept 22 — Stampede, thundering herd, dogpile: three different problems

People use these interchangeably. Distinguishing them is a differentiator:

1. **Single-key stampede (dogpile).** One hot key expires; N concurrent requests miss and all call the origin. Damage is proportional to the key's popularity.
2. **Mass expiry / cold start.** Many keys become invalid at once (deploy, flush, failover, synchronized TTLs). Damage is proportional to *total* traffic — this is Concept 24.
3. **Retry amplification.** The origin slows under the herd, clients time out and retry, and the herd grows. A positive feedback loop that turns a latency blip into an outage. This is the Module 13 material (circuit breakers, budgets) meeting caching.

**The mitigation stack — name several, not one:**

| Mitigation | What it fixes | Cost |
|---|---|---|
| **Single-flight / request coalescing** | (1) — one factory call per key per process | A lock/`Task` per key; still N calls across N instances |
| **Distributed single-flight** (lock in Redis) | (1) across the fleet | Coordination on the miss path; needs lease + fallback (Module 9) |
| **Probabilistic early refresh (XFetch)** | (1), without any locks | Slightly more origin work overall |
| **Stale-while-revalidate / fail-safe** | (1) and (3) — nobody waits, nobody retries | Serves stale data by design |
| **TTL jitter** | (2) | None. Always do it |
| **Negative caching** | Miss storms on non-existent keys | Short-lived wrong answers for new entities |
| **Bounded origin concurrency** (bulkhead/semaphore) | (2) and (3) — caps the blast | Queuing or shedding when saturated |
| **Warm-up on deploy** | (2) | Startup time, and a list of keys to warm |

**In .NET, concretely:**

```csharp
// Per-key single-flight over IMemoryCache, the pattern to know.
// Cache a Task<T>, not a T — so concurrent callers await the same in-flight work.
private readonly ConcurrentDictionary<string, Lazy<Task<Product?>>> _inflight = new();

public Task<Product?> GetAsync(int id, CancellationToken ct)
{
    var key = $"product:{id}";
    if (_memory.TryGetValue(key, out Product? hit)) return Task.FromResult(hit);

    var lazy = _inflight.GetOrAdd(key, k => new Lazy<Task<Product?>>(async () =>
    {
        try
        {
            var p = await _repo.GetAsync(id, ct);
            _memory.Set(key, p, Jitter(TimeSpan.FromMinutes(5)));
            return p;
        }
        finally { _inflight.TryRemove(key, out _); }
    }, LazyThreadSafetyMode.ExecutionAndPublication));

    return lazy.Value;
}
```

**Better answer: don't write that.** `HybridCache.GetOrCreateAsync` has stampede protection built in, and FusionCache adds fail-safe and timeouts on top. The reason to know the hand-rolled version is to explain *what those libraries are doing* — which is the question you'll actually be asked.

---

## Concept 23 — Hot keys

A single key can exceed what one cache node (or one Redis shard, since slots map to one node) can serve. Symptoms: one Redis shard at 100% CPU while others idle; p99 spikes on one key class.

**Detection:**
- Redis `--hotkeys` (uses `OBJECT FREQ`, requires an LFU policy), `MONITOR` (never in production — it serializes everything), `SLOWLOG`, or keyspace notifications.
- Client-side **Count-Min Sketch / Top-K** (RedisBloom's `TOPK`, or an in-process sketch) sampling requests. A Count-Min Sketch gives you approximate per-key frequency in kilobytes, which is the standard solution and a nice thing to name.

**Mitigations:**

| Fix | How | Trade-off |
|---|---|---|
| **In-process L1 for hot keys** | Short-TTL local copy; the hop disappears entirely | Per-instance staleness; **usually the right answer** |
| **Key splitting / replication** | Store `product:42#0..#9`, read a random replica, write all | 10× memory for that key; invalidation must hit all copies |
| **Read replicas** | Serve reads from Redis replicas | Stale reads; not available in every topology |
| **Client-side caching (RESP3 tracking)** | `CLIENT TRACKING` — server pushes invalidations to clients | Redis-native L1 with invalidation; .NET client support is limited, so `HybridCache` + backplane is the practical .NET equivalent |
| **Consistent hashing with bounded loads** | Spill overflow to the next node | Module 8 material; helps distribution, not a single hot key |

**Worth saying:** *"A hot key is not a capacity problem you can shard away, because the key maps to exactly one slot. You fix it by adding a tier above the cache, or by splitting the key into replicas."*

---

## Concept 24 — The cold cache, and the death spiral

**This is the question that separates people who have operated caches from people who have configured them.** When asked "what happens if Redis goes down?", the naive answer is "we fall back to the database." At 95% hit ratio, that fallback is a **20× instantaneous load increase** on a database sized for 5% of traffic. The database saturates, latency climbs, clients time out, clients retry, the effective load grows further, and you are down — even though your "fallback" worked exactly as designed.

Amazon's Builders' Library calls the underlying hazard **modality**: a system whose behaviour differs fundamentally between warm and cold has an untested mode, and untested modes fail. The fix is to reduce the difference between modes, or to make the cold mode survivable.

**The playbook — a strong answer names four or five of these:**

| Defence | Mechanism |
|---|---|
| **Bounded origin concurrency** | A semaphore/bulkhead capping concurrent origin calls; excess requests fail fast or queue briefly. Converts an outage into partial degradation |
| **Load shedding with priority** | Shed non-critical read traffic first (recommendations before checkout) |
| **Request coalescing** | Single-flight means a cold key costs one origin call, not N |
| **Serve stale on error (fail-safe)** | Keep expired entries and serve them when the origin fails. FusionCache's headline feature; `stale-if-error` at the HTTP layer |
| **In-process L1** | Survives an L2 outage entirely for hot keys — the cheapest real insurance |
| **Gutter pool** | Facebook's trick: a small standby pool absorbs traffic for a failed cache node with a short TTL, so the origin never sees the shard's full miss traffic |
| **Staged warm-up** | On startup, pre-populate the top-N keys before taking traffic; join the load balancer only when warm (readiness probe gated on cache warmth) |
| **Don't `FLUSHALL`** | Prefer namespace-version bumps (Concept 16) so old entries age out gradually instead of vanishing at once |
| **Capacity-test cold** | Run the load test with the cache disabled and know the real number. Then decide, explicitly, whether you accept it |

**The architect-level framing:** *"Adding a cache moves load off the database, which tempts you to downsize the database. The moment you do that, the cache becomes a hard dependency — it is no longer an optimization, it's load-bearing. That's a legitimate choice, but it must be a deliberate one, and it changes the availability calculation: your effective availability is now the cache's availability too."* That's the sentence that gets remembered in a debrief.

---

# Part E — Redis, in the depth an interview expects

## Concept 25 — The execution model, and why O(N) commands are latency bombs

**Redis executes commands on a single thread.** (Since 6.0 there are optional I/O threads for socket read/write and TLS, and background threads for `UNLINK`/`FLUSHALL ASYNC`/fsync — but *command execution* is serialized.) Consequences:

1. **Every command is atomic** by construction. No locks needed for `INCR`, `SETNX`, `LPUSH`. This is why Redis is a usable coordination primitive at all (Module 9).
2. **One slow command blocks every client.** `KEYS *` over 10M keys, `HGETALL` on a 500k-field hash, `SMEMBERS` on a huge set, `LRANGE 0 -1`, `DEL` of a multi-GB structure, or a long Lua script — each stalls the whole server for its duration. **This is the number-one cause of mysterious Redis p99 spikes**, and naming it is a strong operational signal.
3. **The fixes:** `SCAN`/`HSCAN`/`SSCAN` with cursors instead of `KEYS`/`HGETALL`; `UNLINK` instead of `DEL` for large values (frees memory on a background thread); keep Lua scripts short; avoid unbounded collections as cache values.
4. **Throughput comes from pipelining, not parallelism.** Redis does ~100k+ simple ops/sec per core; the bottleneck in practice is round trips. **30 sequential `GET`s = 30 RTTs ≈ 15 ms.** Pipelined or `MGET`'d, it's one. In .NET, StackExchange.Redis multiplexes and pipelines automatically across a shared `ConnectionMultiplexer`, but only if you don't `await` each call in sequence — fire the tasks, then `await Task.WhenAll`, or use `MGET`/batches.

---

## Concept 26 — Data structures as cache primitives

The reason to know these is that "cache" often means "the right data structure, server-side" rather than "a serialized blob".

| Structure | Cache use | Key commands |
|---|---|---|
| **String** | Serialized DTO, counter, flag, lock | `GET`/`SET`/`SETEX`/`SET NX PX`/`INCR`/`GETEX` |
| **Hash** | Field-level updates without rewriting the whole object; per-object metadata | `HGET`/`HSET`/`HMGET`; **`HEXPIRE` (Redis 7.4) gives per-field TTL** — new and useful |
| **Sorted set** | Leaderboards, sliding-window rate limits, time-ordered indexes, priority queues | `ZADD`/`ZRANGEBYSCORE`/`ZREMRANGEBYSCORE`/`ZINCRBY` |
| **Set** | Tag membership, dedup, "has this user seen X" | `SADD`/`SISMEMBER`/`SINTERCARD` |
| **List** | Recent-items feed, simple queue | `LPUSH`/`LRANGE`/`LTRIM` (cap the length!) |
| **Bitmap / bitfield** | Per-user flags, daily-active-users, feature exposure — 1 bit per user | `SETBIT`/`BITCOUNT`/`BITFIELD` |
| **HyperLogLog** | Approximate unique counts in **12 KB with ~0.81% error**, regardless of cardinality | `PFADD`/`PFCOUNT`/`PFMERGE` |
| **Stream** | Invalidation event log with replay from an offset; consumer groups | `XADD`/`XREADGROUP`/`XAUTOCLAIM` |
| **Bloom / Cuckoo filter** (module) | "Definitely not present" to avoid origin lookups | `BF.ADD`/`BF.EXISTS` |
| **Count-Min / Top-K** (module) | Hot-key and heavy-hitter detection | `CMS.INCRBY`/`TOPK.ADD` |
| **Geo** | Nearby-entity queries | `GEOADD`/`GEOSEARCH` |

**The interview move:** when asked to design a rate limiter, leaderboard, or "seen it" check, reach for the structure rather than a blob. *"I'd use a sorted set keyed per user with the timestamp as the score, trim with `ZREMRANGEBYSCORE`, and do the whole check-and-increment in one Lua script so it's atomic"* is a complete, correct answer in one sentence.

---

## Concept 27 — Redis consistency: never the source of truth

**Redis replication is asynchronous.** The primary acknowledges a write before replicas have it. On failover, acknowledged writes can be lost. `WAIT numreplicas timeout` blocks until N replicas have acknowledged — but it is **not consensus**: it doesn't prevent a stale replica from being promoted, and it doesn't make the system linearizable. (Jepsen's analyses of Redis are the canonical reference; Kleppmann's Redlock critique in Module 9 is the same theme.)

Therefore:
- **Cached data: fine.** Losing a cache entry costs a miss.
- **Source-of-truth data: not fine.** Session state that must never vanish, balances, idempotency records that gate money movement — these need a durable store, or Redis with AOF `appendfsync always` (which costs you most of the performance you came for) and an acceptance of the residual risk.
- **Locks: only with fencing** (Module 9, Concept 23). A cache that can lose writes cannot provide mutual exclusion.

**Redis Enterprise / Azure Managed Redis active-active geo-replication** is CRDT-based (Module 9, Concept 30): concurrent writes in two regions merge deterministically per data type rather than conflicting. That is genuinely different from async primary-replica, and knowing *why* (CRDT merge semantics, no cross-region invariants) is a strong architect-level detail.

---

## Concept 28 — Redis Cluster, slots, and hash tags

- **16,384 hash slots**; slot = `CRC16(key) mod 16384`; each node owns a range. Clients cache the slot→node map and redirect on `MOVED`/`ASK`.
- **Multi-key operations must target one slot.** `MGET a b c` across slots returns `CROSSSLOT`. This is the constraint that shapes key design in a clustered deployment.
- **Hash tags** force co-location: only the substring inside `{}` is hashed. `{t88}:order:1` and `{t88}:customer:9` land on the same node, so you can `MGET` them or run a Lua script over both. **Danger:** over-tagging (e.g. tagging by tenant when one tenant is huge) recreates a hot shard. Same trade-off as partition keys in Module 8 — co-location vs distribution — and pointing out that it's the *same* trade-off is a nice cross-module connection.
- **Sentinel vs Cluster:** Sentinel = HA for a single primary (failover, no sharding). Cluster = sharding + HA. Choose Cluster when the dataset or throughput exceeds one node; Sentinel is simpler and keeps multi-key operations working.
- **Resharding** moves slots live; clients must handle `ASK` redirects. Managed services do this for you but the latency blip is real.

---

## Concept 29 — Operating Redis as a cache

**`maxmemory-policy` — the trap worth knowing.** Options: `noeviction` (default in OSS), `allkeys-lru`, `allkeys-lfu`, `allkeys-random`, `volatile-lru`, `volatile-lfu`, `volatile-random`, `volatile-ttl`.

- **For a pure cache, use `allkeys-lru` or `allkeys-lfu`.** You want Redis to evict rather than fail.
- **`noeviction` on a cache is an outage waiting to happen**: at `maxmemory`, writes fail with OOM errors while reads keep working, so the symptom is "the cache stopped filling" rather than an obvious failure.
- **`volatile-*` policies only evict keys that have a TTL.** If some keys were written without a TTL, they are unevictable, and once they fill memory you get OOM on writes. This bites people on Azure Cache for Redis, whose default policy is `volatile-lru` — so **"every key gets a TTL"** is a hard rule, not a style preference.

**Persistence for a cache:** usually **disable RDB and AOF**. You don't need durability for derived data, and both cost you: RDB forks (copy-on-write can transiently double memory on a write-heavy instance), AOF costs fsyncs and rewrite CPU. The counter-argument is **warm restart** — persistence lets a restarted node come back warm and avoid Concept 24. Decide deliberately and say which you chose and why.

**Memory metrics to watch:**
- `used_memory` vs `used_memory_rss` → **fragmentation ratio**. >1.5 means real waste (enable `activedefrag`, or right-size); <1.0 means swapping, which is catastrophic for Redis.
- `evicted_keys` rate → you're under-provisioned (or your TTLs are too long for the memory you have).
- `expired_keys`, `keyspace_hits`/`keyspace_misses` → your real hit ratio, server-side.
- `blocked_clients`, `latest_fork_usec`, `instantaneous_ops_per_sec`, `connected_clients`.

**Memory is the cost driver.** A cache is priced in GB-months, and that's the number to compare against the alternative (Concept 41).

**Client-side issues in .NET** — worth having ready, because they're the errors you actually see:
- **Use one `ConnectionMultiplexer` for the app's lifetime**, registered as a singleton. Creating one per request exhausts sockets and destroys throughput. This is the single most common StackExchange.Redis mistake.
- **`RedisTimeoutException` is usually not Redis's fault.** The multiplexer's completion work runs on the thread pool, so **thread-pool starvation** (blocking on async, `.Result`, `.Wait()`) manifests as Redis timeouts. The diagnostic is in the exception payload's queue/busy-worker counts. Connecting this to Module 15's async material is exactly the kind of cross-cutting insight interviewers remember.
- Set sensible `ConnectTimeout`/`SyncTimeout`, enable `AbortOnConnectFail = false` so a transient startup failure doesn't permanently break the client, and treat cache calls as **cancellable and failable** — a cache timeout must degrade to an origin call, never to an error page.

---

## Concept 30 — Azure Redis in 2026: the lineup changed

This matters for an Azure-focused .NET interview because quoting the retired product line dates you.

- **Azure Managed Redis (AMR)** is now the strategic offering: built on Redis Enterprise software, GA since May 2025, with tiers by memory-to-vCPU ratio — **Memory Optimized, Balanced, Compute Optimized, and Flash Optimized** (NVMe-backed for very large, cost-sensitive datasets). It offers zone redundancy, **active geo-replication (CRDT-based)**, Redis modules (JSON, Search/vector, Bloom, TimeSeries), Private Link, and Entra ID authentication.
- **Azure Cache for Redis is being retired.** Enterprise/Enterprise Flash creation was blocked on **April 1, 2026**, with those instances retiring **March 31, 2027**. Basic/Standard/Premium creation is blocked during 2026, with retirement **September 30, 2028**. Microsoft provides migration tooling that repoints the old endpoint at a new AMR instance.
- **For new designs, name AMR.** If asked about the legacy tiers, the distinctions still worth knowing are: Basic = single node, no SLA (dev only); Standard = replicated pair; Premium = clustering, persistence, VNet injection, zone redundancy; Enterprise = Redis Enterprise engine with modules and active geo-replication.
- **Always mention Entra ID authentication over access keys**, and Private Link/VNet over public endpoints. Access keys in configuration is a finding in any security review.
- **Alternatives worth naming:** **Valkey** (the Linux Foundation fork of Redis, which AWS and Google have standardized on — so cross-cloud answers should mention it) and **Garnet**, Microsoft Research's RESP-compatible cache server built on Tsavorite/FASTER, which is wire-compatible with Redis clients including StackExchange.Redis and scales across cores far better than single-threaded Redis. Garnet is a credible answer to "how would you handle a cache hotspot that a single Redis thread can't serve?"

*(Cloud product lineups move; verify current tiers and dates against Microsoft's docs before an interview.)*

---

# Part F — The .NET surface

## Concept 31 — `IMemoryCache`

The in-process L1. Fast (tens of nanoseconds), per-instance, and full of sharp edges.

```csharp
builder.Services.AddMemoryCache(o =>
{
    o.SizeLimit = 50_000;                       // in *your* units — meaningless unless every entry sets a size
    o.CompactionPercentage = 0.25;              // how much to drop when the limit is hit
    o.ExpirationScanFrequency = TimeSpan.FromSeconds(30);
});

var entry = new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),   // bounds staleness
    SlidingExpiration = TimeSpan.FromMinutes(1),                 // idle timeout, *with* the absolute ceiling above
    Size = 1,
    Priority = CacheItemPriority.Normal                          // NeverRemove survives compaction
}.RegisterPostEvictionCallback((k, v, reason, _) => _metrics.Evicted(reason));

_memory.Set(key, value, entry);
```

**The things to know, in priority order:**

1. **`SizeLimit` is unenforced unless every entry sets `Size`.** No size limit = **an unbounded in-process cache = a memory leak with good intentions**. If you say nothing else about `IMemoryCache` in an interview, say this.
2. **`GetOrCreate`/`GetOrCreateAsync` do not deduplicate concurrent factory calls.** N concurrent misses = N factory invocations. (Concept 22.)
3. **`CancellationChangeToken` gives you grouped invalidation.** Register one token source across many entries and cancel it to evict them all at once — the in-process equivalent of tag invalidation, and a neat thing to know.
4. **Eviction is not proactive.** Expired entries are removed lazily on access or by a periodic scan, so memory is held after logical expiry.
5. **GC consequences.** A large `IMemoryCache` is a large set of long-lived references: they get promoted to **gen 2**, they make gen-2 collections more expensive, and any cached object over **85,000 bytes** goes on the **Large Object Heap** (not compacted by default, fragmenting over time). In a container, the .NET heap limit is derived from the cgroup memory limit — an in-process cache sized without reference to the pod limit produces OOMKills rather than evictions. **Cache small, serialized, or pooled representations of large payloads** (and if you must cache large buffers, look at `RecyclableMemoryStream` and `ArrayPool<byte>`). This paragraph is high-value: it's .NET-specific, it's about consequences rather than API surface, and it connects caching to Module 14.

---

## Concept 32 — `IDistributedCache`

The L2 abstraction: `byte[]` in, `byte[]` out, with `Get`, `Set`, `Refresh`, `Remove` (plus async variants). Implementations: Redis (`Microsoft.Extensions.Caching.StackExchangeRedis`), SQL Server, Cosmos DB, Garnet (via the Redis client), NCache, in-memory for tests.

**What it deliberately does not give you**, all of which you must supply yourself:

| Gap | Consequence |
|---|---|
| No stampede protection | Concept 22, unmitigated |
| No L1 | Every hit is a network round trip |
| No tags / grouped invalidation | You build versioned keys yourself |
| No typed API | You choose and own serialization |
| No `null`/negative caching semantics | Easy to conflate "absent" with "cached as null" |
| No per-entry metrics | You wrap it to get hit ratio |

**Serialization is the dominant cost at the L2 tier** — often more CPU than the network hop. `System.Text.Json` with a source-generated context is the sane default; **MessagePack or protobuf** cut payload size 30–60% and deserialize faster, which matters when you're moving 100 MB/s. Compress (Brotli/Gzip) only above a few KB, and measure: for small payloads compression costs more than the bandwidth it saves.

**The one-line verdict for an interview:** *"`IDistributedCache` is a lowest-common-denominator abstraction. For new code I'd use `HybridCache` instead, which sits on top of it and fills in the L1 tier, stampede protection, tags, and serialization."*

---

## Concept 33 — `HybridCache`: the current .NET default

Shipped with .NET 9 (`Microsoft.Extensions.Caching.Hybrid`) and the recommended approach for new .NET 9/10 projects. It is a **two-tier read-through cache** with the gaps above filled in.

```csharp
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = cs);   // optional L2
builder.Services.AddHybridCache(o =>
{
    o.MaximumPayloadBytes = 1024 * 1024;
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration      = TimeSpan.FromMinutes(10),   // L2 (total) lifetime
        LocalCacheExpiration = TimeSpan.FromMinutes(1) // L1 lifetime — shorter, bounds inter-instance divergence
    };
});

// Read-through with stampede protection, tags, and no closure allocation.
var order = await cache.GetOrCreateAsync(
    $"orders:v4:t{tenant}:order:{id}",
    (repo: _repo, id),                                  // state tuple
    static async (s, ct) => await s.repo.GetAsync(s.id, ct),
    tags: [$"order:{id}", $"tenant:{tenant}"],
    cancellationToken: ct);

await cache.RemoveByTagAsync($"tenant:{tenant}", ct);    // grouped invalidation
```

**What it gives you:**
- **L1 + L2 in one API**, with independent expirations — the key design lever: long L2 expiration for load reduction, short `LocalCacheExpiration` to bound how far instances can diverge.
- **Stampede protection** — concurrent `GetOrCreateAsync` calls for the same key collapse to a single factory execution.
- **Tag-based invalidation** via `RemoveByTagAsync`.
- **`static` factory + `state` overload** so the hot path allocates no closure — a small thing that signals you think about allocations.
- **`HybridCacheEntryFlags`** to disable individual tiers per call (`DisableLocalCacheWrite`, `DisableDistributedCacheRead`, `DisableUnderlyingData`, …) — useful for "read cache only, don't hit the DB" and for warm-up paths.
- **Pluggable serialization** (`IHybridCacheSerializer<T>`) so you can drop in MessagePack per type.

**What to check against current docs:** how tag invalidation propagates to *other instances'* L1 — the cross-instance L1 invalidation story is the part of the design that has evolved most, and it's a legitimate thing to say you'd verify. Being able to say *"the inherent hard part of a two-tier cache is invalidating N independent L1 copies; that needs a backplane, and I'd confirm what the framework gives me versus what I have to add"* is better than claiming certainty.

---

## Concept 34 — FusionCache: the resilience-first option

`ZiggyCreatures.FusionCache` is the most capable caching library in the .NET ecosystem and knowing it is a strong signal — it shows you've read beyond the docs. It can also be plugged in as a `HybridCache` implementation, so the choice isn't exclusive.

| Feature | What it does | Why it matters |
|---|---|---|
| **Fail-safe** | Keeps expired entries and serves them if the factory fails | Turns an origin outage into stale-but-up. The single most valuable feature here |
| **Soft/hard timeouts** | Give the factory a short soft budget; on overrun, return stale immediately and let the factory finish in the background | **Caps p99 at the soft timeout** instead of the origin's latency |
| **Eager refresh** | Refresh at a configurable fraction of the TTL, in the background | Concept 12's SWR, declaratively |
| **Backplane** | Redis pub/sub notifications so other instances evict their L1 | **The missing piece for multi-instance L1** |
| **Auto-recovery** | Queues and replays failed distributed operations/notifications after a blip | Invalidations aren't silently lost |
| **Adaptive caching** | The factory can choose the TTL based on the value it produced | Short TTL for volatile data, long for stable, from one code path |
| **Jitter, negative caching, OpenTelemetry** | Built in | The whole Concept 22 mitigation stack, configured rather than coded |

**Also worth naming:** `BitFaster.Caching` (`ConcurrentLru`, `ConcurrentLfu` — W-TinyLFU in .NET, for a bounded high-performance L1), `LazyCache` (simple single-flight over `IMemoryCache`), `EasyCaching`, and `CacheTower`.

---

## Concept 35 — Output caching, response caching, and static assets

Three different ASP.NET Core mechanisms, and conflating them is a common error.

| Mechanism | Who decides | Scope | Invalidation |
|---|---|---|---|
| **Response caching** (`AddResponseCaching`) | **The client's/proxy's HTTP headers** — respects `Cache-Control` from the request, won't cache authenticated responses | Emits correct headers for downstream caches | Expiry only |
| **Output caching** (.NET 7+, `AddOutputCache`) | **The server's policy** — ignores client headers by default | Server-side store; `IOutputCacheStore`, with a Redis-backed store for multi-instance | **Tag-based eviction** via `IOutputCacheStore.EvictByTagAsync` |
| **`MapStaticAssets`** (.NET 9+) | Build-time | Static files with content fingerprints, precompressed (gzip/brotli), ETags | Not needed — **the URL changes when the content changes** |

```csharp
builder.Services.AddOutputCache(o =>
{
    o.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(10)));
    o.AddPolicy("catalog", b => b
        .Expire(TimeSpan.FromMinutes(5))
        .SetVaryByQuery("page", "size")      // the cache key must include everything that changes the response
        .Tag("catalog"));
});
app.UseOutputCache();

app.MapGet("/products", Handler).CacheOutput("catalog");
// later, on a catalog write:
await store.EvictByTagAsync("catalog", ct);
```

**The judgment to voice:** output caching is **server-authoritative**, which is what you want for an API where you control freshness; response caching is for **cooperating with downstream caches** and is mostly the right tool when a CDN or proxy is doing the actual caching. And: **output caching a personalized response without varying by user identity is a data-leak bug** — `SetVaryByHeader`/`SetVaryByValue` or a policy that refuses to cache authenticated requests is mandatory.

---

## Concept 36 — EF Core and the data layer

**EF Core has no second-level cache, by design.** What it has:
- **First-level cache = the change tracker.** Within one `DbContext`, `Find`/tracked queries return the same instance (identity resolution). Scoped to the context's lifetime, which in ASP.NET Core is one request.
- **Query plan / compiled query caching.** EF caches the translation from LINQ to SQL. **`EF.CompileQuery`/`CompileAsyncQuery`** skips even that on hot paths. Note the trap: a query with a **large or variable `IN` list** produces a new SQL shape each time, blowing both EF's and the database's plan cache.
- **`AsNoTracking()`** for read-only queries: no change-tracking overhead, no identity resolution, less allocation. **This is the first thing to try before adding a cache**, along with projecting to a DTO (`Select`) instead of materializing entities.

**Should you add `EFCoreSecondLevelCacheInterceptor`?** It works, it's popular, and it's usually the wrong architectural answer, because it caches at the *query* level: cache keys are query shapes, invalidation is by table, and the result is coarse invalidation plus an opaque layer between your code and the database. **Prefer explicit application-level caching of well-defined read models** — you control the key, the shape, the TTL, and the invalidation, and you can reason about staleness per use case. Saying that, with the reasoning, is a distinctly architect-flavoured answer.

**And say the quiet part:** *"Before caching a slow query, I'd look at the query plan. A missing index, a `SELECT *` pulling 40 columns, an N+1, or a cartesian join from over-eager `Include`s are all cheaper to fix than to cache, and caching them hides the problem while adding a consistency one."* This reliably lands well, because interviewers see a lot of caches that exist to paper over a missing index.

---

# Part G — HTTP caching and the CDN

## Concept 37 — HTTP caching semantics

The layer most backend engineers under-use. Getting the headers right moves work off your infrastructure entirely — a browser or CDN hit costs you nothing.

| Directive | Meaning |
|---|---|
| `max-age=N` | Fresh for N seconds (all caches) |
| `s-maxage=N` | Fresh for N seconds for **shared** caches only (CDN/proxy) — overrides `max-age` there. Lets you set a short browser TTL and a long CDN TTL |
| `public` / `private` | Shared caches may / **may not** store it. **`private` is mandatory for personalized responses** |
| `no-cache` | May store, **must revalidate** before reuse. *Not* "don't cache" |
| `no-store` | Must not store at all. The one for sensitive data |
| `must-revalidate` | Stale responses may not be served, even on error |
| `immutable` | Never revalidate within `max-age` — for fingerprinted assets |
| `stale-while-revalidate=N` | Serve stale up to N s while refreshing in background (RFC 5861). **Concept 12 at the HTTP layer** |
| `stale-if-error=N` | Serve stale up to N s if the origin errors. **Fail-safe at the HTTP layer** |
| `Vary: H` | The cache key includes header H. `Vary: Accept-Encoding` is normal; **`Vary: Cookie` or `Vary: *` effectively disables shared caching** |
| `Age` | Seconds since the shared cache fetched it — how you debug CDN freshness |

**Validators and 304.** `ETag` (strong or weak) + `If-None-Match`, or `Last-Modified` + `If-Modified-Since`. A `304 Not Modified` saves the body but not the round trip — useful for large payloads, useless for small ones where `max-age` is what you actually want. In ASP.NET Core, generate ETags from a version/rowversion rather than by hashing the rendered body, so you can answer conditional requests **without** doing the work.

**The default ruleset worth stating:**
- Fingerprinted static assets: `public, max-age=31536000, immutable`.
- HTML/shell: `no-cache` (revalidate, so you can ship a new build immediately).
- Public API reads: short `max-age` plus generous `s-maxage`, plus `stale-while-revalidate`.
- Anything user-specific: `private, no-store` (or `private, max-age=0` with an ETag).

---

## Concept 38 — CDN mechanics

- **PoPs + anycast.** The same IP is announced from many locations; BGP routes the client to a nearby PoP. The win is **RTT reduction** (TLS handshake and TCP slow-start dominate small-object latency), not just origin offload.
- **Cache key composition.** Default: scheme + host + path + (some or all of) the query. You control whether query parameters, cookies, headers, or device class participate. **Every dimension you add multiplies the keyspace and divides your hit ratio.** Marketing parameters (`utm_*`) are the classic hit-ratio killer — strip or ignore them.
- **Tiered caching / origin shield.** A designated mid-tier PoP sits between edge PoPs and the origin, so a global miss costs the origin one fetch, not one per PoP. **This is the fix for "we have 200 PoPs and our origin still sees load"**, and the term to use is *midgress*.
- **Large-object handling.** Range requests and segmented caching let a CDN cache a 4 GB video in chunks and serve partial content without fetching the whole object.
- **Dynamic content.** Even uncacheable responses benefit: TLS termination at the edge, connection reuse to the origin over warm long-haul connections, and compression. Saying "the CDN helps even for personalized responses, because the expensive part is the handshake and the long-haul RTT" is a nice piece of depth.
- **Azure specifics:** **Azure Front Door** (global anycast L7 with WAF, rules engine, caching, compression, query-string caching behaviour, and origin groups) is the default for a new Azure architecture; the standalone Azure CDN products have been consolidating into it. Static content commonly lives in Blob Storage behind Front Door.

---

## Concept 39 — Edge invalidation, and how to avoid needing it

| Mechanism | Latency | Notes |
|---|---|---|
| **TTL expiry** | Deterministic | The only mechanism you can fully rely on |
| **Purge by URL** | Seconds | Exact and safe; requires knowing every URL (including query variants) |
| **Purge by tag / surrogate key** | Seconds | Tag responses at the origin (e.g. `Surrogate-Key: product-42`), purge the tag. The best tool for content with fan-in |
| **Purge by path / wildcard** | Seconds to minutes | Blunt; can cause an origin stampede across all PoPs — **a purge is a deliberate cold cache** (Concept 24) |
| **Soft purge** | Seconds | Marks stale rather than removing, so `stale-while-revalidate` keeps serving. **The safe purge** |
| **Fingerprinted URLs** | Instant | **The strategy that removes the problem**: content hash in the filename, `immutable`, one-year TTL, and a new URL for new content. No purge, no staleness, no race |

**The rule:** immutable, versioned URLs for assets; tags for content; purge as the exception. For an ASP.NET Core app, `MapStaticAssets` does the fingerprinting for you, and the CDN in front just honours the `immutable` headers.

---

# Part H — Judgment, cost, and security

## Concept 40 — Security and compliance

| Risk | Mechanism | Mitigation |
|---|---|---|
| **Cache poisoning** | An **unkeyed** input (e.g. `X-Forwarded-Host`, a weird header) influences the cached response body; one attacker request poisons the entry for everyone | Keep the cache key and the response-affecting inputs in sync; normalize/strip untrusted headers at the edge; `Vary` correctly |
| **Cross-tenant leakage** | A key missing its tenant or user segment | Tenant/user in every key; separate key prefixes per tenant; test it |
| **Authorization caching** | Cached permission decisions outliving a revocation | Short TTLs (seconds), explicit invalidation on role change, and never cache the decision for destructive operations |
| **PII at rest** | Cached profiles/orders in Redis memory and in RDB/AOF files | `no-store` where possible, short TTLs, encryption in transit (TLS) and at rest, disable persistence, and **document the cache in your data inventory** |
| **Right to erasure (GDPR)** | A deletion request must reach caches too | Deterministic key derivation so you *can* delete; short TTLs so you can wait it out; per-user generation keys so one bump orphans everything |
| **Credential exposure** | Redis access keys in config | **Entra ID authentication**, Key Vault / managed identity, Private Link, TLS-only, and Redis ACLs limiting commands (`FLUSHALL`, `CONFIG`, `KEYS`) |

**The line worth having ready:** *"A cache is a second copy of your data with a different security posture and usually weaker controls. Whatever classification the source data has, the cache inherits — including retention and deletion obligations."*

## Concept 41 — Cost

Compare like an architect: **per GB-month and per unit of load removed.**

- A managed Redis instance is priced primarily on **memory and node size**, and roughly an order of magnitude cheaper per GB than an equivalent amount of relational database compute — which is the actual economic argument for caching, and it's the one that convinces a finance stakeholder.
- **The alternatives to price against:** a **read replica** (no consistency debt, no invalidation code, scales reads, but costs full database pricing and doesn't help with expensive computation), a **materialized view / denormalized read model** (fast, consistent-by-construction within the DB, costs write amplification), **more index/query tuning** (free, often the best ROI), and **a CDN** (cheapest per request by far when the content is cacheable publicly).
- **Bandwidth is a real line item.** Cross-AZ traffic to a distributed cache is charged on most clouds; an in-process L1 for hot keys can cut both the latency and the transfer bill.
- **Engineering cost is the hidden term.** Invalidation logic, cold-start handling, and a new stateful dependency to operate cost engineer-months. A 20 ms query that nobody complains about is not worth a cache.

## Concept 42 — The decision ladder

Ask in this order, and stop as soon as the requirement is met:

1. **Can I make the origin fast enough?** Index, query shape, payload size, N+1, projection. Free, no consistency debt.
2. **Can the client or CDN cache it?** Public, non-personalized content — cheapest possible answer.
3. **Can a read replica or read model absorb it?** Scales reads with no invalidation code and bounded, well-understood staleness.
4. **Cache it in a shared L2 (Redis)** with cache-aside/read-through, TTL, delete-on-write, stampede protection, jitter.
5. **Add L1 in front of L2** for hot keys, with a short local TTL or a backplane. (`HybridCache`/FusionCache.)
6. **Precompute/materialize** (fan-out on write) if read latency still isn't met.
7. **Re-examine the requirement.** If you've reached here and it still doesn't work, the data model or the product requirement is the problem.

## Concept 43 — Observability

The metrics that actually drive decisions:

| Metric | Why |
|---|---|
| **Hit ratio, segmented by key class** | An aggregate hit ratio hides a 20% class inside a 95% average. Segmented ratios tell you where to spend memory |
| **Origin load reduction** | The actual objective. `(1−h)·λ` in requests/sec, tracked as a capacity number |
| **Miss-path latency (p50/p99)** | Your tail *is* the miss path; this is the number a cache can hide from you |
| **Cache operation latency p99** | Redis p99 spikes reveal O(N) commands, big payloads, or thread-pool starvation |
| **Eviction rate & evicted-key reasons** | Rising evictions = under-provisioned memory or over-long TTLs |
| **Fragmentation ratio, `used_memory` vs limit** | Capacity planning and OOM prevention |
| **Stale-serve counter** | If you use fail-safe/SWR, how often you're serving stale is a product-relevant number |
| **Cold-start time to steady-state hit ratio** | Directly measures your Concept 24 exposure |

**Frame it as an SLO conversation:** the cache's contribution isn't "hit ratio ≥ 95%", it's "p99 read latency ≤ 50 ms" and "origin read QPS ≤ X". Hit ratio is an input. (This is the RobinHood paper's thesis: allocate cache capacity to reduce *request tail latency*, not to maximize hit ratio — because a request that fans out to 20 services is only as fast as its slowest dependency.)

## Concept 44 — Anti-patterns

| Anti-pattern | Why it's wrong |
|---|---|
| **Cache as a band-aid for a bad query** | Hides the problem, adds a consistency problem, and the cold path still melts |
| **Unbounded cache** | No `SizeLimit`, no `maxmemory`, no TTL → OOM or unbounded staleness |
| **Cache as source of truth** | Async replication + eviction = your data is gone, by design |
| **Caching without a TTL because "we invalidate explicitly"** | Invalidation is best-effort; the TTL is the correctness backstop |
| **Double/triple caching the same value** | Staleness composes additively; debugging becomes guesswork |
| **`FLUSHALL` as an invalidation strategy** | A self-inflicted 20× origin spike. Use version bumps |
| **Caching per-user data under a shared key** | A data leak, not a bug |
| **Session affinity to make L1 "consistent"** | Fragile, defeats load balancing, breaks on deploy (see Module 6's Data Protection key-ring trap) |
| **No cold-start plan** | The system only works warm; the untested mode is the one you'll meet at 3 a.m. |
| **Hit ratio as the only metric** | You can hit 99% and still have an awful p99 |
| **Caching inside a transaction** | Commit-then-invalidate is the ordering; caching mid-transaction publishes uncommitted state |
| **Sliding expiration on mutable data** | Unbounded staleness for the most popular entries |

---

# Putting it together

## Worked example 1 — Product detail page, 60k RPS

**Requirements.** 60k RPS reads, 200 writes/sec (price/stock updates), 5M SKUs, p99 < 100 ms, price must be accurate within 60 s, stock "roughly right" but never oversell at checkout.

**Estimation.** Working set for 95% hit ratio ≈ top 2% ≈ 100k SKUs × 2 KB ≈ 200 MB → provision ~1 GB. Origin load = 60k × 0.05 = **3,000 RPS**, which one well-indexed primary plus a replica handles. Egress from cache ≈ 120 MB/s ≈ **1 Gbps** — size the tier for bandwidth, not just memory.

**Design.**
- **CDN** in front for the anonymous page shell and images: `s-maxage=60, stale-while-revalidate=30`, fingerprinted assets `immutable`.
- **L1** (`HybridCache` local, 5 s TTL) for the hottest SKUs — kills the Redis round trip for the top of the Zipf curve.
- **L2** Redis, key `catalog:v3:product:{id}:full`, TTL 5 min **with ±15% jitter**, read-through with stampede protection, tags `product:{id}` and `catalog`.
- **Writes:** commit to SQL, then `RemoveByTagAsync($"product:{id}")`. TTL is the backstop.
- **Split the volatile field out.** Price/stock in a **separate, shorter-TTL entry** (`:price`, 30 s) from the stable description (`:full`, 1 h). This is the single best move in the design: it lets you hold long TTLs on the expensive-to-assemble 95% while keeping the volatile 5% fresh, instead of setting one TTL to the strictest requirement.
- **Stock at checkout is not cached** — it's read from the source of truth in the transaction that reserves it. Cached stock is display-only, and the UI says "low stock" rather than an exact count. That's a **requirement negotiation**, and volunteering it is what an architect does.
- **Cold-cache protection:** bounded origin concurrency (e.g. 500 concurrent DB reads), single-flight, `stale-if-error`/fail-safe, readiness gate on warm-up of the top 10k SKUs.

**Trade-off to state:** worst-case price staleness is CDN 60 s + L1 5 s + L2 30 s ≈ 95 s, which exceeds the 60 s requirement — so either the CDN doesn't cache price (render it client-side from a short-TTL API call) or the budget gets renegotiated. **Catching your own violated budget by adding up the layers is exactly the signal being scored.**

## Worked example 2 — "Redis went down and the site fell over"

The incident-review question, and a favourite in architect rounds.

**Diagnosis to give:** at a 95% hit ratio the database was provisioned for 5% of read traffic, so cache loss was a 20× spike. The database saturated, latencies rose past client timeouts, clients retried, offered load grew, and the system entered a **metastable failure state**: even after Redis came back, the cache was cold, so the origin stayed saturated and never got the headroom to refill it. (That last sentence — that the system can't self-recover because recovery requires the resource that's exhausted — is the heart of it, and the "Metastable Failures in Distributed Systems" paper is the reference.)

**Remediation, in order of value:**
1. **Bounded origin concurrency** (bulkhead) — the database gets a hard cap and stays responsive; excess requests shed fast.
2. **Single-flight** — a cold key costs one query, not thousands.
3. **Fail-safe/stale-serving** at both the app and HTTP layers.
4. **L1 in every instance** — hot keys survive an L2 outage entirely.
5. **Retry budgets and jittered backoff** (Module 13) so retries can't amplify.
6. **Right-size the database for the cold case**, or explicitly accept degraded-mode behaviour (read-only mode, cached-only mode with a banner).
7. **Rehearse it** — a game day where you flush the cache under load. Until you've done that, the cold mode is untested.

## Worked example 3 — Multi-tenant SaaS dashboard

Aggregations over tenant data, 30 s freshness acceptable, 4,000 tenants of wildly different sizes, deployments weekly.

- Key: `dash:v7:t{tenant}:widget:{id}:gen:{gen}` — schema version (deploy safety) **and** per-tenant generation (Concept 16).
- Invalidation: CDC from the transactional DB → an invalidation worker that `INCR`s `gen:t{tenant}` on relevant table changes. One increment orphans every widget for that tenant, regardless of how many derived keys exist.
- Big tenants get **precomputed** widgets (fan-out on write, refreshed on a schedule); small tenants are computed **on demand** with SWR. Same code path, different policy, chosen by tenant size — the hybrid from Concept 13.
- Per-tenant memory caps so one tenant can't evict everyone else's entries (the cache equivalent of the noisy-neighbour problem from Module 8).
- Deploy: bump `v7` → `v8` in config. Old entries age out; no flush, no stampede.

---

## Common questions and what a strong answer contains

**"What caching strategies do you know?"** Name the patterns as a taxonomy — read path (cache-aside, read-through, refresh-ahead) vs write path (write-through, write-behind, write-around/invalidate) — then say which you default to and why (cache-aside + delete-on-write + TTL) and when you'd switch.

**"How do you keep the cache consistent with the database?"** You can't, without coordination; you pick a staleness bound. Commit-then-delete plus TTL as the backstop; then draw the stale-set race; then the ladder (versioned keys, lease-on-fill, CDC) and when each is warranted.

**"Update the cache or delete it on write?"** Delete. Three reasons: idempotent/commutative so concurrent writers can't invert the order, no wasted work computing values nobody reads, and the projection logic stays on the read path.

**"What's a cache stampede and how do you prevent it?"** Distinguish the three phenomena. Then: single-flight, XFetch probabilistic early refresh, stale-while-revalidate, TTL jitter, negative caching, bounded origin concurrency. Then note that `IMemoryCache.GetOrCreate` doesn't dedupe but `HybridCache.GetOrCreateAsync` does.

**"How do you choose a TTL?"** From the business freshness requirement, not from feel. Then: split fields by volatility so you're not setting one TTL to the strictest requirement; add jitter; and remember the TTL is the correctness backstop when invalidation fails.

**"What happens if the cache goes down?"** The `1/(1−h)` load multiplier, the retry-amplification loop, metastability, and then the playbook: bulkheads, single-flight, fail-safe, L1, load shedding, and capacity-testing cold.

**"Redis or in-memory?"** Both — L1 for latency and L2-outage survival, L2 for cross-instance consistency and capacity. Then the L1 problem: N stale copies, needing short TTLs or a backplane.

**"How would you cache a personalized feed?"** Fan-out on write vs read, the celebrity problem, the hybrid, and `private`/no shared caching at the HTTP layer. Cache the *components* (per-post objects, shared across users) plus a per-user ID list — not the rendered feed.

**"How do you invalidate everything derived from one entity?"** Generation counters in keys, or tags (`RemoveByTagAsync`, output-cache tags, CDN surrogate keys). Never `FLUSHALL`.

**"Is Redis single-threaded? Does that matter?"** Yes for command execution, which is why commands are atomic and why one O(N) command stalls all clients. Throughput comes from pipelining. Then `KEYS`→`SCAN`, `DEL`→`UNLINK`.

**"Can you use Redis as your database?"** Async replication loses acknowledged writes on failover; eviction can delete your data by design. Fine for cache, session (with an accepted risk), and rate limiting; not for a system of record.

**"How do you measure whether the cache is working?"** Not hit ratio alone: origin load reduction as a capacity number, p99 including the miss path, eviction rate, and staleness/stale-serve counters. Frame it as an SLO.

**"Where would you *not* add a cache?"** Fast indexed reads, write-heavy data, uniform access patterns, strict invariants, and anything where a missing index is the real problem.

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "We'll put Redis in front of the database" and moving on | States the staleness bound, the invalidation strategy, and the cold-cache behaviour in the same breath |
| Treating hit ratio as the goal | Frames it as origin-load reduction and p99; notes the tail is the miss path |
| Cannot describe a stale-set race | Draws the four-step timeline and names the fixes with their guarantees |
| Updating the cache on write | Deletes, and gives the ordering/idempotence/wasted-work reasons |
| Explicit invalidation with no TTL | TTL as the correctness backstop, because invalidation is best-effort |
| One global TTL | Splits by volatility; jitters; derives TTL from a stated freshness requirement |
| Ignoring read-your-writes | Names the bug and picks a fix (write-through on the writer's path, generation bump, skip-cache window) |
| "We'd fall back to the database" | Quotes the `1/(1−h)` multiplier and adds bulkheads, single-flight, and fail-safe |
| `IMemoryCache` with no size limit | Sets `SizeLimit`+`Size`, and mentions gen-2/LOH and container memory-limit consequences |
| Unaware `GetOrCreate` doesn't dedupe | Knows it, and reaches for `HybridCache`/FusionCache with an explanation of what they add |
| Sliding expiration on mutable data | Absolute expiration, or sliding with an absolute ceiling |
| `FLUSHALL` or wildcard purge to invalidate | Version/generation keys and tags; notes that a purge is a deliberate cold cache |
| Caching per-user data under a shared key | Tenant/user/permissions in the key; treats it as a security control |
| No key convention | `app:version:tenant:entity:id:shape`, bounded cardinality, no PII, hashing long composites |
| Thinking Redis is linearizable | Async replication, lost writes on failover, `WAIT` ≠ consensus; fencing for locks |
| Blaming Redis for timeouts | Checks thread-pool starvation and O(N) commands first; one singleton multiplexer |
| Naming Azure Cache for Redis tiers in 2026 | Knows Azure Managed Redis is the target and the retirement timeline; mentions Valkey and Garnet |
| Never questioning the freshness requirement | Negotiates it: display-only stock vs transactional stock; 60 s vs 5 min changes the design |
| No numbers | `(1−h)·λ`; 95% ⇒ 20× cache-loss spike; Redis 0.3–1 ms p50; L1 ~100 ns; 12 KB HLL; 85,000-byte LOH threshold |

---

## Practice exercises

**Exercise 1 — Reproduce the stale-set race.** Two console apps against SQL Server + Redis, cache-aside with commit-then-delete. Insert an artificial delay between the reader's DB read and its cache `SET`, and run a writer in a loop. Assert that the cache ends up holding the pre-write value and stays wrong until the TTL. Then fix it three ways — short TTL, delayed double delete, and a Lua `SET`-if-version-newer — and measure how often each fails under load. **This is the highest-value exercise in the module**, exactly as Module 9's split-brain exercise was: it converts Concept 14 from a fact into something you've seen.

**Exercise 2 — Measure the stampede.** Endpoint backed by a deliberately slow query (500 ms). Fire 500 concurrent requests against a cold key with (a) `IMemoryCache.GetOrCreate`, (b) hand-rolled `Lazy<Task<T>>` single-flight, (c) `HybridCache.GetOrCreateAsync`, (d) FusionCache with soft timeout + fail-safe. Count actual database executions and record p50/p99 for each. You'll get a four-row table you can quote from experience.

**Exercise 3 — Implement XFetch and compare.** Implement Concept 12's probabilistic early expiration, then compare against fixed TTL and against SWR on a Zipf-distributed synthetic workload (α=0.9, 100k keys). Measure origin QPS, p99, and staleness distribution. Then plot hit ratio against cache size and confirm the logarithmic shape from Concept 1 yourself.

**Exercise 4 — Two-tier with a backplane.** Three instances of an API behind a load balancer, `HybridCache` with Redis L2. Update a value and observe how long each instance serves stale data from L1. Then set `LocalCacheExpiration` short, then add a Redis pub/sub backplane (or swap in FusionCache's) and measure the convergence time. Write down what breaks when the backplane message is lost — that's the at-most-once property from Concept 17, experienced directly.

**Exercise 5 — Break Redis on purpose.** Load 5M keys. Then: run `KEYS *` and watch p99 across all clients; run `HGETALL` on a 1M-field hash; set `maxmemory-policy noeviction` and fill memory until writes fail; set `volatile-lru` with half the keys lacking a TTL and do it again. Then fix each. Finally, block the thread pool (`.Result` in a hot path) and observe Redis "timeouts" that Redis had nothing to do with.

**Exercise 6 — The cold-start capacity test.** Load-test your app with the cache disabled. Record the RPS at which p99 breaches your SLO. That ratio versus your warm RPS is your true cache dependency, and most teams have never measured it. Then add a bulkhead capping origin concurrency, re-run, and show that the system degrades instead of collapsing.

**Exercise 7 — The architect write-up (one page).** For a system you know: inventory every cache (browser, CDN, gateway, L1, L2, DB) and for each state the key, TTL, invalidation mechanism, worst-case staleness, and cold-start behaviour. Sum the staleness down each path and check it against the product requirement. Then mark which caches are optimizations and which are load-bearing — i.e. which ones, if lost, take the system down. This is close to a real architect-round take-home and it will find something broken.

---

## Free resources

### Papers and primary sources

| Resource | What it covers | Why read it |
|---|---|---|
| [Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf) — Nishtala et al., NSDI 2013 | **Leases** (the stale-set fix), the gutter pool, mcrouter, `mcsqueal` invalidation from MySQL binlogs, incast congestion, regional invalidation | **The single most important read in this module.** Almost every technique in Part C originates or is validated here |
| [An Analysis of Facebook Photo Caching](https://www.cs.cornell.edu/~qhuang/papers/sosp_fbanalysis.pdf) — Huang et al., SOSP 2013 | Measured traffic through browser cache → edge → origin cache → storage | The best empirical argument for a *layered* cache hierarchy, with real hit-ratio-per-layer numbers |
| [TAO: Facebook's Distributed Data Store for the Social Graph](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf) — ATC 2013 | A graph-aware cache layer as a first-class system, with its own consistency model | What it looks like when the cache becomes the architecture |
| [Optimal Probabilistic Cache Stampede Prevention](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf) — Vattani et al., VLDB 2015 | The XFetch algorithm and its optimality proof | Short, immediately implementable, and naming it is a differentiator |
| [TinyLFU: A Highly Efficient Cache Admission Policy](https://arxiv.org/abs/1512.00727) — Einziger, Friedman & Manes | Count-Min Sketch admission filtering, aging, W-TinyLFU | The modern eviction default, and the "admission not eviction" reframing |
| [ARC: A Self-Tuning, Low Overhead Replacement Cache](https://www.usenix.org/legacy/events/fast03/tech/full_papers/megiddo/megiddo.pdf) — Megiddo & Modha, FAST 2003 | Adaptive recency/frequency balance | The policy everything since is measured against |
| [RobinHood: Tail Latency Aware Caching](https://www.usenix.org/system/files/osdi18-berger.pdf) — Berger et al., OSDI 2018 | Allocating cache space to cut **p99 request** latency rather than maximize hit ratio | **The paper behind "hit ratio is not the goal."** Excellent interview ammunition |
| [Cliffhanger: Scaling Performance Cliffs in Web Memory Caches](https://www.usenix.org/system/files/conference/nsdi16/nsdi16-paper-cidon.pdf) — NSDI 2016 | Incremental hit-rate curves for allocating memory between key classes | Why you should measure hit ratio *per key class* |
| [Metastable Failures in Distributed Systems](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf) — Bronson et al., HotOS 2021 | Systems that stay broken after the trigger is removed; **cold caches are the canonical example** | The formal vocabulary for Worked example 2 |
| *Web Caching and Zipf-like Distributions* — Breslau, Cao, Fan, Phillips & Shenker, INFOCOM 1999 | The measurement behind Concept 1 (α ≈ 0.6–1.0; log-shaped hit-ratio curve) | The empirical foundation; widely mirrored as a PDF |

### Explainers, engineering blogs, and operational wisdom

| Resource | What it covers |
|---|---|
| [Amazon Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/) | **Required reading.** Modality, cold caches, thundering herds, and why caches make systems bimodal — written by people who operate them |
| [Google SRE Book: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) | The retry-amplification and death-spiral mechanics of Concept 24, and how to break them |
| [Caffeine wiki: Efficiency](https://github.com/ben-manes/caffeine/wiki/Efficiency) | Hit-ratio comparisons of LRU/LFU/ARC/W-TinyLFU against **Belady's optimum** on real traces. The best free material on eviction policy |
| [Redis: key eviction](https://redis.io/docs/latest/develop/reference/eviction/) | `maxmemory-policy` options, the sampled-LRU approximation, and the LFU counter design |
| [Redis: client-side caching](https://redis.io/docs/latest/develop/reference/client-side-caching/) | `CLIENT TRACKING`, RESP3 invalidation push, broadcast vs default mode — the protocol-level two-tier cache |
| [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) | Hash slots, hash tags, `MOVED`/`ASK`, resharding |
| [Redis: latency troubleshooting](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/) | Slow commands, fork latency, swap, and the diagnostic order |
| [Redis: distributed locks](https://redis.io/docs/latest/develop/use-cases/patterns/distributed-locks/) + [Kleppmann's critique](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) | The Module 9 connection: why a lock over a cache needs fencing |
| [Jepsen: Redis](https://aphyr.com/posts/283-jepsen-redis) | Empirical demonstration of lost writes under failover — the evidence behind Concept 27 |
| [Netflix: Caching for a Global Netflix](https://netflixtechblog.com/caching-for-a-global-netflix-7bcc457012f1) (and the EVCache posts) | Multi-region cache replication, warming, and cache-as-infrastructure at scale |
| [Twitter: Timelines at Scale](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/) | The canonical fan-out-on-write vs fan-out-on-read talk (Concept 13) |
| [PortSwigger: Practical Web Cache Poisoning](https://portswigger.net/research/practical-web-cache-poisoning) — James Kettle | Unkeyed-input poisoning, with real exploits. Read before you configure a CDN cache key |
| [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) and [web.dev: HTTP cache](https://web.dev/articles/http-cache) | The header semantics of Concept 37, correctly and readably |
| [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html) · [RFC 5861 — stale-while-revalidate / stale-if-error](https://www.rfc-editor.org/rfc/rfc5861.html) | The normative definitions, for when someone argues about `no-cache` |
| [Cloudflare developer docs: Cache](https://developers.cloudflare.com/cache/) | Cache keys, tiered caching, purge mechanics — vendor-neutral enough to learn CDN concepts from |
| [Fastly documentation](https://www.fastly.com/documentation/) | Surrogate keys and soft purge, the best-documented tag-based edge invalidation |

### .NET and Azure documentation

| Resource | What it covers |
|---|---|
| [Caching in .NET — overview](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/overview) | The decision tree across `IMemoryCache`, `IDistributedCache`, `HybridCache`, output and response caching |
| [**HybridCache in ASP.NET Core**](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) | **Read this one properly.** L1/L2, stampede protection, tags, serialization, `HybridCacheEntryFlags`, migration from `IDistributedCache` |
| [In-memory caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory) | `SizeLimit`, expiration, eviction callbacks, `CancellationChangeToken` invalidation |
| [Distributed caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed) | The `IDistributedCache` contract and its Redis/SQL Server implementations |
| [Output caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output) · [Response caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response) | Policies, `VaryBy`, tag eviction, and the header-driven alternative |
| [Azure Architecture Center: Caching guidance](https://learn.microsoft.com/en-us/azure/architecture/best-practices/caching) | The platform-neutral design guidance; good vocabulary for design docs |
| [Cache-Aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) | The pattern as Microsoft documents it, including the consistency caveats |
| [Azure Managed Redis documentation](https://learn.microsoft.com/en-us/azure/redis/) | Tiers, clustering, active geo-replication, Entra ID auth, Private Link |
| [Azure Cache for Redis retirement FAQ](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/retirement-faq) | The dates in Concept 30 and the migration path — worth a skim before an Azure interview |
| [Azure Cache for Redis: development best practices](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices-development) | Connection reuse, timeouts, `maxmemory-policy`, and the "every key gets a TTL" guidance |
| [StackExchange.Redis docs](https://stackexchange.github.io/StackExchange.Redis/) and [Timeouts](https://stackexchange.github.io/StackExchange.Redis/Timeouts) | Multiplexer design, pipelining, and the thread-pool-starvation diagnosis |
| [FusionCache](https://github.com/ZiggyCreatures/FusionCache) | Fail-safe, soft/hard timeouts, eager refresh, backplane, auto-recovery, adaptive caching — with unusually good docs |
| [BitFaster.Caching](https://github.com/bitfaster/BitFaster.Caching) | `ConcurrentLru`/`ConcurrentLfu` (W-TinyLFU) for a bounded, allocation-conscious L1 |
| [Microsoft Garnet](https://microsoft.github.io/garnet/) | RESP-compatible, multi-core cache server from MSR; drop-in for StackExchange.Redis clients |
| [EF Core: querying & no-tracking](https://learn.microsoft.com/en-us/ef/core/querying/tracking) · [compiled queries](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics) | The "make the origin faster first" rung of Concept 42 |
| [`MapStaticAssets`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/static-files) | Fingerprinted, precompressed static assets with immutable caching — Concept 39 for free |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What a cache *is* | An async replica with a staleness bound (the TTL) and no coordination |
| Why caching works | Zipf-like skew; hit ratio grows ~logarithmically with size; fan-out amplification |
| The arithmetic | `origin load = (1−h)·λ`; 90→95% halves it; losing the cache is a `1/(1−h)` spike |
| Tail latency | The p99 is the miss path; bound it with timeouts, SWR, and fail-safe |
| Patterns | Read: cache-aside / read-through / refresh-ahead. Write: write-through / write-behind / write-around |
| Default choice | Cache-aside (read-through) + TTL with jitter + commit-then-delete + stampede protection |
| Update or delete on write | Delete: idempotent and commutative, no wasted work, projection logic stays on the read path |
| Consistency | Bounded staleness at best; read-your-writes and monotonic reads break by default; co-locate values that must agree |
| The stale-set race | Reader's late `SET` of a pre-write value; fixed by versioned keys, versioned sets, or lease-on-miss |
| Invalidation ladder | Don't cache → TTL → TTL+delete → versioned keys → CDC events → lease/fence |
| Mass invalidation | Generation counter or schema version *in the key*; tags; never `FLUSHALL` |
| TTL choice | Derive from the freshness requirement; split fields by volatility; always jitter |
| Stampede | Single-flight, XFetch, SWR, jitter, negative caching, bounded origin concurrency |
| Hot keys | Count-Min/Top-K to detect; L1 tier, key splitting, or client-side tracking to fix |
| Eviction | LRU vs LFU vs **W-TinyLFU** (admission filtering, scan-resistant); Belady as the bound; Redis's sampling is approximate |
| Cache loss | `1/(1−h)` spike → saturation → retries → metastable failure; bulkhead, coalesce, fail-safe, L1, shed, test cold |
| Redis internals | Single-threaded commands ⇒ atomic, and one O(N) command stalls everyone; pipeline; `SCAN`/`UNLINK` |
| Redis consistency | Async replication loses acked writes on failover; `WAIT` ≠ consensus; never a system of record |
| Redis Cluster | 16384 slots, CRC16, `{hash tags}` for co-location, CROSSSLOT limits |
| Redis memory | `allkeys-lru`/`lfu` for a cache, `volatile-*` OOM trap, every key gets a TTL, watch fragmentation |
| Azure | **Azure Managed Redis** (Redis Enterprise, CRDT active geo-replication); Azure Cache for Redis retiring 2027/2028; Valkey and Garnet as alternatives |
| .NET in-memory | `IMemoryCache` — set `SizeLimit` *and* `Size`; absolute over sliding; gen-2/LOH/container-limit consequences |
| .NET distributed | `IDistributedCache` is `byte[]`-only with no L1, tags, or stampede protection; serializer choice dominates |
| .NET default | **`HybridCache`**: L1+L2, stampede protection, tags, `static` factory with state |
| .NET resilience | **FusionCache**: fail-safe, soft/hard timeouts, eager refresh, backplane, auto-recovery |
| HTTP layer | Output caching (server policy + tags) vs response caching (client headers); `s-maxage`, `stale-while-revalidate`, `stale-if-error`, `immutable` |
| CDN | Cache-key composition is your hit ratio; tiered caching/origin shield for midgress; fingerprinted URLs beat purging |
| Security | Poisoning via unkeyed input; tenant/permission in the key; PII retention and erasure; Entra ID over access keys |
| Cost | Cache memory is ~an order of magnitude cheaper per GB than DB compute; compare against a read replica and against tuning the query |
| Metrics | Origin-load reduction and p99, not hit ratio; segment hit ratio by key class; measure time-to-warm |
| When not to cache | Already-fast reads, write-heavy data, uniform access, strict invariants, or a missing index |

---

## Progress

Module 10 complete. Phase 3 now has: **guarantees** (7), **mechanisms** (8), **agreement** (9), and **the deliberate weakening of all three in exchange for latency** (10). Caching is where the theory from 7–9 becomes an everyday engineering decision — and where the ability to name what you gave up is the whole signal.

Next in the curriculum: **Module 11 — Messaging & event-driven systems** (queues vs streams, delivery guarantees, dead-letter queues, the outbox pattern). It picks up two threads left dangling here: the **transactional outbox** appeared in Concept 17 as an invalidation transport and deserves proper treatment, and the **at-most-once semantics of pub/sub** that made the backplane unreliable is exactly the delivery-guarantee taxonomy Module 11 formalizes. It also closes the loop on Module 9's idempotency material, since every message consumer you build in .NET will need it.
