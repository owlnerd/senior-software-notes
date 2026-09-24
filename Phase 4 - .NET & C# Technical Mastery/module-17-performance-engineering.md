# Module 17 — Performance Engineering
*Phase 4: .NET & C# Technical Mastery · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **performance engineering is not "making code fast" — it is building a measured model of where time and memory go, changing the smallest thing that moves the metric that actually matters, and proving that it moved; and in modern .NET that model has three layers you must hold at once: the hardware, the runtime's compilers, and the shape of your APIs.**

That reframing matters because the naive picture of performance work is a bag of tricks: use `Span<T>`, avoid LINQ, pool everything, go Native AOT. A mid-level candidate can list the tricks. A senior candidate asks four questions before touching any of them — *which metric, at which percentile, under which load, and where does the time actually go?* — and can then tell you why their "10× faster" micro-benchmark moved production p99 by nothing (the function was 3% of the profile), why a load test said the service was fine and production said it wasn't (the load generator was closed-loop and hid every stall), why the same code got 40% faster on .NET 11 without a recompile (the JIT removed bounds checks and devirtualized generic virtual calls it couldn't before), why a `HashSet` "beat" an array in a teammate's benchmark for the wrong reason (both sides compared the same interned string literal), and why a 20% efficiency gain saved zero instances (zone redundancy quantizes capacity).

Module 14 said this module would be "Parts E and F applied with a measurement discipline." That is the core, and this module deliberately does not repeat Module 14's allocation cost model, struct-vs-class procedure, pooling rules, hidden-allocation catalogue, GC configuration, or memory diagnostics — it refers back to them. What it adds is everything around them: the method, the statistics, the hardware model, the tools, the JIT's optimizer as something you can observe and cooperate with, the span/memory/buffer ecosystem as an *API design* discipline, Native AOT as a practice rather than a checkbox, and performance as an architectural activity — load testing, capacity planning, startup engineering, regression prevention, and budgets.

It shows up in four places in an interview loop: the **deep technical round** ("walk me through how you'd find out why this is slow"), the **code-review round** ("what's wrong with this benchmark?", "what allocates here?"), the **coding round** (your parser allocates eight objects per line — does it need to?), and the **design round** (capacity numbers, latency budgets, cold-start strategy, JIT vs R2R vs AOT per service).

**Current platform state (verified September 2026).** .NET 10 (November 2025) is the production-current LTS, supported to November 2028. .NET 11 RC1 shipped on 8 September 2026 with a go-live licence — the first of two release candidates — ahead of GA on 10 November 2026; .NET 11 is STS. Stephen Toub's annual *Performance Improvements in .NET 11* was published on 15 September 2026 and covers, among hundreds of changes, a new round of bounds-check elimination, broader deabstraction (devirtualization of generic virtual methods, extended conditional escape analysis, nullable boxing exposed to escape analysis), and runtime async. .NET 11 raises the x86/x64 hardware baseline from x86-64-v1 to x86-64-v2 on all operating systems and moves the ReadyToRun target to x86-64-v3 (AVX2, BMI, FMA, …) on Windows and Linux. Runtime async is still opt-in (`<Features>runtime-async=on</Features>`) but no longer needs `EnablePreviewFeatures` for `net11.0`, and the BCL itself is compiled with it. The .NET 11 SDK's `dotnet` CLI host is Native AOT–compiled by default — the platform team dogfooding exactly the startup argument this module makes. BenchmarkDotNet's latest stable release is 0.15.8 (November 2025); the 0.15.x line added Roslyn analyzers for benchmark mistakes and moved statistics to Perfolizer 0.6's Pragmastat engine, and the .NET team's .NET 11 post uses a 0.16.0 preview. EF Core's Native AOT and precompiled-query support remains explicitly experimental. On Linux, `dotnet-trace collect-linux` (.NET 10+, preview, root, kernel 6.4+) produces unified traces with managed events, native call stacks, and kernel events via `user_events`. On Azure, the Application Insights Profiler for .NET with Code Optimizations remains the production profiler for SDK 2.x; OpenTelemetry-based (SDK 3.x) applications use the Azure Monitor OpenTelemetry Profiler for .NET.

This module has nine jobs:

1. **Replace instinct with method.** A loop you can narrate — define, measure, locate, predict, change one thing, re-measure, keep or revert — and a leverage hierarchy that tells you which level of the system to look at first.
2. **Make your latency statistics correct.** Distributions, percentiles that cannot be averaged, mergeable histograms, tail amplification under fan-out, coordinated omission, and the queueing arithmetic that turns "target CPU utilization" into a latency decision.
3. **Give you a hardware model** good enough to predict when a "worse" algorithm wins: cache lines, prefetching, pointer chasing, branch prediction, dependency chains, and the cost of sharing between cores.
4. **Make you rigorous with BenchmarkDotNet** — what it does, how to write a benchmark that measures what you think it measures, how to read the table, how to compare two numbers honestly, and what a micro-benchmark can never prove.
5. **Make you fluent with profilers**, in the lab and in production: sampling vs tracing, CPU vs wall-clock, allocation profiling, the .NET diagnostics stack, and continuous profiling.
6. **Explain what the JIT does to your code and how to see it** — tiering, dynamic PGO, inlining, devirtualization, bounds-check elimination, struct promotion, ReadyToRun — so you can cooperate with the optimizer instead of fighting a 2016 version of it.
7. **Treat `Span<T>`, `Memory<T>`, and the buffer ecosystem as API design**: ownership rules, discontiguous data, buffer writers, pipelines, the Try pattern, two-tier APIs, and the escape hatches with their contracts.
8. **Make Native AOT a practice**: what the compiler actually does, trimming and its annotation system, getting to zero warnings, the ecosystem map as of .NET 10/11, size and speed knobs, and diagnosing AOT binaries.
9. **Raise it to architecture**: load testing that tells the truth, capacity plans from measured knees, cold-start engineering, regression gates, performance budgets derived from SLOs, runtime upgrades as a strategy — and when not to optimize at all.

Eight framings to carry through:

1. **Measure the thing that matters, where it matters.** The metric is set by a requirement (an SLO, a cost target, a startup budget), not by what is easy to benchmark. A faster function is not a faster service.
2. **Distributions, not numbers.** Every latency is a distribution with a shape; every comparison is between two distributions with noise. Means and single runs are how teams fool themselves.
3. **Leverage is hierarchical.** Removing work beats doing it less often beats batching it beats doing it faster. Architecture and I/O dominate algorithms, which dominate allocation, which dominates instructions.
4. **The machine charges for moving memory, not for arithmetic.** Cache misses, not additions, are the unit of cost for most data-heavy code. Layout is a performance decision.
5. **The JIT is a collaborator with a model.** It rewards idiomatic, small, sealed, bounds-provable code and punishes cleverness that hides intent. Verify with disassembly; don't guess from IL or folklore.
6. **Ownership is the price of zero-copy.** Every buffer you stop allocating is a buffer whose lifetime you now manage. Spans make that safe on the stack; `Memory<T>` and pools make it a contract you must document.
7. **Startup, steady state, and footprint are different products.** JIT, ReadyToRun, and Native AOT each win one of them. Pick per service, deliberately, with numbers.
8. **Performance work is only real when it is repeatable.** A committed benchmark, an allocation assertion, a CI gate, a dashboard, a budget. Otherwise it regresses silently within two releases.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Performance requirements | Metric + percentile + load + environment; name the binding dimension |
| 2 | The optimization loop | Predict the effect size before you change anything; one change per measurement |
| 3 | Where the time goes | Critical path + Amdahl; leverage falls roughly tenfold per level from architecture to instructions |
| 4 | Latency is a distribution | Means hide modes and tails; report p50/p99/p99.9/max with load and window |
| 5 | Aggregating percentiles | Percentiles don't average; merge histograms; fan-out amplifies tails |
| 6 | Coordinated omission | Closed-loop load generators hide stalls; use open-model, constant-arrival-rate load |
| 7 | Utilization and queueing | R = S/(1−ρ); target utilization is a latency decision; variability multiplies waiting |
| 8 | USE and RED | Per resource: utilization, saturation, errors; per service: rate, errors, duration |
| 9 | The memory hierarchy | A DRAM miss ≈ 100 ns ≈ hundreds of instructions; the cache line is the unit of cost |
| 10 | Data layout | Contiguous, dense, hot-fields-together beats clever algorithms on pointer-chasing layouts |
| 11 | Branches and dependency chains | Mispredicts cost ~15–20 cycles; serial dependencies waste the core's width |
| 12 | Cores and sharing | Shared writes serialize; shard, batch, or snapshot; SMT vCPUs aren't cores |
| 13 | Why a Stopwatch loop lies | Tier-0 code, dead-code elimination, constant folding, GC, frequency scaling, order effects, one sample |
| 14 | What BenchmarkDotNet does | Separate process, pilot, overhead subtraction, warm-up, statistics, forced GC between iterations |
| 15 | Correct benchmarks | Return results, inputs from instance fields, realistic sizes and distributions, a baseline |
| 16 | Diagnosers | Memory, disassembly, threading, exceptions, EventPipe, hardware counters — measure more than time |
| 17 | Reading results | Error is half the 99.9% CI; Allocated is deterministic; ZeroMeasurement means you measured nothing |
| 18 | Comparing honestly | Same session, a practical threshold set in advance, robust statistics, reproduce before believing |
| 19 | Jobs and runtimes | `--runtimes`, environment variables, MSBuild arguments, run strategies; .NET 10 vs 11 in one run |
| 20 | Micro vs macro | A micro-benchmark proves relative cost in isolation — nothing about end-to-end latency |
| 21 | Profiler taxonomy | Sampling for *where*, tracing for *why and when*, instrumentation for exact counts |
| 22 | The diagnostics stack | EventPipe/EventSource everywhere; ETW or perf/`user_events` for native; pick the tool by the question |
| 23 | Reading CPU profiles | Exclusive time finds hot code, inclusive finds hot paths; learn the runtime's signatures |
| 24 | Wall-clock analysis | Latency is often waiting; a CPU profile cannot see blocked time |
| 25 | Allocation profiling | Sampled allocation ticks by type and stack; pair bytes with survival |
| 26 | Production profiling | Continuous, sampled, triggered — answer "where did CPU go at 14:05?" without a repro |
| 27 | Tiering in practice | Knobs, startup cost; `AggressiveOptimization` disables PGO; hot R2R code gets rejitted |
| 28 | Dynamic PGO | Guarded devirtualization and hot/cold layout; benchmark type mixes must match production |
| 29 | Inlining | The optimization that enables the others; throw helpers keep hot methods small |
| 30 | Devirtualization | Sealed and exact types, and (.NET 11) generic virtual methods; seal by default |
| 31 | Bounds checks | Write idiomatic loops and check the disassembly; unsafe code to dodge checks is usually obsolete |
| 32 | Structs and registers | Small, readonly, non-address-exposed structs enregister; `SkipLocalsInit` with care |
| 33 | Reading JIT output | `JitDisasm`, `DisassemblyDiagnoser`; look for helper calls, indirect calls, spills |
| 34 | ReadyToRun in depth | A precompiled starting tier; composite images; .NET 11 targets x86-64-v3 on Windows/Linux |
| 35 | The span family | One view type over any contiguous memory; slicing is free; ref-struct rules keep it safe |
| 36 | The span API surface | `MemoryExtensions`, `BinaryPrimitives`, span parsing and formatting — vectorized, allocation-free |
| 37 | `Memory<T>` ownership | A `Memory<T>` parameter is a borrow; `IMemoryOwner<T>` is ownership; dispose exactly once |
| 38 | Discontiguous data | `ReadOnlySequence<T>` + `SequenceReader<T>`; fast-path the single segment |
| 39 | Writing output | `IBufferWriter<T>`, `TryFormat`, `OperationStatus` — the caller supplies the memory |
| 40 | Pipelines | Pooled segments + backpressure + `AdvanceTo(consumed, examined)`; Kestrel's substrate |
| 41 | Span-based API design | Span in, span out, count back; two-tier APIs; `Memory<T>` only for async or storage |
| 42 | Escape hatches | `stackalloc` with a bound, `[InlineArray]`, `Unsafe`/`MemoryMarshal`/`CollectionsMarshal` — contracts, not tricks |
| 43 | Text: UTF-8 first | Stay in bytes; `u8` literals, `IUtf8SpanFormattable`, `Ascii`, ordinal comparisons |
| 44 | Searching and matching | `SearchValues<T>` for chars, bytes, and strings; `[GeneratedRegex]`; `NonBacktracking` for untrusted input |
| 45 | Collections | Choose by access pattern and constant factors; frozen for read-mostly; presize; locality matters |
| 46 | Vectorization | `Vector128/256/512` and `TensorPrimitives`; check the BCL first; vector body + scalar tail |
| 47 | Serialization | STJ source generation, cached options, UTF-8 readers and writers; stream, don't buffer |
| 48 | Exceptions | Microseconds, not nanoseconds; Try patterns; throw helpers; never control flow |
| 49 | Telemetry overhead | `[LoggerMessage]`, `IsEnabled`, sampling, low-cardinality tags, `TagList` |
| 50 | What Native AOT does | ILC whole-program compilation + trimming + an embedded runtime: a closed world |
| 51 | Trimming and annotations | Warnings are future runtime failures; annotate requirements instead of suppressing them |
| 52 | AOT-ready projects | `PublishAot`, `IsAotCompatible`, `CreateSlimBuilder`, source generators, test the native binary |
| 53 | The ecosystem map | Minimal APIs, gRPC, STJ source generation: yes; MVC, Blazor Server: no; EF Core: experimental |
| 54 | Size and speed knobs | `OptimizationPreference`, `IlcInstructionSet`, `InvariantGlobalization`, chiseled `runtime-deps` images |
| 55 | The AOT profile | Milliseconds to start, small RSS, flat post-deploy p99; steady state without dynamic PGO |
| 56 | Diagnosing AOT apps | EventPipe tools work; dumps need native debuggers; keep stack-trace data |
| 57 | Allocation budgets | Bytes per operation as a tested contract — deterministic, so gate CI on it |
| 58 | Steady-state-zero design | Allocate at startup or per connection; reuse on the request path |
| 59 | API shape decides allocation | Caller-provided buffers, visitors with state, struct enumerators, Try patterns |
| 60 | The complexity budget | Performance islands with clean boundaries, review rules, fuzzing |
| 61 | Load testing | Open model, realistic data, warm-up excluded, generator not the bottleneck, environment parity |
| 62 | Capacity plans | Throughput at SLO (the knee), not maximum throughput; redundancy quantizes the benefit |
| 63 | Startup and cold start | Image pull, runtime init, DI, JIT, connections; R2R/AOT, lazy init, readiness warm-up |
| 64 | Preventing regressions | Gate on allocations, track times on stable hardware, canary in production |
| 65 | Budgets and culture | Derive budgets from SLOs; make cost per request visible; give it an owner |
| 66 | Runtime upgrades | Each release is free performance — measured, with the LTS/STS trade-off priced in |
| 67 | When not to optimize | Remove work > do it less > batch it > do it faster; "fast enough" is an SLO |

---

# Part A — The discipline: method before tools

Everything in Parts B–J is technique. This part is the method that decides which technique, if any, to reach for. It is also where interviewers separate candidates fastest: "how would you approach a performance problem?" is asked in nearly every senior loop, and the weak answer names tools while the strong answer names a process.

## Concept 1 — Performance is a set of requirements, not an adjective

"Fast" is not a requirement. A performance requirement has four parts: **a metric, a statistic, a load condition, and an environment.**

> *"p99 server-side latency for `POST /orders` under 150 ms at 3,000 requests per second sustained, on 4-vCPU instances, excluding client network time, measured over any 5-minute window."*

The metric families you should be able to name, because they trade against each other:

| Dimension | Typical metric | Trades against |
|---|---|---|
| **Latency** | p50/p99/p99.9 per operation, server- and client-side | Throughput (batching), cost (headroom) |
| **Throughput** | Requests or messages per second *at a latency target* | Latency, memory |
| **Startup / time-to-ready** | Process start → first successful request; cold-start duration | Peak throughput (AOT vs JIT), image size |
| **Footprint** | Working set, heap size, image size | CPU (caching, Server GC), throughput |
| **Efficiency** | CPU-milliseconds per request, cost per million requests | Development time, readability |
| **Scalability** | How the above change with load and core count (Module 6's USL) | Simplicity (sharding, statelessness) |
| **Predictability** | Variance, jitter, p99/p50 ratio | Peak throughput (e.g., GC latency modes) |

Two points carry most of the weight in an interview. First, **name the binding dimension.** A serverless function is bound by cold start; a trading gateway by p99.9; a batch pipeline by throughput per dollar; a sidecar by footprint × replica count. An optimization that improves a non-binding dimension is, at best, a free option and, at worst, a cost. Second, **"throughput" without a latency condition is meaningless for interactive systems** — any service can accept more requests per second if you let queues grow without bound; what you can actually sell is the throughput at which the latency target still holds (Concept 62 calls this the knee).

This is Module 4's non-functional requirements made precise. When an interviewer says "it needs to be fast," the senior move is to turn it into numbers out loud: *"What's the latency target, at which percentile, and at what load? Is cold start part of the SLO? Is there a cost ceiling?"*

---

## Concept 2 — The optimization loop: hypothesis, one change, measurement

Performance work is the scientific method with a stopwatch. The loop:

1. **Define** the metric and the target (Concept 1). The target is also your stopping condition.
2. **Measure a baseline** in an environment that resembles production — same instance size, same runtime, same data shape, same GC settings, realistic load.
3. **Locate** the cost with a profiler or trace (Part D), not with intuition. Programmers' intuitions about where time goes are wrong often enough that this step is not optional.
4. **Hypothesize with a predicted effect size.** Not "this should help" but *"this path allocates ~6 KB per request, which is ~70% of our allocation rate; removing it should roughly halve gen0 collections and take ~5–10 ms off p99."*
5. **Change one thing.**
6. **Re-measure with the identical method.**
7. **Keep or revert.** An "optimization" that didn't move the metric still costs readability; revert it.
8. **Record it** — the numbers go in the PR description, the benchmark is committed, the allocation assertion is added (Concept 57).

Step 4 is the one that distinguishes engineers from tinkerers. **The prediction is what makes the measurement informative.** If you predicted a 10 ms improvement and got 0.5 ms, your model of the system is wrong, and finding out *why* is more valuable than the optimization itself — perhaps the allocation wasn't on the critical path, perhaps GC wasn't the source of the tail, perhaps the JIT had already eliminated the cost.

Anti-patterns worth naming, because interviewers probe for them:

- **Shotgun optimization** — changing ten things at once, so nobody knows which one mattered or which one regressed something.
- **The streetlight effect** — optimizing what is easy to measure (a micro-benchmark) rather than what matters (the p99 of a service).
- **Optimizing without a target** — there is no stopping condition, so the work continues until readability is gone.
- **Trusting one run** — every measurement has noise (Concept 18); a single before/after pair proves very little.
- **Measuring in the wrong environment** — a laptop in Debug mode with a debugger attached, a load test against a database with a warm cache that production never has.

---

## Concept 3 — Where the time goes: Amdahl, the critical path, and the investigation hierarchy

**Amdahl's law** bounds any optimization: if a fraction *p* of the time is improved by a factor *s*, the overall speedup is `1 / ((1 − p) + p/s)`, with a ceiling of `1/(1 − p)` as *s* → ∞. Make it concrete: a 10× improvement to code that is 30% of the time yields 1.37× overall; the same 10× on 3% yields 1.03×. Module 14 (Concept 55) used this to argue against premature allocation work; here it is the first question in every investigation: **what fraction of the metric does this code account for?**

For latency, the fraction that matters is the fraction of the **critical path**. A request that calls a pricing service and an inventory service in parallel, each taking 40 ms and 25 ms, has a critical path through pricing; making inventory infinitely fast saves nothing. Distributed tracing waterfalls (Modules 11 and 28) are the tool that shows you the critical path; a CPU profile of one process does not.

Then use the **investigation hierarchy** — levels ordered by typical leverage. Each level down is usually an order of magnitude less impact and more engineering cost:

| Level | Question | Typical fixes | Where in the curriculum |
|---|---|---|---|
| 1. Work avoided | Does this need to happen at all, or now, or here? | Caching, precomputation, removing a call, pushing work async | Modules 10, 11 |
| 2. Round trips and I/O | How many network/disk hops, and are they serial? | Batching, eliminating N+1, parallelizing independent calls, fewer bytes | Modules 12, 19 |
| 3. Algorithms and data structures | Is the complexity right for the real *n*? | Better structure, indexing, avoiding quadratic paths | (your strength) |
| 4. Concurrency and contention | Is the work waiting on locks, pools, or threads? | Removing blocking, sharding state, bounding concurrency | Module 15 |
| 5. Memory and GC | Is allocation or retention driving GC cost? | Reducing survivors, LOH churn, hot-path allocations | Module 14, Part I here |
| 6. CPU micro-architecture | Cache misses, branch mispredicts, missed vectorization? | Layout, branchless code, SIMD | Parts B, G here |
| 7. Instruction-level | Bounds checks, inlining, codegen details | Idioms the JIT understands | Part E here |

Most real-world wins are at levels 1–3. Most of *this module's* techniques sit at levels 5–7. Knowing that is itself the senior signal: you reach for spans and SIMD after you've established that the service isn't making four serial database calls where one would do. Module 5's latency numbers make the point: one avoided cross-region round trip (tens of milliseconds) is worth more than every bounds check the JIT will ever remove from your service.

---

## Concept 4 — Latency is a distribution

Real latency distributions are **right-skewed** (a hard floor, a long tail) and frequently **multimodal** (cache hit vs miss, fast path vs slow path, no-GC vs GC, first attempt vs retry). The arithmetic mean of such a distribution can correspond to no request that ever happened.

A concrete case: 90% of requests hit a cache and take 5 ms; 10% miss and take 80 ms. The mean is 12.5 ms. No request takes 12.5 ms. The median is 5 ms; p95 is 80 ms. Any dashboard that shows only the average is showing you a number that describes nothing — and when the cache hit rate drops from 90% to 80%, the mean creeps from 12.5 to 20 ms while p90 jumps from 5 ms to 80 ms.

What to report, and why:

- **p50** — the typical experience.
- **p90/p99** — the experience of a meaningful minority. At 1,000 rps, p99 is ten users every second.
- **p99.9 and max** — where GC pauses, retries, lock convoys, and post-deployment JIT live.
- **The count and the window** — a p99 over 50 samples is noise; a p99 over a day hides a bad five minutes.
- **The load at the time** — latency without load is not a data point.

Users feel the tail more than intuition suggests, because **user-facing operations are composed of many requests.** A session of 20 requests has a `1 − 0.99^20 ≈ 18%` chance of including at least one request slower than the per-request p99. A page that makes 50 backend calls hits the per-call p99 on most loads. This is why large-scale systems set SLOs at p99 or p99.9 rather than at the median.

The usual sources of tail latency in a .NET service, worth being able to list: GC pauses (Module 14), queueing under utilization (Concept 7), ThreadPool starvation and lock contention (Module 15), tier-0 JIT code after deployment (Concept 27), cold caches and connection establishment (Concept 63), retries and timeouts (Module 13), noisy neighbours on shared hosts, and background work such as compaction or log flushing.

**Always look at the histogram, not just the summary statistics.** Bimodality is diagnostic: two humps means two populations of requests, and the fix is usually to find out what distinguishes them.

---

## Concept 5 — Aggregating percentiles, HDR histograms, and tail amplification

**Percentiles cannot be averaged.** The mean of ten instances' p99 values is not the fleet's p99; the mean of sixty per-minute p99s is not the hour's p99. Both errors are common on dashboards and both understate the tail when load is uneven — a single hot instance with a terrible p99 disappears into the average. To compute a percentile over a combined population you need the **underlying distribution**, which means recording **histograms** and merging them.

**HDR Histogram** (Gil Tene) is the canonical design: logarithmic buckets subdivided linearly to a configurable precision (for example, three significant digits), covering nanoseconds to hours in a few kilobytes, mergeable by adding bucket counts. **OpenTelemetry exponential histograms** and **Prometheus native histograms** implement the same idea. The older **explicit-bucket histogram** — fixed boundaries like 5, 10, 25, 50, 100, 250 ms — is mergeable too, but a percentile is only interpolated between boundaries; if your SLO threshold is 150 ms and the buckets are 100 and 250, your "p99" is a guess. When you own the SLO, make sure its threshold is a bucket boundary (Module 28 returns to this).

**Tail amplification under fan-out** ("The Tail at Scale", Dean and Barroso). If a request fans out to *n* backends in parallel and must wait for all of them, and each backend is slow with probability *p*, the request is slow with probability `1 − (1 − p)^n`. With *p* = 1% and *n* = 100, that is **63%**. The per-leaf p99 becomes the aggregate's median. Consequences:

- Fan-out services need **much tighter tails at the leaves** than the SLO at the root suggests.
- Mitigations are architectural: **hedged requests** (send a second copy after a p95 delay and take the first answer — Module 13, and Polly's hedging strategy in Module 25), **tied requests** (the copies cancel each other), **micro-partitioning** and **selective replication** of hot partitions, and **good-enough answers** (return with 98 of 100 shards when the last two are late, where the product allows it).
- Reducing fan-out is often the best mitigation of all.

The senior question to ask whenever someone quotes a latency number: *"Which percentile, over which window, aggregated how?"*

---

## Concept 6 — Coordinated omission and open vs closed load

This is the single most common way load tests lie, and it is a reliable interview differentiator.

A **closed-model** load generator runs a fixed number of virtual users; each sends its next request only after the previous one returns (plus think time). When the system stalls, the users wait — so the generator **stops sending requests exactly when the system is slow.** A one-second stall that should have affected the ~1,000 requests scheduled during that second is recorded as *one* slow sample per virtual user. The generator has "coordinated" with the system under test and "omitted" the requests that would have experienced the stall. The recorded p99 can be understated by orders of magnitude. (The term is Gil Tene's; his talk *How NOT to Measure Latency* is in the resources and is worth an hour.)

An **open-model** generator sends requests on a schedule independent of completions — constant arrival rate or Poisson arrivals — which is how independent users on the internet actually behave. During a stall, requests keep arriving and queue, and every one of them records the full wait. That is the latency your users experience.

Practical guidance:

- **Internet-facing APIs: open model.** k6's `constant-arrival-rate` and `ramping-arrival-rate` executors, NBomber's `Inject` simulations, wrk2's constant-throughput mode, Bombardier with a rate limit. Closed-model tools (NBomber's `KeepConstant`, k6's VU-based executors, plain `wrk`) are fine for *throughput* discovery and misleading for *latency*.
- **Closed systems are real too**: a fleet of 50 worker processes each pulling from a queue, or an internal caller with a fixed-size connection pool, genuinely behaves closed-loop. Model the system you have.
- **Measure latency on the client**, from the intended send time where the tool supports it. Server-side timings (for example, ASP.NET Core's `http.server.request.duration`) start when Kestrel begins processing and miss the time spent in the load balancer, in the TCP accept backlog, and in Kestrel's own connection queue.
- HdrHistogram can correct for coordinated omission after the fact if you know the expected interval between requests, but it is better to generate load correctly in the first place.

A useful story for an interview: *"Our load test showed p99 of 90 ms and production showed 1.4 s during GC-heavy periods. The load test used 200 closed-loop virtual users, so every pause simply slowed the generator down. Switching to a constant-arrival-rate executor reproduced production's tail on the first run."*

---

## Concept 7 — Utilization and queueing: why 90% CPU is a latency decision

Module 6 introduced Little's Law (`L = λW`). The other queueing result every architect should be able to do in their head is the **M/M/1 response time**: with service time *S* and utilization ρ, mean response time is

`R = S / (1 − ρ)`

| Utilization ρ | Mean response time |
|---|---|
| 50% | 2 × S |
| 70% | 3.3 × S |
| 80% | 5 × S |
| 90% | 10 × S |
| 95% | 20 × S |

And the tail is worse than the mean: in M/M/1 the response time is exponentially distributed, so **p99 ≈ 4.6 × R**. The curve is a hockey stick, and the knee is well below 100%. This is why **target utilization is a latency decision, not a cost decision.** Running at 90% CPU "to save money" buys a tenfold queueing multiplier on every request.

Three refinements that make this useful rather than academic:

1. **Pooling helps enormously.** With *c* servers sharing one queue (M/M/c), queueing at the same utilization is far lower than for a single server — a 16-core instance at 70% queues much less than a 1-core instance at 70%. This is the queueing argument for one shared queue over many small ones (join-the-shortest-queue load balancing in Module 6), for larger instances in latency-sensitive tiers, and against aggressive per-endpoint thread partitioning.
2. **Variability multiplies waiting.** Kingman's approximation for a single queue: `wait ≈ (ρ/(1−ρ)) × ((c_a² + c_s²)/2) × S`, where *c_a* and *c_s* are the coefficients of variation of arrivals and service times. A mix of 2 ms requests and 2-second report exports in the same queue has enormous service-time variance and therefore enormous waits for the cheap requests. The fix is **isolation**: separate pools, queues, or instances for heavy work (Module 13's bulkheads), or making the heavy work asynchronous.
3. **In .NET, "CPU utilization" includes the runtime's own work** — GC threads, JIT compilation after deployment, ThreadPool management. A process at 75% CPU with 15% time in GC is effectively a 60%-utilized service with a pause source layered on top.

This is the reasoning behind autoscaling thresholds of 60–70% CPU: part of the headroom is queueing, part is reaction time while new instances start (Concept 63), and part is failure capacity (Concept 62). Say it in those terms and interviewers hear an architect.

---

## Concept 8 — USE and RED: systematic resource analysis

When a service is slow and you don't yet know why, a checklist beats inspiration. Two complementary methods:

**The USE method** (Brendan Gregg): for every *resource*, check **U**tilization (how busy), **S**aturation (how much work is waiting), and **E**rrors. Saturation is the one people skip, and it's usually the answer — a resource can be at 60% utilization and still saturated in bursts.

**The RED method** (Tom Wilkie): for every *service*, track **R**ate, **E**rrors, and **D**uration. Google's SRE book's four golden signals — latency, traffic, errors, saturation — are the same idea with saturation added.

USE applies to software resources as much as hardware, and in a .NET service the software resources are frequently the bottleneck:

| Resource | Utilization | Saturation | Where to read it |
|---|---|---|---|
| CPU | Process CPU % | Run-queue length; ThreadPool queue length | `dotnet.process.cpu.time`, `dotnet.thread_pool.queue.length` |
| Memory / GC | Heap size vs limit | % time in GC, gen2 frequency, pause time | `dotnet.gc.pause.time`, `dotnet.gc.collections`, `dotnet.gc.heap.total_allocated` |
| ThreadPool | Busy threads | Queue length; thread injection | `dotnet.thread_pool.thread.count`, `dotnet.thread_pool.queue.length` |
| Locks | Hold time | Contention count | `dotnet.monitor.lock_contentions` |
| Kestrel | Active connections | Queued connections and requests | `kestrel.active_connections`, `kestrel.queued_connections`, `kestrel.queued_requests` |
| Outbound HTTP | Open connections | Time waiting for a connection | `http.client.open_connections`, `http.client.request.time_in_queue` |
| Database pool | Connections in use | Waiters, wait time | Driver metrics (SqlClient, Npgsql) |
| Channels, semaphores, rate limiters | Depth / permits in use | Waiters, rejections | Your own gauges (Module 15, Concept 72) |
| Network | Bandwidth | Retransmits, socket backlog | Host metrics |

(The metric names are the built-in .NET, ASP.NET Core, and `HttpClient` meters exported through OpenTelemetry; `dotnet-counters monitor` shows the same data live.)

The method: walk the table, find the resource with high saturation or errors, and form your first hypothesis there — then profile to confirm (Part D). It turns "the service is slow" into "requests are waiting 40 ms for an outbound HTTP connection because the handler's connection limit is 10 and the downstream got slower," which is a sentence you can act on.

---

# Part B — The machine under the runtime

You have a strong algorithms foundation, which means you already reason in operation counts. This part adds the correction the hardware applies to those counts. The RAM model of computation — every memory access costs the same — is wrong by up to two orders of magnitude on modern hardware, and knowing *where* it's wrong is what lets you predict when an O(n) scan beats an O(log n) tree or an O(1) hash lookup.

## Concept 9 — The memory hierarchy is the cost model

Approximate latencies on a modern ~3–4 GHz server core (exact numbers vary by microarchitecture; the ratios are what matter):

| Level | Typical size | Latency | In cycles |
|---|---|---|---|
| Register | — | 0 | 0 |
| L1 data cache | 32–64 KB per core | ~1 ns | ~4–5 |
| L2 cache | 1–2 MB per core | ~3–5 ns | ~12–16 |
| L3 cache | tens of MB, shared | ~10–20 ns | ~40–70 |
| DRAM | GBs | ~80–120 ns | ~300+ |
| DRAM on a remote NUMA node | — | ~+50% | — |

Three facts turn this table into a cost model:

1. **The unit of transfer is the cache line** — 64 bytes on x64 and most Arm server cores (128 bytes on Apple's M-series for some levels). Touching one byte of a line costs the whole line. Using all 64 bytes of every line you fetch is the goal.
2. **Hardware prefetchers predict sequential and strided access.** A linear scan over a contiguous array runs at memory *bandwidth* (tens of GB/s), because the next lines are already in flight. Random access or pointer chasing runs at memory *latency*: each dependent miss is ~100 ns, serialized, and the prefetcher cannot help because the next address isn't known until the current load completes.
3. **The TLB is a cache too.** With 4 KB pages, a working set of several GB touched randomly also misses in the translation lookaside buffer, adding a page-walk on top of the cache miss. (The GC supports large pages — `GCLargePages` with a heap hard limit — for the rare workloads where this dominates.)

The consequences for code you write:

- **An array of 1 million `int`s scanned linearly** costs ~4 MB of streaming reads — a millisecond or less.
- **A linked list or tree of 1 million nodes traversed** costs up to ~1 million dependent misses — potentially 100 ms if the nodes are scattered.
- **For small and medium *n*, a linear scan over a contiguous array frequently beats a hash lookup or a binary search tree**, because it touches one or two cache lines and has no dependent loads. The crossover depends on element size and comparison cost; measure it (Concept 45).

In .NET terms: every reference-type object carries a 16-byte header on x64 (object header + MethodTable pointer; Module 14, Concept 3) and lives wherever the allocator put it. A `List<Customer>` is an array of 8-byte references, each pointing somewhere else. Iterating it and reading one field per element is a pointer chase per element.

---

## Concept 10 — Data layout and locality

Once memory movement is the cost, **layout becomes a performance decision** on the same footing as algorithm choice.

**Array of structs vs struct of arrays.** Suppose a simulation holds a million particles with eight fields each (position, velocity, mass, charge, flags), and the hot loop updates only the position from the velocity:

```csharp
// AoS: each 48-byte particle spans most of a cache line; the loop uses 24 of those bytes
struct Particle { public float X, Y, Z, Vx, Vy, Vz; public float Mass, Charge; public int Flags; ... }
Particle[] particles;

// SoA: the hot loop streams through exactly the bytes it needs, and the loop is trivially vectorizable
sealed class Particles
{
    public float[] X, Y, Z, Vx, Vy, Vz;   // plus cold arrays for Mass, Charge, Flags
}
```

The SoA version moves less memory and exposes contiguous `float` runs that `Vector256<float>` can process eight at a time (Concept 46). The AoS version is more natural to program. Real systems often use a hybrid — keep hot fields together in one struct array and move cold fields to another.

**Struct arrays vs class arrays** (Module 14, Concept 47): a `Reading[]` of structs is one contiguous block; a `Reading[]` of classes is an array of references plus a million separately allocated objects. For iteration-heavy workloads — analytics, in-memory indexes, market data, simulations, game state — this is often the single largest performance difference available, and it also shrinks the GC's tracing work.

A nuance worth knowing: **the .NET GC's compaction preserves the relative order of surviving objects** (it slides them down rather than shuffling them), so objects allocated together in a burst tend to stay adjacent after compaction. Allocation-time locality therefore persists — but objects allocated at different times, interleaved with other allocations, end up interleaved in memory.

**Hot/cold splitting.** A class with three frequently read fields and fifteen rarely used ones (audit metadata, descriptions, optional settings) drags the cold fields through the cache on every access. Moving them into a separately referenced "details" object keeps the hot object within one or two cache lines.

**Field order.** The CLR uses automatic layout for classes and reorders fields (references grouped together, then by size) to minimize padding — you cannot rely on declaration order. Structs default to sequential layout, but a struct containing references is laid out automatically as well. `ObjectLayoutInspector` (in the resources) shows you the actual layout and padding.

**Know the layouts of the structures you use.** `Dictionary<TKey,TValue>` stores entries in a contiguous struct array with a separate bucket array — much better locality than node-based maps. `SortedDictionary` and `SortedSet` are red-black trees of individually allocated nodes. `ImmutableList<T>` is a balanced tree — indexing it is O(log n) pointer chasing, which is why `ImmutableArray<T>` exists. `LinkedList<T>` is almost never the right answer in .NET.

Where this matters: large collections iterated in hot loops. Where it doesn't: CRUD services whose time is dominated by I/O. Saying which one you're in is part of the answer.

---

## Concept 11 — Branches, speculation, and dependency chains

Modern cores are deep pipelines that **speculate**: at a conditional branch they predict the outcome and keep executing. A correct prediction costs almost nothing; a **misprediction** discards the speculative work — roughly **15–20 cycles**. Predictors are excellent at regular patterns and useless on random data.

The classic demonstration is a filter loop over random bytes, summing those above 128. On unsorted data the branch is a coin flip; sorting the same data first makes the loop several times faster because the branch becomes perfectly predictable. The data didn't change; its order did.

What to do with this:

- **Branchless forms** remove the prediction entirely: `Math.Min`/`Math.Max`, bit manipulation, arithmetic on comparison results, lookup tables. The JIT converts simple ternaries and `if` assignments into conditional moves (`cmov` on x64, `csel` on Arm64) and, in .NET 11, folds pattern sets like `x is 0 or 1 or 2 or 3 or 4` into branchless range checks. Branchless is not automatically faster: it always pays for both sides, so it wins on unpredictable branches and loses on predictable ones.
- **Data-dependent branching in parsers** is the main reason the BCL's text-processing primitives are vectorized: SIMD comparison finds the positions of delimiters in 16–64 bytes at a time without branching per character (Concepts 44 and 46).
- **Sort or partition data when you'll process it repeatedly** — it helps both the branch predictor and cache locality.

**Dependency chains and instruction-level parallelism.** A modern core can issue four or more independent instructions per cycle. A loop like `sum += a[i]` forms a serial chain through `sum`: each addition must wait for the previous one. For integers that's one addition per cycle; for floating point, whose add latency is about 4 cycles, it's one addition every 4 cycles, using a fraction of the core. **Multiple accumulators** break the chain:

```csharp
double s0 = 0, s1 = 0, s2 = 0, s3 = 0;
int i = 0;
for (; i <= a.Length - 4; i += 4) { s0 += a[i]; s1 += a[i + 1]; s2 += a[i + 2]; s3 += a[i + 3]; }
for (; i < a.Length; i++) s0 += a[i];
double sum = (s0 + s1) + (s2 + s3);
```

Two facts about .NET make this relevant. **The JIT will not reassociate floating-point arithmetic** — doing so changes results — so it cannot make this transformation for you. And **RyuJIT does not generally auto-vectorize your loops**; vectorization in .NET comes from explicit `Vector128/256/512` code and from BCL methods written that way (`TensorPrimitives.Sum`, LINQ's `Sum` over arrays and spans, `MemoryExtensions.IndexOf`, and so on). The practical rule: before hand-writing either, check whether a BCL method already does it (Concept 46).

Two smaller costs worth knowing: **integer division** is expensive (tens of cycles for 64-bit), which is why the JIT turns division by a constant into multiplication and why `Dictionary` uses a multiply-based "fast mod" on 64-bit; and **hardware counters** (branch mispredictions, cache misses) are the way to confirm these effects rather than infer them — BenchmarkDotNet's `[HardwareCounters]` on Windows, `perf stat` on Linux.

---

## Concept 12 — Cores, contention, and the cost of sharing

Module 15 covered the memory model, `Interlocked`, false sharing, and locking in depth. Here is the cost model that connects them to throughput:

| Operation | Approximate cost |
|---|---|
| Uncontended `lock` enter/exit | ~20 ns |
| `Interlocked.Increment` on a line owned by this core | ~5–10 ns |
| `Interlocked.Increment` on a line bouncing between many cores | ~100 ns or more, serialized |
| Contended lock with a thread blocking | Microseconds (a context switch) |

The key property: **a shared write is a serialization point.** A single global counter incremented on every request by 32 cores does not scale — every increment must acquire the cache line exclusively, so the cores take turns. That is Module 6's Universal Scalability Law made physical: the contention term (σ) is the serialized portion, and the coherency term (κ) is the cache-line traffic that grows with the number of participants. Throughput that flattens or *falls* as you add cores is the signature.

The remedies, in order of preference:

1. **Don't share mutable state on the hot path.** Per-request state, immutable inputs.
2. **Shard it.** Striped counters (one per core or per thread, padded to separate cache lines), `ConcurrentDictionary`'s lock striping, per-partition state in stream processors.
3. **Accumulate locally, merge periodically.** Thread-local or per-worker aggregates flushed every N items or T milliseconds — this is how metrics libraries keep `Counter<T>.Add` cheap.
4. **Batch.** One shared write per 1,000 items instead of per item.
5. **Snapshot for read-mostly data.** Build a new immutable object, publish it with a single reference swap; readers never lock (Module 15, Concept 55).

Two hardware realities that affect capacity planning:

- **SMT (hyper-threading).** On most cloud VM families a vCPU is a hardware thread, not a physical core; two vCPUs may share one core's execution units. CPU-bound throughput does not double from 2 to 4 vCPUs if the extra two are siblings. Measure per-instance throughput on the actual SKU (Concept 62).
- **NUMA.** Large VMs can span multiple NUMA nodes; memory attached to another socket is slower. Server GC is NUMA-aware, but a single giant process spanning nodes can still pay for remote memory. Several smaller processes, each within a node, are often faster than one large one — the same conclusion Module 14 (Concept 63) reached from the GC side.

How to see contention: `dotnet.monitor.lock_contentions` in production, `[ThreadingDiagnoser]` in benchmarks, PerfView or the Visual Studio concurrency views for blocked time, and — most tellingly — a **scaling curve**: throughput measured at 1, 2, 4, 8, 16 cores. A flat curve is contention; a curve that bends downward is coherency cost.

---

# Part C — Micro-benchmarking with BenchmarkDotNet

Micro-benchmarks are the most over-trusted artefact in performance work. They are indispensable for one job — comparing the cost of alternatives in isolation — and misleading for almost every other. This part is about doing that one job correctly, and about knowing its limits.

## Concept 13 — Why a Stopwatch loop lies

The hand-rolled benchmark everyone writes once:

```csharp
var sw = Stopwatch.StartNew();
for (int i = 0; i < 1_000_000; i++) Parse("2026-09-22T10:15:30Z");
Console.WriteLine(sw.Elapsed);
```

It is wrong in at least ten independent ways, and being able to list them is a standard interview question:

1. **It measures tier-0 code.** The first calls run unoptimized tier-0 code; promotion to tier-1 happens on a background thread after ~30 calls plus a delay (Concept 27). A short loop measures a mixture of the two, plus the compilation itself.
2. **Dead-code elimination.** The result of `Parse` is discarded. If the JIT can prove the call has no side effects after inlining, it removes the work entirely and you measure an empty loop.
3. **Constant folding.** The input is a literal. After inlining, parts of the computation can be evaluated at compile time. (The same applies to `static readonly` fields, which tier-1 treats as constants.)
4. **GC noise.** One run absorbs a gen0 or gen2 collection triggered by earlier allocations; the next doesn't.
5. **Frequency scaling and thermals.** Turbo boost, power plans, laptop thermal throttling, and battery mode change clock speed between and during runs.
6. **Environment noise.** Background processes, antivirus, noisy neighbours on a VM, the IDE indexing your solution.
7. **Build and debugger.** A Debug build or an attached debugger disables JIT optimizations.
8. **Order and interference effects.** Running A then B in one process lets A warm caches and branch predictors for B, and lets A's PGO data shape B's code if they share call sites.
9. **Unrealistically hot data.** The same small input stays in L1 cache and trains the branch predictor perfectly — production data doesn't.
10. **One sample.** There is no variance estimate, so you cannot tell a 3% difference from noise.

And one subtle one: **code alignment**. Where a hot loop lands relative to 32- or 64-byte boundaries can change its speed by 10–20% on some microarchitectures; the JIT aligns loops (since .NET 6) to reduce this, but it's a reason small differences between two builds can be meaningless.

---

## Concept 14 — What BenchmarkDotNet actually does

BenchmarkDotNet (BDN) exists to neutralize the list above. Knowing its mechanics lets you explain *why* its numbers are trustworthy — and when they aren't.

**Process isolation.** For each benchmark job, BDN generates a project, builds it in Release, and runs each benchmark in a **separate process**. JIT state, PGO profiles, GC heaps, and static state don't leak between benchmarks. (The in-process toolchains exist for environments where spawning processes is impossible, at the cost of this isolation.)

**Stages**, for each benchmark:

1. **Jitting** — invokes the method enough to compile it and to let tiered compilation promote it to optimized code.
2. **Pilot** — finds the number of invocations per iteration *N* such that one iteration takes at least the target iteration time (500 ms by default), so timer resolution becomes irrelevant.
3. **Overhead warm-up and measurement** — measures an empty method with the same signature, so the harness's own call and loop overhead can be subtracted.
4. **Workload warm-up** — runs iterations until measurements stabilize.
5. **Workload measurement** — runs iterations until the statistics are stable enough (adaptively, between a minimum and maximum count), then computes per-operation time as `(workload − overhead) / N`.
6. **Statistics** — removes upper outliers by default, computes mean, standard error, confidence interval, median, quartiles, and flags suspicious distributions.

**Other details that matter:**

- It **unrolls** the invocation loop (16 calls per loop iteration by default) to shrink loop overhead further.
- It **forces a full GC between iterations** by default, so each iteration starts from a clean heap. This makes results reproducible — and means BDN can *understate* GC costs relative to a production process whose heap is large and whose gen2 collections interleave with the work (Concept 20).
- It **consumes return values** so the JIT cannot eliminate the work.
- It prints an **environment header** — OS, CPU model, runtime version, JIT, enabled instruction sets (for example, "X64 RyuJIT AVX2"), GC mode. Always include it when sharing results; a number without its environment is not a result.
- Since 0.15.7 it ships **Roslyn analyzers** that flag structural mistakes at compile time — non-public or sealed benchmark classes, malformed `[Arguments]`/`[Params]`, multiple baselines in a category.

---

## Concept 15 — Writing a correct benchmark

A template that avoids the common mistakes:

```csharp
using System.Collections.Frozen;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

BenchmarkSwitcher.FromAssembly(typeof(HeaderLookupBenchmarks).Assembly).Run(args);

[MemoryDiagnoser]
[HideColumns("Error", "StdDev", "RatioSD")]      // keep them while investigating; hide for reports
public class HeaderLookupBenchmarks
{
    private string[] _keys = default!;             // instance fields, not constants or static readonly
    private Dictionary<string, int> _dict = default!;
    private FrozenDictionary<string, int> _frozen = default!;

    [Params(8, 64, 4_096)]                         // small (cache-resident) through large
    public int Size;

    [Params(0.9)]                                  // realistic hit ratio, not "always found"
    public double HitRatio;

    [GlobalSetup]
    public void Setup()
    {
        var rng = new Random(42);                  // fixed seed: reproducible, but not sorted or trivial
        var names = Enumerable.Range(0, Size).Select(i => $"x-header-{i:x6}").ToArray();
        _dict = names.ToDictionary(n => n, n => n.Length, StringComparer.OrdinalIgnoreCase);
        _frozen = _dict.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);

        // Lookup keys are NEW string instances (as they would be when parsed from a request),
        // mixing hits and misses in random order.
        _keys = Enumerable.Range(0, 1_024).Select(_ =>
                rng.NextDouble() < HitRatio
                    ? names[rng.Next(Size)].ToUpperInvariant()
                    : $"x-missing-{rng.Next():x8}")
            .ToArray();
    }

    [Benchmark(Baseline = true, OperationsPerInvoke = 1_024)]
    public int Dictionary()
    {
        int found = 0;
        foreach (string k in _keys) if (_dict.TryGetValue(k, out _)) found++;
        return found;                                 // return something the JIT cannot discard
    }

    [Benchmark(OperationsPerInvoke = 1_024)]
    public int Frozen()
    {
        int found = 0;
        foreach (string k in _keys) if (_frozen.TryGetValue(k, out _)) found++;
        return found;
    }
}
```

Run it with `dotnet run -c Release -- --filter "*HeaderLookup*"`.

The rules this encodes:

- **Return the result.** BDN consumes return values; `void` benchmarks invite dead-code elimination. For lazy sequences use `Consumer` and `.Consume(consumer)` so the enumeration actually happens.
- **Inputs come from instance fields initialized in `[GlobalSetup]`** — not literals, not `const`, not `static readonly` (all of which the JIT can fold).
- **Sweep sizes with `[Params]`** across cache boundaries: something that fits in L1, something in L2/L3, something that doesn't fit at all. Performance cliffs at cache boundaries are often the most important result.
- **Use realistic distributions**: hit/miss ratios, key lengths, skew, random order with a fixed seed. Sorted or single-valued inputs train the branch predictor and flatter the code.
- **Use fresh instances where production has them.** A lookup key that is the *same string instance* as the stored key hits `string.Equals`'s reference-equality fast path — production keys parsed from input never do (worked example 3 turns this into a full critique).
- **Match production's type mix at polymorphic call sites.** If production sees five implementations of an interface and your benchmark sees one, dynamic PGO will devirtualize the benchmark's call site and not production's (Concept 28).
- **Batch tiny operations with `OperationsPerInvoke`** and a loop over a prepared input set, rather than benchmarking a 3 ns operation directly.
- **Avoid `[IterationSetup]` for operations shorter than ~100 ms.** It forces one invocation per iteration, so you end up measuring timer resolution and setup overhead. Make the operation repeatable instead (reset state cheaply, or operate on a pool of prepared inputs).
- **Keep setup out of the measured method** unless setup cost *is* the question (constructing a `Regex` per call is a legitimate benchmark if that's what the code under review does).
- **Name a baseline** and read the `Ratio` column; absolute nanoseconds are machine-specific, ratios travel better.
- **Async benchmarks** may return `Task`/`ValueTask`; BDN awaits them. Remember you're measuring the await machinery too.
- **Spans as arguments**: since 0.15.6, `[ArgumentsSource]` supports ref-struct parameters such as `ReadOnlySpan<char>`.

---

## Concept 16 — Diagnosers: seeing more than time

A time is a symptom. Diagnosers attach explanations:

| Diagnoser | What it adds | Notes |
|---|---|---|
| `[MemoryDiagnoser]` | **Allocated bytes per operation**; Gen0/Gen1/Gen2 collections per 1,000 operations | Managed allocations only, all threads; accuracy improved in 0.15.2 around tiered JIT. The most useful diagnoser by far |
| `[DisassemblyDiagnoser]` | The JIT's machine code for the benchmark (and callees to `maxDepth`), code size, optional diffs | Concept 33. Check it whenever a result surprises you |
| `[ThreadingDiagnoser]` | Completed work items and lock contentions per operation | Module 15's contention questions, measured |
| `[ExceptionDiagnoser]` | Exceptions thrown per operation | Catches exceptions hidden inside library calls |
| `[EventPipeProfiler(EventPipeProfile.CpuSampling)]` | A CPU-sampling trace per benchmark (viewable in Speedscope or PerfView) | Cross-platform "where inside the benchmark does the time go?" |
| `[HardwareCounters(...)]` | Branch mispredictions, cache misses, retired instructions per op | Windows, via ETW, requires elevation; on Linux use `perf stat` around the run |
| `[InliningDiagnoser]`, `[TailCallDiagnoser]` | JIT inlining and tail-call decisions | Windows ETW-based |
| `[NativeMemoryProfiler]` | Native allocations and leaks | Windows ETW-based |
| dotTrace / dotMemory / Visual Studio profiler diagnosers | A full profiler session per benchmark | Separate packages |

Two habits: **always run with `[MemoryDiagnoser]`** — allocation is deterministic and often explains time; and **open the disassembly whenever a result is surprising** (a benchmark that is suspiciously fast has usually been optimized away; one that is suspiciously slow often has a bounds check, a box, or an indirect call you didn't expect).

---

## Concept 17 — Reading the results table

A typical result:

```
| Method     | Size | Mean      | Error     | StdDev    | Median    | Ratio | Gen0   | Allocated | Alloc Ratio |
|----------- |----- |----------:|----------:|----------:|----------:|------:|-------:|----------:|------------:|
| Dictionary | 4096 | 21.43 ns  | 0.212 ns  | 0.198 ns  | 21.40 ns  |  1.00 |      - |         - |          NA |
| Frozen     | 4096 | 12.87 ns  | 0.141 ns  | 0.132 ns  | 12.85 ns  |  0.60 |      - |         - |          NA |
```

(Illustrative numbers — the point is how to read them, not the values.)

- **Mean** — the arithmetic mean per operation after outlier removal.
- **Error** — **half the width of the 99.9% confidence interval of the mean.** "21.43 ± 0.21 ns" is the honest reading.
- **StdDev** — the spread of iteration results. Large StdDev relative to Mean means a noisy or multimodal benchmark.
- **Median** — robust to outliers; when Mean and Median disagree noticeably, the distribution is skewed.
- **Ratio / RatioSD** — relative to the baseline, with its own variability. A ratio of 0.97 with RatioSD 0.04 is *no difference*.
- **Gen0/Gen1/Gen2** — collections per 1,000 operations. "-" means none observed.
- **Allocated** — bytes per operation. "-" means zero. **This column is deterministic**: unlike time, it doesn't vary between runs or machines (for the same runtime), which makes it the ideal thing to assert on in CI (Concept 64).
- **Code Size** (with the disassembly diagnoser) — bytes of machine code.

Warnings and hints to take seriously:

- **`ZeroMeasurement`** — the method is indistinguishable from an empty method. You measured nothing; the work was eliminated or is below resolution.
- **"The minimum observed iteration time is … which is very small"** — the operation is too small for reliable measurement; batch it (`OperationsPerInvoke`).
- **Multimodal distribution warnings** (BDN reports an "mvalue") — two or more humps: alignment effects, GC landing in some iterations, input-dependent paths, or background noise. Look at the histogram before trusting the mean.
- **Outliers removed** — a few is normal; many means an unstable environment.

---

## Concept 18 — Statistics: comparing two numbers honestly

A benchmark comparison is a statistical claim: *distribution A is shifted relative to distribution B by at least X.* Treat it that way.

- **Set a practical threshold before you look.** "We care about differences of 5% or more." With enough samples, any difference becomes statistically significant; significance is not relevance.
- **Performance data is rarely normal.** It's skewed, has outliers, and is often multimodal. Robust statistics — medians, quantiles, and non-parametric tests such as Mann–Whitney — behave better than means and t-tests. Andrey Akinshin (BDN's author) has written extensively on robust estimators for exactly this data (the Hodges–Lehmann shift estimator, quantile-respectful density estimation); BDN 0.15.x moved its statistics onto Perfolizer 0.6 and its Pragmastat engine. BDN can add a statistical-test column comparing each benchmark to the baseline against a threshold — the API has evolved with the new engine, so check the current docs.
- **Compare within one session.** Put A and B in the same benchmark class with a baseline, or compare runtimes with multiple jobs in one run (Concept 19). Numbers from different days, machines, or VM instances are different experiments.
- **Reproduce before you believe.** Rerun; ideally on a second machine. A result you can't reproduce is noise until proven otherwise.
- **Beware multiple comparisons.** Run 200 benchmarks and some will "regress" by chance at any confidence level. Regression detection needs a threshold, a re-run, and a trend (Concept 64).
- **Laptops are hostile environments** — turbo, thermal throttling, power management. For claims you will act on, use a quiet dedicated machine or at least a consistent cloud SKU; BDN's `[WakeLock]` (0.15.0) at least stops the machine from sleeping mid-run.
- **Report fully**: mean ± error, median, ratio, allocations, the environment header, and the exact command to reproduce.

The senior phrasing: *"It's 8% faster with an error of ±1%, reproduced twice on the same host, and it allocates 120 bytes less per call — and it's in a path that's 20% of our CPU profile, so I'd expect ~1.5% end to end, which I'd verify with the load test."*

---

## Concept 19 — Jobs, runtimes, and configuration

A **job** is an execution environment: runtime, JIT, GC settings, environment variables, build properties, run strategy. BDN runs every benchmark under every job, in one session, which is how you get fair comparisons.

**Comparing runtimes** — the cleanest way to see what an upgrade buys (this is exactly how the .NET team's annual performance post is produced):

```bash
dotnet run -c Release -f net10.0 -- --filter "*" --runtimes net10.0 net11.0
```

The project multi-targets (`<TargetFrameworks>net11.0;net10.0</TargetFrameworks>`) and BDN builds and runs each benchmark on each runtime, printing them side by side. `--runtimes nativeaot10.0` adds a Native AOT job. Attribute-based jobs (`[SimpleJob(RuntimeMoniker…)]`) do the same in code — check the moniker names for the newest runtimes against your BDN version.

**Comparing configurations**, via a custom config:

```csharp
var config = DefaultConfig.Instance
    .AddJob(Job.Default.WithId("PGO on"))
    .AddJob(Job.Default.WithId("PGO off")
        .WithEnvironmentVariable("DOTNET_TieredPGO", "0"))
    .AddJob(Job.Default.WithId("Server GC").WithGcServer(true))
    .AddJob(Job.Default.WithId("Runtime async")
        .WithMsBuildArguments("/p:Features=runtime-async=on"));
```

Environment variables let you test runtime knobs (`DOTNET_TieredPGO`, `DOTNET_TieredCompilation`, `DOTNET_ReadyToRun`) and hardware fallbacks (`DOTNET_EnableAVX2=0`, `DOTNET_EnableAVX512F=0`, `DOTNET_PreferredVectorBitWidth`) — the latter is how you verify that your vectorized code's scalar fallback is correct and acceptable on older hardware. `WithMsBuildArguments` (which replaced the deprecated `WithNuget` in 0.15.3) compares package versions or build-time feature switches.

**Run strategies:**

| Strategy | What it measures | Use for |
|---|---|---|
| `Throughput` (default) | Steady-state cost after warm-up | Almost everything |
| `ColdStart` | First invocations, no pilot or warm-up | First-call costs: serializer metadata, regex construction, lazy initialization |
| `Monitoring` | Fixed iterations, no pilot; each iteration is one long operation | Macro-benchmarks of whole operations lasting hundreds of milliseconds or more |

**Accuracy presets**: `[ShortRunJob]` for fast exploration (lower accuracy), `[DryJob]` to validate that benchmarks run at all (useful in CI), `[MediumRunJob]`/`[LongRunJob]` when you need tighter intervals.

**Exporters**: GitHub-flavoured Markdown for PRs, JSON for automated comparison (Concept 64), CSV, and — since 0.15.8 — OpenMetrics for Prometheus-compatible pipelines.

---

## Concept 20 — What a micro-benchmark can and cannot prove

**It can prove:**

- The relative cost of two implementations *in isolation*.
- Allocation per operation — precisely and deterministically.
- Codegen differences (with the disassembly diagnoser).
- How cost scales with input size, including cache cliffs.
- What a runtime upgrade does to a specific operation.

**It cannot prove:**

- **End-to-end latency impact.** Amdahl applies: a 10× win on 3% of the profile is 2.7%.
- **Behaviour under concurrency.** Contention, cache-line bouncing, ThreadPool effects, and lock convoys don't appear in a single-threaded benchmark.
- **GC interplay.** BDN starts each iteration with a clean heap; a production process has a large gen2, card-marking costs, and collections triggered by *other* code's allocations.
- **Real cache behaviour.** Benchmark data is hot; production code competes for cache with everything else the process is doing. Micro-benchmarks systematically favour code that uses more memory to save instructions (lookup tables, caches, unrolled code) — the icache and dcache pressure it creates shows up only in the real workload.
- **I/O.** Network, disk, and database behaviour need component or load tests.

Think in a **benchmark pyramid**, and require evidence at the level of the claim:

1. **Micro** — BenchmarkDotNet, one operation. "Parser A allocates 0 B/op; parser B allocates 480 B/op."
2. **Component** — a subsystem under a realistic workload in-process (a serializer plus the pipeline around it; BDN's `Monitoring` strategy, or a custom harness). "The ingestion path processes 1.8× more events per second."
3. **Service** — a load test against a deployed instance (Concept 61). "p99 at 3,000 rps dropped from 140 to 95 ms."
4. **Production** — canary metrics and traces (Concept 64). "CPU per request fell 11% across the fleet."

An interviewer who hears you say *"the micro-benchmark justified the change; the load test is what justified the claim"* hears someone who has been burned before and learned from it.

---

# Part D — Profiling: finding where the time goes

Benchmarks answer "which of these is cheaper?" Profilers answer the prior question: "what is actually expensive?" Step 3 of the optimization loop (Concept 2) lives here.

## Concept 21 — Profiler taxonomy: sampling, instrumentation, tracing

Three techniques, each answering a different question:

| Technique | How it works | Strength | Weakness |
|---|---|---|---|
| **Sampling** | Interrupt periodically (≈1 kHz is typical), record each thread's stack | Low overhead, safe in production, shows *where* time goes statistically | Misses short-lived functions; no call counts; accuracy grows with sample count |
| **Instrumentation** | Insert probes at method entry/exit | Exact call counts and per-call timings | High overhead; distorts small methods (the probe costs more than the method); rarely production-safe |
| **Tracing** | Record timestamped events the code or runtime emits (GC start/end, JIT, request start/stop, lock contention) | Shows *when* and *why* — causality, ordering, durations of specific activities | Only sees what emits events; high-volume events have overhead |

Two orthogonal distinctions matter as much as the technique:

- **CPU time vs wall-clock time.** A CPU profile samples threads *that are running*. A request that takes 800 ms but spends 790 ms waiting on a database, a lock, or the ThreadPool queue barely appears in a CPU profile. Latency problems need wall-clock or thread-time analysis (Concept 24).
- **Allocation profiling** is its own view — what is allocated, by whom, and how much of it survives (Concept 25).

Every profiler perturbs what it measures (the observer effect). Sampling perturbs least, which is why it's the default for production and the first tool in the lab.

---

## Concept 22 — The .NET diagnostics stack

The plumbing, from the bottom up:

- **`EventSource`** — the managed API for emitting events. The runtime (GC, JIT, ThreadPool, exceptions, contention, assembly loading), ASP.NET Core, `HttpClient`, and many libraries emit through it.
- **EventPipe** — the runtime's cross-platform, in-process event transport. No admin rights needed; tools connect over the **diagnostic port** (a Unix domain socket or named pipe). Its sampling profiler captures **managed stacks only**.
- **ETW** (Windows) and **perf_events / LTTng** (Linux) — the operating systems' tracing systems, which also see native code, the kernel, and other processes, but need elevation.
- **`user_events`** (Linux, .NET 10+) — EventPipe can emit its events as `user_events`, so a single trace captures managed events, native call stacks, and kernel events together. This is what `dotnet-trace collect-linux` uses; it's a preview feature needing root, kernel 6.4+, and glibc 2.27+.
- **`System.Diagnostics.Metrics`** — counters and histograms (the meters in Concept 8), consumed live by `dotnet-counters` and exported by OpenTelemetry.

The tools, organized by the question they answer:

| Question | Tool |
|---|---|
| What's the process doing right now, broadly? | `dotnet-counters monitor` (CPU, GC, ThreadPool, exceptions, contention, ASP.NET Core, `HttpClient`) |
| Where is CPU going? | `dotnet-trace collect --profile cpu-sampling` → PerfView or Speedscope; on Windows PerfView/ETW; on Linux .NET 10+ `dotnet-trace collect-linux` for native frames too; `perf` + `DOTNET_PerfMapEnabled=1` |
| What's allocating? | `dotnet-trace --profile gc-verbose`; Visual Studio's .NET Object Allocation tool; dotMemory |
| What's retained on the heap? | `dotnet-gcdump` (Module 14, Concept 58) |
| Why is it hung? | `dotnet-stack` (live stacks, no dump), `dotnet-dump` + SOS (Module 15, Concept 73) |
| All of the above in Kubernetes, without shelling in | **`dotnet-monitor`** — a sidecar exposing an HTTP API for traces, dumps, gcdumps, and metrics, plus **collection rules** ("when CPU > 80% for 60 s, capture a 30-second trace to blob storage") |
| Rich GUI analysis | PerfView (free, Windows, the most powerful), Visual Studio Performance Profiler (CPU, allocations, async, database, events), JetBrains dotTrace/dotMemory (commercial), Ultra (a free ETW-based profiler with a Firefox-profiler UI) |
| Linux-native without the .NET SDK | `perfcollect` (perf + LTTng), still the route for native stacks before .NET 10 |

Operational details that trip people up: the tools must run as the same user as the target process (or root) and share its `/tmp` for the diagnostic port — in containers that means installing the tools in the image, using a sidecar that shares the process namespace and `/tmp`, or using `dotnet-monitor`. Native frames need symbols on disk to resolve. Managed frames are resolved from runtime rundown events or perf maps.

---

## Concept 23 — Reading CPU profiles and flame graphs

The vocabulary:

- **Inclusive time** — time in a method *and everything it calls*. Sort by inclusive time to find hot *paths* ("request handler → serializer → property getter").
- **Exclusive (self) time** — time in the method's own code. Sort by exclusive time to find hot *code* — the leaves where cycles actually burn.
- **Top-down (call tree)** vs **bottom-up (by function)** views. Bottom-up by exclusive time is usually the fastest route to "what's hot"; then walk up the callers to find out *why* it's called so much.

**Flame graphs** (Brendan Gregg) plot stacks with the root at the bottom; **width is proportional to time**, the vertical axis is stack depth, and colour is usually arbitrary. The x-axis is *not* time order — frames are sorted alphabetically for merging. What to look for:

- **Wide plateaus at the top** — a leaf function doing a lot of self time.
- **Wide towers** — a call path that dominates; follow it down to see where it enters your code.
- **Many thin spikes under one frame** — death by a thousand cuts; the fix is usually at the frame they share.
- **Differential flame graphs** (before vs after, coloured by difference) — ideal for regression hunting (worked example 1).

Runtime signatures worth recognizing in .NET profiles:

| You see | It usually means |
|---|---|
| Lots of time in `clrjit` or JIT helpers | Startup or post-deployment JIT; many dynamic methods; a code path hit for the first time |
| `gc_heap::*` frames (`WKS::`/`SVR::`) | GC work — correlate with GC events and allocation rate |
| Allocation helpers (`JIT_New`, `AllocateObject`, …) | Allocation-heavy code; switch to an allocation profile |
| `Monitor`/`Lock` spinning and waiting, `SpinWait` | Lock contention (Module 15) |
| `Buffer.Memmove`, `SpanHelpers.*` | Copying — often `ToArray`, `Substring`, or stream buffering |
| Hashing and `Dictionary.FindValue` | Lookups in hot loops; comparer cost (culture-aware or case-insensitive) |
| `Regex` interpreter frames | A regex constructed per call, or an interpreted regex in a hot path (Concept 44) |
| Exception dispatch frames | Exceptions used for control flow (Concept 48) |
| Serializer internals with reflection frames | Reflection-based serialization, or `JsonSerializerOptions` created per call (Concept 47) |

One caveat to name: **EventPipe's sampling profiler has safe-point bias** — managed threads are sampled at points where the runtime can safely inspect them, so time in a tight loop may be attributed to a nearby location. OS-level sampling (ETW, `perf`, `collect-linux`) doesn't have this bias. For "which method," EventPipe is fine; for "which line inside a hot loop," use OS-level sampling or the disassembly.

---

## Concept 24 — Wall-clock and thread-time analysis

Most latency problems in services are **waiting**, not computing: waiting for a database, an HTTP call, a lock, a connection from a pool, a thread from the ThreadPool, or a GC suspension to end. Module 15's starvation signature — low CPU, high latency — is the canonical example: a CPU profile of a starving service shows nothing interesting at all.

Tools for the waiting:

- **Distributed tracing** (Modules 11 and 28) is the macro view: which span on the critical path is long. Always start here for a slow request in a distributed system.
- **PerfView's Thread Time view**, especially **"Thread Time (with StartStop Activities)"**, groups time by request activity and splits it into CPU time and blocked time with the blocking stacks. It answers "this request took 800 ms: 30 ms CPU, 700 ms blocked in `SqlDataReader.Read`, 70 ms waiting to be scheduled."
- **Visual Studio's .NET Async tool and Events viewer** show async operations and their durations.
- **`dotnet-stack`** snapshots every thread's stack — a poor man's wall-clock sampler for hangs and pile-ups.
- **The ThreadPool and Kestrel queue metrics** (Concept 8) measure scheduling delay directly.

**Async makes stacks harder.** After an `await` suspends, the physical stack unwinds; the continuation runs later on a ThreadPool thread whose stack starts in the pool's dispatch loop. A sampling profiler sees the physical stack, not the logical async call chain. Tools have reconstructed logical chains from `Task` events, which are chatty enough to perturb production workloads. .NET 11 adds a lightweight async-profiler event stream — compact per-thread records flushed in batches, plus an anchor frame on the physical stack — that lets profilers join CPU samples to logical async stacks; the .NET team reports under 1% overhead in some measurements, and it covers both runtime-async and classic compiler-generated state machines. Runtime async (Module 15, Concept 16) also makes *live* stacks show the real call chain.

---

## Concept 25 — Allocation and GC profiling

Module 14 (Concepts 56–61) covered memory *diagnostics* — leaks, retention, dumps. Allocation *profiling* is the performance-side question: **what is creating the GC work?**

- The runtime emits a **`GCAllocationTick`** event roughly every 100 KB of allocation (per heap), carrying the type and — with stacks enabled — the call stack. It's sampled, so it's cheap, and over a few seconds of load it gives a statistically accurate "bytes allocated by type and by stack."
- **`dotnet-trace collect --profile gc-verbose`** captures allocation ticks and GC events; PerfView's **"GC Heap Alloc Ignore Free"** view and its **GCStats** report (pause times per GC, generation counts, allocation rate, % time in GC, promoted bytes) are the classic analysis.
- **Visual Studio's .NET Object Allocation Tracking** and **dotMemory** give the same data with friendlier UIs.
- **BenchmarkDotNet's `[MemoryDiagnoser]`** is the micro-scale equivalent.

What to look for, using Module 14's cost model (you pay for survivors, not for garbage):

1. **Top types by bytes and their stacks** — the usual suspects are strings (formatting, concatenation, `Substring`, encoding), `byte[]`/`char[]` buffers, boxed values, closures and delegates, LINQ iterators, and serializer intermediates.
2. **LOH allocations** (≥ 85,000 bytes) — each one is expensive immediately (Module 14, Concept 28). Filter for them specifically.
3. **Survival** — promoted bytes per GC. An allocation profile shows *volume*; pair it with gen1/gen2 promotion data or a `gcdump` diff to find the allocations that actually cost you.
4. **Allocation rate per request** — total allocated bytes divided by requests (Concept 57). A number like "38 KB per request" is the most useful single allocation metric a service can publish.

---

## Concept 26 — Production and continuous profiling

Laboratory profiling reproduces the problem you thought of. Production profiling finds the one you didn't: the tenant with the pathological payload, the endpoint that only gets slow on Mondays, the regression that only shows on one instance type.

The options on Azure and beyond:

- **Application Insights Profiler for .NET** — captures sampled traces from live applications (on a schedule, and on CPU or memory triggers), links them to slow requests, and feeds **Code Optimizations**, an AI-based analysis of CPU and memory hot spots with recommended fixes that can hand context to GitHub Copilot. The profiler packages target the classic Application Insights SDK 2.x; applications on the **OpenTelemetry-based SDK 3.x** use the **Azure Monitor OpenTelemetry Profiler for .NET** instead. On App Service for Windows it's preinstalled; a codeless site-extension enablement path is in beta.
- **`dotnet-monitor` collection rules** — self-hosted, trigger-based capture of traces and dumps, written to storage. Good for AKS and anything not on App Service.
- **Third-party continuous profilers** — Datadog, Grafana Pyroscope, Elastic, Dynatrace and others sample continuously and let you query "CPU by function, for service X, on instance Y, between 14:00 and 14:10."
- **OpenTelemetry** is standardizing a *profiles* signal alongside traces, metrics, and logs; it's maturing and worth watching, not yet something to build on without checking current status.

Operational considerations to raise in a design review:

- **Overhead budget**: sampling profilers typically cost a few percent while active, and triggered or scheduled sessions keep the average far lower.
- **Security**: traces contain method names and sometimes event payloads; **dumps contain everything in memory — secrets, tokens, customer data.** Treat them as sensitive data with access control and retention limits.
- **Symbols and build identity**: you need to map a production binary to its source and symbols; embed source link and keep symbols for every release.

The architect's standard: *"We should be able to answer 'where did CPU go on instance 7 at 14:05 yesterday?' without attempting a repro."*

---

# Part E — Working with the JIT

Module 14 (Concepts 4–7) explained how IL becomes machine code and Module 14 (Concept 53) how deabstraction removes allocations. This part is the performance engineer's view: what the optimizer does, what helps and hinders it, and how to check its work. The theme: **write code the JIT's model understands, then verify with disassembly — never optimize against an imagined JIT.**

## Concept 27 — Tiered compilation as a performance engineer sees it

The model (Module 14, Concept 6): methods start at **tier-0** (fast to compile, barely optimized, optionally instrumented for PGO), are promoted to **tier-1** (fully optimized, using the PGO data) after about 30 calls plus a short delay, with **OSR** moving long-running loops to optimized code mid-execution, and **ReadyToRun** code serving as a precompiled starting tier.

The knobs and what they actually do:

| Setting (MSBuild / environment) | Effect | When it's right |
|---|---|---|
| `TieredCompilation=false` / `DOTNET_TieredCompilation=0` | Every method compiled fully optimized on first call; no instrumentation, no dynamic PGO | Almost never; costs startup *and* steady state (no PGO) |
| `TieredPGO=false` / `DOTNET_TieredPGO=0` | Tiering without instrumentation — tier-1 without profile data | Diagnosing whether PGO causes a behaviour; rarely in production |
| `TieredCompilationQuickJit=false` | Methods without loops still tier, but start optimized | Rarely; hurts startup |
| `TieredCompilationQuickJitForLoops` | Default on since .NET 7 thanks to OSR | Leave it |
| `DOTNET_TC_CallCountThreshold`, `DOTNET_TC_CallCountingDelayMs` | Promotion thresholds | Experiments only |
| `DOTNET_ReadyToRun=0` | Ignore R2R code; JIT everything | Measuring what R2R buys you |

Consequences:

- **Startup pays for tier-0 compilation plus call counting**, and early requests run slow code. Post-deployment p99 spikes are frequently just this (Concept 63).
- **`[MethodImpl(MethodImplOptions.AggressiveOptimization)]` is a trap.** It skips tier-0, so the method is compiled optimized immediately — but **without instrumentation, hence without dynamic PGO** — and it isn't served from ReadyToRun either, so it's always jitted at startup. It was a workaround for loop-heavy methods before OSR existed; in modern .NET it usually makes steady-state code *slower*. If you see it in a codebase, ask for the benchmark that justified it.
- **Hot ReadyToRun code is re-jitted** — since .NET 8 hot R2R methods go through an instrumented tier before tier-1, so they get PGO too.
- **Hand-rolled benchmarks must run long enough** to reach tier-1 and let background compilation finish; BDN handles this (Concept 14).

Observing it: `DOTNET_JitDisasmSummary=1` (with `DOTNET_JitStdOutFile=path`) lists every method the JIT compiles with its tier — `Tier0`, `Instrumented Tier0`, `Tier1`, `OSR` — which makes tiering visible in a way nothing else does. The JIT also publishes metrics (compilation time, compiled methods and IL bytes) through the runtime meter.

---

## Concept 28 — Dynamic PGO and guarded devirtualization

Dynamic PGO (on by default since .NET 8) is the reason idiomatic, interface-heavy C# performs far better than its IL suggests. Tier-0 code carries probes: **class probes** record which concrete types appear at virtual and interface call sites, delegate probes record delegate targets, and block counts record which branches are taken. Tier-1 uses them:

- **Guarded devirtualization (GDV)** — if a call site saw `Dog` almost every time, the JIT emits `if (obj.GetType() == typeof(Dog)) { /* direct, inlinable Dog.Speak */ } else { /* original virtual call */ }`. The common case becomes a direct call that can be inlined and optimized with its caller; correctness is preserved by the fallback.
- **Hot/cold block layout** — rarely executed blocks move out of line, so the hot path is straight-line code with fewer taken branches and better instruction-cache use.
- **Better inlining decisions** — hot call sites get more inlining budget.
- **Casting and delegate optimizations** — the same guess-and-verify applied to type checks and delegate invocations.

What this means for the code you write and the benchmarks you trust:

1. **Monomorphic interface call sites are nearly free after PGO; megamorphic ones aren't.** An `IValidator` call site that sees one implementation in production is effectively a direct call; one that sees twelve is a real interface dispatch. Abstraction is cheap where it is used uniformly.
2. **Benchmarks must reproduce production's type distribution** (Concept 15). A benchmark that exercises one `IPricingRule` implementation will be devirtualized; the production call site with seven rules won't be.
3. **PGO data is per process and is lost on restart.** Every new pod relearns its profile during warm-up, which is part of why post-deployment latency is worse and why scale-out instances are briefly slower.
4. **Don't hand-devirtualize prematurely.** Writing `if (x is Dog d) d.Speak(); else x.Speak();` duplicates what PGO does, and it goes stale when the distribution changes.
5. **Native AOT has no dynamic PGO** — it compensates with whole-program analysis (Concept 55).

---

## Concept 29 — Inlining: the optimization that enables the others

Inlining replaces a call with the callee's body. The saved call overhead is the small benefit; the large one is that **the callee's code is now optimized together with the caller's** — constants propagate into it, bounds checks become provable, allocations are shown not to escape, and further calls become devirtualizable. A chain of small abstractions that each look expensive can collapse into a handful of instructions.

The JIT's heuristics weigh the callee's IL size and content, the call site's hotness (with PGO), constant arguments that would simplify the body, and the caller's growing size. Very small methods are always inlined; larger ones only when the heuristics predict a payoff.

**What blocks inlining:**

- The target is **virtual or interface** and wasn't devirtualized (Concepts 28 and 30).
- The callee is **too large**, or the caller has already used its budget.
- The callee contains **exception-handling constructs** (`try/catch`, filters) — progressively relaxed across releases (.NET 10 can inline some methods containing `try/finally`) but still a common blocker.
- The callee uses **`stackalloc`**, is **recursive**, or is marked `[MethodImpl(MethodImplOptions.NoInlining)]`.
- In **ReadyToRun** code, callees across a version-bubble boundary (Concept 34).

**The throw-helper pattern** keeps hot methods small and inlinable by moving the cold, code-heavy throw path out:

```csharp
public T this[int index]
{
    get
    {
        if ((uint)index >= (uint)_count) ThrowIndexOutOfRange();   // tiny, cold call
        return _items[index];
    }
}

[DoesNotReturn]
private static void ThrowIndexOutOfRange() =>
    throw new ArgumentOutOfRangeException("index");                // string formatting and allocation live here
```

The BCL's `ArgumentNullException.ThrowIfNull`, `ArgumentOutOfRangeException.ThrowIfNegative`, and friends are exactly this pattern — use them.

**`[MethodImpl(MethodImplOptions.AggressiveInlining)]`** overrides most size heuristics. Use it sparingly, only with a benchmark: it grows callers, which costs instruction cache and can push a caller past the size at which the JIT optimizes it well. In library code on genuinely hot small methods it's reasonable; sprinkled across application code it's cargo cult.

---

## Concept 30 — Devirtualization and sealing

A virtual or interface call becomes a direct call when the JIT knows the exact type:

- the object was just created with `new` in the same (inlined) scope;
- the static type is a **sealed** class, or the method is sealed;
- the value comes from a `static readonly` field whose runtime type tier-1 can observe;
- the type is a **value type in generic code** — each value-type instantiation is specialized, so constrained calls are direct (Module 14, Concepts 5 and 15);
- on .NET 11, **`Activator.CreateInstance<T>()`** results are treated as exact types, and **generic virtual methods** — both non-shared and shared, plus default interface methods on generic interfaces — can now be devirtualized;
- in **Native AOT**, whole-program analysis proves there is only one implementation.

Otherwise it relies on PGO's guarded devirtualization (Concept 28).

**Seal classes by default.** It costs nothing, gives the JIT exact-type knowledge (cheaper type checks, direct calls, better inlining), and is a design statement — "this type wasn't designed for inheritance." Analyzer **CA1852** flags internal types that can be sealed; many teams enable it as a warning. For public library types, sealing is a compatibility decision (unsealing later is safe; sealing later is breaking), which makes "sealed unless designed for extension" the right default there too.

---

## Concept 31 — Bounds-check elimination

C# guarantees that array, string, and span accesses are in bounds. The JIT implements the guarantee with a compare-and-branch before each access — and eliminates it whenever it can **prove** the index is in range. A remaining check shows up in disassembly as a call to `CORINFO_HELP_RNGCHKFAIL` on the failure path.

**Patterns the JIT has long handled:**

```csharp
for (int i = 0; i < array.Length; i++) sum += array[i];   // canonical: no check
foreach (int x in span) sum += x;                          // no check
if ((uint)i < (uint)span.Length) return span[i];           // the unsigned-compare idiom: one check covers both bounds
```

**Patterns that historically defeated it** — each of which recent JITs increasingly handle, and each of which you should *measure* rather than work around:

- A loop bound that isn't the array's length (`i < count`) — handled by **loop cloning**: one guard up front, a check-free fast loop, and the original as fallback.
- Two arrays indexed by the same variable — slicing the second to the first's length (`b = b[..a.Length]`) lets the JIT prove both.
- Derived indices (`i + 1`, `i - 1`, lookahead in parsers), index-from-end (`^1`), and computed table indices.

**.NET 11 closed a long list of such gaps**: tightening ranges from `!= constant` checks (which is what C# list patterns compile to), bounds derived from bitwise OR of masked values (table lookups in decoders), known result ranges of leading/trailing-zero-count and population-count instructions, assertions established earlier in the same basic block, coalescing several constant-index checks into one (`a[0] + a[1] + a[2] + a[3]` now needs a single check), loop cloning for `!=`-terminated loops, lookahead guards such as `(uint)(i + 2) < (uint)span.Length`, reading a fixed-size value from the end of a span, repeated slicing in vectorized loops, and removing redundant `checked` overflow tests and narrowing-cast checks when ranges are already proven.

**The practical guidance** — worth quoting nearly verbatim from the .NET team, because it reverses a decade of folklore: much of this work came from auditing *unsafe* code in the BCL that existed only to avoid bounds checks; often the safe rewrite was as fast or faster, and where it wasn't, the JIT was improved and the library rewritten in plain, bounds-checked C#. If you own hand-written pointer or `Unsafe.Add` loops whose only purpose is avoiding bounds checks, **rewrite them safely and measure on .NET 11** — the gap is usually gone or too small to justify the risk (Concept 42).

---

## Concept 32 — Structs, promotion, and register allocation

Registers are the fastest storage there is, and the JIT's job is to keep hot values in them. For structs, that means **promotion**: treating a struct local's fields as independent scalar locals that can each live in a register, so the struct never exists in memory at all. .NET 8's **physical promotion** generalized the older, narrower rules to many more struct shapes, which is part of why small value types — `Span<T>`, `ValueTuple`, your `Money` and `OrderId` — cost nothing in hot code.

What prevents it:

- **Address exposure.** Taking a local's address — passing it by `ref` or `in` to a method that isn't inlined, or using `&` — forces it into memory. (Several .NET 11 improvements are about the JIT *not* marking locals exposed unnecessarily.)
- **Large structs.** Beyond a size threshold, copies become `memcpy`-like moves and promotion is abandoned. Keep structs used in hot loops small — tens of bytes, not hundreds.
- **Non-readonly structs passed by `in`** — defensive copies (Module 14, Concept 16). Make hot structs `readonly`.

**Zero-initialization.** The runtime zeroes locals and `stackalloc` buffers by default. `[SkipLocalsInit]` (assembly, type, or method level; requires `AllowUnsafeBlocks`) removes the zeroing, which matters for large `stackalloc` buffers in hot paths. The obligation it creates is absolute: **never read a buffer before writing it**, or you read stale stack memory — a correctness and potential information-disclosure bug. `Unsafe.SkipInit(out T)` is the per-variable equivalent.

---

## Concept 33 — Reading the JIT's output

You don't need to write assembly to benefit from reading it. Four ways in:

1. **`[DisassemblyDiagnoser(maxDepth: 2)]`** in BenchmarkDotNet — the tier-1 code actually benchmarked, with callees; `exportDiff: true` diffs jobs (for example, .NET 10 vs 11).
2. **`DOTNET_JitDisasm="Namespace.Type:Method"`** — works on the shipping release runtime (since .NET 7), prints each tier's code for matching methods as they're compiled; combine with `DOTNET_JitStdOutFile=path` and `DOTNET_JitDisasmSummary=1` (Concept 27).
3. **sharplab.io** — paste code, see lowered C#, IL, and JIT assembly instantly. Fast for questions like "what does this pattern compile to?"; it doesn't reflect your application's PGO profile.
4. **godbolt.org** (Compiler Explorer) — supports C# with several .NET versions side by side.

What to look for — mostly helper calls and memory traffic, not instruction counts:

| In the disassembly | Meaning |
|---|---|
| `CORINFO_HELP_RNGCHKFAIL` | A bounds check that wasn't eliminated |
| `CORINFO_HELP_NEWSFAST`, `CORINFO_HELP_NEWARR_1_*` | Heap allocation (an object or array) |
| `CORINFO_HELP_BOX*` | Boxing |
| `CORINFO_HELP_CHKCAST*`, `CORINFO_HELP_ISINST*` | Type checks the JIT couldn't prove |
| `CORINFO_HELP_ASSIGN_REF`, `CORINFO_HELP_CHECKED_ASSIGN_REF` | GC write barrier on a reference store (Module 14, Concept 26) |
| `CORINFO_HELP_OVERFLOW` | A `checked` arithmetic test |
| `call [rax+…]`, `call qword ptr [...]` | Indirect (virtual or interface) calls — not devirtualized |
| `mov [rsp+…]` / `mov …, [rsp+…]` around a loop | Register spills |
| `vmovdqu`, `vpcmpeqb`, `ymm`/`zmm` registers (x64) or `ld1`, `cmeq`, `v` registers (Arm64) | Vectorized code |
| "Total bytes of code" | Code size — relevant to inlining and instruction cache |

A caution: fewer instructions is not the same as faster. Latency, dependency chains, memory access, and execution-port pressure decide speed. Disassembly explains a benchmark result; it doesn't replace one.

---

## Concept 34 — ReadyToRun in depth

Module 14 (Concepts 7 and 64) positioned ReadyToRun as the middle option between JIT and Native AOT. The engineering details:

- **What it is**: `crossgen2` precompiles IL to native code at publish time (`<PublishReadyToRun>true</PublishReadyToRun>` with a runtime identifier) and stores it alongside the IL in the same assembly. At runtime, that code is used instead of tier-0; hot methods are still promoted to tier-1 with PGO.
- **Why its code is lower quality**: R2R code must stay valid when *other* assemblies are serviced, so it goes through indirections (fixups) for cross-assembly references and **won't inline across a version-bubble boundary**. Generic instantiations over types from other assemblies may not be precompiled and fall back to the JIT.
- **Composite R2R** (`<PublishReadyToRunComposite>true</PublishReadyToRunComposite>`, self-contained) compiles the framework and the application into one image with a single version bubble: better cross-assembly inlining, faster startup, larger output, and you must redeploy to pick up runtime servicing.
- **Size**: R2R assemblies are substantially larger than IL-only ones, which matters for container image size and pull time (Concept 63).
- **.NET 11 changes worth knowing**: the R2R target instruction set moved to **x86-64-v3** (AVX2, BMI1/2, FMA, LZCNT, MOVBE) on Windows and Linux and to `armv8.2-a + RCPC` on Windows Arm64, so R2R code is better on modern hardware — but on supported CPUs lacking those features the precompiled code can't be used and methods fall back to the JIT, costing startup. And `Comparer<T>.Default`/`EqualityComparer<T>.Default` are now specialized in R2R images instead of falling back to the JIT, with the .NET team reporting up to 20× improvements for comparer-heavy collection operations.
- **When to use it**: anything whose startup is visible — containers that restart on every deployment and scale-out, serverless functions, CLI tools, desktop applications. It is close to free and requires no code changes; the only reason not to is image size. Measure time-to-first-request with and without (`DOTNET_ReadyToRun=0` disables it at runtime for an A/B test).
- **Advanced**: partial R2R guided by a profile (so only startup-relevant methods are precompiled) exists via `dotnet-pgo` and MIBC profiles — the BCL itself ships this way. Rarely worth it for applications.

---

# Part F — `Span<T>`, `Memory<T>`, and the buffer ecosystem

Module 14 (Concepts 19–20) explained what a span *is* at runtime; Module 16 (Concepts 52–56) explained the language features around it. This part treats the span family as what it has become: **a complete, zero-copy I/O and text-processing ecosystem with an ownership model** — and, above all, an API-design discipline.

## Concept 35 — The span family as one abstraction

Before spans, "a sequence of contiguous elements" had half a dozen representations — `T[]`, `string`, an `(array, offset, count)` triple, `ArraySegment<T>`, a pointer and a length, a `stackalloc` buffer — and every API had to pick one or be written several times. `Span<T>` and `ReadOnlySpan<T>` unify them: **one view type over contiguous memory of any provenance**.

```csharp
static int CountDigits(ReadOnlySpan<char> text)
{
    int n = 0;
    foreach (char c in text) if (char.IsAsciiDigit(c)) n++;
    return n;
}

CountDigits("order-12345");                 // a string — no copy
CountDigits(chars);                         // a char[]
CountDigits(chars.AsSpan(4, 6));            // a slice of an array — no copy
Span<char> scratch = stackalloc char[32];   // stack memory
int written = FormatSomething(scratch);
CountDigits(scratch[..written]);
unsafe { CountDigits(new ReadOnlySpan<char>(nativePtr, nativeLength)); }  // native memory
```

The mental model to hold: **a span is a view, not a container.** Slicing creates a new view (a reference and a length) at zero cost; the underlying memory is never copied. Indexing is bounds-checked (and the checks are usually eliminated — Concept 31). The `ref struct` restrictions — no boxing, no heap fields, no capture, no crossing an `await` — exist to guarantee the view never outlives the memory it points to (Module 14, Concept 19; Module 16, Concept 52). When you need to keep a view across an `await` or store it in a field, you need `Memory<T>` (Concept 37).

---

## Concept 36 — The span API surface and `MemoryExtensions`

Most of the value of spans comes from the APIs that accept them — almost all vectorized internally, almost all allocation-free:

```csharp
ReadOnlySpan<char> line = "GET /api/orders?id=42 HTTP/1.1";

int sp = line.IndexOf(' ');
ReadOnlySpan<char> method = line[..sp];
ReadOnlySpan<char> rest   = line[(sp + 1)..];
ReadOnlySpan<char> target = rest[..rest.IndexOf(' ')];
int q = target.IndexOf('?');
ReadOnlySpan<char> path = q >= 0 ? target[..q] : target;

bool isGet = method is "GET";                          // constant-pattern match on a span (C# 11)
bool api   = path.StartsWith("/api/", StringComparison.Ordinal);

foreach (Range r in "a,b,,c".AsSpan().Split(','))      // .NET 9: splitting without substrings
    Process("a,b,,c".AsSpan()[r]);
```

The families to know:

- **Searching** — `IndexOf`, `LastIndexOf`, `IndexOfAny`, `IndexOfAnyExcept`, `IndexOfAnyInRange`, `ContainsAny`, `Count`, `CommonPrefixLength`, and the `SearchValues<T>` overloads (Concept 44).
- **Comparing** — `SequenceEqual`, `SequenceCompareTo`, `StartsWith`/`EndsWith` (always pass `StringComparison.Ordinal` or `OrdinalIgnoreCase` for text), `Equals(other, comparison)`.
- **Transforming in place** — `Reverse`, `Sort` (with keys or a comparison), `Fill`, `Clear`, `Replace`, `CopyTo`/`TryCopyTo`, `Trim` and its variants.
- **Splitting** — the .NET 8 `Split(Span<Range> destination, char separator)` overload that writes ranges into a caller-provided buffer, and the .NET 9 enumerators.
- **Binary data** — `BinaryPrimitives.ReadInt32BigEndian`, `WriteUInt64LittleEndian`, and friends: endian-correct reads and writes on byte spans, the building block of every binary protocol.
- **Parsing and formatting** — `int.TryParse(ReadOnlySpan<char>, …)`, `Guid.TryParse`, `DateTimeOffset.TryParseExact` over spans; the generic `ISpanParsable<T>`/`IUtf8SpanParsable<T>`; `TryFormat` on every primitive (Concept 39).
- **Encoding** — `Encoding.UTF8.GetBytes(ReadOnlySpan<char>, Span<byte>)`, `Utf8.FromUtf16`/`ToUtf16` (returning `OperationStatus`), `Convert.TryToBase64Chars`, `Base64Url` (.NET 9), `Convert.FromHexString(ReadOnlySpan<char>)`.

The habit worth building: **before writing a loop over characters or bytes, look for the `MemoryExtensions` method that does it.** It will be vectorized, bounds-check-free internally, and maintained by people who benchmark it on every hardware generation.

---

## Concept 37 — `Memory<T>` ownership rules

`Memory<T>` (and `ReadOnlyMemory<T>`) is the heap-storable counterpart of `Span<T>`: an ordinary struct holding an object (array, string, or `MemoryManager<T>`) plus an offset and length, from which you get a span with `.Span` at the point of use. It can cross `await`s and live in fields. That freedom comes with a question spans never raise: **who owns the underlying memory, and for how long is my view valid?**

Microsoft's *Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines* (in the resources) state ten rules. They condense to four ideas:

1. **Synchronous APIs take spans.** If the method doesn't need to store the buffer or use it across an `await`, take `Span<T>`/`ReadOnlySpan<T>`. Take the read-only form whenever you don't write.
2. **A `Memory<T>` parameter is a borrow for the duration of the call** — or, if the method returns a `Task`, until the task completes. After that, the callee must not touch it.
3. **`IMemoryOwner<T>` is ownership.** Whoever holds it must dispose it exactly once or transfer it; an API that accepts an `IMemoryOwner<T>` is taking ownership. (`MemoryPool<T>.Shared.Rent(minimumLength)` returns one; its `Memory` may be *longer* than requested — slice it.)
4. **Storing a `Memory<T>` — in a field, via a constructor or a settable property — is a long-lived borrow.** Document the lifetime contract, because the type system won't enforce it.

The bug these rules prevent is a use-after-free with a managed accent:

```csharp
// BUG: stores a borrowed view past the end of the call
public sealed class TelemetryBatcher
{
    private readonly List<ReadOnlyMemory<byte>> _pending = [];
    public void Add(ReadOnlyMemory<byte> payload) => _pending.Add(payload);   // retains the borrow
}

// A caller doing the "right thing" with a pooled buffer:
byte[] rented = ArrayPool<byte>.Shared.Rent(size);
int n = Serialize(evt, rented);
batcher.Add(rented.AsMemory(0, n));
ArrayPool<byte>.Shared.Return(rented);     // the batcher now points at a buffer someone else will overwrite
```

Nothing crashes. Some time later, a different request rents the same array, and the batch is flushed containing another request's bytes. It looks like a concurrency bug and is found, if ever, by fuzzing or by accident. Fixes, in order of preference: take ownership explicitly (`Add(IMemoryOwner<byte> payload)`, disposed after flush); copy on entry (`payload.ToArray()`) and say so; or change the design so the batcher serializes into its own buffer.

---

## Concept 38 — Discontiguous data: `ReadOnlySequence<T>` and `SequenceReader<T>`

Network data doesn't arrive in one contiguous block. A message can span two or three pooled buffers. **`ReadOnlySequence<T>`** represents a logical sequence as a linked list of segments; it's what `PipeReader` hands you (Concept 40), and `Utf8JsonReader` accepts it directly.

Its API mirrors spans where it can — `Length`, `Slice`, `PositionOf`, `CopyTo` — and adds `IsSingleSegment` and `FirstSpan`. **`SequenceReader<T>`** (a `ref struct`) reads through the segments without you managing boundaries:

```csharp
static bool TryReadLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> line)
{
    var reader = new SequenceReader<byte>(buffer);
    if (reader.TryReadTo(out line, (byte)'\n'))    // handles a delimiter in any segment
    {
        buffer = buffer.Slice(reader.Position);    // everything after the delimiter
        return true;
    }
    line = default;
    return false;
}

static void HandleLine(ReadOnlySequence<byte> line)
{
    if (line.IsSingleSegment)
    {
        Parse(line.FirstSpan);                     // the common, fast path: no copy
        return;
    }
    // The rare case: the line straddles segments. Copy to contiguous scratch memory.
    int len = checked((int)line.Length);
    byte[]? rented = null;
    Span<byte> scratch = len <= 512 ? stackalloc byte[512] : (rented = ArrayPool<byte>.Shared.Rent(len));
    try   { line.CopyTo(scratch); Parse(scratch[..len]); }
    finally { if (rented is not null) ArrayPool<byte>.Shared.Return(rented); }
}
```

The pattern — **fast-path the single segment, copy only when a value straddles a boundary** — is how Kestrel, the Redis clients, and the gRPC stack parse network data. `SequenceReader<T>` also offers `TryRead`, `TryReadLittleEndian`/`TryReadBigEndian`, `IsNext`, `Advance`, and `TryReadExact`.

---

## Concept 39 — Writing output: `IBufferWriter<T>` and the Try pattern

The allocation-free way to *produce* data is to **let the caller supply the memory**. Two complementary shapes:

**The Try pattern** — the caller passes a destination span; the method reports how much it wrote, or that it didn't fit:

```csharp
public readonly record struct Money(decimal Amount, string Currency) : ISpanFormattable, IUtf8SpanFormattable
{
    public bool TryFormat(Span<char> destination, out int charsWritten,
                          ReadOnlySpan<char> format, IFormatProvider? provider) =>
        destination.TryWrite(provider ?? CultureInfo.InvariantCulture,
                             $"{Amount:F2} {Currency}", out charsWritten);

    public bool TryFormat(Span<byte> utf8Destination, out int bytesWritten,
                          ReadOnlySpan<char> format, IFormatProvider? provider) =>
        Utf8.TryWrite(utf8Destination, provider ?? CultureInfo.InvariantCulture,
                      $"{Amount:F2} {Currency}", out bytesWritten);

    public string ToString(string? format, IFormatProvider? provider) =>
        string.Create(provider ?? CultureInfo.InvariantCulture, $"{Amount:F2} {Currency}");

    public override string ToString() => ToString(null, null);
}
```

Implementing `ISpanFormattable` and `IUtf8SpanFormattable` means your type formats directly into interpolated strings, `StringBuilder`, `Utf8.TryWrite`, and UTF-8 writers **without an intermediate string** — the framework checks for these interfaces. The interpolation handlers here format straight into the destination.

**`IBufferWriter<T>`** — when the output size isn't known in advance, the writer supplies memory on request:

```csharp
static void WriteMoney(IBufferWriter<byte> writer, Money m)
{
    Span<byte> span = writer.GetSpan(64);               // at least 64 bytes (possibly more)
    if (!m.TryFormat(span, out int written, default, null))
    {
        span = writer.GetSpan(1024);                   // rare: ask for more and retry
        if (!m.TryFormat(span, out written, default, null)) throw new InvalidOperationException();
    }
    writer.Advance(written);                          // commit exactly what was written
}
```

Implementations include `ArrayBufferWriter<T>` (a growable buffer), `PipeWriter` (Concept 40), and anything you write yourself on pooled memory; `Utf8JsonWriter` writes into an `IBufferWriter<byte>`.

**`OperationStatus`** is the richer form of the Try pattern used by streaming transformations — `Base64.EncodeToUtf8`, `Utf8.FromUtf16`, and similar — returning `Done`, `DestinationTooSmall`, `NeedMoreData`, or `InvalidData`, with separate counts for input consumed and output written. It's the right shape whenever input may arrive in chunks.

---

## Concept 40 — `System.IO.Pipelines`

`Stream` makes the caller manage buffers: allocate one, read into it, handle messages that straddle reads, grow it for large messages, and copy partial data around. Every protocol parser written on `Stream` reimplements this, usually with bugs. **Pipelines** moves that work into the library:

- A **`Pipe`** has a `PipeWriter` end and a `PipeReader` end. The writer fills pooled segments; the reader sees the unconsumed data as a `ReadOnlySequence<byte>`.
- **Backpressure** is built in: when unconsumed data exceeds `PauseWriterThreshold`, the writer's `FlushAsync` waits until the reader drains below `ResumeWriterThreshold`. A slow consumer slows the producer instead of growing memory without bound (Module 15's bounded-channel reasoning, applied to bytes).
- **`AdvanceTo(consumed, examined)`** is the concept that makes it work. *Consumed* tells the pipe which bytes can be released; *examined* tells it how far you've looked. If you've examined everything and consumed nothing (an incomplete message), the next `ReadAsync` waits for **more** data rather than returning the same bytes again.

```csharp
async Task ProcessAsync(PipeReader reader, CancellationToken ct)
{
    try
    {
        while (true)
        {
            ReadResult result = await reader.ReadAsync(ct);
            ReadOnlySequence<byte> buffer = result.Buffer;

            while (TryReadLine(ref buffer, out ReadOnlySequence<byte> line))
                HandleLine(line);                 // must not retain 'line' after AdvanceTo

            reader.AdvanceTo(buffer.Start, buffer.End);   // consumed the complete lines; examined all
            if (result.IsCompleted) break;
        }
    }
    finally { await reader.CompleteAsync(); }
}

// Adapting an existing stream: PipeReader.Create(networkStream), PipeWriter.Create(networkStream)
```

Kestrel is built on pipelines, which is why ASP.NET Core's request parsing allocates so little. Use them for **custom TCP protocols, high-throughput stream parsing, and proxies**; for ordinary request/response code, `Stream` and the framework's abstractions are fine. The one rule that bites: **data in the `ReadResult` buffer is valid only until the next `AdvanceTo`** — the same borrow semantics as Concept 37.

---

## Concept 41 — Designing span-based APIs

The API-design guidance the BCL follows, which is what an interviewer means when they ask how you'd design a high-performance library surface:

1. **Inputs: `ReadOnlySpan<T>`** for synchronous methods. It accepts arrays, strings (for `char`), slices, and stack memory with one overload — and since C# 14's first-class spans, extension methods on `ReadOnlySpan<T>` apply to arrays too (Module 16, Concept 54).
2. **Outputs: a caller-provided `Span<T>` destination plus a written count**, as `bool TryX(ReadOnlySpan<byte> source, Span<byte> destination, out int bytesWritten)` — or `OperationStatus` for streaming. Use `IBufferWriter<T>` when the size is unknowable up front.
3. **`Memory<T>`/`ReadOnlyMemory<T>` only for async methods or APIs that retain the buffer**, with the lifetime documented (Concept 37). Async methods cannot take spans.
4. **Offer two tiers.** An easy, allocating convenience API (`string Format(Money m)`) layered on the efficient core (`bool TryFormat(Money m, Span<char> destination, out int written)`). The BCL does this everywhere: `int.Parse(string)` over `int.TryParse(ReadOnlySpan<char>, …)`; `Convert.ToBase64String` over `Base64.EncodeToUtf8`.
5. **Implement the standard interfaces** — `ISpanFormattable`, `IUtf8SpanFormattable`, `ISpanParsable<T>`, `IUtf8SpanParsable<T>` — so your types plug into the framework's allocation-free paths.
6. **Generic over ref structs where appropriate**: `where T : allows ref struct` (Module 16, Concept 53) lets a generic API accept spans.
7. **Use `scoped`** on ref-struct parameters that don't escape, so callers can pass stack memory.
8. **Never expose pooled arrays** from a public API; return owned data or `IMemoryOwner<T>`.
9. **Keep spans out of domain models and service contracts.** Spans are for parsers, serializers, protocol handlers, formatters, and hot I/O paths. A `ReadOnlySpan<char>` property on a domain entity is the wrong tool — it can't even exist there.

---

## Concept 42 — Escape hatches: `stackalloc`, `[InlineArray]`, `Unsafe`, `MemoryMarshal`

The tools below turn off guardrails. Each has a **contract** — the conditions under which it's correct — and using one without stating its contract in a comment is a review failure.

**`stackalloc`.** Stack memory is free to allocate and release, but the stack is small (~1 MB by default on the main thread, less in some environments) and **stack overflow cannot be caught** — it terminates the process. Contract: **the size is bounded by a constant you chose**, typically a few hundred bytes to ~1 KB, with a pooled fallback above it (Module 14, Concept 49's idiom, used in Concept 38 above). Never `stackalloc` a size derived from user input without a bound. Combine with `[SkipLocalsInit]` only if you never read before writing (Concept 32).

**`[InlineArray(N)]`** (C# 12 / .NET 8) declares a struct containing *N* contiguous elements — a fixed-size buffer usable in safe code, convertible to a span:

```csharp
[InlineArray(8)]
public struct Buffer8<T> { private T _element0; }

var small = new Buffer8<int>();
Span<int> s = small;       // implicit conversion
s[0] = 42;                 // bounds-checked
```

Useful for small fixed-capacity collections embedded in structs (the "small-vector" optimization) and for the compiler's own `params ReadOnlySpan<T>` lowering (Module 16, Concept 55).

**`CollectionsMarshal`** — access to collection internals:
- `AsSpan(List<T>)` — iterate or modify a list's backing store without the enumerator. Contract: **don't add or remove items while holding the span** (it views the old array after a resize).
- `GetValueRefOrAddDefault(dictionary, key, out bool exists)` — a single lookup for get-or-add, returning a `ref` to the value slot. Contract: don't modify the dictionary while holding the ref.
- `SetCount(List<T>, n)` (.NET 8) — set a list's count directly after filling its span.

**`MemoryMarshal`** — reinterpretation and raw access: `Cast<TFrom, TTo>` (reinterpret a span of one unmanaged type as another), `AsBytes`, `GetReference` (a `ref` to element 0 with no bounds check), `CreateSpan` (a span from a `ref` and a length — unchecked), `Read<T>`/`Write<T>`. Contracts: alignment, endianness, and — for anything producing an unchecked `ref` — that you've proven the bounds yourself.

**`Unsafe`** — `Unsafe.Add`, `Unsafe.As`, `Unsafe.ReadUnaligned`: fully unchecked. Contract: everything.

**`GC.AllocateUninitializedArray<T>(n)`** skips zeroing (worth it for large buffers you'll overwrite entirely); **`GC.AllocateArray<T>(n, pinned: true)`** allocates on the Pinned Object Heap for long-lived I/O buffers (Module 14, Concept 29).

**The direction of the platform matters here.** .NET 11's JIT work was driven largely by auditing unsafe code in the BCL and replacing it with safe code at equal or better speed (Concept 31), and C# 15's memory-safety preview begins turning `unsafe` into an auditable, caller-visible contract (Module 16, Concept 52). The architect's policy: **unsafe code requires a benchmark proving the safe version is too slow on the current runtime, a comment stating the contract, and tests (ideally fuzzing) that exercise the boundaries.** Re-evaluate it on every runtime upgrade; much of it will become deletable.

---

# Part G — Throughput techniques in the BCL and in your code

Most throughput wins in real .NET services come from using the platform's already-optimized primitives correctly rather than from writing clever code. This part is the catalogue — organized by the kind of work, each with the trap that makes it slow and the primitive that makes it fast.

## Concept 43 — Text: UTF-8 first

Networks, files, and databases speak bytes, and the web speaks UTF-8. .NET strings are UTF-16. Every `string` you create from incoming bytes is a **transcode plus an allocation**, and every response you build as a string is another transcode on the way out. The fastest text processing is the processing that **stays in UTF-8 bytes from input to output**:

- **UTF-8 literals** — `"Content-Type"u8` is a `ReadOnlySpan<byte>` pointing at constant data in the assembly; no allocation, no transcoding (Module 16, Concept 58).
- **`IUtf8SpanFormattable`** (.NET 8) on the primitives — `int`, `long`, `double`, `Guid`, `DateTime`, `DateTimeOffset`, … — and **`Utf8.TryWrite`** with interpolation format values straight into a UTF-8 buffer.
- **`Utf8Parser`/`Utf8Formatter`** (`System.Buffers.Text`) and **`IUtf8SpanParsable<T>`** parse numbers and dates from bytes without a string.
- **The `Ascii` class** (.NET 8) — `Ascii.IsValid`, `Ascii.EqualsIgnoreCase`, `Ascii.ToUpper` — vectorized operations for protocol text that is ASCII by specification (HTTP header names, method tokens).
- **`Base64Url`** (.NET 9) for tokens and IDs, span-based.

When you do need strings:

- **`string.Create(length, state, (span, state) => …)`** builds a string with exactly one allocation.
- **`CompositeFormat.Parse`** (.NET 8) pre-parses a format string used repeatedly with `string.Format`, avoiding re-parsing on every call.
- **`StringBuilder` with a capacity estimate**, reused where appropriate (`ObjectPool<StringBuilder>`), and appended to via interpolation (which formats directly into the builder).
- **Always specify ordinal comparison** for identifiers, keys, protocol tokens, and file paths: `StringComparison.Ordinal`/`OrdinalIgnoreCase`, `StringComparer.Ordinal`. Culture-aware comparison is dramatically slower and, for identifiers, usually *wrong* (the Turkish-I problem). A `Dictionary<string, T>` created without a comparer uses ordinal comparison — but `string.StartsWith(string)` without a `StringComparison` is culture-sensitive, and analyzers CA1307/CA1309/CA1310 exist to catch it.
- **`<InvariantGlobalization>true</InvariantGlobalization>`** for services that don't need culture-specific behaviour removes the ICU dependency, speeds up culture-related operations, and shrinks images (Concept 54) — but it's a behaviour change; confirm that nothing formats for end users with culture data.

---

## Concept 44 — Searching and matching: `SearchValues<T>` and regular expressions

**`SearchValues<T>`** (.NET 8 for `char` and `byte`; .NET 9 added `SearchValues<string>`) precomputes an optimal search strategy for a fixed set of values — bitmaps, vectorized range checks, or, for multiple strings, multi-substring algorithms — and then runs vectorized:

```csharp
private static readonly SearchValues<char> s_delimiters = SearchValues.Create(" \t,;|");
private static readonly SearchValues<string> s_secretMarkers =
    SearchValues.Create(["password=", "secret=", "api_key="], StringComparison.OrdinalIgnoreCase);

int i = text.IndexOfAny(s_delimiters);
bool leaks = logLine.AsSpan().ContainsAny(s_secretMarkers);
```

Two rules: **create it once and store it in a `static readonly` field** (creation is the expensive part), and use it anywhere you'd have written `IndexOfAny(char[])`, a loop comparing against a set of characters, or several consecutive `Contains` calls.

**Regular expressions** have three performance modes and one safety mode:

| Mode | Cost profile | When |
|---|---|---|
| Interpreted (default `new Regex(pattern)`) | Cheap to construct, slowest to match | Patterns known only at runtime, used rarely |
| `RegexOptions.Compiled` | Expensive construction (emits IL at runtime), fast matching; not available as IL emission under Native AOT | Legacy — prefer the source generator |
| **`[GeneratedRegex]` source generator** | Zero runtime construction cost, fast matching, readable and debuggable generated code, trimming- and AOT-friendly | **The default for any pattern known at compile time** |
| `RegexOptions.NonBacktracking` | Linear-time matching guaranteed; some features unsupported | **Untrusted patterns or inputs** — the ReDoS defence |

```csharp
public static partial class Patterns
{
    [GeneratedRegex(@"^ord_[0-9a-f]{12}$", RegexOptions.CultureInvariant, matchTimeoutMilliseconds: 100)]
    public static partial Regex OrderId();
}

bool ok = Patterns.OrderId().IsMatch(input);
foreach (ValueMatch m in Patterns.OrderId().EnumerateMatches(text)) { /* m.Index, m.Length — no Match objects */ }
```

The classic regex performance bug is **constructing a `Regex` per call** (per request, per line) — construction includes parsing and, for `Compiled`, IL emission and JIT. It shows up in CPU profiles as regex parser frames and in memory as dynamic-method growth (Module 14, Concept 8). The static `Regex.IsMatch(input, pattern)` methods use a small internal cache, which hides the problem until the number of distinct patterns exceeds it. Also: `EnumerateMatches` (returning `ValueMatch`) and `EnumerateSplits` (.NET 9) avoid allocating `Match` objects and substrings. And sometimes the fastest regex is no regex — `StartsWith`, `IndexOf`, and `SearchValues` cover a surprising fraction of real patterns.

---

## Concept 45 — Collections chosen for the access pattern

You know the complexity classes; the performance engineering layer is **constant factors and memory layout** (Concepts 9–10):

- **Presize.** `new List<T>(expectedCount)`, `new Dictionary<K,V>(expectedCount)`, `EnsureCapacity`. Growth doubles and copies; for large collections the intermediate arrays land on the LOH.
- **Small *n* favours arrays.** For a handful of elements, a linear scan over an array beats hashing — no hash computation, one or two cache lines, predictable branches. The crossover for string keys is often somewhere in the tens, for integers higher; measure with your key type (the benchmark in Concept 15 is the template).
- **Read-mostly lookups built once: `FrozenDictionary<K,V>`/`FrozenSet<T>`** (.NET 8). Creation is deliberately expensive — it analyzes the keys and picks a specialized strategy (for strings, for example, hashing only a distinguishing substring, or bucketing by length) — and lookups become faster than `Dictionary`. Ideal for configuration, routing tables, feature maps, and allow-lists built at startup. Wrong for anything that changes.
- **Comparers matter.** `StringComparer.OrdinalIgnoreCase` has fast paths; a culture-aware comparer, or a custom comparer with an expensive `GetHashCode`, dominates lookup cost. Struct keys must implement `IEquatable<T>` and a good `GetHashCode` (`HashCode.Combine`) or they box and reflect (Module 14, Concept 17).
- **Alternate lookup** (.NET 9) lets span-keyed lookups hit string-keyed dictionaries without allocating (Module 16, Concept 53).
- **`CollectionsMarshal.GetValueRefOrAddDefault`** makes get-or-update a single lookup (Concept 42).
- **Immutable collections**: `ImmutableArray<T>` is an array wrapper — fast reads, copy-on-write. `ImmutableList<T>`/`ImmutableDictionary<K,V>` are trees — O(log n) pointer-chasing reads, chosen for cheap *modification* of persistent versions. Picking `ImmutableList<T>` for a read-heavy snapshot is a common, silent slowdown; `ImmutableArray<T>` or a frozen collection is usually what was wanted.
- **Specialized structures**: `PriorityQueue<TElement, TPriority>` (a 4-ary heap), `BitArray` and `BitOperations` for dense flags, the generic `OrderedDictionary<TKey,TValue>` (.NET 9) when insertion order matters, `SortedDictionary` (a red-black tree) only when you genuinely need ordered iteration with frequent updates.
- **Concurrent collections** are for concurrent access (Module 15), not a default: `ConcurrentDictionary` reads are lock-free and fast, but its enumeration, `Count`, and `ToArray` take locks across all stripes.

---

## Concept 46 — Vectorization in practice

SIMD registers hold 128, 256, or 512 bits — 16, 32, or 64 bytes processed per instruction. .NET exposes this through **cross-platform vector types** that compile to SSE/AVX/AVX-512 on x64 and AdvSimd (with SVE work in progress) on Arm64:

- **`Vector128<T>`, `Vector256<T>`, `Vector512<T>`** — fixed-width, with `IsHardwareAccelerated` checks and a large operator set (arithmetic, comparisons, `ExtractMostSignificantBits`, shuffles, conversions; .NET 11 adds lane-construction and composition APIs such as `Zip`/`Unzip`, `Concat*`, and geometric/alternating sequence construction).
- **`Vector<T>`** — width chosen by the runtime for the current hardware.
- **Platform intrinsics** (`System.Runtime.Intrinsics.X86.Avx2`, `…Arm.AdvSimd`) — for instructions without a cross-platform equivalent. Prefer the cross-platform APIs; they're what the BCL uses.

**Step one is always: is the BCL already vectorized for this?** Usually, yes. `MemoryExtensions` (`IndexOf*`, `Count`, `SequenceEqual`, `Contains*`, `Replace`, `Reverse`), `SearchValues`, LINQ's `Sum`/`Min`/`Max`/`Average` over arrays and spans, `Encoding` and `Base64` conversions, `string` operations, and **`TensorPrimitives`** (`System.Numerics.Tensors`: `Sum`, `Dot`, `CosineSimilarity`, `Max`, `SoftMax`, `Exp`, element-wise arithmetic over spans) — the right tool for numeric kernels such as embedding similarity.

When you do hand-vectorize, the loop shape is always the same — a vector body plus a scalar tail:

```csharp
static int CountGreaterThan(ReadOnlySpan<byte> data, byte threshold)
{
    int count = 0, i = 0;
    if (Vector256.IsHardwareAccelerated && data.Length >= Vector256<byte>.Count)
    {
        Vector256<byte> t = Vector256.Create(threshold);
        for (; i <= data.Length - Vector256<byte>.Count; i += Vector256<byte>.Count)
        {
            Vector256<byte> v = Vector256.Create(data.Slice(i, Vector256<byte>.Count));
            Vector256<byte> gt = Vector256.GreaterThan(v, t);             // 0xFF in lanes where v > t
            count += BitOperations.PopCount(gt.ExtractMostSignificantBits());
        }
    }
    for (; i < data.Length; i++)                                          // scalar tail
        if (data[i] > threshold) count++;
    return count;
}
```

Practical rules:

- **Test the scalar fallback and every width.** Use BDN jobs with `DOTNET_EnableAVX2=0`, `DOTNET_EnableAVX512F=0` (Concept 19) and run the correctness tests under them too; CI hardware is rarely your production hardware.
- **Wider isn't always faster.** On some hardware 512-bit instructions reduce clock frequency; the runtime may report `Vector512.IsHardwareAccelerated == false` where it judges 512-bit a net loss, and `DOTNET_PreferredVectorBitWidth` adjusts the preference.
- **Alignment rarely matters now**; unaligned loads are cheap on modern cores.
- **Vectorization pays on large, contiguous, homogeneous data** — thousands of elements or more. For a 12-element span, the setup and tail cost more than the loop.
- **Mind the baseline.** .NET 11 guarantees x86-64-v2 (SSE4.2, POPCNT) as a minimum; Native AOT compiles for the baseline unless told otherwise (Concept 54).

---

## Concept 47 — Serialization

Serialization is frequently the largest CPU consumer in API services, and the fixes are well known:

- **Cache `JsonSerializerOptions`.** Each options instance owns a metadata cache; creating one per call (`JsonSerializer.Serialize(x, new JsonSerializerOptions { … })`) rebuilds reflection metadata every time and is one of the most common serious performance bugs in .NET codebases. Analyzer **CA1869** flags it; `JsonSerializerOptions.Web` (.NET 9) is a ready-made cached instance.
- **Use source generation** — required under Native AOT and beneficial everywhere (no reflection warm-up, faster startup, less memory):

```csharp
[JsonSourceGenerationOptions(JsonSerializerDefaults.Web)]
[JsonSerializable(typeof(OrderDto))]
[JsonSerializable(typeof(List<OrderDto>))]
internal partial class AppJsonContext : JsonSerializerContext { }

byte[] bytes = JsonSerializer.SerializeToUtf8Bytes(order, AppJsonContext.Default.OrderDto);

// ASP.NET Core minimal APIs:
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

  The generator produces metadata by default and can also generate a **fast-path serializer** (`GenerationMode = JsonSourceGenerationMode.Serialization`) that writes straight to `Utf8JsonWriter` without the metadata-driven engine.
- **Stay in UTF-8 and stream.** `SerializeToUtf8Bytes`, `SerializeAsync(stream, …)`, and writing to a `PipeWriter` avoid building a `string` and then encoding it. For large payloads, `DeserializeAsyncEnumerable` processes top-level arrays incrementally instead of materializing them.
- **Drop to `Utf8JsonReader`/`Utf8JsonWriter`** for the rare hot path where you need to extract two fields from a large document or emit a fixed shape at maximum speed.
- **Dispose `JsonDocument`** — it rents pooled buffers; forgetting to dispose it leaks pool capacity. `JsonNode` is the convenient mutable DOM and costs more.
- **Consider the format itself.** Binary formats (Protocol Buffers via gRPC, MessagePack with its source generator) are smaller and cheaper to parse, at the cost of schema management and human readability — a service-boundary decision (Module 11), not a micro-optimization.
- **Compression is CPU for bandwidth.** Brotli at high levels is expensive; for dynamic API responses, fast levels (`CompressionLevel.Fastest`) usually give most of the size win at a fraction of the CPU. Measure both sides of the trade.

---

## Concept 48 — Exceptions and failure paths

Throwing an exception captures a stack trace, unwinds frames, runs filters and `finally` blocks, and allocates. .NET 9 replaced the exception-handling implementation with a managed one derived from Native AOT's, making throw/catch several times faster, but it still costs on the order of **microseconds, not nanoseconds** — a thousand times a cheap method call. In async code the cost compounds: with classic compiler-generated state machines, an exception crossing ten async frames is caught and re-thrown at each one; the .NET 11 measurements for runtime async show that path becoming several times cheaper, because pass-through frames no longer catch and rethrow.

The rules:

- **Exceptions are for exceptional conditions.** Expected outcomes — validation failures, "not found," a parse that might fail — belong in return values: the `Try` pattern, result types (Module 16, Concept 39), `OperationStatus`.
- **Check the counters.** `dotnet.exceptions` (and `dotnet-counters`'s exception count) exposes first-chance exceptions per second. A healthy service throws close to zero in steady state; hundreds per second usually means a library or a code path is using exceptions for control flow — and `[ExceptionDiagnoser]` finds them in benchmarks.
- **Use throw helpers** to keep hot methods small and inlinable (Concept 29).
- **Price the failure path in design.** Retries multiply load; timeouts that fire late hold resources; a dependency failure that turns every request into a thrown-and-caught exception can raise CPU exactly when the system is least able to afford it (Module 13's retry storms and cascading failures, seen from the CPU side).

---

## Concept 49 — Telemetry overhead

Observability is not free, and in high-throughput services it's often a top-five cost:

- **Logging**: the `[LoggerMessage]` source generator (Module 16, Concept 58) — strongly typed, checks `IsEnabled` before doing any work, avoids boxing and template parsing. Never interpolate into log calls. Keep `Debug`/`Trace` logging out of hot loops even when disabled, unless it's generator-backed. Avoid logging whole payloads.
- **Tracing**: `ActivitySource.StartActivity()` returns `null` when nothing is listening or the sampler drops the trace, so un-sampled spans cost almost nothing — **sample** (head sampling in the SDK, tail sampling in a collector) rather than recording everything. Don't create spans for sub-microsecond operations.
- **Metrics**: `Counter<T>.Add` and `Histogram<T>.Record` are designed to be cheap; the traps are **tag cardinality** (a user ID, full URL, or exception message as a tag creates unbounded time series — memory in-process, cost in the backend; OpenTelemetry's SDK caps series per metric and overflows beyond it) and allocation from tags (pass tags via `TagList`, a struct, rather than arrays).
- **Exporters** batch and run in the background — but an exporter that can't keep up buffers in memory; check its queue limits.
- **Measure telemetry like any other cost**: a CPU profile of a busy service with telemetry on will show you its share. Budgets for it are reasonable (for example, "telemetry under 5% of CPU").

---

# Part H — Native AOT in practice

Module 14 (Concepts 7, 54, 64) framed Native AOT as a startup-and-footprint decision and Module 16 (Concept 62) explained why the language has been moving work to compile time to enable it. This part is the practitioner's view: what the toolchain does, what breaks and how you find out, how to get a real service to zero warnings, and what the result actually looks like.

## Concept 50 — What Native AOT actually does

`dotnet publish -c Release -r linux-x64` with `<PublishAot>true</PublishAot>` runs a different pipeline from the normal build:

1. The C# compiler produces IL as usual (with source generators having already replaced reflection where they can).
2. **ILC**, the Native AOT compiler, starts from the application's roots (`Main`, plus anything explicitly rooted) and performs a **whole-program dependency analysis**: it follows every call, type reference, virtual method override, and generic instantiation that is statically reachable. Trimming is inherent — unreachable code is simply never compiled.
3. ILC compiles the reachable IL to native code using the same RyuJIT back end the JIT uses, but with whole-program knowledge.
4. The platform linker (MSVC `link` on Windows, `clang`/`ld` on Linux and macOS) links that code with the **Native AOT runtime** — a slimmed runtime including the same GC — into **one self-contained native executable**. No `dotnet` host, no IL, no JIT.

The **closed-world assumption** follows directly: everything that will ever execute must be known at build time. Therefore:

- **No runtime code generation** — `Reflection.Emit` and `DynamicMethod` are unavailable; `System.Linq.Expressions` falls back to an interpreter, so `Expression.Compile()` works but slowly.
- **No loading assemblies** that weren't compiled in — no plugin systems via `Assembly.LoadFrom`.
- **Reflection works only over what ILC kept.** Reflecting over a type's members is fine if the compiler knew to keep them (because of annotations — Concept 51 — or because the code was otherwise reachable); `Type.GetType("Some.Name")` on a type nothing else references fails at runtime.
- **Generic instantiations must be knowable.** `typeof(List<>).MakeGenericType(t)` works only if that instantiation was compiled.
- **`dynamic`, built-in COM interop, and C++/CLI** are unsupported (source-generated `ComWrappers` interop is the replacement for COM).
- **Build on the target OS.** Cross-OS compilation isn't supported — build Linux binaries on Linux (a container in CI is the usual answer); cross-architecture builds are possible with the right toolchain.

ILC can also **pre-initialize** simple static constructors at compile time, baking their results into the binary's data — one of several reasons AOT startup is so fast.

---

## Concept 51 — Trimming and the annotation system

The trimmer (ILLink for trimmed JIT apps, ILC's own analysis for AOT) removes what it can prove is unused. Its safety comes from **static analysis of reflection**: when code reflects in ways the analysis can't follow, it emits a **warning**, and the contract is simple and strong — **an application that publishes with zero trim/AOT warnings behaves the same as its JIT version.** A warning is therefore not noise; it is a report of a place where the published app may fail at runtime.

The warnings you'll meet:

| Warning | Meaning |
|---|---|
| **IL2026** | Calling a member marked `[RequiresUnreferencedCode]` — "this may not work when trimmed" |
| **IL2067–IL2075** family | A `Type`, string, or member flows into a reflection API without the annotation that says which members must be kept |
| **IL2104** | A referenced assembly produced trim warnings (the details are hidden unless you ask for them) |
| **IL3050** | Calling a member marked `[RequiresDynamicCode]` — "this needs runtime code generation," so it can't work under AOT |
| **IL3000** | `Assembly.Location` in a single-file or AOT app returns an empty string |

The annotations are how library and application code *declare* reflection needs so the analysis can satisfy them — instead of hiding them:

```csharp
// "Whoever calls this must guarantee T's public parameterless constructor is kept."
public static T Create<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicParameterlessConstructor)] T>()
    => Activator.CreateInstance<T>();

// "This API can't be made trim-safe; callers inherit the warning."
[RequiresUnreferencedCode("Scans assemblies for handlers. Use the source-generated RegisterHandlers() when trimming.")]
public static IServiceCollection AddHandlersFromAssembly(this IServiceCollection services, Assembly assembly) { /* … */ }
```

- **`[DynamicallyAccessedMembers]`** on a `Type`, a string, or a generic parameter states which members must be preserved; the requirement flows through assignments and parameters, and the analysis checks every call site.
- **`[RequiresUnreferencedCode]` / `[RequiresDynamicCode]`** mark APIs that fundamentally can't be safe; the warning propagates to callers, who either propagate it further or stop using the API.
- **`[UnconditionalSuppressMessage]`** suppresses a warning where you can **prove** it's a false positive — with a justification in the attribute. Suppressing to get a green build is how production outages under AOT happen.
- **`[DynamicDependency]`** and root descriptor files keep specific members explicitly — the escape hatch for code you can't annotate.
- **Feature switches** — `AppContext` switches whose guarded code the trimmer can remove when the switch is off at build time (`InvariantGlobalization`, `EventSourceSupport`, `UseSystemResourceKeys`, and your own via .NET 9's `[FeatureSwitchDefinition]`).

---

## Concept 52 — Making a project AOT-ready

For an application:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <PublishAot>true</PublishAot>                       <!-- also enables the trim/AOT analyzers during normal builds -->
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

For a library: **`<IsAotCompatible>true</IsAotCompatible>`**, which marks it trimmable and turns on the trim, AOT, and single-file analyzers, so incompatibilities surface when the *library* builds rather than when a consumer publishes.

An AOT-friendly ASP.NET Core service (the `dotnet new webapiaot` template is the starting point):

```csharp
var builder = WebApplication.CreateSlimBuilder(args);

builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var app = builder.Build();
app.MapGet("/orders/{id:int}", (int id) => new OrderDto(id, "pending"));
app.Run();

public sealed record OrderDto(int Id, string Status);

[JsonSerializable(typeof(OrderDto))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

**`CreateSlimBuilder`** omits features that are rarely needed and costly for size — IIS integration, EventLog and Debug logging providers, hosting startup assemblies, static web assets, regex and `alpha` route constraints, and Kestrel's HTTPS/HTTP/3 configuration by default (restore HTTPS with `builder.WebHost.UseKestrelHttpsConfiguration()` if the service terminates TLS itself rather than behind a proxy). **`CreateEmptyBuilder`** goes further for minimal hosts.

The work of getting a real service to zero warnings is mostly **replacing reflection with source generation**:

| Reflection-based pattern | AOT-friendly replacement |
|---|---|
| Reflection-based JSON | `JsonSerializerContext` (Concept 47) |
| `IConfiguration.Bind`/`Get<T>` | Configuration-binding source generator (enabled automatically with `PublishAot`) |
| Data-annotation options validation | Options-validation source generator (`[OptionsValidator]`) |
| Minimal API delegate compilation | Request Delegate Generator (automatic) |
| `Regex` with `Compiled` | `[GeneratedRegex]` |
| `LoggerMessage.Define` / reflection-y logging | `[LoggerMessage]` |
| `[DllImport]` with runtime marshalling | `[LibraryImport]` |
| Assembly scanning for DI registration | Explicit registration, or a source generator |
| Reflection-based object mappers | Source-generated mappers (for example, Mapperly) |

**Test the published binary, not just the build.** Run the integration suite against the native executable in CI — in a container matching production — and **fail the pipeline on any IL2xxx/IL3xxx warning.** Zero warnings is the only state in which the equivalence guarantee holds.

---

## Concept 53 — The ecosystem map (as of .NET 10/11)

What an architect should know before promising AOT (check the current docs before committing — this moves every release):

| Area | Status |
|---|---|
| **ASP.NET Core minimal APIs** | Supported (partially — the Request Delegate Generator covers the common surface) |
| **gRPC** | Fully supported |
| **MVC controllers, Razor Pages** | **Not supported** |
| **Blazor Server** | **Not supported** |
| **SignalR** | Partially supported |
| **Middleware**: CORS, health checks, HTTP logging, localization, output caching, rate limiting, request decompression, response caching and compression, rewrite, static files, WebSockets | Supported |
| **Authentication** | JWT bearer supported; other schemes largely not |
| **System.Text.Json** | Source-generated contexts only (reflection-based serialization is disabled by default under AOT) |
| **Newtonsoft.Json** | Not compatible |
| **`Microsoft.Extensions.*`** (DI, configuration via the binder generator, options, logging, hosting) | Supported |
| **EF Core** | **Experimental** — compiled models and precompiled queries generated by `dotnet ef dbcontext optimize --precompile-queries --nativeaot`; documented as not yet suited for production |
| **Dapper** | The source-generated Dapper.AOT variant targets AOT; verify against your runtime version |
| **Reflection-scanning libraries** (classic AutoMapper configuration, assembly-scanning mediators, some DI extensions) | Problematic; source-generated alternatives exist |
| **OpenTelemetry, Azure SDK clients, other third-party packages** | Increasingly annotated — **verify each package**: build with the analyzers and read the warnings |

The practical consequence: **Native AOT today is a great fit for new minimal-API and gRPC services, workers, CLIs, and functions with modest dependency graphs; it's a poor fit for an existing MVC + EF Core monolith.** The data-access story is the usual blocker — raw ADO.NET or a source-generated micro-ORM is the current production-safe route for AOT services that need a database.

---

## Concept 54 — Size and speed knobs

The settings that matter, and what each one trades:

| Setting | Effect | Trade-off |
|---|---|---|
| `<OptimizationPreference>Speed</OptimizationPreference>` / `Size` | Bias ILC's optimization decisions | Speed grows the binary; Size can cost throughput |
| `<IlcInstructionSet>x86-x64-v3</IlcInstructionSet>` (or a specific list, or `native`) | Let compiled code assume newer instruction sets (AVX2, BMI, FMA, …) without runtime checks | Faster vector and bit-manipulation paths; **the binary won't start on CPUs lacking them** |
| `<InvariantGlobalization>true</InvariantGlobalization>` | Drop ICU and culture data | Big size and startup win; culture-specific formatting and comparison stop working |
| `<UseSystemResourceKeys>true</UseSystemResourceKeys>` | Strip exception message resources | Smaller; exception messages become resource keys |
| `<StackTraceSupport>false</StackTraceSupport>` | Remove method-name metadata for stack traces | Smaller; **stack traces lose method names** — rarely worth it for services |
| `<EventSourceSupport>false</EventSourceSupport>` | Remove EventSource/EventPipe support | Smaller; **breaks `dotnet-counters`, `dotnet-trace`, and runtime metrics** — don't, for services |
| `StripSymbols` (default on Linux) | Move native symbols to a separate `.dbg` file | Keep the symbol files for every release — you'll need them for crash analysis (Concept 56) |

For **binary-size analysis**, `<IlcGenerateMstatFile>true</IlcGenerateMstatFile>` produces a map of what ended up in the binary, and **Sizoscope** (by Michal Strehovský of the Native AOT team) visualizes it and answers "why is this type included?" — usually a single reflection-heavy dependency dragging in a subsystem.

For **containers**, an AOT binary needs only the native dependencies, not the .NET runtime: base it on the **`runtime-deps`** images, ideally the **chiseled** (distroless) variants — `mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled` — for a small image with a minimal attack surface. Image size matters for cold start because pulling the image is often the slowest step of scaling out (Concept 63).

---

## Concept 55 — The AOT performance profile, measured

The shape of the trade-off across the three deployment models, for a typical HTTP service (orders of magnitude; your numbers will differ — measure them):

| | JIT | ReadyToRun | Native AOT |
|---|---|---|---|
| Time to first response | Hundreds of ms | Noticeably faster | **Tens of ms** |
| Post-deployment p99 | Elevated during warm-up | Less elevated | **Flat from the first request** |
| Steady-state throughput | **Best** (tier-1 + dynamic PGO) | Best (hot code re-jitted) | Close; sometimes lower (no dynamic PGO), sometimes higher (whole-program optimization) |
| Working set | Highest (JIT, metadata, tiering state) | High | **Lowest** |
| Deployment size | Small IL + shared runtime | Larger | Single native binary; no runtime needed |
| Compatibility | Everything | Everything | Closed world (Concepts 50–53) |
| Build time | Fast | Slower | **Slowest** (whole-program compile + link) |

Where AOT's steady state gains: **whole-program devirtualization** (an interface with one implementation in the entire program becomes a direct call), no version-resilience indirections, pre-initialized statics, and — in .NET 11 — faster interface dispatch through a shared, patchable dispatch helper that also shrinks call sites. Where it loses: **no dynamic PGO**, so no guarded devirtualization of polymorphic sites and no profile-driven layout; and code compiled for a baseline instruction set unless you raise it (Concept 54).

The platform's own choice is instructive: in .NET 11 the `dotnet` CLI host is Native AOT–compiled by default, serving help, parsing, and many commands natively and launching tools without the 600–700 ms managed CLI startup. That is the canonical AOT workload — **short-lived processes where startup is most of the lifetime.**

Measure it properly: a BDN job with `--runtimes nativeaot10.0` for micro-level comparisons; for services, time-to-first-successful-request (from container start), RSS under steady load, throughput at the latency SLO (Concept 62), and image size — side by side for JIT, R2R, and AOT builds of the same code (practice exercise 8).

---

## Concept 56 — Diagnosing Native AOT applications

- **EventPipe works**, so `dotnet-counters`, `dotnet-trace`, `dotnet-monitor`, and the runtime metrics work — unless you disabled `EventSourceSupport` (Concept 54).
- **Managed dump analysis with SOS doesn't apply.** There's no managed metadata for SOS to walk; crash dumps are analysed with **native debuggers** (WinDbg, lldb, Visual Studio's native debugging) using the native symbols — which is why keeping the stripped `.dbg`/`.pdb` files for every release is non-negotiable.
- **Stack traces** need `StackTraceSupport` (on by default) to show method names; line numbers need symbols.
- **Debugging** works at source level through native debuggers (Visual Studio can debug an AOT binary), but there's no Edit and Continue or hot reload — debug the JIT build day to day and treat AOT as a publish-time concern, verified by tests against the native binary.
- **The most common production failure** — a `MissingMetadataException`-style runtime error, or a serializer that "can't find" a type — is a trim warning someone suppressed or ignored. The fix is always upstream: an annotation, a source-generated context entry, or removing the reflection.

---

# Part I — Allocation-conscious design as engineering

Module 14 (Part E) gave you the allocation cost model, the struct-vs-class procedure, pooling, the zero-allocation patterns, and the hidden-allocation catalogue. This part is about turning those into **engineering practice**: budgets that are tested, designs whose steady state doesn't allocate, APIs whose shape prevents allocation, and a policy for where the complexity is allowed to live.

## Concept 57 — Allocation budgets as contracts

A hot path's allocation behaviour should be a **stated, tested property**, like its correctness:

> *"Parsing a message allocates 0 bytes on the success path. Handling a `GET /orders/{id}` request allocates under 4 KB end to end."*

Allocation is the ideal thing to put under test because **it is deterministic**: for a given runtime version and input, a code path allocates the same number of bytes every run, on every machine. Time varies with noise; allocation doesn't. Three levels of enforcement:

1. **Unit-level assertions** — measure allocated bytes around the operation:

```csharp
[Fact]
public void Parse_success_path_does_not_allocate()
{
    var line = "2026-09-22T10:15:30Z GET /api/orders 200 1532 12.5";
    Assert.True(LogParser.TryParse(line, out _));            // warm up: JIT, statics, caches

    long before = GC.GetAllocatedBytesForCurrentThread();
    for (int i = 0; i < 1_000; i++) LogParser.TryParse(line, out _);
    long allocated = GC.GetAllocatedBytesForCurrentThread() - before;

    Assert.Equal(0, allocated);
}
```

   `GetAllocatedBytesForCurrentThread` is per thread, so keep the measured code synchronous (or use `GC.GetTotalAllocatedBytes(precise: true)` in an isolated test process). Warm up first so one-time initialization isn't counted.

2. **Benchmark-level gates** — BenchmarkDotNet's JSON export compared against a stored baseline in CI; fail on allocation increases above a small threshold (Concept 64).

3. **Production metric** — allocation *rate per request*: the runtime's total-allocated-bytes counter divided by request count over the same window. Publish it per service; watch it per release. A jump from 18 KB to 60 KB per request after a deployment is a regression you can see before it becomes a GC-driven p99 problem.

---

## Concept 58 — Steady-state-zero design

High-performance servers share a design principle: **allocate at startup or at connection establishment, then reuse on the request path.** The steady state — the millionth request — allocates nothing, or close to it. Kestrel's own pipeline is the reference example.

The techniques:

- **Per-connection or per-worker buffers** rented once and reused for the life of the connection (pooled, and on the Pinned Object Heap if they're handed to native I/O).
- **Reusable state objects** — parser state, serializer writers (`Utf8JsonWriter.Reset` onto a new buffer), `StringBuilder` — from `ObjectPool<T>` or owned per worker.
- **Cached delegates** — `static` lambdas are cached by the compiler; capturing lambdas allocate a closure and a delegate per creation (Module 14, Concept 50).
- **Precomputed constants** — UTF-8 header names and values as `u8` literals, frozen lookup tables, `SearchValues` instances, cached `JsonSerializerOptions`, `CompositeFormat` instances.
- **Reusable asynchronous operations** — `IValueTaskSource`-backed awaitables (as `Socket` uses internally) for per-operation async without a `Task` allocation (Module 15, Concept 14).

The discipline has limits worth stating in an interview: it applies to **infrastructure and hot paths** — protocol handlers, ingestion loops, gateways, serializers — where the operation count is enormous. Business logic that runs a few thousand times per second, touches a database, and allocates a few kilobytes is dominated by I/O; making it allocation-free buys nothing measurable and costs clarity.

---

## Concept 59 — API shape decides allocation

The most effective allocation work happens **before the code is written, in the signature**. The same capability in three shapes:

```csharp
// Shape 1 — allocating by construction: a list per call, a string per item
IReadOnlyList<string> GetTags(ReadOnlySpan<char> header);

// Shape 2 — the caller owns the memory: write ranges into a caller-provided buffer
int GetTags(ReadOnlySpan<char> header, Span<Range> destination);

// Shape 3 — visitor with explicit state: no allocation, no capture
void ForEachTag<TState>(ReadOnlySpan<char> header, TState state, TagVisitor<TState> visit)
    where TState : allows ref struct;
delegate void TagVisitor<TState>(ReadOnlySpan<char> tag, TState state) where TState : allows ref struct;
```

Shape 1 is the easiest to use and allocates on every call. Shape 2 allocates nothing and needs the caller to supply a buffer (usually `stackalloc`). Shape 3 allocates nothing *and* handles unbounded counts, at the cost of callback-style code; passing state explicitly lets the caller use a `static` lambda. The two-tier pattern (Concept 41) often exposes Shape 1 as a convenience over Shape 2.

Other shape-level decisions with allocation consequences:

- **Struct enumerators.** A collection whose `GetEnumerator()` returns a struct (as `List<T>` does) is enumerated by `foreach` without allocation; returning `IEnumerable<T>` from an API forces the interface path (partly rescued by .NET 10–11 escape analysis — Module 14, Concept 53 — but not something to design around).
- **`Try` patterns over exceptions** for expected failures (Concept 48).
- **`ValueTask<T>`** for async methods that usually complete synchronously (Module 15, Concept 14).
- **`ReadOnlySpan<byte>` properties for constant data** — `static ReadOnlySpan<byte> Magic => [0x50, 0x4B, 0x03, 0x04];` compiles to a view over the assembly's data section, not an array allocation.
- **`params ReadOnlySpan<T>`** instead of `params T[]` (Module 16, Concept 55).
- **Returning spans of internal buffers** only with a documented, short validity window — or not at all.

---

## Concept 60 — The complexity budget

Every technique in Parts F, G, and I buys performance with complexity, and complexity has a cost in bugs, onboarding, and review time. The architect's job is to decide **where the complexity is allowed to live**:

- **Performance islands.** Confine span-heavy, pooled, unsafe, or vectorized code to a small number of well-bounded components — the parser, the serializer, the protocol handler — with a clean, conventional API facing the rest of the codebase. The domain layer should look like ordinary C#.
- **Review rules** for code inside the islands, ideally as a checklist in the PR template:
  - every `stackalloc` has a constant bound and a pooled fallback;
  - every `Rent` has a `Return` in a `finally`, and no rented buffer escapes;
  - every stored `Memory<T>` has a documented owner and lifetime;
  - every `Unsafe`/`MemoryMarshal` use has a comment stating its contract and a benchmark justifying it on the current runtime;
  - every vectorized path has a scalar fallback tested under disabled instruction sets;
  - a benchmark (and ideally an allocation test) is committed with the change.
- **Ownership.** `CODEOWNERS` entries for the islands so changes get reviewed by someone who knows the invariants.
- **Fuzzing.** Parsers and protocol code built on spans are exactly where off-by-one and boundary bugs hide; property-based tests and fuzzers (SharpFuzz for .NET) find the inputs your unit tests didn't think of.
- **Re-evaluation.** On each runtime upgrade, re-run the island's benchmarks against the simple version. The JIT keeps closing gaps (Concepts 31 and 66); code that was necessary in 2022 may now be slower than the naive version.

---

# Part J — Performance at system scope: the architect's view

## Concept 61 — Load testing that tells the truth

A load test is an experiment. It's only as good as its fidelity to production, and most load tests are wrong in predictable ways:

**Know what kind of test you're running:**

| Test | Question it answers |
|---|---|
| **Baseline / load** | Does the service meet its SLO at expected peak load? |
| **Capacity / breakpoint** | At what load does the SLO break? (the knee — Concept 62) |
| **Stress** | How does it fail beyond capacity — gracefully (shedding, 429s) or catastrophically? (Module 6's overload protection) |
| **Spike** | Does it survive a sudden step change — autoscaling lag, cold instances, connection storms? |
| **Soak** | Does anything degrade over hours — leaks, fragmentation, growing caches, connection churn? |

**The fidelity checklist:**

- **Open-model load** for internet-facing services (Concept 6).
- **Realistic data**: payload size distribution, key skew (a Zipfian distribution of product IDs, not uniform), cache hit ratios, the real mix of endpoints, authenticated and anonymous traffic.
- **Warm-up excluded** from the measurement window — or measured separately, if post-deployment behaviour is the question (it often should be).
- **Environment parity**: the same instance SKU, CPU and memory limits, GC settings, runtime version, and number of instances; dependencies either real (at realistic scale) or faked with realistic latency distributions — a fake database that answers in 50 µs makes every service look fast.
- **The load generator is not the bottleneck**: check its CPU, its connection count, and its own latency; distribute it if necessary.
- **Measure both sides**: client-side latency histograms *and* server-side USE/RED metrics (Concept 8), on the same timeline, so you can explain what you see.
- **Run long enough** for GC, caches, and autoscaling to reach steady state; for soak tests, hours.

**Tools**: **k6** (JavaScript scenarios, arrival-rate executors, good CI integration), **NBomber** (.NET-native scenarios in C#, open and closed simulations — check its licence terms for commercial use), **Bombardier** and **oha** (quick HTTP blasting), **wrk2** (constant-throughput, coordinated-omission-aware), **Azure Load Testing** (managed, runs JMeter and Locust scripts at scale, correlates with Azure Monitor server-side metrics, and gates CI/CD pipelines on pass/fail criteria), and **crank** (the ASP.NET team's tool for orchestrating benchmark agents on dedicated machines — how the team produces its own numbers).

---

## Concept 62 — From benchmark to capacity plan

Capacity planning turns a measured per-instance number into an instance count, and the per-instance number is **throughput at the SLO, not maximum throughput.**

**Find the knee.** Run a capacity test that steps load up in increments on one instance, recording p99 at each step. Plot p99 against requests per second: it's flat, then rises, then goes vertical (Concept 7's hockey stick). The **knee** is the highest load at which p99 is still comfortably inside the SLO. Maximum throughput — the rate at which the instance saturates — is typically well beyond the knee and is useless for planning an interactive service.

**Then add the headroom factors**, each with a reason:

- **Queueing and burst headroom** — operate below the knee, not at it: variability, GC, noisy neighbours, and traffic bursts within the scaling reaction time.
- **Failure capacity** — with *z* availability zones and a requirement to survive losing one, the remaining `(z − 1)/z` of the fleet must carry the full load. With three zones, each instance must run at no more than about two-thirds of its safe rate in normal operation.
- **Deployment capacity** — rolling deployments remove instances temporarily unless you surge (`maxUnavailable: 0`).
- **Growth** — the planning horizon's expected traffic growth.

Worked example 5 does the arithmetic end to end. Two insights from it are worth carrying into any design round:

1. **Performance improvements pay off in steps.** Instance counts are integers and must usually be divisible across zones; a 15% efficiency gain may save no instances at all, while a 25% gain saves three. Know where your next step is before arguing that an optimization will "save money."
2. **Little's Law sizes the pools.** Fleet-wide concurrency = arrival rate × latency: 20,000 rps × 50 ms mean = 1,000 requests in flight across the fleet; if each holds a database connection for 10 ms, that's ~200 concurrent connections fleet-wide — divided by instance count, that's your per-instance pool size, and summed across all services, that's a number your database's connection limit must accommodate.

Express the result in the currency executives use: **cost per million requests**, and **headroom in months of growth**.

---

## Concept 63 — Startup and cold-start engineering

Startup matters whenever instances are created on the critical path: scale-out under load, scale-from-zero, deployments, crash recovery, serverless invocations. **Time-to-ready** decomposes into phases, and the slow phase is often not the one people optimize:

| Phase | Typical drivers | Mitigations |
|---|---|---|
| **Schedule and pull the image** | Image size, registry proximity, node cache | Smaller images (chiseled/`runtime-deps`, AOT), image streaming or pre-pulling, a registry in-region |
| **Process and runtime start** | Runtime init, assembly loading | ReadyToRun, Native AOT, fewer assemblies |
| **Host build** | Configuration providers (especially remote ones like Key Vault/App Configuration), DI container construction, assembly scanning | Trim reflection-heavy startup (scanning, mapper configuration validation), defer non-critical initialization, cache remote configuration |
| **JIT** | Tier-0 compilation of everything touched at startup and on first requests | R2R, AOT; avoid `AggressiveOptimization` (Concept 27) |
| **First requests** | Cold caches, TLS handshakes, connection pools empty, lazy initialization in libraries, serializer metadata | **Warm up before reporting ready**: open pool connections, touch hot paths, prime caches — in the **readiness** probe or a startup hook |

Rules that separate good cold-start engineering from bad:

- **Measure each phase.** Timestamps from container start to process start, host start (`IHostApplicationLifetime.ApplicationStarted`), first successful request, and "p99 back to normal." Without the breakdown, teams optimize JIT when the image pull took 20 seconds.
- **Readiness, not liveness, gates warm-up.** A liveness probe that fails during a slow warm-up gets the pod killed in a restart loop; readiness keeps traffic away until the instance is actually ready. Startup probes exist for slow initializers.
- **Never block startup synchronously on a remote dependency without a timeout**; a slow dependency then becomes a fleet-wide inability to scale.
- **Don't scale to zero if cold start violates the SLO.** On Azure Container Apps and Functions, a minimum replica count (or always-ready/pre-warmed instances) trades a small standing cost for eliminating cold starts on the critical path — often the right architectural answer before any code-level optimization (Module 26 develops the compute-platform side).
- **Autoscaling reaction time** is metrics interval + decision interval + time-to-ready. Faster startup makes smaller headroom safe (Concept 62) — which is the *economic* argument for R2R or AOT.

---

## Concept 64 — Preventing regressions

Performance regresses silently: one logging change, one new dependency, one "harmless" LINQ query per request. Prevention needs gates at each level of the benchmark pyramid (Concept 20):

- **Gate on allocations; track times.** Allocation assertions and BDN allocation columns are deterministic, so they make reliable CI gates (Concept 57). Timings from shared CI runners are noisy enough that hard gates on them produce flaky builds and then get disabled; **track** timings as trends instead, and gate only with generous thresholds or on dedicated hardware.
- **Compare within one run.** The robust CI pattern is A/B in the same job on the same machine: build the base branch and the PR, run both benchmark sets, and compare with a statistical threshold. The .NET team's `dotnet/performance` repository does this at scale — its `ResultsComparer` tool compares two BDN result sets with a noise threshold, and its lab runs benchmarks on dedicated hardware continuously and files regressions automatically.
- **Nightly or pre-release load tests** with SLO-based pass/fail criteria (Azure Load Testing's test criteria or a k6 threshold) catch regressions no micro-benchmark sees — the extra database round trip, the new synchronous call.
- **Canary analysis in production.** Deploy to a small slice, compare its latency distribution, CPU per request, allocation per request, and error rate with the baseline fleet, and roll back automatically on significant regression (Module 28 covers the observability side).
- **Make results visible.** A bot comment on each PR with the benchmark comparison changes behaviour more than any policy document.

---

## Concept 65 — Performance budgets and culture

**Budgets turn SLOs into engineering constraints.** Decompose the end-to-end target along the critical path:

> End-to-end p99 of 300 ms for checkout → API gateway 10 ms, checkout service 60 ms of its own work, pricing call 80 ms, inventory call 60 ms (in parallel with pricing), payment authorization 120 ms, network and serialization slack 30 ms.

Each owning team now has a number, and a change that spends another team's budget is visible in the design review rather than in production. The same works for **cost budgets** — CPU-milliseconds or dollars per thousand requests — and **startup budgets** for anything that scales out on demand.

Cultural practices that keep budgets real:

- **Every service publishes its numbers**: p50/p99 per endpoint, CPU and allocation per request, cost per million requests, time-to-ready.
- **Performance is part of the definition of done** for designated hot paths — the benchmark and allocation test ship with the feature.
- **A performance review checklist** in design reviews: round trips on the critical path, fan-out and its tail amplification, payload sizes, caching and its consistency cost, synchronous dependencies, startup work, telemetry cost.
- **Performance debt is tracked** like other technical debt (Module 33): known hot spots, workarounds awaiting a runtime upgrade, unsafe code to re-evaluate.
- **Credit deletion.** The best performance changes often remove code, calls, or whole components.

---

## Concept 66 — Runtime upgrades as a performance strategy

Each .NET release delivers performance you don't have to write. .NET 11's list alone includes more bounds-check elimination, devirtualization of generic virtual methods, extended escape analysis, faster delegates, runtime async (with the BCL already compiled that way), R2R comparer specialization, and faster Native AOT interface dispatch. Many of these apply **without recompiling your code**, because they're JIT or runtime changes; others — like new span-based overloads that existing call sites start binding to — apply when you **recompile** against the new framework (Module 16, Concept 55).

How to use this as an architect:

- **Measure the upgrade.** Run your benchmark suite with both runtimes in one session (Concept 19) and a load test on each; publish the delta. Upgrades are one of the few performance changes that are both broad and cheap.
- **Price in the support model.** .NET 10 is LTS (supported to November 2028); .NET 11 is STS (24 months). Organizations that upgrade every release capture the gains yearly at the cost of more frequent validation; LTS-only organizations capture them every two years with less churn. Either is defensible if stated.
- **Mind the hardware baseline.** .NET 11 requires x86-64-v2 and prefers x86-64-v3 for R2R; old on-premises hosts or unusual VM SKUs may run slower or not at all.
- **Remove workarounds.** Every upgrade is an opportunity to delete code written around old JIT limitations — the unsafe loops, manual devirtualization, hand-rolled caches of things the BCL now caches. Re-run the benchmark; if the simple version is as fast, delete the complex one.
- **Opt into new features deliberately.** Runtime async is opt-in in .NET 11: try it in a canary, measure, keep or revert — exactly the loop from Concept 2.

---

## Concept 67 — When not to optimize

The architect's version of Module 14's Concept 55 is economic:

- **Is it binding?** If the service meets its SLO with headroom and its cost is a rounding error on the bill, the correct amount of performance work is zero. "Fast enough" is defined by the SLO, not by what's possible.
- **What's it worth?** An engineer-month costs a known amount. A 10% CPU reduction on a service running 6 instances saves, perhaps, half an instance — which may be zero instances after redundancy rounding (Concept 62). The same month spent on the service with 400 instances, or on removing a round trip from a user-facing critical path, is worth far more.
- **What does it cost to maintain?** Complexity is a recurring cost (Concept 60); the runtime keeps making simple code faster (Concept 66), so hand-tuned code depreciates.
- **Is there a bigger lever?** The leverage order is fixed: **remove the work** (don't compute it, don't call it) > **do it less often** (cache, precompute, deduplicate) > **do it in bulk** (batch, pipeline) > **do it faster** (algorithms, then allocation, then instructions). Parts B–I of this module are almost entirely the last category.

The sentence to have ready: *"I'd want to know which metric is actually binding and what it's worth to move it. If we're inside our SLO and the cost is small, I'd leave this code alone and spend the effort on the round trips on our critical path — and I'd keep the benchmark so we notice if that changes."*

---

# Putting it together

Five worked examples, each shaped like an interview question. They exercise the method (Part A) more than any individual technique — which is the point.

## Worked example 1 — "After last week's release, p99 on our orders API went from 40 ms to 180 ms. Walk me through it."

**1. Characterize before hypothesizing.** Is it every endpoint or one? Every instance or some? The whole distribution shifted, or only the tail? Constant, or periodic? At the same traffic level as before?

Suppose the dashboards say: only `GET /orders/{id}`; all instances; p50 moved from 12 to 18 ms but p99 from 40 to 180 ms; traffic unchanged; started exactly at the deployment. A tail that moved much more than the median points at something intermittent — GC, contention, a slow sub-path — rather than uniformly more work.

**2. Correlate with the change.** Diff the release: code, package versions, configuration, runtime version, infrastructure. Suppose the release added a "sensitive-data redaction" step to response logging and bumped a JSON package.

**3. Walk the USE table (Concept 8).** CPU per request up ~30%. `dotnet.gc.pause.time` and gen0 collection rate roughly tripled. Allocation per request: 22 KB → 140 KB. ThreadPool queue and lock contention unchanged. The shape says **allocation-driven GC**, which explains a tail that moved more than the median: requests that overlap a collection wait for it (Module 14, Concept 62).

**4. Locate with evidence.** Capture a `dotnet-trace --profile gc-verbose` on one instance of each version under the same load (or pull Application Insights Profiler traces for both), and compare allocation-by-stack. The new version's top allocator is the redaction step: it constructs a `Regex` per log call, calls `Replace` on the serialized response *string* (so the response is now also serialized to a string before being written), and uses string interpolation in the log call (Concepts 44 and 49; Module 16, Concept 58).

**5. Hypothesize with a size.** Removing per-call regex construction and the extra response serialization should bring allocation back near 25 KB per request and GC frequency back near the old rate — so p99 back near 40–50 ms.

**6. Fix one thing at a time, measure each.** First the regex: `[GeneratedRegex]`, static. Re-measure: allocation 140 → 95 KB, p99 180 → 130 ms. Then redact on the UTF-8 bytes with `SearchValues<string>` to find markers, only when a marker is present, instead of serializing the response to a string: allocation → 26 KB, p99 → 45 ms. Then `[LoggerMessage]` for the log call: allocation → 23 KB, p99 → 41 ms.

**7. Prevent recurrence.** Add an allocation test for the endpoint's handler path (Concept 57), a CA1869/regex analyzer rule to the build, and allocation-per-request to the release dashboard and the canary analysis (Concept 64).

**What makes this answer senior:** it characterized the distribution before guessing, used the resource table to classify the problem as GC-driven, compared versions with the same method, predicted effect sizes, changed one thing per measurement, and ended with a gate rather than a fix.

---

## Worked example 2 — "This log-processing job is too slow. Make the parser faster."

The job reads access-log lines like

```
2026-09-22T10:15:30Z GET /api/orders 200 1532 12.5
```

and aggregates request count and total duration per path. The current code:

```csharp
public sealed record LogEntry(DateTimeOffset Timestamp, string Method, string Path,
                              int Status, long Bytes, double DurationMs);

public static class NaiveParser
{
    public static LogEntry Parse(string line)
    {
        string[] p = line.Split(' ');
        return new LogEntry(
            DateTimeOffset.Parse(p[0], CultureInfo.InvariantCulture),
            p[1], p[2], int.Parse(p[3]), long.Parse(p[4]),
            double.Parse(p[5], CultureInfo.InvariantCulture));
    }
}

public sealed class NaiveAggregator
{
    private readonly Dictionary<string, (long Count, double TotalMs)> _stats = new();
    public void Add(LogEntry e) =>
        _stats[e.Path] = _stats.TryGetValue(e.Path, out var s) ? (s.Count + 1, s.TotalMs + e.DurationMs) : (1, e.DurationMs);
}
```

**Step 0 — method before code.** Is the parser actually the bottleneck? Profile the job: suppose 70% of CPU is in parsing and aggregation, 20% in reading the file, 10% in GC. Amdahl says a 5× parser win would make the job roughly 2.3× faster — worth doing. (If parsing had been 10%, the answer would be "look at I/O instead.")

**Step 1 — count the allocations per line.** A `string[]` plus six substrings from `Split`; a `LogEntry` object; inside `DateTimeOffset.Parse`, general-purpose parsing work; in the aggregator, the tuple is a value type, but the path string is a new instance per line even though there are only a few dozen distinct paths. Roughly eight objects per line, every line.

**Step 2 — a span-based parser that allocates nothing per line:**

```csharp
public readonly ref struct LogLine
{
    public DateTimeOffset Timestamp { get; init; }
    public ReadOnlySpan<char> Method { get; init; }
    public ReadOnlySpan<char> Path { get; init; }
    public int Status { get; init; }
    public long Bytes { get; init; }
    public double DurationMs { get; init; }
}

public static class SpanParser
{
    public static bool TryParse(ReadOnlySpan<char> line, out LogLine entry)
    {
        entry = default;
        Span<Range> f = stackalloc Range[7];                    // 6 fields + 1 to detect extras
        if (line.Split(f, ' ') != 6) return false;

        if (!DateTimeOffset.TryParseExact(line[f[0]], "yyyy-MM-dd'T'HH:mm:ss'Z'",
                CultureInfo.InvariantCulture, DateTimeStyles.AssumeUniversal, out var ts) ||
            !int.TryParse(line[f[3]], NumberStyles.None, CultureInfo.InvariantCulture, out int status) ||
            !long.TryParse(line[f[4]], NumberStyles.None, CultureInfo.InvariantCulture, out long bytes) ||
            !double.TryParse(line[f[5]], NumberStyles.AllowDecimalPoint, CultureInfo.InvariantCulture, out double ms))
            return false;

        entry = new LogLine
        {
            Timestamp = ts, Method = line[f[1]], Path = line[f[2]],
            Status = status, Bytes = bytes, DurationMs = ms,
        };
        return true;
    }
}

public sealed class SpanAggregator
{
    private sealed class PathStats { public long Count; public double TotalMs; }

    private readonly Dictionary<string, PathStats> _stats = new(StringComparer.Ordinal);
    private readonly Dictionary<string, PathStats>.AlternateLookup<ReadOnlySpan<char>> _byPath;

    public SpanAggregator() => _byPath = _stats.GetAlternateLookup<ReadOnlySpan<char>>();

    public void Add(in LogLine e)
    {
        if (!_byPath.TryGetValue(e.Path, out PathStats? s))
        {
            s = new PathStats();
            _byPath[e.Path] = s;             // allocates the key string only for a path seen for the first time
        }
        s.Count++;
        s.TotalMs += e.DurationMs;
    }
}
```

What changed, and why each change is safe: `Split` into a stack-allocated `Range` buffer replaces the array and substrings (Concept 36); `TryParseExact` with a fixed format and the invariant culture replaces general-purpose date parsing, and every `TryParse` returns failure instead of throwing on malformed lines (Concept 48); the entry is a `ref struct` of spans into the original line, so nothing is copied; the dictionary's **alternate lookup** finds the stats for a path span without creating a string (Module 16, Concept 53), and the mutable `PathStats` class avoids re-inserting a tuple on every update. The parsers now also *validate* — the naive version threw on the first malformed line.

**Step 3 — benchmark it, correctly.**

```csharp
[MemoryDiagnoser]
public class LogParsingBenchmarks
{
    private string[] _lines = default!;

    [Params(10_000)]
    public int Count;

    [GlobalSetup]
    public void Setup()
    {
        var rng = new Random(42);
        string[] paths = ["/api/orders", "/api/orders/42", "/api/customers", "/health", "/api/cart"];
        _lines = new string[Count];
        for (int i = 0; i < Count; i++)
            _lines[i] = string.Create(CultureInfo.InvariantCulture,    // invariant: "12.5", never "12,5"
                $"2026-09-22T10:{i % 60:00}:{i * 7 % 60:00}Z GET {paths[rng.Next(paths.Length)]} 200 {rng.Next(100, 5000)} {rng.NextDouble() * 50:F1}");
    }

    [Benchmark(Baseline = true)]
    public int Naive()
    {
        var agg = new NaiveAggregator();
        foreach (string l in _lines) agg.Add(NaiveParser.Parse(l));
        return _lines.Length;
    }

    [Benchmark]
    public int Spans()
    {
        var agg = new SpanAggregator();
        int ok = 0;
        foreach (string l in _lines) if (SpanParser.TryParse(l, out LogLine e)) { agg.Add(e); ok++; }
        return ok;
    }
}
```

Note the trap caught in `Setup`: an interpolated `{value:F1}` formats with the **current culture**, and on a machine with a Serbian (or German, or French) culture it produces `12,5` — which the invariant-culture parser correctly rejects. A benchmark that silently parses nothing looks spectacularly fast. `string.Create(CultureInfo.InvariantCulture, …)` pins the format; returning the count of successfully parsed lines from each benchmark makes such a failure visible.

**What to expect, and how to report it.** The span version should allocate a handful of strings in total (one per distinct path) instead of roughly eight objects per line, and typically runs several times faster — but the ratio depends on your hardware and runtime, so run it and report what *you* measured, with the environment header. Don't quote numbers you didn't measure; in an interview, say "I'd expect several-fold, driven mostly by allocation and the date parsing, and I'd confirm with BenchmarkDotNet."

**Step 4 — the next level, and whether to take it.** The file is UTF-8; `File.ReadLines` transcodes every line to a UTF-16 `string` — one allocation per line, now the dominant remaining cost. Going further means reading bytes through a `PipeReader` or `RandomAccess` with pooled buffers, finding line breaks with `IndexOf((byte)'\n')`, splitting on bytes, parsing with `Utf8Parser`/`IUtf8SpanParsable<T>`, and using an alternate lookup keyed on `ReadOnlySpan<byte>` with a custom UTF-8 comparer. That's perhaps another 2× — and considerably more code to own. Whether it's worth it depends on the job's SLA and cost (Concept 67). Saying that out loud is part of the answer.

---

## Worked example 3 — "A teammate's benchmark shows `HashSet` is 10× faster than an array for our lookups. Do you believe it?"

The benchmark:

```csharp
public class ContainsBenchmarks
{
    private static readonly string[] Words = ["alpha", "bravo", "charlie", "delta", "echo", "foxtrot", "golf", "hotel"];
    private HashSet<string> _set = default!;

    [IterationSetup]
    public void Setup() => _set = new HashSet<string>(Words);

    [Benchmark] public void Array() => Words.Contains("hotel");
    [Benchmark] public void Hash()  => _set.Contains("hotel");
}
```

**The critique, in order of severity:**

1. **`void` benchmarks** — the results are discarded; parts of the work may be eliminated. Return the `bool`.
2. **The lookup key is the same interned literal as the stored element.** `"hotel"` in the benchmark and `"hotel"` in the array are the *same string object*, so equality succeeds on the reference-equality fast path without comparing characters. Production keys are parsed from input — different instances — and must be compared character by character. Both sides are flattered, unevenly.
3. **One key, always present, always last in the array.** The array search is at its worst case every time, the branch predictor learns the pattern perfectly, and misses — perhaps the majority of production lookups — are never measured.
4. **`static readonly` input** — tier-1 can treat the array as a constant, enabling folding that production code wouldn't get.
5. **`[IterationSetup]` on a nanosecond-scale operation** forces one invocation per iteration; the measurement is dominated by timer and harness overhead. (BDN will also warn that the iteration time is very small.)
6. **n = 8 only.** The interesting question — where is the crossover? — can't be answered from one size.
7. **No baseline, no `[MemoryDiagnoser]`, no environment header in the report.**
8. **Which `Contains` is this?** With C# 14's first-class spans, `array.Contains(x)` binds to `MemoryExtensions.Contains` over a span rather than LINQ's `Enumerable.Contains` — the same source measures different code depending on the language version (Module 16, Concept 54). Know what you're benchmarking.

**The rewrite:** instance-field inputs built in `[GlobalSetup]`; `[Params]` over sizes 4, 16, 64, 1,024; lookup keys that are **new string instances** (for example `new string(word.AsSpan())`) mixing hits and misses in random order with a fixed seed; a loop over 1,024 keys per invocation with `OperationsPerInvoke = 1_024`; a returned count of hits; the array version as baseline; `[MemoryDiagnoser]` on. Concept 15's template is exactly this shape.

**What you'd expect to learn:** at very small sizes the array scan is competitive or faster (no hashing, one or two cache lines); the hash set wins increasingly as *n* grows; and a `FrozenSet<string>` built once is likely faster still for read-only data. The honest answer to the teammate is *"the 10× is mostly an artefact of the benchmark; the real answer depends on n and the hit ratio — here's the benchmark that measures it."* Then pick the structure for the sizes production actually has.

---

## Worked example 4 — "Should we move our services to Native AOT?"

A design-round question disguised as a technical one. Structure the answer as a decision, not an opinion.

**1. Classify the estate by the binding dimension (Concept 1):**

| Service type | Binding dimension | AOT verdict |
|---|---|---|
| Public API on MVC controllers + EF Core, long-running, 12 instances | Steady-state throughput, p99 | **No** — MVC unsupported, EF Core AOT experimental; JIT + PGO gives the best steady state. Consider **R2R** for deployment warm-up |
| Webhook receiver scaled from zero on Container Apps | Cold start | **Strong candidate** — minimal API, few dependencies, cold start is the SLO |
| Queue-driven worker, burst scaling | Time-to-ready during bursts, footprint | **Candidate** if its dependencies are AOT-clean |
| Sidecar or agent deployed next to every pod | Footprint × replica count | **Strong candidate** |
| Internal CLI tools | Startup (the whole lifetime) | **Strong candidate** — the .NET 11 CLI made the same choice |

**2. State the costs honestly:** build time and a native toolchain in CI (Linux builds on Linux); a closed world (no plugins, no runtime code generation); a dependency audit — every package must be warning-free; a new class of runtime failure if warnings are suppressed; no dynamic PGO; dumps analysed with native tooling; and team skill with the annotation system.

**3. Propose a pilot, not a migration:** one strong candidate (the webhook receiver). Success criteria written in advance: zero trim/AOT warnings with no unjustified suppressions; the integration suite passing against the native binary; time-to-first-request under 100 ms from container start; RSS under an agreed limit; throughput at the SLO within 10% of the JIT build (Concept 55); image size reduction measured.

**4. Offer the cheap alternative for everything else:** enable **ReadyToRun** for all container services now — no code changes, most of the startup win (Concept 34) — and set minimum replicas above zero where cold start matters more than the standing cost (Concept 63).

**5. Record it as an ADR** (Module 31): context, options (JIT / R2R / AOT / min-replicas), decision per service class, consequences, and the conditions that would make you revisit it — for example, "EF Core AOT support leaves experimental status" or "cold-start SLO introduced for the public API."

**The one-sentence version:** *"AOT is a startup-and-footprint decision, so I'd adopt it where one of those is binding and the dependency graph is clean — starting with one pilot and measured criteria — and turn on ReadyToRun everywhere else, because it gets most of the startup benefit for free."*

---

## Worked example 5 — "We need to handle 20,000 requests per second at p99 under 100 ms. How many instances?"

**1. Measure the knee (Concept 62).** A capacity test on one 4-vCPU instance, stepping load in 200-rps increments with an open-model generator (Concept 6), gives (illustrative): p99 flat around 45 ms up to 2,000 rps; 70 ms at 2,400 rps with CPU ~72%; 160 ms at 2,800 rps. The knee is ~2,400 rps. Choose a **safe operating rate** below it — say 2,000 rps (p99 ~45 ms, CPU ~60%) — to absorb bursts, GC variance, and noisy neighbours.

**2. Steady state:** 20,000 / 2,000 = **10 instances**.

**3. Zone failure:** three zones; losing one leaves two-thirds of the fleet. In a failure you can accept running near the knee (p99 ~70 ms, still inside the SLO) but not beyond it: `(2/3) × N × 2,400 ≥ 20,000` → N ≥ 12.5. Zone symmetry requires a multiple of 3: **15 instances** (5 per zone). Check: 12 (4 per zone) leaves 8 × 2,400 = 19,200 — insufficient.

**4. Deployments:** roll with `maxSurge` and `maxUnavailable: 0` so capacity never dips during a release.

**5. Autoscaling:** minimum 15 at peak hours (or a schedule), scaling on RPS per instance or CPU with the target at the safe rate; scale in off-peak with a floor that still survives a zone loss at off-peak traffic.

**6. Pools via Little's Law:** at a mean latency of 30 ms, fleet-wide concurrency is 20,000 × 0.03 = 600 requests in flight, ~40 per instance. If each request holds a database connection for ~8 ms, that's 20,000 × 0.008 = 160 connections fleet-wide, ~11 per instance — size the pool at roughly 20 per instance for variance, and confirm the database accepts 15 × 20 = 300 connections from this service alongside every other client.

**7. The economics:** 15 × 4 vCPU = 60 vCPU. Now the insight from Concept 62: suppose an optimization raises per-instance knee throughput by 20% (2,880 rps). Recompute: `(2/3) × N × 2,880 ≥ 20,000` → N ≥ 10.4 → **12 instances** (4 per zone). A 20% efficiency gain saved 3 instances (20%) — but a 10% gain (knee 2,640 → N ≥ 11.4 → still 12) saves the same three, and a 5% gain (N ≥ 11.9 → 12) also saves three, while a 3% gain (N ≥ 12.1 → 15) saves nothing. **Capacity is quantized; know where the next step is before selling an optimization on cost.**

**8. State the sensitivities:** this plan assumes the dependencies (database, downstream services) scale with it; the per-instance numbers came from a test with realistic data and cache hit ratios; and the knee must be re-measured after significant releases and runtime upgrades (Concept 66).

---

## Common questions and what a strong answer contains

**"How do you approach a performance problem?"** The loop: define the metric and target, measure a baseline in a production-like environment, locate with a profiler or trace, predict the effect of a change, change one thing, re-measure, keep or revert, record — and a hierarchy that looks at work avoided and round trips before allocations and instructions (Concepts 2–3).

**"Why not use average latency?"** Latency distributions are skewed and multimodal; the mean can describe no real request and hides the tail users feel. Report percentiles with load and window — and remember percentiles can't be averaged across instances or time; merge histograms (Concepts 4–5).

**"What is coordinated omission?"** A closed-loop load generator stops sending requests while the system is stalled, so the stall is recorded as a few slow samples instead of every request that would have arrived. Use open-model, constant-arrival-rate load for internet-facing services (Concept 6).

**"Why not run at 90% CPU?"** Queueing: mean response time ≈ S/(1−ρ), so 90% utilization means ~10× service time and a much worse p99. Target utilization comes from the latency SLO plus failure and scaling headroom (Concepts 7, 62).

**"How does BenchmarkDotNet avoid the pitfalls of a Stopwatch loop?"** Separate processes per benchmark, Release builds, a pilot stage to size iterations, overhead measurement and subtraction, warm-up past tier-0, consumed return values, forced GC between iterations, and statistics with confidence intervals (Concepts 13–14).

**"What does the Error column mean?"** Half the width of the 99.9% confidence interval of the mean. Differences smaller than the combined error — or smaller than your pre-agreed practical threshold — are not results (Concepts 17–18).

**"What can't a micro-benchmark tell you?"** End-to-end impact (Amdahl), behaviour under concurrency, GC interplay with a real heap, real cache pressure, and I/O. Micro justifies the change; load tests and production metrics justify the claim (Concept 20).

**"CPU is low but requests are slow — what now?"** It's waiting, not computing: wall-clock/thread-time analysis, distributed traces for the critical path, ThreadPool and pool-wait metrics, `dotnet-stack` for pile-ups. A CPU profile won't show blocked time (Concepts 24, 8; Module 15).

**"What is dynamic PGO and does it change how you write code?"** Tier-0 instrumentation records types, delegates, and branches; tier-1 uses them for guarded devirtualization, inlining, and hot/cold layout. It makes monomorphic abstractions cheap, so don't hand-devirtualize — but benchmarks must reproduce production's type mix, and `AggressiveOptimization` switches it off (Concepts 27–28).

**"Why seal classes?"** It gives the JIT exact-type knowledge — direct calls, cheaper type checks, better inlining — costs nothing, and states design intent. CA1852 enforces it for internal types (Concept 30).

**"How do you know whether a bounds check was eliminated?"** Read the disassembly — `[DisassemblyDiagnoser]` or `DOTNET_JitDisasm` — and look for `CORINFO_HELP_RNGCHKFAIL`. And on .NET 11, rewrite unsafe loops that exist only to avoid bounds checks back into safe code and measure; the gap is usually gone (Concepts 31, 33).

**"What's the difference between `Span<T>` and `Memory<T>`?"** A span is a stack-only view (a reference plus a length), usable in synchronous code over any contiguous memory; `Memory<T>` is a heap-storable handle you convert to a span at the point of use, for async methods and stored buffers — and it brings ownership rules: a `Memory<T>` parameter is a borrow; `IMemoryOwner<T>` is ownership, disposed exactly once (Concepts 35, 37).

**"What does `System.IO.Pipelines` give you over `Stream`?"** Pooled buffer management, messages that straddle reads handled for you via `ReadOnlySequence<T>`, built-in backpressure, and `AdvanceTo(consumed, examined)` semantics that wait for more data instead of re-reading. It's what Kestrel is built on (Concepts 38, 40).

**"How would you design a high-performance parsing API?"** `ReadOnlySpan<T>` in; a caller-provided span out with a written count, or `IBufferWriter<T>`, or `OperationStatus` for streaming; implement `ISpanParsable`/`IUtf8SpanFormattable`; offer an easy allocating tier on top; `Memory<T>` only for async or retained buffers (Concepts 39, 41, 59).

**"When would you vectorize by hand?"** Rarely: first check `MemoryExtensions`, `SearchValues`, LINQ over arrays, and `TensorPrimitives`. If still needed, use the cross-platform `Vector128/256/512` APIs with a vector body and scalar tail, test every width and the fallback, and only for large contiguous data (Concept 46).

**"Is LINQ slow?"** It allocates iterators and delegates and can't be as tight as a hand-written loop, and the gap has narrowed a lot (vectorized aggregates, escape analysis of enumerators). In a hot path at 100k ops/s, measure and rewrite that method; elsewhere, readability wins (Module 14, Concepts 50, 53).

**"Native AOT or ReadyToRun?"** AOT for startup- and footprint-bound workloads with AOT-clean dependencies — functions scaled from zero, CLIs, sidecars, workers — accepting a closed world and no dynamic PGO. R2R for everything else that restarts often: most of the startup win, no restrictions, larger binaries (Concepts 34, 55; worked example 4).

**"How do you know an app is safe to publish with Native AOT?"** Zero trim/AOT warnings with no unjustified suppressions — that's the equivalence guarantee — plus the integration suite passing against the native binary, and dependencies declared `IsAotCompatible` or verified (Concepts 51–53).

**"How would you make startup faster?"** Measure the phases first — image pull, runtime start, host build, JIT, first requests. Then smaller images, R2R or AOT, less reflection at startup, deferred initialization, warm-up gated by readiness, and a minimum replica count where cold start violates the SLO (Concept 63).

**"How do you prevent performance regressions?"** Allocation assertions and allocation-based benchmark gates in CI (deterministic), A/B timing comparisons on stable hardware tracked as trends, SLO-based load tests before release, and canary analysis in production (Concepts 57, 64).

**"How many instances do we need for X rps?"** Measure the per-instance knee at the SLO, choose a safe operating rate below it, add failure capacity for a zone loss, round to zone symmetry, size pools with Little's Law — and note that improvements save instances in steps (Concept 62; worked example 5).

**"What's new in .NET performance recently?"** .NET 11: more bounds-check elimination, generic-virtual-method devirtualization, extended escape analysis, runtime async (opt-in, with the BCL compiled that way), R2R comparer specialization, faster AOT interface dispatch, an x86-64-v2 baseline and x86-64-v3 R2R target. .NET 10: stack allocation of small arrays and delegates, `collect-linux` tracing. BenchmarkDotNet 0.15: analyzers and a new statistics engine (Concepts 31, 34, 66).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "Make it fast" with no target | Turns it into metric + percentile + load + environment and names the binding dimension |
| Optimizes the code they suspect | Profiles first; predicts the effect size before changing anything |
| Changes five things, then measures | One change per measurement; reverts changes that didn't move the metric |
| Quotes average latency | Quotes p50/p99/p99.9 with load and window; looks at the histogram for modes |
| Averages p99s across instances | Merges histograms; asks "aggregated how?" |
| Ignores fan-out | Computes tail amplification, `1 − (1−p)^n`, and proposes hedging or less fan-out |
| Load-tests with closed-loop virtual users | Uses open-model constant arrival rate; knows coordinated omission by name |
| Sizes for maximum throughput | Sizes for throughput at the SLO (the knee) plus zone-failure and burst headroom |
| Runs "efficiently" at 85–90% CPU | Knows R = S/(1−ρ) and derives target utilization from the SLO |
| Reasons only in Big-O | Adds the memory hierarchy: cache lines, pointer chasing, prefetching, and measured crossovers |
| Stopwatch loop in Debug | BenchmarkDotNet in Release; can list ten ways the Stopwatch loop lies |
| `void` benchmarks with constant inputs | Returns results; inputs from instance fields set in `[GlobalSetup]` |
| Benchmarks with one input, always found, same string instance | Realistic sizes, hit ratios, fresh instances, production type mix |
| `[IterationSetup]` on nanosecond operations | Batches with `OperationsPerInvoke`; makes operations repeatable |
| Declares a 3% difference a win | Checks Error and RatioSD, sets a threshold in advance, reproduces |
| Treats a micro-benchmark win as a service win | Applies Amdahl; confirms with a load test and production metrics |
| Only ever uses CPU profiles | Uses wall-clock/thread-time and traces for latency; allocation profiles for GC |
| Can't read a flame graph | Distinguishes inclusive and exclusive time; knows width is time, not order |
| Adds `AggressiveOptimization` or `AggressiveInlining` "for speed" | Knows the first disables PGO; uses the second only with a benchmark |
| Hand-devirtualizes with type checks | Knows PGO's guarded devirtualization does this, and seals classes instead |
| Writes `Unsafe.Add` loops to dodge bounds checks | Writes idiomatic loops, checks the disassembly, measures on the current runtime |
| Uses `Memory<T>` everywhere "for flexibility" | Spans for sync code; `Memory<T>` only for async or storage, with documented ownership |
| Stores a borrowed `Memory<T>` in a field | Takes `IMemoryOwner<T>` or copies; can describe the use-after-return bug |
| Parses network data with `Stream` + manual buffers | Uses pipelines or at least `SequenceReader<T>` with a single-segment fast path |
| `new Regex(...)` per call; `RegexOptions.Compiled` by habit | `[GeneratedRegex]`, static; `NonBacktracking` for untrusted input |
| New `JsonSerializerOptions` per call | Cached options (CA1869) and source-generated contexts |
| Culture-sensitive string comparisons on identifiers | Ordinal comparisons everywhere for keys and protocol text |
| Exceptions for expected outcomes | Try patterns and result types; watches the exception-rate counter |
| High-cardinality metric tags | Bounded tag values, sampling, `TagList`, a telemetry CPU budget |
| "Let's go Native AOT" as a slogan | Classifies services by binding dimension; pilots with success criteria; R2R elsewhere |
| Suppresses trim warnings to get a green build | Treats warnings as future runtime failures; annotates or removes the reflection |
| Optimizes JIT time when cold start is slow | Measures the phases and finds the image pull or the remote-config call |
| Gates CI on timings from shared runners | Gates on allocations; tracks timings as trends; A/B on stable hardware |
| Pools small, short-lived objects | Knows pooling moves objects to gen2 and adds lifetime bugs; pools buffers, not graphs |
| Treats runtime upgrades as risk only | Measures them as free performance; deletes obsolete workarounds; prices LTS vs STS |
| Optimizes whatever is interesting | Asks what the change is worth, and is willing to say "leave it alone" |

---

## Practice exercises

**Exercise 1 — The lying Stopwatch (45 min).** Write a Stopwatch loop timing `int.Parse` on a literal, then the same measurement in BenchmarkDotNet done properly. Explain every discrepancy using Concept 13's list. Then deliberately break the BDN version three ways — `void` return, constant input, `[IterationSetup]` — and record what each does to the numbers and which warnings BDN prints. This builds the instinct that makes worked example 3 easy.

**Exercise 2 — Cache cliffs (1 hr).** Benchmark summing one field over *n* elements for (a) a class array, (b) a struct array, (c) a struct-of-arrays layout, with `[Params]` from 1 K to 64 M elements. Plot time per element against *n* and mark your CPU's L1, L2, and L3 sizes on the axis. Explain each knee. Then shuffle the class instances in memory (allocate in random order) and watch pointer chasing get worse.

**Exercise 3 — Branch prediction and ILP (45 min).** Filter-and-sum bytes above 128 on sorted vs unsorted data; then a branchless version; then a multiple-accumulator floating-point sum. Measure with `[HardwareCounters(BranchMispredictions)]` on Windows or `perf stat` on Linux, and tie each result to Concept 11.

**Exercise 4 — The parser, end to end (2 hrs).** Implement worked example 2 yourself — naive version, span version, benchmark, allocation test — and then step 4 (UTF-8 bytes with pipelines). Write the PR description you'd submit: numbers, environment, what each change contributed, and whether step 4 is worth maintaining. This is the highest-value exercise in the module.

**Exercise 5 — PGO in the lab (45 min).** A benchmark calling an interface method through a call site that sees 1, 2, 3, and 8 implementations (round-robin). Run with `DOTNET_TieredPGO` on and off and read the disassembly for each. Find where guarded devirtualization stops paying. Then write down what this means for the benchmarks your team has written.

**Exercise 6 — Reading the JIT (1 hr).** Write four loop shapes over a span: canonical `for`, loop to a separate `count`, lookahead `span[i + 1]`, and a list-pattern classifier. Run with `[DisassemblyDiagnoser]` on `net10.0` and `net11.0` and find every `CORINFO_HELP_RNGCHKFAIL`. Explain each difference using Concept 31. Then add `AggressiveOptimization` to one method and observe what changes.

**Exercise 7 — Coordinated omission, demonstrated (1 hr).** A minimal API endpoint that normally takes 5 ms but stalls for 1 second every 10 seconds (a static `SemaphoreSlim` gate or a timer). Load it with a closed-model tool (100 VUs) and with k6's `constant-arrival-rate` at the same average throughput. Compare the p99s and explain the difference in one paragraph. Then turn the stall into a GC-style pause and repeat.

**Exercise 8 — Native AOT, measured (2–3 hrs).** Take a small minimal API with JSON, configuration binding, options validation, logging, and one outbound `HttpClient` call. Publish it as JIT, R2R, composite R2R, and Native AOT in containers. Reach zero trim/AOT warnings without unjustified suppressions. Measure: time from container start to first successful response, RSS under steady load, throughput at a fixed p99, image size, and build time. Write a one-page ADR choosing a model for this service. Inspect the AOT binary with Sizoscope and remove the largest avoidable dependency.

**Exercise 9 — Profiling under load (1.5 hrs).** Run a service under load and capture `dotnet-trace` with `cpu-sampling` and `gc-verbose` (and, on Linux with .NET 10+ and root, `collect-linux`). Open them in PerfView or Speedscope. Identify the top three CPU costs and the top three allocators by stack. For each, state the fix and its predicted effect size before implementing it; then implement one and check the prediction.

**Exercise 10 — A regression gate (1.5 hrs).** Add an allocation test (Concept 57) for a hot path in a real codebase, and a CI step that runs a BDN suite on both the base branch and the PR in the same job, exports JSON, and fails if any benchmark's allocation increases by more than 5% or its time by more than a generous threshold. Deliberately introduce a regression (string interpolation in a log call) and confirm the gate catches it.

**Exercise 11 — A capacity plan (1 hr).** For a service you know, run the knee-finding test from worked example 5, then write the capacity plan: safe rate, steady-state count, zone-failure count, pool sizes via Little's Law, cost per million requests, and the efficiency gain needed to reach the next instance-count step down.

---

## Free resources

### Primary sources — the .NET team's own writing and design documents

| Resource | What it covers | Why read it |
|---|---|---|
| [Performance Improvements in .NET 11](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-11/) (Stephen Toub, September 2026) | Deabstraction, bounds checks, runtime async, libraries — hundreds of changes, each with a BenchmarkDotNet benchmark | **The single highest-value document for this module.** Its benchmarking setup section is also a model of how to compare runtimes |
| [Performance Improvements in .NET 10](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/) · [.NET 9](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/) · [.NET 8](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/) | The same series for earlier releases | A year-by-year history of the JIT and BCL; the .NET 8 post is the best introduction to dynamic PGO |
| [What's new in the .NET 11 runtime](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/runtime) | JIT, hardware intrinsics, R2R, AOT interface dispatch, runtime async, diagnostics | The official current-state summary (Concepts 31, 34, 55, 66) |
| [What's new in the .NET 11 SDK](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/sdk) | Including the Native AOT–compiled CLI host | Concept 55's dogfooding example |
| [Minimum hardware requirements updated (.NET 11)](https://learn.microsoft.com/en-us/dotnet/core/compatibility/jit/11/minimum-hardware-requirements) | x86-64-v2 baseline, new R2R targets | Concepts 34 and 66 |
| [RyuJIT overview](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/jit/ryujit-overview.md) | The JIT's phases and internal representation | Background for all of Part E |
| [Viewing JIT dumps](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/jit/viewing-jit-dumps.md) | `JitDisasm`, `JitStdOutFile`, and related switches | Concept 33 from the source |
| [Tiered compilation design](https://github.com/dotnet/runtime/blob/main/docs/design/features/tiered-compilation.md) | Tiers, call counting, background compilation | Concept 27 |
| [Guarded devirtualization design](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/jit/GuardedDevirtualization.md) | How GDV works and why | Concept 28 |
| [Vectorization guidelines](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/vectorization-guidelines.md) | How the BCL writes vectorized code, with the testing strategy | **Read before writing any `Vector128/256/512` code** (Concept 46) |
| [Memory safety in .NET (design)](https://github.com/dotnet/designs/blob/main/accepted/2025/memory-safety/memory-safety.md) | The accepted design behind the unsafe-code audit and C# 15's preview | Concepts 31 and 42 |
| [dotnet/performance](https://github.com/dotnet/performance) | The .NET team's benchmark suite, tooling, and `ResultsComparer` | Concept 64 at industrial scale |
| [Microbenchmark design guidelines](https://github.com/dotnet/performance/blob/main/docs/microbenchmark-design-guidelines.md) | How the .NET team writes benchmarks | **The best free guide to Concept 15** |

### BenchmarkDotNet and statistics

| Resource | What it covers |
|---|---|
| [BenchmarkDotNet](https://benchmarkdotnet.org/) · [Overview](https://benchmarkdotnet.org/articles/overview.html) | The official site and getting-started guide |
| [Good practices](https://benchmarkdotnet.org/articles/guides/good-practices.html) | The maintainers' own list of pitfalls (Concepts 13, 15) |
| [How it works](https://benchmarkdotnet.org/articles/guides/how-it-works.html) | The stages in Concept 14 |
| [Diagnosers](https://benchmarkdotnet.org/articles/configs/diagnosers.html) · [Jobs](https://benchmarkdotnet.org/articles/configs/jobs.html) | Concepts 16 and 19 |
| [BenchmarkDotNet releases](https://github.com/dotnet/BenchmarkDotNet/releases) | Changelogs: analyzers (0.15.7), OpenMetrics export (0.15.8), Pragmastat statistics (0.15.6) |
| [Andrey Akinshin's blog](https://aakinshin.net/) | BDN's author on robust statistics for performance data — quantiles, shift estimators, multimodality (Concept 18) |
| [Adam Sitnik's blog](https://adamsitnik.com/) | BDN maintainer and .NET performance engineer; spans, pipelines, profiling walkthroughs |

### Methodology, latency, and queueing

| Resource | What it covers |
|---|---|
| [The USE Method](https://www.brendangregg.com/usemethod.html) · [Performance methodologies](https://www.brendangregg.com/methodology.html) | Brendan Gregg's systematic approaches (Concept 8) |
| [Flame graphs](https://www.brendangregg.com/flamegraphs.html) | The inventor's explanation (Concept 23) |
| [The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) | Tom Wilkie's service-level counterpart to USE |
| [Monitoring distributed systems (Google SRE book)](https://sre.google/sre-book/monitoring-distributed-systems/) | The four golden signals; why tails matter |
| [How NOT to Measure Latency](https://www.infoq.com/presentations/latency-response-time/) — Gil Tene | **The talk on coordinated omission and percentiles** (Concepts 4–6). Watch it once; it changes how you read every dashboard |
| [HdrHistogram](http://hdrhistogram.org/) | The histogram design behind Concept 5 |
| [wrk2](https://github.com/giltene/wrk2) | A constant-throughput load generator; its README explains coordinated omission concisely |
| [The Tail at Scale](https://research.google/pubs/the-tail-at-scale/) — Dean & Barroso | Tail amplification and hedged requests (Concept 5) |
| [Interactive latency numbers](https://colin-scott.github.io/personal_website/research/interactive_latency.html) | Module 5's numbers, by year (Concept 9) |

### Hardware and mechanical sympathy

| Resource | What it covers |
|---|---|
| [Algorithms for Modern Hardware](https://en.algorithmica.org/hpc/) — Sergey Slotin | **Free online book; ideal for someone strong in algorithms** — cache-aware data structures, SIMD, branchless code (Part B) |
| [Performance Analysis and Tuning on Modern CPUs](https://github.com/dendibakh/perf-book) — Denis Bakhvalov | Free edition; top-down microarchitecture analysis, hardware counters |
| [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) — Ulrich Drepper | The classic long-form treatment of Concept 9 |
| [Gallery of Processor Cache Effects](https://igoro.com/archive/gallery-of-processor-cache-effects/) — Igor Ostrovsky | Seven short C# experiments that make cache behaviour visible (Exercise 2) |
| [Agner Fog's optimization manuals](https://www.agner.org/optimize/) | Instruction latencies and microarchitecture details |
| [MIT 6.172 Performance Engineering of Software Systems](https://ocw.mit.edu/courses/6-172-performance-engineering-of-software-systems-fall-2018/) | A full free university course: lectures, notes, problem sets |
| [Mechanical Sympathy](https://mechanical-sympathy.blogspot.com/) — Martin Thompson | False sharing, queues, and hardware-aware design (Concept 12) |

### Microsoft Learn — spans, buffers, pipelines, SIMD

| Resource | What it covers |
|---|---|
| [Memory- and span-related types](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/) | The family overview (Concept 35) |
| [Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines) | **The ten ownership rules** condensed in Concept 37 |
| [Work with buffers in .NET](https://learn.microsoft.com/en-us/dotnet/standard/io/buffers) | `IBufferWriter<T>`, `ReadOnlySequence<T>`, `SequenceReader<T>` (Concepts 38–39) |
| [System.IO.Pipelines](https://learn.microsoft.com/en-us/dotnet/standard/io/pipelines) | **The official guide to Concept 40**, including `AdvanceTo` semantics and common mistakes |
| [System.IO.Pipelines: High performance IO in .NET](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/) — David Fowler | The motivation, from the designer |
| [Pipe Dreams, part 1](https://blog.marcgravell.com/2018/07/pipe-dreams-part-1.html) — Marc Gravell | Porting a real Redis client to pipelines — the practitioner's view |
| [All About Span](https://learn.microsoft.com/en-us/archive/msdn-magazine/2018/january/csharp-all-about-span-exploring-a-new-net-mainstay) — Stephen Toub | The original deep explanation of why spans exist |
| [SIMD-accelerated types in .NET](https://learn.microsoft.com/en-us/dotnet/standard/simd) | Vector types (Concept 46) |
| [`TensorPrimitives`](https://learn.microsoft.com/en-us/dotnet/api/system.numerics.tensors.tensorprimitives) · [`SearchValues`](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.searchvalues) · [`System.Collections.Frozen`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.frozen) | API references for Concepts 44–46 |

### Microsoft Learn — libraries, ASP.NET Core, and runtime configuration

| Resource | What it covers |
|---|---|
| [System.Text.Json source generation](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation) | Metadata vs fast-path modes (Concept 47) |
| [Regular expression source generators](https://learn.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-source-generators) · [Regex best practices](https://learn.microsoft.com/en-us/dotnet/standard/base-types/best-practices-regex) | Concept 44 |
| [Regular Expression Improvements in .NET 7](https://devblogs.microsoft.com/dotnet/regular-expression-improvements-in-dotnet-7/) — Stephen Toub | How the regex engines, the source generator, and `NonBacktracking` work |
| [High-performance logging](https://learn.microsoft.com/en-us/dotnet/core/extensions/high-performance-logging) · [Logging source generation](https://learn.microsoft.com/en-us/dotnet/core/extensions/logger-message-generator) | Concept 49 |
| [ASP.NET Core performance best practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices) | The official checklist; Module 18 goes deeper |
| [Runtime configuration: compilation](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/compilation) | Tiered compilation, PGO, R2R, and quick-JIT settings (Concept 27) |
| [Built-in runtime metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-runtime) · [ASP.NET Core metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-aspnetcore) | The metric names in Concept 8 |
| [Azure Well-Architected: Performance Efficiency](https://learn.microsoft.com/en-us/azure/well-architected/performance-efficiency/) | Capacity planning, performance targets, and testing at the architecture level (Part J) |

### Native AOT, trimming, and ReadyToRun

| Resource | What it covers |
|---|---|
| [Native AOT deployment overview](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) | Prerequisites, limitations, publishing (Concept 50) |
| [Optimizing Native AOT deployments](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/optimizing) | Size/speed preferences and instruction sets (Concept 54) |
| [ASP.NET Core support for Native AOT](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot) | **The compatibility table behind Concept 53**, `CreateSlimBuilder`, the `webapiaot` template |
| [Trimming options](https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/trimming-options) · [Fixing trim warnings](https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/fixing-warnings) · [Prepare .NET libraries for trimming](https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/prepare-libraries-for-trimming) | The annotation system in Concept 51, from the source |
| [ReadyToRun compilation](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run) | Composite images, cross-targeting, restrictions (Concept 34) |
| [EF Core: NativeAOT and precompiled queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries) | The experimental status and its limitations (Concept 53) |
| [Sizoscope](https://github.com/MichalStrehovsky/sizoscope) | Binary-size analysis for AOT apps, by a Native AOT team member |
| [Andrew Lock's blog](https://andrewlock.net/) | Practical, detailed posts on Native AOT, source generators, and ASP.NET Core internals |
| [.NET container images (dotnet-docker)](https://github.com/dotnet/dotnet-docker) | `runtime-deps` and chiseled images for AOT binaries |

### Diagnostics and profiling

| Resource | What it covers |
|---|---|
| [.NET diagnostic tools overview](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) | The whole tool family (Concept 22) |
| [dotnet-trace](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) | Profiles, providers, and `collect-linux` (.NET 10+) |
| [dotnet-counters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters) · [EventPipe](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/eventpipe) | Live metrics; the transport and its `user_events` mode |
| [Debug high CPU usage](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-highcpu) | A worked tutorial for Concept 23, on Windows and Linux |
| [dotnet-monitor](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-monitor) | The sidecar and its collection rules (Concept 26) |
| [perfcollect](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/trace-perfcollect-lttng) | Native stacks on Linux before .NET 10 |
| [Visual Studio profiling tools](https://learn.microsoft.com/en-us/visualstudio/profiling/) | CPU usage, .NET object allocation, async, and events tools |
| [PerfView](https://github.com/microsoft/perfview) | The most powerful free .NET profiler; the repository links its tutorials |
| [Application Insights Profiler for .NET](https://learn.microsoft.com/en-us/azure/azure-monitor/profiler/profiler) · [Code Optimizations](https://learn.microsoft.com/en-us/azure/azure-monitor/optimization-insights/code-optimizations-profiler-overview) | Production profiling on Azure (Concept 26) |
| [Azure Monitor OpenTelemetry Profiler for .NET](https://github.com/Azure/azuremonitor-opentelemetry-profiler-net) | The profiler for OpenTelemetry-based Application Insights (SDK 3.x) |
| [ASP.NET Core diagnostic scenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios) — David Fowler | Real-world anti-patterns with explanations; overlaps Modules 15 and 18 |

### Load testing and capacity

| Resource | What it covers |
|---|---|
| [k6 documentation](https://grafana.com/docs/k6/latest/) · [k6 executors](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/) | Scenarios, thresholds, and the arrival-rate (open-model) executors (Concepts 6, 61) |
| [NBomber](https://nbomber.com/) | .NET-native load testing with `Inject` (open) and `KeepConstant` (closed) simulations |
| [Bombardier](https://github.com/codesenberg/bombardier) · [oha](https://github.com/hatoo/oha) | Fast HTTP load generators for quick experiments |
| [Azure Load Testing](https://learn.microsoft.com/en-us/azure/load-testing/) | Managed JMeter/Locust at scale with server-side metrics and CI gates |
| [crank](https://github.com/dotnet/crank) · [aspnet/Benchmarks](https://github.com/aspnet/Benchmarks) | How the ASP.NET team runs its own benchmark infrastructure |
| [TechEmpower Framework Benchmarks](https://www.techempower.com/benchmarks/) | Industry-wide framework comparisons — useful context, easy to over-read |

### Tools

| Tool | What it's for |
|---|---|
| [sharplab.io](https://sharplab.io/) · [Compiler Explorer](https://godbolt.org/) | Lowered C#, IL, and JIT assembly in the browser (Concept 33) |
| [Speedscope](https://www.speedscope.app/) | Flame graphs from `dotnet-trace` and BDN's EventPipe profiler |
| [Ultra](https://github.com/xoofx/ultra) | A free ETW-based sampling profiler for Windows with a Firefox-profiler UI |
| [ObjectLayoutInspector](https://github.com/SergeyTeplyakov/ObjectLayoutInspector) | Actual field layout and padding of your types (Concept 10) |
| [SharpFuzz](https://github.com/Metalnem/sharpfuzz) | Fuzzing .NET parsers (Concept 60) |
| [Mapperly](https://github.com/riok/mapperly) | A source-generated object mapper — an AOT-friendly replacement for reflection mappers (Concept 52) |

### Talks, video, and blogs

| Resource | What it covers |
|---|---|
| [.NET on YouTube (including *Deep .NET*)](https://www.youtube.com/@dotnet) | Stephen Toub's deep dives on spans, async, and performance internals |
| [.NET Conf](https://www.dotnetconf.net/) | Annual performance, runtime, and Native AOT sessions |
| [NDC Conferences](https://ndcconferences.com/) | Free talk recordings on their YouTube channel; search "performance," "Span," "BenchmarkDotNet" |
| [Dotnetos](https://dotnetos.org/) | .NET performance conference and courses; much of it free |
| [Matt Warren's blog](https://mattwarren.org/) | Long-running series on .NET runtime internals and performance |

### Books (not free, listed for completeness)

- **Pro .NET Benchmarking** — Andrey Akinshin. The definitive book on Part C: methodology, statistics, pitfalls, environment effects.
- **Writing High-Performance .NET Code, 2nd edition** — Ben Watson. Practical, measurement-first; some specifics dated, method timeless.
- **Systems Performance, 2nd edition** — Brendan Gregg. The reference for methodology and OS/hardware performance analysis.
- **Pro .NET Memory Management, 2nd edition** — Kokosa, Nasarre, Gosse. The memory side (Module 14) in exhaustive depth.
- **Performance Modeling and Design of Computer Systems** — Mor Harchol-Balter. Queueing theory for engineers (Concept 7), rigorous and readable.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| A performance requirement | Metric + percentile + load + environment; name the binding dimension |
| Your approach | Define → baseline → locate → predict → change one thing → re-measure → keep/revert → record |
| Where to look first | Work avoided > round trips > algorithms > contention > GC > micro-architecture > instructions |
| Amdahl | `1 / ((1−p) + p/s)`; 10× on 3% of the time = 1.03× |
| Average latency | Hides modes and tails; report p50/p99/p99.9/max with load and window |
| Aggregating percentiles | Can't average them; merge histograms (HDR, OTel exponential, Prometheus native) |
| Fan-out | `P(slow) = 1 − (1−p)^n`; 1% × 100 leaves = 63%; hedge, tie, reduce fan-out |
| Coordinated omission | Closed-loop generators stop sending during stalls; use constant arrival rate |
| Utilization | `R = S/(1−ρ)`: 80% → 5×, 90% → 10×; M/M/1 p99 ≈ 4.6 × mean |
| Variability | Kingman: waits scale with `(c_a² + c_s²)/2`; isolate heavy work |
| USE / RED | Utilization, saturation, errors per resource; rate, errors, duration per service |
| Memory hierarchy | L1 ~1 ns, L2 ~4, L3 ~15, DRAM ~100; 64-byte lines; prefetch loves sequential access |
| Data layout | SoA for hot loops over few fields; struct arrays over class arrays; hot/cold splitting |
| Branches | Mispredict ~15–20 cycles; sorted data predicts; branchless isn't always faster |
| ILP | Break dependency chains with multiple accumulators; the JIT won't reassociate floats |
| Auto-vectorization | RyuJIT generally doesn't; vectorization comes from explicit APIs and the BCL |
| Sharing | Shared writes serialize; shard, accumulate locally, batch, snapshot |
| SMT | A vCPU is usually a hyperthread; measure per-instance throughput on the real SKU |
| Stopwatch benchmarks | Tier-0, DCE, constant folding, GC, frequency scaling, order effects, one sample |
| BenchmarkDotNet mechanics | Separate process, pilot, overhead subtraction, warm-up, statistics, forced GC per iteration |
| Benchmark hygiene | Return results; instance-field inputs; realistic sizes, hit ratios, fresh instances, type mix |
| `[IterationSetup]` | Not for sub-100 ms operations; use `OperationsPerInvoke` |
| Diagnosers | Memory always; Disassembly when surprised; Threading, Exception, EventPipe, HardwareCounters |
| Error column | Half the 99.9% confidence interval of the mean |
| Allocated column | Deterministic — the thing to gate CI on |
| ZeroMeasurement | You measured an empty method; the work was eliminated |
| Comparing results | Same session, threshold set in advance, robust statistics, reproduce |
| Comparing runtimes | `--runtimes net10.0 net11.0` in one run |
| Micro vs macro | Micro justifies the change; load tests and production justify the claim |
| Profiler types | Sampling (where), tracing (why/when), instrumentation (exact counts) |
| Low CPU, high latency | Waiting — thread-time analysis, traces, pool metrics; not a CPU profile |
| Flame graphs | Width = time, not order; plateaus = hot leaves; diff before/after |
| EventPipe sampling | Managed stacks, safe-point bias; ETW/perf/`collect-linux` for native and precise |
| `collect-linux` | .NET 10+, root, kernel 6.4+, `user_events`: managed + native + kernel in one trace |
| Allocation profiling | `GCAllocationTick` every ~100 KB; by type and stack; pair with survival |
| Production profiling | App Insights Profiler + Code Optimizations (SDK 2.x); Azure Monitor OTel Profiler (SDK 3.x); `dotnet-monitor` rules |
| Tiering | Tier-0 (instrumented) → tier-1 after ~30 calls; OSR for loops; hot R2R code re-jitted |
| `AggressiveOptimization` | Skips tier-0 → no PGO → often slower; not R2R-precompiled |
| Dynamic PGO | Guarded devirtualization, hot/cold layout; per process, lost on restart |
| Inlining | Enables everything else; blocked by virtual calls, EH, `stackalloc`, size; use throw helpers |
| Sealing | Exact types for the JIT; CA1852; sealed by default |
| Bounds checks | Idiomatic loops + disassembly (`RNGCHKFAIL`); .NET 11 closed many gaps; rewrite unsafe loops safely |
| Struct promotion | Small, readonly, non-address-exposed structs live in registers |
| `SkipLocalsInit` | Removes zeroing; never read before write |
| Reading JIT output | `DisassemblyDiagnoser`, `DOTNET_JitDisasm`; look for helpers, indirect calls, spills |
| ReadyToRun | Precompiled start tier, lower code quality, larger files; .NET 11 R2R targets x86-64-v3 |
| Span | A view over any contiguous memory; slicing is free; stack-only by design |
| `Memory<T>` | Heap-storable handle; a parameter is a borrow; `IMemoryOwner<T>` is ownership |
| `ReadOnlySequence<T>` | Segmented data; `SequenceReader<T>`; fast-path the single segment |
| Output APIs | Try pattern with a destination span, `IBufferWriter<T>`, `OperationStatus` |
| Pipelines | Pooled buffers, backpressure, `AdvanceTo(consumed, examined)`; buffers valid until the next advance |
| API design | Span in, span out, count back; two tiers; `Memory<T>` only for async/storage |
| `stackalloc` | Constant bound with a pooled fallback; overflow can't be caught |
| Unsafe code | Needs a benchmark on the current runtime, a stated contract, and fuzz tests |
| Text | UTF-8 end to end; `u8` literals; `IUtf8SpanFormattable`; ordinal comparisons |
| `SearchValues` | Create once, static; chars, bytes, and (.NET 9) strings |
| Regex | `[GeneratedRegex]` by default; never construct per call; `NonBacktracking` for untrusted input |
| Collections | Presize; arrays win at small n; frozen for read-mostly; `ImmutableList` is a tree |
| Vectorization | BCL and `TensorPrimitives` first; vector body + scalar tail; test every width |
| JSON | Cached options (CA1869), source generation, UTF-8 and streaming |
| Exceptions | Microseconds each; Try patterns; watch the exception-rate counter |
| Telemetry | `[LoggerMessage]`, sampling, bounded tag cardinality, `TagList` |
| Native AOT | ILC whole-program compile + trimming + embedded runtime; closed world |
| Trim warnings | Zero warnings = same behaviour as JIT; suppressing them is how outages happen |
| AOT readiness | `PublishAot`/`IsAotCompatible`, `CreateSlimBuilder`, source generators, test the native binary |
| AOT ecosystem | Minimal APIs, gRPC, STJ-gen: yes; MVC, Blazor Server: no; EF Core: experimental |
| AOT knobs | `OptimizationPreference`, `IlcInstructionSet`, `InvariantGlobalization`; keep EventSource and stack traces |
| AOT profile | Tens of ms to start, low RSS, flat post-deploy p99; no dynamic PGO; whole-program devirtualization |
| AOT diagnostics | EventPipe tools work; dumps via native debuggers; keep symbols |
| Allocation budgets | Bytes per operation as a tested contract; allocation per request as a production metric |
| Steady-state-zero | Allocate at startup/connection; reuse on the request path; only for hot infrastructure |
| Load tests | Open model, realistic data, warm-up excluded, generator checked, environment parity |
| Capacity | Knee at SLO, safe rate below it, zone-failure headroom, zone symmetry, Little's Law for pools |
| Quantized savings | Instance counts step; know the gain needed for the next step |
| Cold start | Measure phases; image pull often dominates; R2R/AOT; readiness-gated warm-up; min replicas |
| Regression gates | Gate allocations; track timings; A/B in one job; canary analysis |
| Budgets | Decompose the SLO along the critical path; cost per million requests |
| Runtime upgrades | Free, measurable gains; LTS vs STS; delete obsolete workarounds |
| When not to optimize | Not binding, not worth it, or a bigger lever exists: remove > reduce > batch > speed up |

---

## Progress

Module 17 complete — **Phase 4 has two modules left.** Modules 14 and 15 described what the runtime does with your memory and your time; Module 16 described the code you write; this module gave you the discipline for measuring all three and deciding what, if anything, to change.

This module closes several loops from earlier phases:

- **Module 5's latency numbers** became a cost model (Concept 9) and a leverage hierarchy (Concept 3): one avoided round trip outweighs every micro-optimization in Parts B–I.
- **Module 6's Little's Law, USL, and autoscaling** now have their quantitative companions — M/M/1 and Kingman for target utilization (Concept 7), the physical meaning of USL's contention and coherency terms (Concept 12), and knee-based capacity planning with zone redundancy (Concept 62).
- **Module 10's caching** is confirmed as level 1 of the hierarchy — removing work — and gains a GC cost from Module 14 and a measurement method from this module.
- **Module 13's hedging and timeouts** reappear as the architectural answer to tail amplification under fan-out (Concept 5).
- **Module 14's allocation cost model** became engineering practice: allocation budgets as tested contracts, steady-state-zero design, and API shapes that prevent allocation (Part I).
- **Module 15's runtime async and ThreadPool analysis** gained the profiling side — thread-time analysis and .NET 11's lightweight async-profiler events (Concept 24).
- **Module 16's performance-shaped features and AOT motivation** — spans, `allows ref struct`, `params` spans, source generators, interceptors — now have their full practice in Parts F and H.

Threads left open on purpose:

- **ASP.NET Core's request pipeline costs** — Kestrel, middleware ordering, DI resolution and lifetime costs at startup and per request, output caching, response compression — are **Module 18**.
- **EF Core performance** — the N+1 problem, change-tracking cost, compiled and precompiled queries, and why EF Core is the usual AOT blocker — is **Module 19**.
- **Polly's hedging strategy** as the implementation of Concept 5's mitigation is **Module 25**.
- **Compute-platform cold start** — App Service vs AKS vs Functions vs Container Apps, minimum replicas, and where Native AOT and R2R fit each — is **Module 26**.
- **Histograms, SLOs, exemplars, continuous profiling, and canary analysis** as an observability platform are **Module 28**.
- **Cost per request, build-vs-buy of performance, and performance debt** in executive language are **Module 33**.

Next in the curriculum: **Module 18 — ASP.NET Core internals** (the middleware pipeline, the DI container and lifetime pitfalls — singleton, scoped, transient — and minimal APIs vs controllers), which picks up the captive-dependency problem from Module 15, the startup phases from Concept 63, and the AOT constraints from Part H.
