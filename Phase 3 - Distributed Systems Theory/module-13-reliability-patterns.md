# Module 13 — Reliability Patterns
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **every reliability pattern is a decision about what to do with work you cannot finish right now — wait for it, repeat it, refuse it, isolate it, route it elsewhere, or do something cheaper instead — and every one of those decisions can turn a small failure into a large one if it is made badly.**

That reframing matters because the naive picture of reliability is additive: add a retry, add a circuit breaker, add a second region, and the system gets more reliable. The real picture is that each mechanism changes the *dynamics* of the system under stress. A retry is extra load delivered precisely when the dependency is weakest. A timeout that is too long holds threads hostage; one that is too short manufactures failures. A failover path you have never exercised is a second, untested system that you will meet for the first time during the worst hour of the year. Senior candidates recite patterns. Architects explain the feedback loops each pattern creates and how they bound them.

Everything from Phase 3 lands here, because this module formalizes the failure-handling tactics the previous modules introduced piecemeal. Module 6 introduced load shedding, backpressure, and "three layers of three retries is 27 attempts." Module 7 showed that availability and consistency trade against each other under partition. Module 8 gave you replication topologies, which are what "active-active" and "active-passive" *are* at the data layer. Module 9 gave you leader election and fencing tokens, which are what makes a failover safe. Module 10's cold-cache death spiral is a metastable failure. Module 11 introduced retries, backoff, jitter, dead-lettering, and the retry-amplification failure mode. Module 12 closed with backups, PITR, and RPO/RTO. This module ties them into one model.

This module has seven jobs:

1. **Install a precise vocabulary of failure.** Reliability vs availability vs resilience; fault vs error vs failure; crash vs omission vs timing vs gray failure; correlated vs independent. Interviews are lost on imprecise words ("the service was down") far more often than on missing patterns.
2. **Make availability arithmetic reflexive.** Nines to minutes, series and parallel composition, MTBF and MTTR, and the uncomfortable result that your service cannot be more available than its critical dependencies allow.
3. **Teach the four request-level patterns properly** — timeouts, retries, circuit breakers, bulkheads — including their configuration, their interactions, the order in which they compose, and the specific way each one can cause an outage.
4. **Name the dynamics that cause large outages.** Cascading failure, retry amplification, metastable failure, gray failure, correlated failure, and change as the dominant trigger. Recent real incidents (AWS us-east-1 October 2025, Azure Front Door October 2025, Cloudflare November 2025, Google Cloud June 2025) are used as evidence, not decoration.
5. **Make active-active vs active-passive a real decision** rather than a preference — with the data layer, capacity headroom, failover mechanics, control-plane dependencies, and DR tiers made explicit.
6. **Cover verification.** Safe deployment, chaos engineering, DR drills, and observability for the resilience mechanisms themselves — because an untested resilience mechanism is a hypothesis.
7. **Map it all onto .NET and Azure** — Polly v8, `Microsoft.Extensions.Http.Resilience`, the resilience already built into the Azure SDKs, EF Core and SqlClient (and the double-retry trap that creates), ASP.NET Core rate limiting and health checks, and the Azure reliability surface service by service. Module 25 goes deep on Polly; this module gives you the theory Polly implements and the architectural decisions around it.

Seven framings to carry through:

1. **Failure is the normal case at scale.** Design for partial failure first and treat the happy path as a special case of it.
2. **Every resilience mechanism is load.** Retries, hedges, health checks, failover warm-up, and cache refills all consume capacity. Budget them explicitly.
3. **Slow is worse than dead.** A crashed dependency fails fast; a slow one holds your resources hostage. Most cascading failures start with latency, not errors.
4. **Contain before you recover.** Bulkheads, cells, and shedding limit the blast radius; retries and failover try to recover. Containment is the one that saves you when recovery misfires.
5. **Correlation defeats redundancy.** Two replicas that share a config push, a certificate, a DNS name, or a control plane are one replica.
6. **Change is the most common trigger.** Deployments and configuration pushes cause more outages than hardware. Progressive exposure is a reliability pattern.
7. **Untested recovery does not exist.** A failover you have not rehearsed has an unknown RTO and an unknown probability of working.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Four words | Reliability = correct over time; availability = fraction of time usable; resilience = recover and degrade gracefully; durability = data survives |
| 2 | Fault, error, failure | A fault is a cause, an error is a bad internal state, a failure is visible to the user; fault tolerance stops the chain before the last step |
| 3 | Failure modes | Crash, omission, timing, Byzantine — and fail-slow, which is the one that causes cascades |
| 4 | Gray failure | The component looks healthy to its monitors but not to its users; differential observability is the defining property |
| 5 | Availability arithmetic | Nines to minutes; A = MTBF/(MTBF+MTTR); series multiplies, parallel multiplies unavailability; MTTR is the cheaper lever |
| 6 | Dependencies cap availability | A service is bounded by its critical dependencies; each critical dependency needs roughly an extra nine |
| 7 | Failure domains and correlation | Process → host → rack → zone → region → provider → control plane → config; redundancy only works across independent domains |
| 8 | Change causes outages | Deployments and config pushes are the leading trigger; propagation speed is blast radius |
| 9 | Cascading failures | Resource exhaustion + positive feedback; latency is the usual carrier |
| 10 | Metastable failure | A trigger pushes the system into a bad state that persists after the trigger is gone; goodput collapses while throughput stays high |
| 11 | Timeouts, and choosing them | Every remote call needs one; derive it from the dependency's latency distribution and the caller's budget |
| 12 | Kinds of timeout in .NET | Connect, per-attempt, total, idle, pool lifetime; `HttpClient.Timeout` defaults to 100 s |
| 13 | Deadline propagation | Pass the remaining budget downstream; never let a callee work longer than the caller will wait |
| 14 | Timeouts create ambiguity | A timeout does not mean "it didn't happen" — it means "I don't know"; idempotency is the price of retrying |
| 15 | Classify before retrying | Transient, permanent, ambiguous; HTTP and gRPC semantics decide which is which |
| 16 | Backoff | Constant, linear, exponential, capped — spread retries out in time so the dependency can recover |
| 17 | Jitter | Randomize the delay; full or decorrelated jitter prevents synchronized waves |
| 18 | Retry amplification | Retries multiply across layers (attempts^depth); retry at one layer and signal "don't retry" upward |
| 19 | Retry budgets | Cap retries as a fraction of traffic with a token bucket; the retry rate falls as the failure rate rises |
| 20 | Retry safety | Only idempotent operations are safe to retry; idempotency keys make unsafe operations safe |
| 21 | Where retries live | In-process, SDK, broker redelivery, workflow — pick deliberately and avoid stacking them |
| 22 | Server-directed backoff | 429/503 + `Retry-After` is the dependency telling you its capacity; honour it over your own schedule |
| 23 | Circuit breaker | Stop calling a failing dependency to fail fast and let it recover; closed → open → half-open |
| 24 | Trip conditions | Consecutive failures vs failure ratio over a window with minimum throughput; slow calls count |
| 25 | What counts, and probing | Count dependency failures and timeouts, never caller errors; half-open lets a trickle through |
| 26 | Breaker scope | Per dependency, per endpoint, per partition, per tenant — too coarse turns partial failure into total failure |
| 27 | When the circuit is open | Fail fast, degrade, serve stale, queue for later — decide per flow, not per library |
| 28 | The critique of breakers | Modal behaviour is hard to test; token buckets and outlier ejection solve parts of the problem without the cliff |
| 29 | Composition order | Rate limit → total timeout → retry → breaker → attempt timeout; order changes semantics |
| 30 | Bulkheads | Partition resources so one dependency or workload cannot exhaust what everyone shares |
| 31 | Bulkheads in .NET | Semaphores and concurrency limiters, per-client connection pools, not thread pools |
| 32 | Architectural bulkheads | Per dependency, tenant, priority, and workload — separate pools, queues, deployments |
| 33 | Cells and stamps | Independent full copies of the stack serving disjoint customer sets; the router must be thin |
| 34 | Shuffle sharding | Give each customer a random subset of workers; combinatorics shrink the blast radius of a poison tenant |
| 35 | Load shedding and criticality | Reject early, cheaply, and by priority; overload is a product decision about who waits |
| 36 | Adaptive concurrency and queues | Limits that learn from latency; LIFO and CoDel under overload; bounded queues always |
| 37 | Four words for "too much" | Rate limiting, throttling, load shedding, backpressure — different owners, different signals |
| 38 | Hedged requests | Send a second request after the p95 to cut the tail; budget it and only hedge idempotent work |
| 39 | Fallbacks and degradation | Degrade non-critical features deliberately; beware fallbacks that are untested second systems |
| 40 | Static stability | Keep working without making changes during a failure; pre-provision, do constant work |
| 41 | Health checks | Liveness, readiness, startup; shallow vs deep; fail open when everything looks unhealthy |
| 42 | Crash-only and self-healing | Make restart the recovery path; supervise, restart, replace — but bound restart storms |
| 43 | Redundancy vocabulary | N+1, N+2, 2N; surviving the loss of one of N domains needs N/(N−1) capacity |
| 44 | Active-passive | Cold, warm, hot standby; RTO = detect + decide + promote + redirect + warm |
| 45 | Active-active | Every site serves; requires a data strategy, capacity for failover, and routing |
| 46 | Choosing between them | Decide from RPO/RTO, data conflict tolerance, cost, and team maturity — hybrids are normal |
| 47 | Failover mechanics | Detection without flapping, a decision owner, fencing, redirection, warm-up, and failback |
| 48 | Zones vs regions | Zones allow synchronous replication and automatic failover; regions usually mean async and a decision |
| 49 | DR strategy ladder | Backup & restore → pilot light → warm standby → multi-site active-active, priced against RPO/RTO |
| 50 | Multi-region data | The replication mode is the RPO; the write topology is the conflict model |
| 51 | Control plane vs data plane | Recovery must not depend on the control plane; the data plane should keep working alone |
| 52 | Global routing | The global entry point is a single point of failure; alternate ingress is expensive and usually unnecessary |
| 53 | Hidden shared dependencies | DNS, identity, secrets, certificates, registries, pipelines, status pages — the things both regions share |
| 54 | Safe change | Progressive exposure, bake time, automated rollback, feature flags off by default, config as code |
| 55 | Chaos and drills | Hypothesis-driven fault injection, game days, and real failovers on a schedule |
| 56 | Observing resilience | Measure retries, breaker transitions, rejections, hedges, and failover time — not just errors |
| 57 | FMEA and the health model | Enumerate flows and failure modes; classify dependencies as critical or degradable |
| 58 | Polly v8 | `ResiliencePipeline`, strategies, the registry, telemetry, and chaos strategies |
| 59 | `Microsoft.Extensions.Http.Resilience` | Standard and hedging handlers, their defaults, and how to override them |
| 60 | Resilience you already have | Azure SDK, EF Core, SqlClient, Cosmos, Service Bus — and the double-retry trap |
| 61 | Server-side resilience in ASP.NET Core | Rate limiting, concurrency limits, health checks, Kestrel limits, graceful shutdown |
| 62 | The Azure reliability surface | Zones, SLAs, and the HA/DR features of SQL, Cosmos, Service Bus, Storage, Redis, Front Door |
| 63 | Azure Well-Architected Reliability | RE:01–RE:10 as a review vocabulary, and the mission-critical guidance |
| 64 | Anti-patterns | The mistakes that turn resilience into the outage |
| 65 | The reliability review | A per-dependency and per-service checklist you can narrate |
| 66 | When not to | Most systems do not need multi-region active-active, global ingress redundancy, or a breaker on every call |

---

# Part A — The vocabulary of failure

## Concept 1 — Four words: reliability, availability, resilience, durability

These four words are used interchangeably in casual conversation. In an architecture review they mean different things, and using them precisely is a cheap, reliable seniority signal.

**Reliability** is the probability that a system performs its intended function *correctly* over a period of time. A system that is always reachable but returns wrong balances is available and unreliable. Reliability is the umbrella term — the Azure and AWS Well-Architected "Reliability" pillars include everything in this module.

**Availability** is the fraction of time (or of requests) for which the system is usable. It is usually expressed in nines and measured either as **time-based** availability ("the service was up 99.95% of minutes") or **request-based** availability ("99.95% of valid requests succeeded"). Request-based is more honest for large services, because a system that fails 5% of requests for an hour is neither "up" nor "down." Google's SRE practice, and most modern SLOs, use request-based availability.

**Resilience** is the ability to *absorb* faults, *degrade* gracefully, and *recover* to full function. Resilience is about behaviour under stress: does the system shed optional work, isolate the failing part, and come back without a human restarting things in the right order? Two systems can have identical availability over a year and very different resilience — one had many small, contained blips; the other had one catastrophic outage that took eight hours of manual recovery.

**Durability** is the probability that data, once acknowledged, is not lost. It is a separate axis from availability: a storage account can be unavailable for an hour and lose nothing (high durability, reduced availability), or be available while silently corrupting data. Azure Storage quotes durability separately from availability for exactly this reason — LRS, ZRS, and GRS differ mostly in durability and in which failures they survive.

Two related terms you should also use precisely:

- **High availability (HA)** handles *expected* component failures (an instance, a disk, a zone) automatically, within the normal design, usually in seconds to minutes.
- **Disaster recovery (DR)** handles *disasters* — loss of a region, a destructive bug, ransomware, operator error — through a plan with a stated RPO and RTO, usually involving a decision. HA does not replace DR, and replication does not replace backups (Module 12, Concept 54): a `DELETE` without a `WHERE` replicates perfectly.

**The interview-grade sentence:** *"I'll separate four things: availability, which is the fraction of requests that succeed; durability, which is whether acknowledged data survives; resilience, which is how the system degrades and recovers; and correctness, which is whether the answers are right. The patterns for each are different, and they trade against each other."*

---

## Concept 2 — Fault, error, failure: the chain you are trying to break

The dependability literature (Avižienis, Laprie, Randell, and Landwehr's taxonomy) gives three words that make incident discussions precise:

- A **fault** is the underlying cause: a bug, a bad disk sector, a misconfigured timeout, a network partition, an expired certificate.
- An **error** is the incorrect internal state that a fault produces when it is activated: a corrupted cache entry, a thread blocked forever, a connection pool with zero free connections.
- A **failure** is when the error becomes visible at the service boundary: a user receives a 500, a wrong answer, or nothing at all.

The chain is **fault → error → failure**, and it is recursive: the *failure* of a dependency is a *fault* from the perspective of its caller. That recursion is exactly how cascading failures travel (Concept 9).

This gives you three places to intervene, which map cleanly onto the patterns in this module:

| Intervention point | Goal | Examples |
|---|---|---|
| **Fault prevention** | Stop faults from being introduced | Code review, type systems, safe deployment, config validation, capacity planning |
| **Fault tolerance** | Stop faults from becoming failures | Redundancy, retries, timeouts, circuit breakers, bulkheads, failover |
| **Fault removal / forecasting** | Find faults before they activate | Testing, chaos engineering, FMEA, load testing, game days |

**Fault tolerance is the important mental shift.** At scale, faults are not exceptional. With 10,000 disks and an annual failure rate of 1–2%, you replace a disk most days. With thousands of instances, several are unhealthy at any moment. A design that assumes components do not fail is not a design that works "most of the time" — it is a design that is broken somewhere continuously.

A second useful distinction from the same body of work: **fail-stop** components halt cleanly and others can detect that they halted; **fail-silent** components stop producing output without announcing it. Most real components are neither; they degrade (Concepts 3 and 4). A very effective design principle is to *convert* messy failures into fail-stop ones — crash the process on an unexpected invariant violation rather than limping on — which is the essence of crash-only design (Concept 42).

A finding worth quoting: the 2014 OSDI study *Simple Testing Can Prevent Most Critical Failures* (Yuan et al.) analysed catastrophic failures in Cassandra, HBase, HDFS, Hadoop MapReduce, and Redis and found that the large majority were caused by **incorrect handling of non-fatal errors that the software had already detected** — empty catch blocks, `// TODO` handlers, over-broad aborts. The fault was caught; the error handling turned it into a failure. That is an argument for reviewing error paths as carefully as happy paths, and for testing them deliberately.

---

## Concept 3 — The failure-mode taxonomy, and why "fail-slow" is the dangerous one

Classic distributed systems theory classifies how a component can fail, from most benign to most malicious:

| Mode | What happens | How it presents | Handled by |
|---|---|---|---|
| **Crash (fail-stop)** | The component halts and stays halted | Connection refused, no heartbeat | Redundancy + failure detection |
| **Crash-recovery** | Halts, then restarts — possibly with lost volatile state | Gaps, duplicates after restart | Durable state, idempotency (Module 11) |
| **Omission** | Some messages are dropped (send or receive) | Timeouts, missing responses, lost acks | Timeouts, retries, acks, sequence numbers |
| **Timing / performance** | Responses arrive, but too late | Latency spikes, p99 blowouts | Timeouts, hedging, load shedding |
| **Byzantine** | Arbitrary behaviour, including wrong or inconsistent answers | Corrupt data, conflicting responses | Checksums, validation, BFT protocols (rare outside blockchains) |

In a cloud system, the Byzantine category is mostly *accidental* rather than malicious: a bit flip in memory, a bad CPU core that computes wrong results ("silent data corruption"), a misconfigured node returning stale data, a deployment with a logic bug. Checksums end to end, input validation at trust boundaries, and "treat configuration as untrusted input" (Concept 8) are the practical defences.

**Fail-slow is the one that causes large outages.** A crashed dependency is almost friendly: connections are refused in milliseconds, callers get an error, and the failure does not consume the caller's resources. A dependency that accepts connections and then responds in 30 seconds instead of 30 milliseconds is devastating:

- Each in-flight call holds a thread, a socket, a connection-pool slot, memory, and often a database transaction or lock in the caller.
- Callers keep sending at the same rate, so concurrent in-flight work grows linearly with the latency: at 1,000 RPS, 30 ms latency means ~30 concurrent calls; 30 s latency means ~30,000 (Little's Law, Module 6 — `L = λW`).
- The caller's own latency grows, so *its* callers start timing out and retrying.

Research on "fail-slow" hardware (Gunawi et al., FAST 2018, *Fail-Slow at Scale*) documented disks, NICs, and SSDs that kept working at a fraction of their normal speed and degraded entire clusters, because the software treated "slow" as "healthy."

**The practical consequence:** your timeouts, your circuit breakers, and your health checks must treat excessive latency as failure. A breaker that only counts exceptions will stay closed while every thread in your process waits on a dying dependency. This is why Resilience4j has a "slow call rate" threshold, why Polly's standard pipeline places the attempt timeout *inside* the circuit breaker (so timeouts count as failures), and why Envoy's outlier detection can eject hosts on latency.

**The interview-grade sentence:** *"I worry more about slow dependencies than dead ones. A dead dependency fails fast; a slow one exhausts my threads and connection pool and turns its problem into mine — so every call has a timeout and my breaker counts timeouts as failures."*

---

## Concept 4 — Gray failure and differential observability

Microsoft Research's HotOS 2017 paper *Gray Failure: The Achilles' Heel of Cloud-Scale Systems* (Huang et al.) named a pattern every large-system operator recognizes: a component is **unhealthy from the perspective of the things that use it, while appearing healthy to the things that monitor it.**

The defining property is **differential observability**: the failure detector and the application observe the component through different paths, and they disagree. Examples:

- A VM responds to the load balancer's health probe on `/health` (a trivial handler) but its disk is so slow that real requests time out.
- A node's heartbeat to the cluster manager travels over a management network that is fine, while the data network drops 5% of packets.
- A service returns 200 OK for every request, but with empty results because a downstream cache is misconfigured.
- An SSD reports no errors but has latency spikes of several seconds (fail-slow, Concept 3).

Gray failures are dangerous in three ways:

1. **They defeat redundancy.** The failure detector does not remove the bad replica, so the load balancer keeps sending it a share of traffic — and with random or round-robin balancing, a single slow node in a fan-out request slows *every* request that touches it.
2. **They are slow to diagnose.** Dashboards are green. The only symptom is user-visible latency or errors that nobody can localize.
3. **They cascade.** The healthy replicas absorb retries from requests that failed on the gray one, and their latency rises too.

Defences, which you should name in an interview:

- **Observe from the client's perspective.** Measure success and latency *as seen by callers*, per backend instance. Client-side outlier detection (Envoy, gRPC, service meshes) ejects instances whose *observed* error rate or latency is anomalous, which closes the observability gap.
- **Make health checks exercise the real path** — but carefully (Concept 41).
- **Use least-outstanding-requests or latency-aware load balancing** (Module 6, Concept 12), which naturally route away from slow instances.
- **Synthetic transactions** that perform a real user flow end to end.
- **Compare peers.** An instance whose p99 is 10× its peers' p99 is suspect even if it passes every check.

**The interview-grade sentence:** *"A lot of real incidents are gray failures: the node passes its health check but fails real traffic. I close that gap by measuring health from the caller's side — per-instance error rate and latency — and ejecting outliers, rather than trusting the component's own report."*

---

## Concept 5 — Availability arithmetic: nines, MTBF, MTTR, series and parallel

You should be able to do these conversions without hesitation:

| Availability | Downtime per year | Downtime per month (30 d) | Downtime per week |
|---|---|---|---|
| 99% | 3.65 days | 7.2 hours | 1.68 hours |
| 99.5% | 1.83 days | 3.6 hours | 50.4 minutes |
| 99.9% | 8.76 hours | 43.2 minutes | 10.1 minutes |
| 99.95% | 4.38 hours | 21.6 minutes | 5.04 minutes |
| 99.99% | 52.6 minutes | 4.32 minutes | 1.01 minutes |
| 99.995% | 26.3 minutes | 2.16 minutes | 30.2 seconds |
| 99.999% | 5.26 minutes | 25.9 seconds | 6.05 seconds |

A useful sanity check: **each additional nine divides the allowed downtime by ten**, and at four nines a single human-driven incident response (page → acknowledge → diagnose → mitigate) typically consumes the entire monthly budget. **Five nines is not achievable with human-in-the-loop recovery**; it requires automatic detection and mitigation measured in seconds.

**MTBF and MTTR.** For a repairable component:

```
A = MTBF / (MTBF + MTTR)
```

where MTBF is the mean time between failures and MTTR is the mean time to recovery (often decomposed as time to detect + time to diagnose + time to mitigate). The key insight is that **the two levers are not equally expensive.** Doubling MTBF usually means better hardware, fewer changes, more testing — expensive and slow. Halving MTTR means faster detection, automated rollback, pre-built failover, and runbooks — usually cheaper and faster. A system that fails every week but recovers in 10 seconds (A ≈ 99.998%) is more available than one that fails once a year and takes 10 hours to fix (A ≈ 99.89%). This is the argument behind crash-only design, automated rollback, and "recover fast rather than never fail."

**Series composition (all components required).** If a request needs components A, B, and C, and failures are independent:

```
A_total = A_1 × A_2 × … × A_n
```

Five dependencies at 99.9% each give `0.999^5 ≈ 99.50%` — you lost roughly four hours of annual budget to arithmetic alone. Twenty dependencies at 99.9% give ~98.0%. This is why microservice call chains are an availability tax, and why **synchronous depth is an architectural cost** (a direct argument for the asynchronous patterns of Module 11).

**Parallel composition (any one component suffices).** With independent replicas:

```
A_total = 1 − (1 − A_1) × (1 − A_2) × … × (1 − A_n)
```

Two independent replicas at 99% give `1 − 0.01² = 99.99%`. Three give 99.9999%. This is the arithmetic that justifies redundancy — **with the enormous caveat that it assumes independence** (Concept 7). Real replicas share software, configuration, deployment pipelines, and control planes, so the true combined availability is far lower than the formula suggests.

**Partial redundancy (k-of-n).** A quorum system that needs 2 of 3 replicas has availability `3p²(1−p) + p³`. With p = 99%, that is 99.97% — better than a single node, worse than "any 1 of 3," which is the price of consistency (Module 9).

**Composite SLAs in Azure.** Microsoft publishes per-service SLAs, and the Azure Architecture Center explicitly teaches composite SLA calculation using the series and parallel formulas. The important senior caveats: SLAs are *financial commitments with service credits*, not engineering predictions; they exclude many failure types; and the composite of published SLAs is a floor for a well-designed system and a ceiling for a poorly designed one.

**The interview-grade sentence:** *"Five critical dependencies at three nines cap me at about 99.5% before my own bugs. So I either reduce synchronous critical dependencies, make some of them non-critical with degradation, or add redundancy — and I'm explicit that the parallel formula assumes independence I probably don't have."*

---

## Concept 6 — Your service cannot be more available than its critical dependencies

Google's *The Calculus of Service Availability* (Treynor, Dahlin, Rau, and Beyer; ACM Queue, 2017) turns Concept 5 into a design rule that is worth quoting by name in an architect interview.

**The rule of the extra nine.** If your service targets availability A and has a *critical* dependency — one whose failure causes your failure — then that dependency must be substantially more available than A, because your budget must also cover your own bugs, deployments, and other dependencies. The heuristic is that **each critical dependency should offer roughly one more nine than the service that depends on it.** A 99.99% service cannot be built on a 99.9% critical database; the arithmetic is already lost.

This gives you exactly three levers when the dependencies do not support your target:

1. **Make the dependency non-critical.** Degrade gracefully when it fails: serve without recommendations, accept the order and charge the card later, show cached data with a staleness banner. This is by far the most powerful lever and it is a *product* decision as much as an engineering one (Concept 39).
2. **Add redundancy to the dependency.** A second, independent instance of it (a second region, a second provider, a replica with failover) — subject to the correlation problem.
3. **Reduce the frequency and duration of its failures as seen by you.** Caching (Module 10), asynchronous decoupling (Module 11), retries for transient faults, and fast failover.

**Classifying dependencies is the core artefact.** For every flow, list each dependency and mark it:

| Class | Meaning | Required behaviour on failure |
|---|---|---|
| **Hard / critical** | The flow cannot complete without it | Must meet the extra-nine rule, or be made redundant |
| **Soft / degradable** | The flow can complete with reduced function | Timeout short, fallback defined, breaker in front |
| **Asynchronous** | The flow completes and the dependency is used later | Queue in between; its outage becomes a backlog, not an error |

A mature design moves as many dependencies as possible from the first row to the second and third. Azure's Well-Architected guidance on failure mode analysis (Concept 57) asks for exactly this classification per flow.

**A subtle point that distinguishes experienced candidates:** *hidden* critical dependencies are the ones that break the calculation. Your service may call only a database and a cache, but it also depends on DNS, the identity provider for token validation, the secret store at startup, the container registry during scale-out, and the certificate that expires on a Sunday. Concept 53 is about finding those.

**The interview-grade sentence:** *"For a 99.95% target I list each dependency on the critical path and ask whether it's at least 99.99%. If it isn't, I either make it degradable, put a queue in front of it, or make it redundant — and I look for the hidden ones like DNS, identity, and secrets that never show up on the architecture diagram."*

---

## Concept 7 — Failure domains and correlated failure

A **failure domain** is a set of components that can fail together because of a single cause. Redundancy only helps when replicas are in *different* failure domains. The hierarchy most cloud designs reason about:

| Domain | Typical shared cause | Mitigation |
|---|---|---|
| Process | Crash, memory leak, deadlock, thread-pool starvation | Multiple instances; supervisor restart |
| Host / VM | Hardware fault, host OS patch, noisy neighbour | Instances spread across hosts (availability sets, anti-affinity) |
| Rack / fault domain | Top-of-rack switch, power distribution unit | Fault domains; spread constraints |
| Availability zone | Datacenter power, cooling, network | Zone-redundant deployment |
| Region | Regional control plane, regional network, natural disaster | Multi-region with DR or active-active |
| Cloud provider / global service | Global control plane, global config push, global DNS/edge | Multi-provider or alternate path (rarely justified) |
| **Software version** | A bug present in every replica | Staged rollout, version diversity during rollout |
| **Configuration** | A bad value pushed everywhere | Staged config rollout, validation, last-known-good |
| **Operator / automation** | A command or script run against everything | Scoped tooling, blast-radius limits, two-person rules |
| **Time** | Leap second, certificate expiry, license expiry, DST bug | Monitoring expiry dates; testing time-based code |
| **Load** | A traffic spike that hits every replica at once | Headroom, shedding, autoscaling |

The bottom half of that table is where the parallel-availability formula lies to you. Three replicas across three zones are three failure domains for *power* and *hardware*, but **one** failure domain for the software version, the configuration, and the operator's script. Correlated failures dominate in practice. Google's OSDI 2010 study of its storage systems (*Availability in Globally Distributed Storage Systems*, Ford et al.) found that failures were strongly correlated — they arrived in bursts affecting many nodes at once — and that models assuming independence overestimated availability by orders of magnitude.

**Practical techniques to decorrelate:**

- **Deploy software and configuration progressively across failure domains**, never to all at once (Concept 54). The rollout order itself is a bulkhead.
- **Pin replicas to different zones explicitly** (topology spread constraints in Kubernetes, zone-redundant SKUs in Azure).
- **Keep the control plane out of the data path** so a control-plane failure does not take down running workloads (Concept 51).
- **Avoid global singletons**: one global config store, one global feature-flag service, one global DNS zone with no fallback.
- **Stagger time-based work** (jittered cron, jittered TTLs — Module 10) so it does not become a synchronized load spike.

**The interview-grade sentence:** *"Redundancy helps only across independent failure domains. My three zones protect me from a datacenter losing power, but not from a bad config push, so I also treat the deployment pipeline and config rollout as failure domains and stage both."*

---

## Concept 8 — Change is the leading cause of outages

Across public postmortems and internal incident reviews at large providers, the most common trigger of significant outages is not hardware — it is **change**: code deployments, configuration changes, feature enablement, certificate rotation, capacity changes, and automated operations. Google's SRE book states that a large majority of outages it observed were triggered by changes to live systems, and every recent large cloud incident fits the pattern. Four from the last eighteen months are worth knowing in detail, because interviewers increasingly ask about them:

**Google Cloud, June 12, 2025.** A new quota-policy feature had been deployed to Google's Service Control system two weeks earlier, without error handling and without feature-flag protection. A policy change containing unexpected blank fields was then written to regional Spanner tables and replicated globally within seconds; the unprotected code path hit a null pointer and Service Control crashed in a loop in every region. Google's report adds a second lesson that belongs to Concept 18: as tasks restarted in large regions, they created a herd effect on their Spanner dependency because **randomized exponential backoff had not been implemented**, prolonging recovery. Remediations included enforcing feature flags on changes to critical binaries.

**AWS us-east-1, October 19–20, 2025.** A latent race condition in DynamoDB's automated DNS management left the regional endpoint with an empty DNS record. DynamoDB itself recovered in under three hours, but EC2's DropletWorkflow Manager, which depends on DynamoDB to maintain leases on physical hosts, then tried to re-establish leases across the entire fleet at once, took so long that leases timed out again, and entered congestive collapse that needed manual intervention — followed by a network-configuration backlog and Network Load Balancer health-check flapping. The trigger was an automation defect; the long tail was a metastable failure (Concept 10). Running instances kept working while new launches failed — a clean illustration of control plane vs data plane (Concept 51).

**Azure Front Door, October 29, 2025.** An inadvertent tenant configuration change bypassed safety validation because of a software defect in the deployment protection mechanism and propagated globally, causing a large number of AFD nodes to fail to load. Mitigation was to block all configuration changes — including customers' — and roll out a last-known-good configuration in phases. Many Microsoft services that sit behind AFD, including portal and identity experiences, were affected for roughly eight hours. The lesson for architects is Concept 52: a global ingress is a shared failure domain, and during the incident customers could not change its configuration to route around it.

**Cloudflare, November 18, 2025.** A permissions change in a ClickHouse cluster caused a query to return duplicate rows, doubling the size of a machine-learning "feature file" used by the Bot Management module. The file was regenerated every few minutes and pushed to every machine; the proxy software had a hard limit below the new size and failed, producing global 5xx errors for about three hours. Because the ClickHouse change was rolling out gradually, some generations of the file were good and some bad, so the error rate oscillated and the team initially suspected an attack.

**The pattern in all four:** a change in one place, **propagated fast and globally** (Spanner replication, a config push, a feature file refreshed every five minutes), **consumed by code that did not handle unexpected input gracefully**, with no staged exposure to catch it. The generalizable lessons:

1. **Propagation speed is blast radius.** Anything that reaches every region in seconds — configuration, feature flags, ML models, policy data, DNS — needs the same staged rollout as code.
2. **Treat configuration and data from the control plane as untrusted input.** Validate it, bound it, and on failure *keep running with the last known good version* rather than crashing.
3. **New code paths should be dark by default** behind a flag, enabled region by region.
4. **Kill switches ("red buttons") must exist before you need them.** Google's mitigation was a pre-built switch to disable the serving path.

**The interview-grade sentence:** *"Most big outages are changes, and the recent ones are mostly configuration or data that propagated globally in seconds. So I stage config exactly like code, validate it at the consumer, fall back to last-known-good instead of crashing, and put new paths behind flags that start off."*

---

## Concept 9 — Cascading failures: the mechanisms

A **cascading failure** is one that grows through positive feedback: the failure of one part increases the probability that other parts fail. Google's SRE book chapter *Addressing Cascading Failures* is the canonical treatment, and the mechanisms fall into a small number of families.

**1. Resource exhaustion.** The most common carrier. Something makes requests slower; slower requests hold resources longer; resources run out; new requests queue or fail. The resources to name:

| Resource | .NET-specific face |
|---|---|
| Threads | ThreadPool starvation from sync-over-async (`.Result`, `.Wait()`), which Module 15 covers in depth; the pool injects threads slowly, so latency climbs for seconds before recovering |
| Connections | `SqlConnection` pool exhausted (`Max Pool Size` defaults to 100), `HttpClient` sockets, SNAT port exhaustion on Azure outbound connections |
| Memory | Unbounded queues and buffers, large in-flight request bodies, retained response objects; GC pauses lengthen as heap grows |
| CPU | Retry and serialization overhead, TLS handshakes after connection churn, GC under memory pressure |
| File descriptors / ports | Sockets in `TIME_WAIT` from creating an `HttpClient` per request |
| Downstream quota | Cosmos DB RU/s throttling (429s), Azure API rate limits |

**2. Load redistribution.** A replica fails; the load balancer moves its traffic to the survivors; the survivors are now over capacity and fail; their traffic moves again. With N replicas each at utilization u, losing one pushes the rest to `u × N/(N−1)`. At 80% utilization across 5 replicas, losing one pushes the remaining four to 100%. This is why headroom is a reliability requirement (Concept 43), and why aggressive health checks that eject slightly slow instances can *cause* collapse.

**3. Retry amplification.** Failures produce retries, which increase load, which produce more failures (Concept 18). Module 11 introduced this; Concept 18 quantifies it.

**4. Latency propagation.** A slow dependency makes its callers slow, which makes their callers time out and retry, which adds load back onto the slow dependency. In a deep call graph, latency propagates upward faster than errors do.

**5. Cache and state loss.** A cache flush or failover to a cold replica turns a mostly-cached read path into a mostly-origin read path; Module 10's `1/(1−h)` multiplier (a 95% hit rate means the origin sees 20× load when the cache is lost).

**6. Health-check-induced death spirals.** Overloaded instances fail their health checks, are removed, and their load moves to the remaining instances, which then also fail health checks (Concept 41). Kubernetes liveness probes that check dependencies can restart an entire fleet at once.

**7. Startup storms.** After a mass restart, every instance simultaneously warms caches, opens connections, fetches configuration, and JIT-compiles — at the moment the system is least able to absorb it. Google's June 2025 recovery is an example.

**How the patterns in this module break each loop:**

| Mechanism | Primary defence |
|---|---|
| Resource exhaustion | Timeouts, bulkheads, bounded queues, concurrency limits |
| Load redistribution | Headroom, load shedding, fail-open health checks |
| Retry amplification | Single-layer retries, backoff + jitter, retry budgets, circuit breakers |
| Latency propagation | Deadline propagation, aggressive attempt timeouts, hedging (carefully) |
| Cache loss | Static stability, cache warm-up, request coalescing, shedding |
| Health-check spirals | Shallow liveness, readiness with care, fail-open behaviour |
| Startup storms | Jittered startup, slow start at the load balancer, staged restarts |

**The interview-grade sentence:** *"Cascades need a positive feedback loop — usually latency turning into resource exhaustion, plus retries or load redistribution. Every pattern I add is meant to cut one of those loops: timeouts and bulkheads for exhaustion, budgets for retries, headroom and shedding for redistribution."*

---

## Concept 10 — Metastable failures and goodput collapse

The most useful theory for understanding big outages is **metastability**, formalized in *Metastable Failures in Distributed Systems* (Bronson, Aghayev, Charapko, and Zhu; HotOS 2021) and studied empirically in *Metastable Failures in the Wild* (Huang et al.; OSDI 2022).

**Definition.** A metastable failure occurs in an open system (one where load keeps arriving regardless of how the system is doing) when a **trigger** pushes the system from a stable, efficient state into a **bad state that persists even after the trigger is removed**, because a **sustaining effect** keeps it there.

The system has three regimes:

| State | Description |
|---|---|
| **Stable** | Load is below capacity; any perturbation dies out |
| **Vulnerable** | Load is still below *normal* capacity, but a trigger would push the system into a self-sustaining bad state |
| **Metastable (failed)** | The system is busy, doing work, but almost none of it is useful; removing the trigger does not help |

The crucial and counter-intuitive point: **systems are often run in the vulnerable state on purpose**, because it is efficient. A cache that absorbs 95% of reads lets you run a small database — which is exactly what makes the cache-loss trigger fatal.

**Common sustaining effects:**

- **Retries.** Each failure generates more attempts, keeping load above capacity after the original spike is gone.
- **Timeouts with abandoned work.** Clients time out, but the server keeps processing requests nobody is waiting for. **Throughput stays high while goodput — work completed in time to be useful — falls toward zero.** This is the classic queue-based metastable failure Marc Brooker describes.
- **Cache misses.** A cold cache sends load to the origin, which is too slow to refill the cache, which keeps it cold (Module 10, Concept 24).
- **Lease and heartbeat storms.** Leases expire during an outage; renewing all of them at once overloads the lease manager, so they expire again. AWS's October 2025 DropletWorkflow Manager collapse is a textbook case.
- **Garbage collection and memory pressure.** Overload raises heap size, which lengthens GC pauses, which reduces capacity further.

**Why recovery is hard.** Because the bad state is self-sustaining, fixing the trigger (restoring DNS, redeploying the good config) is not enough. Recovery requires pushing load **below the recovery threshold**, which is usually far lower than normal load. Standard recovery moves:

1. **Shed load hard at the edge** — reject a large percentage of traffic outright so the core can drain.
2. **Disable or reduce retries** — globally, via a flag or a budget.
3. **Restart in waves**, admitting load gradually, rather than all at once.
4. **Drop queued work** that is already too old to be useful (goodput over throughput).
5. **Throttle recovery work** (lease renewals, cache refills, replication catch-up) so it cannot itself overload the system.

**Design moves that prevent it:**

- Bounded queues and deadlines, so abandoned work is dropped rather than done (Concepts 13, 36).
- Retry budgets rather than fixed retry counts (Concept 19).
- LIFO or CoDel under overload, so recent requests (whose clients are still waiting) are served first (Concept 36).
- Load shedding that is cheaper than serving (Concept 35).
- Capacity tests that deliberately remove the cache or restart the fleet, to find where the vulnerable region starts.
- Static stability, so recovery does not depend on doing extra work (Concept 40).

**Goodput vs throughput** is the phrase to use. Under overload, a well-designed system's goodput plateaus near capacity; a badly designed one's goodput *collapses* as offered load rises past capacity, even though throughput and CPU look busy. The difference between those two curves is almost entirely the patterns in this module.

**The interview-grade sentence:** *"The outages that last hours are usually metastable: the trigger is fixed in minutes, but retries, cold caches, or lease storms keep the system saturated doing useless work. So I design for goodput — deadlines, bounded LIFO queues, retry budgets, shedding — and my recovery runbook starts with cutting load below the recovery threshold, not with restarting everything."*

---

# Part B — Timeouts and deadlines

## Concept 11 — Every remote call needs a timeout, and how to choose one

**Timeouts are the first reliability pattern**, because without them none of the others work: a retry cannot fire, a breaker cannot count, and a bulkhead cannot release a slot while a call is still waiting forever. Amazon's Builders' Library guidance is blunt: set a timeout on every remote call, and generally on any call across processes, even on the same host — including both a connection timeout and a request timeout.

**Why a timeout is a trade-off, not a safety feature.**

- **Too long:** the caller holds resources (threads, connections, memory, transactions) while waiting, so a slow dependency exhausts the caller (Concept 3). A 100-second timeout in front of a service that normally answers in 50 ms is effectively no timeout.
- **Too short:** healthy-but-slightly-slow requests are abandoned and retried, which *adds* load to a dependency that is already slow — the retry storm. Worse, the work may have completed server-side, so the timeout manufactures duplicates (Concept 14).

**The standard method: choose an acceptable false-timeout rate and read the percentile.** Amazon describes picking an acceptable rate of false timeouts — for example 0.1% — and setting the timeout at the corresponding latency percentile of the downstream service (the p99.9) measured over a representative period, with some margin. Concretely:

1. Measure the dependency's latency distribution *as seen by the caller* (including network), across peak periods.
2. Pick the false-timeout rate you can tolerate (0.1% → p99.9; 1% → p99).
3. Set the attempt timeout at that percentile plus a margin, then **check it against the caller's own budget** (Concept 13). If p99.9 of the dependency is 800 ms and the caller's SLO is 300 ms, no timeout value fixes that — the dependency is too slow for the design and you need caching, async, hedging, or a different dependency.
4. Revisit when the dependency changes. Timeouts copied from a config file three years ago are a common latent fault.

**Traps in picking the value:**

- **Measuring the median instead of the tail.** A timeout at 2× p50 will fire on a meaningful percentage of requests.
- **Latency distributions are multi-modal.** Cold starts, cache misses, GC pauses, and TLS handshakes create separate humps. Pre-warming connections before taking traffic avoids timing out the first requests after deployment.
- **Different operations deserve different timeouts.** A point read and a report export do not share a timeout. Per-operation (or per-endpoint) timeouts are normal.
- **Timeouts must shrink as you go down the stack**, never grow. A callee that works for 30 seconds for a caller who gave up after 5 is wasted work at best and a metastable sustaining effect at worst (Concept 10).

**The interview-grade sentence:** *"I set each attempt timeout from the dependency's measured p99 or p99.9 plus a margin — the percentile is the false-timeout rate I'm willing to accept — and then check it fits inside my caller's latency budget. If it doesn't fit, a timeout won't fix it; the design needs to change."*

---

## Concept 12 — The kinds of timeout, and the .NET defaults you must know

"The timeout" is several different timers, and confusing them is a common source of production incidents.

| Timeout | What it bounds | Why it's separate |
|---|---|---|
| **DNS resolution** | Name lookup | Resolver problems look like connection hangs |
| **Connect** | TCP (and TLS) handshake | A dead host should fail in hundreds of milliseconds, not in the request timeout |
| **Per-attempt (request)** | One try, from send to full response | The main tool against slow dependencies |
| **Total / overall** | All attempts including retries and backoff | Keeps retries inside the caller's budget |
| **Idle / keep-alive** | How long an unused pooled connection lives | Stale connections through NAT/load balancers fail on reuse |
| **Connection lifetime** | Maximum age of a pooled connection | Forces DNS re-resolution — essential for failover |
| **Server-side request timeout** | How long the server keeps working on a request | Bounds abandoned work |
| **Lock/transaction/command** | A single database operation | Bounds blocking inside the store |

**.NET defaults every architect should be able to recite:**

- **`HttpClient.Timeout` defaults to 100 seconds.** This is the most common "no effective timeout" bug in .NET services. It covers the whole `SendAsync` call (including reading headers, and the body when buffered) but it is a single number with no notion of per-attempt vs total when you add retries through handlers. With `IHttpClientFactory` and the resilience handlers (Concept 59), set per-attempt and total timeouts in the pipeline and treat `HttpClient.Timeout` as an outer guard.
- **`SocketsHttpHandler.ConnectTimeout`** defaults to infinite; set it explicitly (often 1–5 s within a region) when you build handlers yourself.
- **`SocketsHttpHandler.PooledConnectionLifetime`** defaults to infinite, which means **a long-lived `HttpClient` never re-resolves DNS** — after a DNS-based failover (Traffic Manager, a database failover group listener, a Service Bus geo-replication promotion), your process keeps talking to the old IP. `IHttpClientFactory` mitigates this by rotating handlers (default handler lifetime is 2 minutes); if you use a singleton `HttpClient`, set `PooledConnectionLifetime` (a few minutes is common). This is a **failover** bug disguised as a configuration detail, and naming it is a strong signal.
- **SqlClient:** `Connect Timeout` defaults to 15 s; `SqlCommand.CommandTimeout` defaults to 30 s. EF Core's `CommandTimeout` follows the provider unless set.
- **gRPC for .NET:** no deadline by default — a call without a deadline can wait forever. Always set `deadline:` on calls (Concept 13).
- **Azure SDK clients** have their own `NetworkTimeout` and retry settings in `ClientOptions.Retry` (Concept 60).
- **ASP.NET Core** has no default per-request execution timeout; the `Microsoft.AspNetCore.Http.Timeouts` request-timeouts middleware (`AddRequestTimeouts`, `[RequestTimeout]`) lets you bound server-side work and signal it via `HttpContext.RequestAborted`. Kestrel has separate limits for headers, keep-alive, and minimum data rates (Concept 61).

**The cancellation contract.** In .NET, a timeout is only as good as the cancellation it triggers. A timeout strategy cancels a `CancellationToken`; if the code underneath ignores the token (a blocking call, a library that doesn't accept one, a `Task.Run` that doesn't observe it), the caller stops waiting but the work continues and the resource stays held. Polly's timeout strategy is explicitly cooperative for this reason. Every async method on a remote path should accept and pass a `CancellationToken`.

**The interview-grade sentence:** *"In .NET I set connect, per-attempt, and total timeouts separately — HttpClient's 100-second default is effectively none — and I set a pooled connection lifetime so DNS-based failover actually takes effect. Timeouts are cooperative, so every async call on the path takes the cancellation token."*

---

## Concept 13 — Deadline propagation and timeout budgets

A request that enters your system at the edge has a **latency budget** — say 1 second, from the user's perspective. Every hop downstream should know how much of that budget remains, and should refuse to start or continue work that cannot finish in time.

**The problem without propagation.** The edge gives up at 1 s. Service A calls B with a 2 s timeout; B calls C with a 5 s timeout. After the user has gone, B and C keep working — for nobody. Under load, this abandoned work is precisely the sustaining effect of a metastable failure (Concept 10).

**Deadlines vs timeouts.** A *timeout* is relative ("5 seconds from now"). A *deadline* is absolute ("by 12:00:03.250"). Deadlines compose naturally: each hop computes `remaining = deadline − now − safety margin` and uses that as the timeout for its own calls. gRPC builds this in: the client sets a deadline, it travels in the `grpc-timeout` header, and the server can read it and pass it downstream (gRPC for .NET can propagate deadlines and cancellation automatically from an incoming call to outgoing calls when you configure the client factory with `EnableCallContextPropagation()`).

**For HTTP**, there is no standard header, so teams define one (for example an `x-request-deadline` or remaining-milliseconds header) and propagate it via middleware and a `DelegatingHandler`. Clock skew makes absolute timestamps risky across machines; passing **remaining duration** and recomputing at each hop is the more robust approach.

**In C#**, the shape is a linked cancellation token with the remaining budget:

```csharp
public async Task<Quote> GetQuoteAsync(QuoteRequest request, CancellationToken requestAborted)
{
    // Budget for this operation: the smaller of our own SLO and what the caller allows.
    var remaining = _deadlineAccessor.Remaining ?? TimeSpan.FromMilliseconds(800);
    var budget = remaining - TimeSpan.FromMilliseconds(50);   // leave time to respond
    if (budget <= TimeSpan.Zero)
        throw new DeadlineExceededException();                 // don't start work nobody will wait for

    using var cts = CancellationTokenSource.CreateLinkedTokenSource(requestAborted);
    cts.CancelAfter(budget);

    var price = await _pricing.GetPriceAsync(request.Sku, cts.Token);
    var stock = await _inventory.GetAvailabilityAsync(request.Sku, cts.Token);
    return new Quote(price, stock);
}
```

**Budget allocation.** A 1 s budget across a fan-out of four parallel calls plus a sequential call is an engineering exercise: parallel calls share the same window; sequential calls divide it; retries must fit inside it (the total timeout of Concept 12). A useful rule is that **retries are only allowed if the remaining budget can accommodate at least one more full attempt** — otherwise, fail fast.

**Server-side enforcement.** A server should also check the deadline **when dequeuing work**: if a request waited in a queue past its deadline, drop it without processing. This single check converts a queue from a goodput killer into a shock absorber.

**The interview-grade sentence:** *"I propagate the remaining deadline with every call — gRPC does it natively, for HTTP I pass remaining milliseconds — so no service works longer than its caller will wait, and servers drop queued requests whose deadline has already passed."*

---

## Concept 14 — Timeouts create ambiguity: "I don't know" is a third outcome

A remote call has three outcomes, not two: **success**, **failure**, and **unknown**. A timeout, a connection reset after the request was sent, or a 504 from a gateway all mean *unknown*: the request may never have arrived, may have been processed and the response lost, or may still be in progress.

This is the same insight as the Two Generals' Problem (Module 9) and the at-least-once delivery discussion (Module 11), applied to synchronous calls. It has direct consequences:

- **A retry after an ambiguous failure can duplicate the effect.** Charging a card, sending an email, creating an order, incrementing a counter.
- **Reporting "failed" to the user after an ambiguous failure can be a lie.** The order may exist.
- **Compensating after an ambiguous failure can be wrong.** Refunding a charge that never happened is its own incident.

The resolutions, in order of preference:

1. **Make the operation idempotent** so that retrying is safe (Concept 20). Natural idempotency (`PUT` of a full resource, "set status to shipped") or an idempotency key.
2. **Make the outcome queryable.** Give the operation a client-generated ID so that after an ambiguous failure the client can ask "did order `7f3c…` get created?" before deciding.
3. **Move the operation to an asynchronous, durable channel** (Module 11) where at-least-once delivery plus idempotent processing is the explicit contract.
4. **Surface the ambiguity honestly.** "We're confirming your payment" is a valid UI state; "Payment failed" when you don't know is not.

**A nuance that interviewers like:** some failures are *known* not to have reached the server — DNS failure, connection refused, TLS handshake failure — and are therefore safe to retry even for non-idempotent operations. Some HTTP clients and proxies distinguish these ("retry only if the request was not sent"). Everything after the request bytes left the process is ambiguous.

**The interview-grade sentence:** *"A timeout means 'I don't know,' not 'it failed.' So anything I retry after a timeout has to be idempotent, and anything with side effects gets a client-generated ID so the caller can check the outcome instead of guessing."*

---

# Part C — Retries

## Concept 15 — What retries are for: classify the failure first

Retries exist for exactly one purpose: **to mask transient faults** — failures that are likely to succeed if the same request is sent again shortly. Cloud platforms produce these constantly by design: a load balancer draining an instance, a database failing over, a throttling response, a brief network blip, a deadlock victim, a connection reset during a rolling deployment.

Retries are *not* for: permanent errors (retrying cannot fix a validation failure), overload (retrying makes it worse unless paced), or bugs (retrying a null reference exception three times produces three null reference exceptions and a slower alert).

**The classification you should apply before any retry decision:**

| Class | Examples | Retry? |
|---|---|---|
| **Transient** | Connection reset, 503 during failover, 502 from a proxy losing an instance, SQL transient error numbers (e.g., failover, throttling), deadlock victim, Cosmos 429/410/449 | Yes, with backoff and jitter, within budget |
| **Throttled** | HTTP 429, 503 with `Retry-After`, Azure/Cosmos throttling | Yes, but **wait at least as long as the server says** (Concept 22) |
| **Permanent** | 400, 401, 403, 404, 409 (usually), 422, deserialization error, business rule violation | No — fail fast and surface the error |
| **Ambiguous** | Timeout, 504, connection dropped after send | Only if the operation is idempotent (Concept 14) |
| **Overload signal** | "Server overloaded — do not retry" | No — and propagate that signal upward (Concept 18) |

**HTTP semantics that matter** (RFC 9110):

- **Safe methods** (`GET`, `HEAD`, `OPTIONS`) and **idempotent methods** (`PUT`, `DELETE`, plus the safe ones) may be retried by the protocol's definition. `POST` and `PATCH` are not idempotent unless the API makes them so.
- **408 Request Timeout**, **429 Too Many Requests** (RFC 6585), **502**, **503**, and **504** are the common retryable statuses; **500** is ambiguous — it is often a bug (not transient) but sometimes a transient dependency failure, which is why many teams retry it at most once or not at all.
- **`Retry-After`** can accompany 503 and 429 (and 3xx) with either a number of seconds or an HTTP date.

**gRPC status codes** encode this more precisely: `UNAVAILABLE` is the canonical retryable code; `DEADLINE_EXCEEDED` is ambiguous; `RESOURCE_EXHAUSTED` is throttling; `INVALID_ARGUMENT`, `NOT_FOUND`, `PERMISSION_DENIED`, `FAILED_PRECONDITION` are permanent. gRPC's retry design (proposal A6) also supports server pushback — the server can tell the client not to retry, or when to.

**Two .NET notes:**

- The **standard resilience handler's retry** (Concept 59) handles `HttpRequestException`, attempt timeouts, 408, 429, and 5xx by default, and **retries all HTTP methods unless you disable unsafe ones** (`options.Retry.DisableForUnsafeHttpMethods()`). Leaving POST retries on for a non-idempotent endpoint is a real duplicate-charge bug.
- **SQL transient error numbers** are a curated list (connection loss during failover, resource governance throttling, deadlock 1205, etc.). EF Core's `SqlServerRetryingExecutionStrategy` and SqlClient's configurable retry logic both ship lists; do not write your own unless you must.

**The interview-grade sentence:** *"Before I retry, I classify: transient errors get retried with backoff; throttling waits for Retry-After; permanent errors fail fast; and ambiguous ones like timeouts only get retried if the operation is idempotent."*

---

## Concept 16 — Backoff: spreading retries out in time

If a dependency failed because it is momentarily unavailable or overloaded, retrying *immediately* gives it no time to recover and adds load at the worst moment. **Backoff** inserts a delay between attempts, and the shape of that delay matters.

| Strategy | Delay before attempt *n* | Character |
|---|---|---|
| **Immediate** | 0 | Only for faults that clear in microseconds (a single stale pooled connection); at most once |
| **Constant** | `d` | Simple; synchronized retries arrive in waves |
| **Linear** | `d × n` | Grows slowly |
| **Exponential** | `d × 2ⁿ` (or another multiplier) | Backs off fast; the standard choice |
| **Capped exponential** | `min(cap, d × 2ⁿ)` | Prevents absurd waits; the version you actually deploy |

**Why exponential.** If the failure is caused by overload, the load you add must fall quickly while the failure persists. Exponential backoff reduces each client's retry rate geometrically, which is the same principle as TCP's congestion backoff and Ethernet's collision backoff.

**Why capped.** Without a cap, ten attempts with a 1 s base produce a final wait of over 17 minutes (2¹⁰ s). The cap bounds the tail; a *total* timeout (Concept 12) bounds the whole operation. Amazon's guidance calls this capped exponential backoff.

**Choosing the base and cap:**

- **In a synchronous request path**, the entire retry sequence must fit inside the caller's budget (Concept 13). That usually means a base of tens to a few hundred milliseconds, a cap of a second or two, and **one or two retries** at most.
- **In background or queue processing**, delays can be seconds to minutes, and the queue's delayed redelivery (Module 11) is often a better mechanism than an in-process sleep that holds a message lock.
- **For long outages**, backoff alone is not enough — the fleet's aggregate retry rate plateaus once everyone is at the cap. That is the job of retry budgets and circuit breakers (Concepts 19 and 23).

**The interview-grade sentence:** *"I use capped exponential backoff — in a synchronous path that's one or two retries starting around 100–200 milliseconds and capped at a second or two, all inside a total timeout — because the point is to give the dependency time to recover, not to hammer it."*

---

## Concept 17 — Jitter: why randomness wins

Exponential backoff alone has a flaw: clients that failed at the same moment compute the **same** delays and retry at the **same** moments. A thousand clients that failed together retry together at +100 ms, +200 ms, +400 ms — synchronized waves that look like load spikes to the recovering dependency.

**Jitter** randomizes each delay so retries spread out in time. Marc Brooker's AWS Architecture Blog post *Exponential Backoff and Jitter* (2015) simulated the common variants and is the canonical reference:

| Variant | Formula | Behaviour |
|---|---|---|
| **No jitter** | `sleep = min(cap, base × 2ⁿ)` | Synchronized waves; most total work |
| **Full jitter** | `sleep = random(0, min(cap, base × 2ⁿ))` | Excellent spread; the usual recommendation |
| **Equal jitter** | `t = min(cap, base × 2ⁿ); sleep = t/2 + random(0, t/2)` | Guarantees some minimum wait; slightly more work than full jitter |
| **Decorrelated jitter** | `sleep = min(cap, random(base, previous × 3))` | Delays grow randomly from the previous delay; performs well in the simulation |

The simulation's result: **jittered strategies completed with far fewer total calls than un-jittered exponential backoff**, and full jitter and decorrelated jitter were the best performers. The intuition is that jitter converts a burst of retries into an approximately constant rate, which a recovering server can absorb.

**In C#** (full jitter, as a helper you would rarely write yourself because Polly does it):

```csharp
static TimeSpan FullJitterDelay(int attempt, TimeSpan baseDelay, TimeSpan maxDelay)
{
    // attempt is 0-based; guard the exponent to avoid overflow
    var exp = Math.Min(maxDelay.TotalMilliseconds,
                       baseDelay.TotalMilliseconds * Math.Pow(2, Math.Min(attempt, 30)));
    return TimeSpan.FromMilliseconds(Random.Shared.NextDouble() * exp);
}
```

Polly v8's retry strategy exposes this as `BackoffType = DelayBackoffType.Exponential` with `UseJitter = true`; its exponential-with-jitter implementation is based on the "decorrelated jitter v2" algorithm that originated in the `Polly.Contrib.WaitAndRetry` project, which smooths out the clustering that naive decorrelated jitter can produce.

**Jitter beyond retries.** The same logic applies to **any periodic or synchronized work**: cache TTLs (Module 10), cron jobs, health-check intervals, lease renewals, reconnection after a broker failover, polling loops, and the startup sequence of a fleet. Google's June 2025 recovery prolonged by a herd effect from restarting tasks without randomized backoff is the cautionary example (Concept 8). A subtle refinement from Amazon's guidance: for scheduled work, **consistent** per-host jitter (derived from a hash of the host ID) keeps the spread while making behaviour reproducible for debugging.

**The interview-grade sentence:** *"Backoff without jitter just moves the thundering herd to a later timestamp. I use exponential backoff with full or decorrelated jitter, and I jitter every synchronized thing — TTLs, cron, reconnects, and fleet restarts."*

---

## Concept 18 — Retry amplification and retrying at a single layer

Module 6 stated it and Module 11 named it; here is the arithmetic. If each layer of a call chain makes up to **r** attempts (the original plus retries) and there are **d** layers, a persistent failure at the bottom produces up to **rᵈ** attempts there for every user request:

| Attempts per layer (r) | Layers (d) | Attempts at the bottom |
|---|---|---|
| 2 (1 retry) | 3 | 8 |
| 3 (2 retries) | 3 | 27 |
| 3 | 5 | 243 |
| 4 (3 retries) | 5 | 1,024 |

That multiplication happens exactly when the bottom layer is already failing, often because it is overloaded. **Well-intentioned retries at every layer are one of the most reliable ways to turn a partial outage into a total one.** The Yandex engineering write-up *Good Retry, Bad Retry: An Incident Story* is a readable account of this happening in production, with simulations of the fixes.

**Defences, in order of importance:**

1. **Retry at one layer only.** Amazon's stated practice for low-cost operations is to retry at a single point in the stack. Which layer? Usually the one **closest to the failing dependency** (it has the most context about whether the error is transient and the cheapest retry), or the **outermost client** (the only one that knows the full budget) — pick one per call graph and make it explicit in your service contract.
2. **Propagate a "do not retry" signal upward.** Google's SRE guidance is to have a distinct status for "overloaded; don't retry" so that a layer that has already retried (or decided not to) stops its callers from retrying too. In HTTP, a common convention is returning 503 with a header such as `x-should-retry: false`, or mapping downstream exhaustion to a non-retryable status at your boundary.
3. **Cap per-request attempts** — Google suggests a small fixed number, such as three total attempts, as a per-request limit.
4. **Add a per-client retry budget** so that retries are a bounded fraction of traffic (Concept 19).
5. **Put a breaker in front of the dependency** so a sustained failure stops generating attempts at all (Concept 23).
6. **Respect deadlines** — a retry that cannot finish before the caller's deadline should not be attempted (Concept 13).

**The layering in practice for a .NET service:** a request handler calls an internal API with `HttpClient` + the standard resilience handler (retries), which calls a database through EF Core with `EnableRetryOnFailure` (retries), which uses SqlClient (which may have configurable retry logic enabled). If the API's caller also uses a resilient client, you have four retry layers without writing a single `for` loop. Concept 60 shows how to find and remove them.

**The interview-grade sentence:** *"Retries multiply: three attempts at each of five layers is 243 attempts at the bottom. So I retry at exactly one layer per call path, cap attempts, add a retry budget, and have the lower layer return a 'don't retry' signal once it has given up."*

---

## Concept 19 — Retry budgets and adaptive retry

Fixed retry counts have a fatal property: **at 100% failure, N retries multiply load by (1 + N)**, precisely when the dependency is weakest. A **retry budget** makes the retry rate fall as the failure rate rises.

**The token-bucket retry budget** (Marc Brooker's post *Fixing retries with token buckets and circuit breakers* is the clearest explanation). Each successful call deposits a fraction of a token (say 0.1) into a bucket of bounded size; each retry withdraws one whole token; if the bucket is empty, the retry is not attempted. Behaviour:

- At low failure rates (below ~10% with a 0.1 deposit), the bucket stays full and the client behaves like "N retries" — transient faults are masked.
- At high failure rates, retries are limited to roughly 10% of traffic — load on the failing dependency rises by at most ~10%, not by 1 + N.
- It needs no mode switch, so there is no "open" state to test (contrast with circuit breakers, Concept 28).

The **AWS SDKs** implement a version of this in their *standard* and *adaptive* retry modes: each retry consumes tokens from a client-side bucket (timeouts cost more than other errors), successes refund tokens, and when the bucket is empty, retries stop. **Finagle** (Twitter) and **Envoy** implement retry budgets as a percentage of active requests — Envoy's circuit-breaker thresholds include a `retry_budget` whose default budget percentage is 20%, with a minimum retry concurrency so low-traffic services can still retry.

**Client-side adaptive throttling.** Google's SRE book (*Handling Overload*) describes a related mechanism that throttles *all* requests, not just retries. Each client tracks, over a recent window, the number of **requests** it attempted and the number of **accepts** the backend actually served. It then rejects new requests locally with probability:

```
p_reject = max(0, (requests − K × accepts) / (requests + 1))
```

With K = 2, the client lets through up to twice as many requests as the backend has recently been accepting, so the backend still sees enough traffic to signal recovery, while the client sheds the excess locally — without the backend spending anything on requests it would reject anyway.

**A C# sketch of a retry budget** — useful to understand even though you would normally configure one in a library or mesh:

```csharp
public sealed class RetryBudget
{
    private readonly double _depositPerSuccess;   // e.g. 0.1 → retries ≈ 10% of traffic under failure
    private readonly double _maxTokens;           // e.g. 100
    private double _tokens;
    private readonly object _gate = new();

    public RetryBudget(double depositPerSuccess = 0.1, double maxTokens = 100)
        => (_depositPerSuccess, _maxTokens, _tokens) = (depositPerSuccess, maxTokens, maxTokens);

    public void RecordSuccess()
    {
        lock (_gate) _tokens = Math.Min(_maxTokens, _tokens + _depositPerSuccess);
    }

    public bool TryAcquireRetry()
    {
        lock (_gate)
        {
            if (_tokens < 1) return false;
            _tokens -= 1;
            return true;
        }
    }
}
```

**Scope the budget to the shared fate.** One budget per dependency (or per dependency endpoint), per client process. A single global budget lets a failing optional dependency consume the retry capacity of a critical one.

**The interview-grade sentence:** *"A fixed retry count multiplies load at exactly the wrong time. I prefer a retry budget — a token bucket where successes earn a fraction of a token and retries spend one — so under a full outage retries are capped around ten or twenty percent of traffic instead of tripling it."*

---

## Concept 20 — Retry safety: idempotency and idempotency keys

A retry is safe only if performing the operation twice has the same effect as performing it once. Module 11 covered idempotent *consumers*; this concept covers idempotent *APIs*, because every synchronous retry policy depends on them.

**Naturally idempotent operations:** reads; `PUT` of a full representation; "set X to value"; `DELETE` of a specific resource (the second delete returns 404 or 204, but the state is the same); conditional updates with an ETag (`If-Match`), where a duplicate simply fails the precondition.

**Not idempotent:** "create an order," "charge the card," "increment the counter," "append to the list," "send the email."

**The idempotency-key pattern** (popularized by Stripe's API, and being standardized as the `Idempotency-Key` HTTP header in an IETF HTTPAPI draft):

1. The **client** generates a unique key (a UUID) per logical operation — *not* per attempt — and sends it on every attempt.
2. The **server**, in the same transaction as the side effect, records `(key → outcome)` with a unique constraint (Module 12, Concept 46).
3. On a repeated key, the server returns the stored outcome instead of repeating the effect.
4. On a repeated key with a **different payload**, the server returns an error (422 or 409) — the key was reused incorrectly.
5. On a repeated key while the first attempt is **still in flight**, the server returns 409 (or makes the second attempt wait) so two concurrent executions cannot both proceed.
6. Keys expire after a retention window (Stripe uses 24 hours) that exceeds any plausible retry horizon.

```csharp
// Client side: one key per logical operation, reused across retries
var idempotencyKey = Guid.NewGuid().ToString();
using var request = new HttpRequestMessage(HttpMethod.Post, "/payments")
{
    Content = JsonContent.Create(payment)
};
request.Headers.Add("Idempotency-Key", idempotencyKey);
// The retry strategy re-sends this same request message (and therefore the same key).
```

Amazon's Builders' Library article *Making retries safe with idempotent APIs* describes the same design with a client request token, and adds the important detail that the server should **return a semantically equivalent response** to the retry, so the client's logic does not have to distinguish "created" from "already created."

**Senior nuances:**

- The key must be **stored atomically with the effect**. Recording the key in Redis and the order in SQL is a dual write (Module 11).
- **Downstream effects need their own keys.** If your payment service calls a card processor, pass *its* idempotency key too — derived deterministically from yours so a retry of your operation produces the same downstream key.
- **Hedging (Concept 38) and multi-region write failover (Concept 47) also require idempotency**, because they are retries in disguise.

**The interview-grade sentence:** *"Only idempotent operations get retried. For creates and payments I require a client-generated idempotency key, stored with a unique constraint in the same transaction as the effect, returning the original result on replay — and I derive the downstream provider's key from it."*

---

## Concept 21 — Where retries live, and the double-retry trap

A retry can happen in many places, and you should decide deliberately which one owns it.

| Location | Examples | Best for | Cost |
|---|---|---|---|
| **Inside the client library/SDK** | Azure SDK `RetryOptions`, Cosmos SDK 429 handling, AWS SDK modes | Protocol-specific transient faults the SDK understands | Often invisible; easy to double up |
| **In the application pipeline** | Polly / `Microsoft.Extensions.Http.Resilience`, EF Core execution strategy | Your policy for your dependency | Must coordinate with the SDK's retries |
| **In a proxy / service mesh** | Envoy, Istio, Linkerd, YARP | Uniform policy, outlier detection, budgets | Adds a layer that also retries |
| **In a broker (redelivery)** | Service Bus abandon + `MaxDeliveryCount`, delayed retries | Asynchronous work, long outages | Latency in minutes; lock and ordering implications (Module 11) |
| **In a workflow engine** | Durable Functions, Temporal, Dapr workflows | Multi-step business processes | Heavy machinery; durable state |
| **At the user** | "Try again" button | Rare, human-scale failures | Humans retry in bursts too |

**The double-retry trap in .NET** is common enough to name specifically:

- An **Azure SDK client** (Blob Storage, Key Vault, Service Bus, etc.) retries internally by default, with exponential backoff, via `ClientOptions.Retry`.
- The team wraps calls to it in a **Polly retry** "for resilience."
- The HTTP transport underneath was registered with **`AddStandardResilienceHandler()`** via `ConfigureHttpClientDefaults` (for example, from an Aspire service-defaults project), which the Azure SDK may use if you plumb an `HttpClient` into its transport.

Now a single call can produce 3 × 4 × 4 attempts. **Pick one owner per dependency:** usually configure the SDK's own retry (it understands the service's error codes, throttling headers, and failover semantics) and do *not* add an outer retry — add only an outer total timeout and, if needed, a circuit breaker.

**Asynchronous vs synchronous retry.** If a request path cannot succeed now, consider whether the work can be accepted and completed later: write it to a queue (Module 11) and let the broker's redelivery handle the outage, rather than holding the user's request open while retrying. This converts a dependency outage from an error into a backlog.

**The interview-grade sentence:** *"Each dependency gets exactly one retry owner. For Azure SDK clients that's usually the SDK itself, because it understands the service's throttling and failover — so I don't wrap it in Polly retries, I just add a total timeout. And if work can be done later, I queue it instead of retrying inline."*

---

## Concept 22 — Server-directed backoff: 429, 503, and `Retry-After`

When a server throttles you, it is telling you something your client-side backoff formula cannot know: **how much capacity it currently has for you.** Honour it.

- **HTTP 429 Too Many Requests** means *you* (or your key, tenant, or IP) exceeded a rate limit.
- **HTTP 503 Service Unavailable** means the *service* is temporarily unable to handle the request (overload or maintenance).
- **`Retry-After`** on either tells you when to try again, as seconds or an HTTP date.

**Rules for clients:**

1. **Wait at least `Retry-After`**, even if your backoff schedule says less. Add jitter *on top* so all throttled clients do not return at the same instant.
2. **If `Retry-After` exceeds your remaining deadline, fail fast** instead of waiting (Concept 13).
3. **Throttling is not a failure of the dependency**, so it generally **should not trip a circuit breaker** — you would stop sending traffic that the server would have accepted a moment later. (Some teams count sustained 429s to trigger client-side rate reduction instead.)
4. **Reduce your sending rate**, not just the retry: a client that keeps sending new requests at full rate while retrying throttled ones is still over the limit. Client-side rate limiting (Concept 37) or adaptive throttling (Concept 19) belongs here.

**Platform specifics worth knowing:**

- **Cosmos DB** returns 429 with `x-ms-retry-after-ms`. The .NET SDK retries these automatically — by default up to 9 times with a maximum cumulative wait of 30 seconds (`MaxRetryAttemptsOnRateLimitedRequests`, `MaxRetryWaitTimeOnRateLimitedRequests`). Sustained 429s are a capacity signal (RU/s, hot partition — Module 12, Concept 34), not a retry-tuning problem.
- **Azure Resource Manager and many Azure data-plane APIs** return 429 with `Retry-After`; the Azure SDK's retry policy honours it.
- **Polly / `Microsoft.Extensions.Http.Resilience`**: the HTTP retry options honour `Retry-After` by default (`ShouldRetryAfterHeader`), and a custom `DelayGenerator` can read other headers.
- **Service Bus / Event Hubs** surface throttling as a specific exception reason (`ServiceBusFailureReason.ServiceBusy`), and the SDK backs off accordingly.

**Server side**, the corresponding obligation is to **send** `Retry-After` when you throttle, and to make the 429 path cheap (Concept 35). ASP.NET Core's rate limiter can set it via `OnRejected` using the lease's `MetadataName.RetryAfter` (Concept 61).

**The interview-grade sentence:** *"Throttling responses are the server telling me its capacity, so Retry-After overrides my backoff, jitter goes on top, a wait longer than my deadline fails fast, and 429s don't trip the breaker — they slow me down."*

---

# Part D — Circuit breakers

## Concept 23 — The circuit breaker: purpose and state machine

Michael Nygard popularized the software circuit breaker in *Release It!* (2007), borrowing the idea from electrical engineering: when a circuit draws dangerous current, a breaker trips and stops the flow until someone resets it. In software, **a circuit breaker wraps calls to a dependency and, when the dependency appears to be failing, stops making the calls for a while.**

It serves **three purposes**, and a strong answer names all three:

1. **Protect the caller.** Fail fast instead of waiting on timeouts, so threads, connections, and the request budget are not burned on calls that will fail.
2. **Protect the dependency.** Stop sending load it cannot handle, giving it room to recover (the opposite of retry amplification).
3. **Make failure explicit.** An open breaker is a clear, observable signal that can drive fallbacks, degrade features, and page someone.

**The state machine:**

```
          failure threshold exceeded
  ┌────────┐ ───────────────────────────► ┌────────┐
  │ CLOSED │                              │  OPEN  │  calls fail immediately
  └────────┘ ◄──┐                         └────────┘  (BrokenCircuitException)
      ▲         │ probe(s) succeed             │
      │         │                              │ break duration elapses
      │     ┌───────────┐                      │
      └─────│ HALF-OPEN │ ◄────────────────────┘
            └───────────┘
               │ probe fails
               └──────────────► back to OPEN (possibly for longer)
```

- **Closed:** calls flow normally; the breaker records outcomes.
- **Open:** calls are rejected immediately without touching the dependency, for the **break duration**.
- **Half-open:** after the break duration, a limited number of trial calls are allowed. If they succeed, the breaker closes; if they fail, it opens again.

Some implementations add **isolated** (manually forced open — Polly's `CircuitBreakerManualControl.IsolateAsync()`), useful as an operational kill switch for a dependency during an incident, and **disabled / forced-closed** states for testing.

**What a breaker is not:** it is not a retry mechanism (Polly's breaker rethrows the handled exception and does not retry — combine it with a retry strategy), and it is not a rate limiter (it responds to *failures*, not *volume*).

**The interview-grade sentence:** *"A circuit breaker watches a dependency's failure rate and, above a threshold, fails calls immediately for a break period, then lets a few probes through to test recovery. It protects my resources, gives the dependency room to recover, and gives me an explicit signal to degrade on."*

---

## Concept 24 — Trip conditions: count, ratio, window, and minimum throughput

How a breaker decides to open is its most important configuration, and there are two families.

**Consecutive-failure (count-based) breakers.** Open after *N* consecutive failures (Polly v7's basic `CircuitBreaker`, many simple implementations). Easy to understand; poorly suited to high-volume traffic, where a steady 30% failure rate interleaved with successes never produces N consecutive failures, and suited mostly to low-volume dependencies.

**Failure-ratio (rate-based) breakers over a sliding window.** Open when the **proportion** of failed calls in a recent window exceeds a threshold, **provided at least a minimum number of calls occurred** in that window. This is the model in Polly v8 (the only model — its v8 breaker is the former "advanced" breaker), Resilience4j, and Hystrix:

| Parameter (Polly v8 name) | Meaning | Polly default |
|---|---|---|
| `FailureRatio` | Proportion of handled failures that trips the breaker | 0.1 (10%) |
| `SamplingDuration` | Window over which the ratio is computed | 30 seconds |
| `MinimumThroughput` | Calls required in the window before the ratio counts | 100 |
| `BreakDuration` | How long the breaker stays open | 5 seconds |
| `BreakDurationGenerator` | Optional function to vary break duration (e.g., grow with repeated failures) | none |

**Why minimum throughput matters.** Without it, two failures out of two calls at 3 a.m. is a 100% failure ratio and opens the breaker on noise. With it, low-traffic periods cannot trip the breaker at all — which is the flip side: **a minimum throughput set too close to typical traffic means the breaker rarely evaluates**, so set it well below normal volume in the window. Polly's documentation notes that a breaker's reaction time to a total failure is roughly proportional to `SamplingDuration × FailureRatio`, so a 30-second window with a 10% threshold reacts within a few seconds at full failure.

**Tuning guidance.** The .NET team's 2025 devblog on circuit-breaker fine-tuning recommends that, absent guidance from the dependency's owners, a **failure ratio of 0.5 or higher** is a better starting point than the 0.1 default (which is aggressive), and that a **short break duration (around 5 seconds)** is usually right — longer breaks block healthy traffic after recovery. It also argues that the threshold should ideally be defined by the *callee*, which knows at what availability traffic still makes sense.

**Time-based vs count-based windows.** Resilience4j lets you choose a sliding window of the last *N* calls or of the last *N* seconds. Time-based windows adapt to traffic volume; count-based windows give consistent statistical weight. Both are implemented with fixed buckets that roll over, which is memory-cheap.

**Slow calls.** Resilience4j's breaker can open on a **slow-call rate** (the percentage of calls slower than a threshold) as well as a failure rate — the direct defence against fail-slow dependencies (Concept 3). In Polly, you get the same effect by placing an **attempt timeout inside the breaker** so slow calls become `TimeoutRejectedException` failures that the breaker counts.

**The interview-grade sentence:** *"I use a failure-ratio breaker over a sliding window with a minimum-throughput floor, so it reacts to sustained failure rather than noise, and I put the per-attempt timeout inside it so slow calls count as failures. For thresholds I start around fifty percent with a short break and tune from telemetry."*

---

## Concept 25 — What counts as a failure, and how half-open probing works

**Count only failures that indicate the dependency is unhealthy.**

| Outcome | Count as breaker failure? | Why |
|---|---|---|
| Connection refused / reset, DNS failure | Yes | Dependency or path unhealthy |
| Attempt timeout | Yes | Fail-slow is failure |
| 5xx (500, 502, 503, 504) | Usually yes | Server-side trouble (500 is debatable if it's a deterministic bug for one input) |
| 429 throttling | Usually **no** | Capacity signal; slow down instead (Concept 22) |
| 4xx client errors (400, 401, 403, 404, 409, 422) | **No** | The caller's fault; opening the breaker would block valid requests |
| `OperationCanceledException` from the *caller's* token | **No** | The caller gave up; the dependency may be fine (Polly's default predicate excludes it) |
| Business-rule rejections | **No** | A correct answer |

A breaker that counts 404s will open when a client requests many nonexistent resources — a self-inflicted outage for every other client. A breaker that counts the caller's cancellations will open when users navigate away quickly.

**Half-open probing strategies:**

- **Single probe:** after the break duration, the first call is allowed through; success closes the breaker, failure reopens it. Simple; slow to recover under high traffic; one unlucky call reopens the breaker. Polly v8 behaves this way (the first action after the break decides).
- **Limited probes:** allow *k* calls (Resilience4j's `permittedNumberOfCallsInHalfOpenState`) and evaluate their failure ratio.
- **Gradual ramp:** increase the allowed fraction over time (similar to a load balancer's slow start, Module 6, Concept 13), which avoids slamming a recovering dependency with the full traffic the instant the breaker closes.
- **Out-of-band health probing:** a background task pings a health endpoint and closes the breaker when it succeeds, so user requests are never used as probes. More machinery, but it separates health detection from user traffic.

**Break duration strategy.** A fixed short break (seconds) recovers quickly when the fault is brief. A growing break (via `BreakDurationGenerator`, e.g., doubling on each consecutive reopen up to a cap) reduces probe traffic during a long outage. Add jitter to break durations across instances so a fleet of breakers does not probe in lockstep.

**The interview-grade sentence:** *"The breaker counts connection failures, timeouts, and 5xx — never 4xx, never throttling, and never the caller's own cancellation — and half-open lets a small number of probes through, ideally ramping traffic back rather than flipping straight to full load."*

---

## Concept 26 — Breaker scope and granularity

**What does one breaker protect?** This question is where many designs go wrong.

| Scope | Example | Risk if chosen wrongly |
|---|---|---|
| **Per dependency** | One breaker for "Payments API" | One bad endpoint or one bad instance opens it for everything |
| **Per endpoint / operation** | `GET /prices` vs `POST /orders` | More state; better isolation |
| **Per host / authority** | One breaker per base URL (Polly's `SelectPipelineByAuthority`) | Needed when a client talks to several hosts or regions |
| **Per partition / shard** | Cosmos partition-level breaker; one breaker per database shard | Without it, one hot partition blocks all partitions |
| **Per tenant** | Breaker keyed by tenant for per-tenant backends | Without it, one tenant's broken integration blocks everyone |
| **Per instance of the dependency** | Client-side outlier detection per backend host | Usually belongs in the load balancer or mesh |

**The core risk: a coarse breaker converts a partial failure into a total one.** If shard 3 of 10 is failing and your breaker covers all shards, a 10% failure rate can trip it and take down 100% of traffic. The Azure Cosmos DB .NET SDK added a **partition-level circuit breaker** precisely for this reason: failures on one partition can be routed around (to another region) without affecting others.

**Where the breaker state lives.**

- **Per process (the default).** Each instance of your service has its own breaker. State converges quickly because every instance observes the same dependency; there is no shared-state dependency to fail.
- **Shared / distributed state** (e.g., in Redis). Occasionally proposed so all instances open together. Usually a bad idea: it adds a critical dependency to your failure-handling path, and the per-instance view is often *more* accurate (different instances may reach the dependency through different paths — gray failure, Concept 4).

**Breakers vs load-balancer health.** If the dependency has many instances behind a load balancer, a client breaker keyed by hostname sees only the aggregate. Unhealthy *instances* are better handled by **health checks and outlier ejection** at the load balancer or mesh (Concept 28); the client breaker handles *the whole dependency* being unhealthy.

**The interview-grade sentence:** *"I scope breakers to the unit that fails together — per host, per partition, or per tenant when those fail independently — because a single breaker over a sharded dependency turns one bad shard into a full outage. State stays per process; a shared breaker store is just another thing to fail."*

---

## Concept 27 — What to do when the circuit is open

An open breaker produces an immediate exception. What the caller does with it is a **product decision** that should be made per flow:

| Response | When it fits | Example |
|---|---|---|
| **Fail fast with a clear error** | The dependency is critical; there is no meaningful substitute | Payment authorization unavailable → "try again shortly" (503 with `Retry-After`) |
| **Degrade the feature** | The dependency is non-critical | Product page without recommendations or reviews |
| **Serve stale data** | Data is cacheable and staleness is tolerable | Last-known prices with an "as of" timestamp (Module 10's stale-while-revalidate / fail-safe) |
| **Use a default** | A safe static answer exists | Feature flags default to off; shipping estimate shows a range |
| **Queue for later** | The work can complete asynchronously | Accept the order, publish to a queue, confirm by email later (Module 11) |
| **Route to an alternate** | An independent equivalent exists | Secondary region, secondary provider — beware Concept 39 |
| **Pause consumption** | Asynchronous consumer of a failing dependency | Stop the Service Bus processor while the breaker is open; the queue is the buffer (Module 11, Concept 13) |

**Principles:**

- **The choice follows from the dependency classification** (Concept 6). Hard dependencies fail fast; soft dependencies degrade.
- **Tell the user the truth.** Degraded states should be visible where they matter ("Recommendations are temporarily unavailable").
- **Keep the degraded path cheap.** A fallback that calls another expensive service moves the overload instead of removing it.
- **Signal upward.** Return a status that tells your callers not to retry (Concept 18).
- **Emit telemetry on every transition** — breaker opened, half-opened, closed — and alert on sustained open state for critical dependencies (Concept 56).

**The interview-grade sentence:** *"What happens when the breaker is open depends on the flow: critical dependencies fail fast with a retry-after, optional ones degrade, cacheable data is served stale with a timestamp, and work that can wait goes to a queue. That decision is made with product, not buried in a library default."*

---

## Concept 28 — The critique of circuit breakers, and the alternatives

A senior answer does not just advocate breakers; it knows their weaknesses. Marc Brooker's posts on the subject make the critique crisply:

1. **Breakers are modal.** They introduce a mode (open) that the system rarely enters. Rarely exercised modes are poorly tested and surprise you in production.
2. **Client-side breakers can make partial outages worse.** If a dependency is failing for 20% of requests (one bad shard, one bad host, one bad tenant), a breaker that opens on aggregate failure rate rejects the 80% that would have succeeded. Without deep knowledge of the dependency's internal structure, the client cannot know whether the failure is total or partial.
3. **They add time to recovery.** After the dependency recovers, traffic resumes only after the break duration and a successful probe.
4. **They interact subtly with retries and load balancing**, and with each other across layers.

**The alternatives and complements:**

- **Retry budgets (token buckets)** — Concept 19. Break *retries* rather than all traffic: first attempts always go through; retries stop when the budget is empty. No mode, graceful behaviour at partial failure, and a bounded load multiplier at total failure. This is Brooker's preferred approach, and he notes a variant suggested to him — using breakers only to break retries, not first attempts.
- **Client-side adaptive throttling** — Concept 19. Rejects locally in proportion to how much the backend is refusing.
- **Outlier detection / ejection** — the load balancer or mesh (Envoy, Istio, Linkerd, gRPC's outlier detection) tracks per-instance error rates and latency and temporarily removes outliers from the pool. This handles the "one bad host" case far better than a client breaker, because it acts on the unit that is actually failing.
- **Concurrency limits and bulkheads** — Concepts 30–36. A slow dependency saturates its bulkhead; excess calls fail fast *without* any failure-rate statistics. Envoy's "circuit breaking" is actually this: limits on connections, pending requests, requests, and retries per upstream cluster.
- **Server-side load shedding** — Concept 35. The dependency protects itself; clients only need to honour its signals.

**When a classic breaker is still the right tool:** a dependency with a single failure domain (a third-party API, a single database), relatively low fan-out, where "all or nothing" is a good model of its failures, and where failing fast frees significant caller resources.

**The interview-grade sentence:** *"Breakers are useful, but they're modal and can turn a partial outage into a full one. For multi-instance dependencies I prefer outlier ejection at the load balancer, retry budgets instead of fixed retries, and bulkheads for slowness — and I keep classic breakers for single-failure-domain dependencies like a third-party API."*

---

## Concept 29 — Composition order: how the strategies stack

When you combine rate limiting, timeouts, retries, and a breaker, **the order changes the behaviour**. The widely used order — the one `Microsoft.Extensions.Http.Resilience`'s standard handler uses, from outermost to innermost — is:

```
Rate limiter (bulkhead)
  └─ Total request timeout
       └─ Retry
            └─ Circuit breaker
                 └─ Attempt timeout
                      └─ the actual call
```

**Why each layer is where it is:**

- **Rate limiter / bulkhead outermost.** It caps concurrent work *before* any other resource is spent. A request rejected here costs almost nothing and is not retried by this pipeline.
- **Total timeout outside retry.** It bounds the whole operation, including backoff delays, so retries cannot exceed the caller's budget.
- **Retry outside the breaker.** Each retry attempt passes through the breaker, so the breaker sees every attempt; and when the breaker is open, the attempt fails immediately with `BrokenCircuitException`. Your retry predicate should normally **not** retry `BrokenCircuitException` (or should wait at least the break duration) — otherwise the retry loop simply burns through attempts against an open breaker.
- **Breaker outside the attempt timeout.** Timeouts are then failures the breaker counts, which makes the breaker responsive to slowness (Concept 24).
- **Attempt timeout innermost.** It bounds each individual try.

**What goes wrong with other orders:**

| Mistake | Consequence |
|---|---|
| Timeout *inside* retry only (no total timeout) | Total latency = attempts × timeout + delays — easily exceeds the caller's deadline |
| Breaker *outside* retry | The breaker sees one outcome per logical call (after all retries), reacts slowly, and never sees the individual failures |
| Attempt timeout *outside* the breaker | Timeouts are invisible to the breaker; slow dependencies never trip it |
| Retry outside the rate limiter | Rejections by your own limiter get retried, adding pressure to the thing you are protecting |
| Two retry layers | Amplification (Concept 18) |

**A sanity constraint worth knowing:** the standard resilience handler validates that the circuit breaker's sampling duration is at least twice the attempt timeout, so the breaker's window can actually contain completed attempts; and the total timeout should comfortably exceed `attempts × attempt timeout + delays`, or the later retries will never run.

**The interview-grade sentence:** *"Outermost is the concurrency limit, then a total timeout, then retries, then the breaker, then the per-attempt timeout — so rejections are cheap, retries fit the budget, every attempt is counted by the breaker, and slow attempts count as failures."*

---

# Part E — Isolation and overload control

## Concept 30 — Bulkheads: the idea

A ship's hull is divided into watertight compartments — bulkheads — so that a breach floods one compartment rather than sinking the ship. In software, **a bulkhead partitions a shared resource so that a failure or overload in one partition cannot exhaust the resource for the others.**

The motivating failure: a service calls five dependencies using a shared pool of threads (or connections, or memory). One dependency becomes slow. Calls to it pile up and consume the entire shared pool. Now calls to the four healthy dependencies cannot get a thread either, and the whole service is down — because of one non-critical dependency (Concept 9).

With bulkheads, each dependency gets its own bounded allocation. The slow dependency saturates **its** allocation; excess calls to it are rejected immediately; the other four keep their capacity.

**The properties that make a bulkhead work:**

1. **Bounded.** A fixed maximum concurrency or pool size per partition.
2. **Fail-fast when full.** Excess work is rejected (or queued very briefly with a small bounded queue), not waited on indefinitely.
3. **Sized from data.** Enough capacity for normal peak load of that dependency with headroom; too small and the bulkhead causes its own rejections.
4. **Observable.** Rejections are metrics you alert on.

**Bulkheads complement circuit breakers.** A breaker reacts to *failure statistics* after the fact; a bulkhead reacts to *concurrency* immediately, which is precisely what a slow dependency produces. The two together handle both "failing fast" and "failing slowly" dependencies.

**The interview-grade sentence:** *"A bulkhead gives each dependency or workload its own bounded slice of a shared resource, so one slow dependency saturates its own slice and gets fast rejections while everything else keeps working. It's the pattern that handles slowness before any breaker has enough data to trip."*

---

## Concept 31 — Bulkhead implementations, and why .NET uses semaphores

**Thread-pool bulkheads** (Hystrix's default model) give each dependency its own dedicated thread pool — say 10 threads — and execute calls on it. This provides strong isolation (a hung call blocks only its pool's threads) and allows timeouts to abandon the calling thread. It fits the synchronous, thread-per-request world of classic Java.

**Semaphore bulkheads** limit the *number of concurrent calls* with a counter, without dedicated threads. They are cheaper and fit asynchronous code, where a waiting call holds no thread at all.

**In modern .NET, bulkheads are semaphores, not thread pools**, because:

- Asynchronous I/O means a call awaiting a slow dependency does not hold a ThreadPool thread; what it holds is a *logical* slot — a socket, a pooled connection, memory, and a place in your concurrency budget.
- The .NET ThreadPool is shared process-wide; creating dedicated thread pools per dependency fights the runtime rather than working with it.
- The failure you actually need to bound is *concurrent in-flight work*, which a semaphore bounds exactly.

**The tools:**

| Tool | Use |
|---|---|
| `System.Threading.RateLimiting.ConcurrencyLimiter` | Concurrency cap with an optional bounded queue and ordering (`QueueProcessingOrder.OldestFirst` or `NewestFirst`) |
| Polly `AddConcurrencyLimiter(permitLimit, queueLimit)` (Polly.RateLimiting) | The same limiter as a pipeline strategy; throws `RateLimiterRejectedException` when full |
| Standard resilience handler's rate limiter | A concurrency limiter per `HttpClient` pipeline (default 1,000 permits, no queue) |
| `SemaphoreSlim` | Hand-rolled bulkhead; use `WaitAsync(timeout)` and fail when it returns `false` |
| `SocketsHttpHandler.MaxConnectionsPerServer` | Connection-level bulkhead per host (default: unlimited) |
| Separate named/typed `HttpClient`s via `IHttpClientFactory` | Separate handler chains and connection pools per dependency |
| SqlClient `Max Pool Size` / separate connection strings | Connection-pool bulkheads per workload (pools are keyed by connection string) |
| Bounded `Channel<T>` | Bulkhead for in-process work queues (Module 11, Concept 15) |

```csharp
// A per-dependency bulkhead as a Polly strategy: 50 concurrent calls, 10 queued, then fail fast
builder.Services.AddResiliencePipeline("inventory", pipeline =>
{
    pipeline
        .AddConcurrencyLimiter(permitLimit: 50, queueLimit: 10)
        .AddTimeout(TimeSpan.FromSeconds(2));
});
```

**Sizing.** Start from the dependency's normal peak concurrency (by Little's Law: peak RPS to that dependency × p99 latency) and add headroom — for example 2×. Validate under load test. A bulkhead that rejects during normal peaks is a self-inflicted outage.

**Queue or not.** A small queue smooths microbursts. A large queue defeats the purpose — it converts overload into latency, and latency into timeouts upstream. **Prefer no queue or a tiny one**, and consider `NewestFirst` under overload (Concept 36).

**The interview-grade sentence:** *"In async .NET a bulkhead is a concurrency limiter, not a thread pool — a ConcurrencyLimiter or Polly's limiter per dependency, sized from peak RPS times p99 latency with headroom, with at most a tiny queue — plus separate HttpClients and connection pools so dependencies don't share sockets."*

---

## Concept 32 — Bulkheads at the architecture level

The same idea scales up from "a semaphore per dependency" to entire deployments.

**Per dependency** — Concept 31.

**Per workload type.** Separate the interactive API from background jobs, report generation, and batch imports: different deployments, different compute pools, different database connection pools (or read replicas), different queues. A runaway export should never consume the capacity that serves checkout. In Kubernetes, separate node pools and resource quotas; in Azure App Service, separate plans; in SQL, Resource Governor or separate replicas.

**Per priority / criticality.** Separate paths for critical and non-critical traffic: a dedicated queue (the Azure *Priority Queue* pattern), dedicated consumers, dedicated capacity. Under overload, the non-critical partition is shed first (Concept 35).

**Per tenant.** In a multi-tenant system (Module 12, Concept 53), one tenant's traffic spike, poison message, or pathological query should not affect others:

- Per-tenant rate limits and concurrency limits at the edge (ASP.NET Core partitioned rate limiters, Concept 61).
- Per-tenant queues or sessions (Service Bus sessions keyed by tenant), so one tenant's backlog does not block another's.
- Tier-based pools: premium tenants on dedicated capacity, the long tail pooled.
- Silo or bridge models for the largest tenants.

**Per consumer of your API.** Different client applications get different quotas and possibly different deployments (the *Backends for Frontends* shape), so a buggy mobile release cannot overload the partner API.

**Per region and zone.** A deployment in each zone that can serve independently (Concept 40).

**The cost.** Every partition is capacity that cannot be borrowed by others — **bulkheads trade utilization for isolation.** Pooling maximizes efficiency; partitioning maximizes containment. Mature designs usually combine a shared pool with per-partition *limits* (so no partition can take more than, say, 30% of the pool) rather than fully dedicated capacity, and reserve dedicated capacity for the most critical or most dangerous workloads.

**The interview-grade sentence:** *"I apply bulkheads at every level: per dependency in the process, per workload in the deployment — interactive separate from batch — per priority in the queues, and per tenant at the edge with partitioned limits. The trade is utilization for containment, so I usually cap shares of a shared pool rather than fully dedicate capacity."*

---

## Concept 33 — Cells and deployment stamps

**Cell-based architecture** takes bulkheads to their logical conclusion: instead of one large deployment serving all customers, you run **many independent, complete copies of the stack** — compute, data, queues, caches — each serving a **disjoint subset of customers**. A **thin routing layer** maps each request to its cell.

AWS's whitepaper *Reducing the Scope of Impact with Cell-Based Architecture* describes the pattern, and Azure's Architecture Center describes the same idea as the **Deployment Stamps** pattern (with the **Geode** pattern as a related globally distributed variant). Slack's engineering blog describes migrating to a cellular architecture where it can drain traffic away from an availability zone during gray failures.

**What cells buy you:**

- **Blast radius equals one cell.** A bad deployment, a poison tenant, a runaway query, or a corrupted datastore affects only the customers in that cell. With 20 cells, a cell-scoped failure touches about 5% of customers.
- **Deployments become naturally progressive** — deploy to one cell, bake, then the next (Concept 54).
- **Known maximum scale per cell.** You test a cell at its maximum size once, then scale out by adding cells rather than by growing a single system past tested limits.
- **Data locality and compliance**: cells can be regional.

**What cells cost:**

- **The router is a new critical component.** It must be simple, highly available, and statically stable — ideally a lookup (customer → cell) cached aggressively, with no complex logic. If the router fails, every cell is unreachable.
- **Cross-cell operations are hard.** Global queries, cross-customer features, and aggregates need a separate path (often asynchronous replication into an analytics store).
- **Migrating customers between cells** (rebalancing, a customer outgrowing its cell) is a data-migration project — Module 8's resharding problem.
- **Operational fan-out.** N cells means N of everything to deploy, monitor, patch, and pay for.

**When to reach for cells:** multi-tenant SaaS at significant scale; systems where a single tenant can plausibly harm others; regulated or regional data requirements; organizations that need the blast-radius guarantee more than they need cross-tenant features. **When not to:** early-stage products, small tenant counts, or domains dominated by cross-customer interactions.

**The interview-grade sentence:** *"Cells, or deployment stamps in Azure's vocabulary, are complete independent copies of the stack for disjoint customer sets behind a thin router. They cap the blast radius of anything — a deploy, a poison tenant, a bad datastore — at one cell, at the cost of a highly available router and harder cross-cell work."*

---

## Concept 34 — Shuffle sharding

Shuffle sharding, described in Amazon's Builders' Library article *Workload isolation using shuffle-sharding*, improves on plain sharding for the "one bad customer" problem.

**Plain sharding:** 8 workers split into 4 shards of 2; each customer is assigned to one shard. If a customer sends a poison request that crashes its workers, it takes down its shard — and **every other customer on that shard** (25% of customers).

**Shuffle sharding:** each customer is assigned a **random combination** of workers (e.g., 2 of the 8). The number of distinct combinations is `C(8, 2) = 28`. A poison customer destroys its 2 workers; another customer is fully affected only if it has **exactly the same pair**, which is 1 in 28. Customers who share one of the two workers still have one healthy worker and — with retries or a client that tries both — continue to be served.

The combinatorics grow fast. With 100 workers and a shard size of 5, there are `C(100, 5) = 75,287,520` combinations, so the probability that two customers share an identical shard is negligible. Amazon uses this for Route 53's DNS infrastructure, where each customer domain is served by a distinct combination of name servers.

**Requirements for shuffle sharding to work:**

- **Clients must be fault tolerant across their shard** — they retry on another member of their combination. Without that, partial overlap still hurts.
- **The poison must be tied to the customer's identity**, so it lands only on that customer's shard. Shuffle sharding does not help against failures that affect every worker (a bad deploy, a global config).
- **Assignment must be stable** (store it, or derive it deterministically from the customer ID with a hash), so a customer does not wander across the fleet.

**Where it applies:** multi-tenant queues and worker pools, API front ends, per-tenant caches, DNS and edge infrastructure. It is often combined with cells: shuffle-shard within a cell.

**The interview-grade sentence:** *"Shuffle sharding assigns each tenant a random subset of workers, so a poison tenant takes out only its own combination; with a hundred workers and shards of five there are about seventy-five million combinations, so almost nobody shares the whole blast radius — as long as clients retry across their shard."*

---

## Concept 35 — Load shedding, admission control, and criticality

Module 6 introduced load shedding as a scaling concept; here it is as a reliability pattern. **Load shedding is the deliberate rejection of some work so that the rest can be served within its deadline.** Amazon's Builders' Library article *Using load shedding to avoid overload* makes the key point: a server accepting more work than it can complete in time drives goodput toward zero (Concept 10); rejecting the excess keeps goodput near capacity.

**Principles:**

1. **Reject early and cheaply.** The cost of rejecting must be far lower than the cost of serving; otherwise rejection itself becomes the overload. Reject at the edge, before authentication-heavy work, deserialization of large bodies, or database calls where possible.
2. **Reject based on a real signal.** Concurrency in flight, queue depth, queue age (the time the oldest request has waited), CPU, or measured latency — not a static RPS number that was right last year. Concurrency and queue age are the most robust because they reflect actual capacity.
3. **Reject by priority.** Classify requests by **criticality** — Google's SRE book describes levels like `CRITICAL_PLUS`, `CRITICAL`, `SHEDDABLE_PLUS`, and `SHEDDABLE` — and shed the least critical first. Checkout before recommendations; user-facing before batch; retries before first attempts; health checks never.
4. **Propagate criticality downstream** with the request (a header), so dependencies shed consistently.
5. **Return a response that stops retries** — 503 with `Retry-After`, or a "don't retry" marker (Concept 18).
6. **Protect the shedding mechanism itself** — it must keep working when everything else is overloaded.

**Where shedding happens:**

| Layer | Mechanism |
|---|---|
| Edge / API gateway | Rate limits per client, WAF, Azure Front Door rules, API Management policies |
| Load balancer | Connection limits, queue limits |
| Service (ASP.NET Core) | Concurrency limiter middleware, request queue limits, Kestrel `MaxConcurrentConnections` |
| Handler | Deadline check before expensive work (Concept 13) |
| Queue consumer | Drop or dead-letter messages older than their usefulness (Module 11) |
| Client | Adaptive throttling (Concept 19) |

**Brownout / feature shedding.** A softer form of shedding reduces the *cost* of each request rather than the number: disable expensive optional features (personalization, related items, rich rendering) when load exceeds a threshold. This keeps every user served with a lighter experience, and it is a product decision to be designed in advance.

**The interview-grade sentence:** *"Under overload I'd rather serve 80% of requests on time than 100% late. So I shed early and cheaply, based on concurrency or queue age rather than a static RPS, in priority order — optional features, then batch, then retries — and return a 503 with Retry-After so clients don't make it worse."*

---

## Concept 36 — Adaptive concurrency limits and queue discipline

**Static limits go stale.** A concurrency limit of 200 is right until a deployment makes requests 30% slower, a dependency degrades, or the instance size changes. **Adaptive concurrency limits** discover the right limit continuously from latency, borrowing from TCP congestion control.

**Netflix's `concurrency-limits` library** (and its accompanying blog post *Performance Under Load*) applies algorithms such as **Vegas** and **Gradient** to request concurrency: measure the minimum (no-queueing) latency and the current latency; if latency rises relative to the baseline, queueing is happening, so reduce the limit; if it stays flat, probe upward. The result is a limit that tracks actual capacity and sheds excess automatically. Envoy offers an adaptive concurrency filter based on the same idea, and Uber, Google, and others run similar systems internally.

**Queue discipline under overload.** Most queues are FIFO, which is exactly wrong when overloaded: the request at the head of the queue has been waiting longest and its client is the most likely to have already given up. Facebook's *Fail at Scale* (Ben Maurer, ACM Queue, 2015) describes two techniques used across its services:

- **Controlled Delay (CoDel)**, adapted from network queue management (Nichols & Jacobson, *Controlling Queue Delay*, ACM Queue 2012): if the queue has not been empty within a recent interval (Facebook's example uses 100 ms), apply a very short timeout (5 ms in their example) to items waiting in the queue, so a standing queue is drained aggressively instead of growing. When load is normal, the queue drains periodically and generous timeouts apply.
- **Adaptive LIFO**: under normal conditions process FIFO; when a queue has built up, switch to LIFO so the **newest** requests — whose clients are still waiting — are served first, and old requests time out at the back.

In .NET, `ConcurrencyLimiter` supports `QueueProcessingOrder.NewestFirst`, which gives you LIFO admission from a bounded queue — a small detail that directly implements the goodput-preserving discipline.

**Bounded queues, always.** Every queue in a request path — in-process channels, thread-pool work items, broker queues with a TTL, Kestrel's request queue — needs a bound, and ideally an age limit. An unbounded queue is a latency and memory bomb (Module 6, Concept 17; Module 11, Concept 15).

**The interview-grade sentence:** *"Static concurrency limits go stale, so for high-traffic services I prefer adaptive limits that back off when latency rises, like TCP congestion control. And under overload I switch queues to newest-first with a short queue timeout — CoDel plus adaptive LIFO — because the oldest request's client has usually already left."*

---

## Concept 37 — Four words for "too much": rate limiting, throttling, load shedding, backpressure

These are often used interchangeably. They are different mechanisms with different owners and signals, and separating them is a crisp seniority signal.

| Mechanism | Who decides | Based on | Purpose | Typical response |
|---|---|---|---|---|
| **Rate limiting** | The provider, by policy | A client's request *rate* against a quota (token bucket, fixed/sliding window) | Fairness, abuse prevention, cost control, contractual tiers | 429 + `Retry-After` |
| **Throttling** | The provider, dynamically | Current resource consumption (RUs, CPU, IOPS) | Protect a shared resource from a heavy consumer | 429 / 503, or slowed processing |
| **Load shedding** | The service, dynamically | Its own capacity right now (concurrency, queue age, latency) | Protect goodput under overload | 503 (fast), priority-based |
| **Backpressure** | The consumer, implicitly | Its own processing rate | Slow the producer to match the consumer | Blocking, credit-based flow control, bounded buffers |

**Key distinctions:**

- **Rate limiting can reject when the system is idle** (the client exceeded its quota); load shedding **never rejects when the system has spare capacity**.
- **Rate limiting is per client; load shedding is per server.** A system usually needs both: per-client limits so one client cannot starve others, and server-side shedding so the aggregate of well-behaved clients cannot exceed capacity.
- **Backpressure is a property of the protocol or pipeline** (TCP windows, AMQP credits, `Channel<T>` bounded writes, reactive streams), not a policy. Once you put a durable queue in the middle, you have removed backpressure by design and must add load shedding at the front (Module 11, Concept 15).
- **Throttling** is the word Azure uses for resource-governance rejections (Cosmos RU throttling, Azure SQL resource limits, ARM request limits), and it is the concern of the Azure *Throttling* pattern.

In ASP.NET Core, `Microsoft.AspNetCore.RateLimiting` provides fixed window, sliding window, token bucket, and **concurrency** limiters, optionally partitioned per key. The first three are rate limiting; the concurrency limiter is load shedding (Concept 61).

**The interview-grade sentence:** *"Rate limiting enforces a client's quota even when I'm idle; load shedding protects my capacity regardless of who's asking; throttling is a provider protecting a shared resource from a heavy consumer; and backpressure is the consumer slowing the producer through the protocol. A real system needs per-client limits and server-side shedding together."*

---

# Part F — Degradation, hedging, and health

## Concept 38 — Hedged requests and the tail at scale

**The tail-latency problem.** Dean and Barroso's *The Tail at Scale* (Communications of the ACM, 2013) showed why rare slowness dominates large systems. If a single server has a p99 latency of 1 second, a request that fans out to **100** such servers and waits for all of them sees at least one 1-second response with probability `1 − 0.99¹⁰⁰ ≈ 63%`. At scale, **the tail of the components becomes the median of the system.**

**Hedged requests** attack this directly: send the request to one replica; if no response arrives within a short delay (typically around the p95 of expected latency), send a second copy to another replica; use whichever responds first and cancel the other. Because the hedge fires only for the slowest ~5% of requests, the additional load is small. The paper reports a Google BigTable benchmark in which sending a hedge after 10 ms reduced the p99.9 latency of a 1,000-key read from 1,800 ms to 74 ms while sending only about 2% more requests.

**Variants:**

- **Hedged (delayed) requests** — as above; the standard form.
- **Parallel (speculative) requests** — send to multiple replicas immediately; lower latency, much higher load; rarely justified.
- **Tied requests** — send to two servers with a note identifying the other; whichever starts processing first tells the other to cancel. Reduces duplicated work at the cost of coordination.
- **Cross-region hedging** — the Cosmos DB .NET SDK's threshold-based **availability strategy** (`AvailabilityStrategy.CrossRegionHedgingStrategy(threshold, thresholdStep)`) sends a read to the next preferred region if the first has not answered within the threshold, and uses the first response. It can optionally hedge writes on multi-write accounts, with the documented caveat of more 409/412 conflicts.

**Rules for safe hedging:**

1. **Only hedge idempotent operations** — reads, or writes protected by idempotency keys and conflict handling (Concept 20). A hedged "create order" without a key creates two orders.
2. **Budget it.** Hedging is load. Cap hedges to a small percentage of traffic, and **stop hedging when the system is overloaded** — otherwise hedging becomes retry amplification at the moment the system is saturated. Hedge only against replicas that are healthy (the standard hedging handler uses a circuit breaker per endpoint for exactly this).
3. **Cancel the loser** promptly and propagate the cancellation so the losing replica stops working.
4. **Hedge to independent replicas** — a hedge to the same overloaded host or the same slow partition gains nothing.
5. **Set the delay from data** — around p90–p95 of the operation's latency, not a guess.

In .NET, Polly v8 has a **hedging strategy** (`AddHedging`, default one extra attempt after 2 seconds), and `Microsoft.Extensions.Http.Resilience` has a **standard hedging handler** that can route hedges to different endpoints (Concept 59).

**The interview-grade sentence:** *"When tail latency matters and the operation is idempotent, I hedge: send a second request to a different replica after about the p95, take the first answer, cancel the other. It costs a few percent of extra load, so I budget it and turn it off under overload."*

---

## Concept 39 — Fallbacks, graceful degradation, and the case against fallback

**Graceful degradation** means the system keeps delivering its core value when parts fail, with reduced function. It is the payoff of classifying dependencies (Concept 6): soft dependencies get a designed degraded mode.

**A degradation ladder** for an e-commerce product page, designed before any incident:

| Level | Condition | Behaviour |
|---|---|---|
| 0 — Full | All healthy | Personalized recommendations, live inventory, reviews, dynamic pricing |
| 1 — Reduced | Recommendations unhealthy | Generic "popular items" from a static list |
| 2 — Lean | Load above threshold | Reviews and related items hidden; cached inventory "in stock / low stock" only |
| 3 — Essential | Severe overload or partial outage | Static product data from CDN, add-to-cart and checkout only |
| 4 — Queue | Payment dependency down | Accept orders, authorize later, notify by email |

Each level is a **feature flag or a runtime switch** that can be flipped automatically (by a breaker, a load signal) or manually (by an incident commander), and each is **tested**.

**The case against fallback.** Amazon's Builders' Library article *Avoiding fallback in distributed systems* makes an argument that every architect should be able to reproduce, because it runs against intuition:

- **Fallback paths are rarely exercised**, so they are poorly tested and often broken when needed.
- **A fallback often has different load characteristics** — e.g., falling back from a local cache to a remote database — and **activates at the worst moment**, when many clients fail over at once, overloading the fallback target (a latent bimodal behaviour).
- **Fallback can mask failures**, hiding a real problem until the fallback also fails.
- **Fallback code can itself contain bugs** that turn a partial failure into a total one.

Amazon's preferred alternatives: make the **primary path more reliable** (redundancy, static stability); **push data proactively** rather than fetching it on demand; and if a fallback exists, **exercise it continuously** so it is not a special mode — for example, serve a fraction of production traffic through it at all times.

**Reconciling the two.** Degradation that *reduces* work (hide a feature, serve static content, queue for later) is low-risk because it is cheaper than the primary path. Fallback that *redirects* work to another system (another database, another region, another provider) is high-risk because it assumes spare capacity and tested behaviour on that other system. Prefer the first; treat the second as a system you must test like production.

**The interview-grade sentence:** *"I design degradation that does less work — hide optional features, serve cached or static content, queue for later — and I'm wary of fallbacks that redirect load to another system, because those paths are rarely tested and activate exactly when everything is stressed. If I keep one, I run real traffic through it continuously."*

---

## Concept 40 — Static stability and constant work

**Static stability** (Amazon's Builders' Library: *Static stability using Availability Zones*) is the property that a system **keeps working in its current state during a failure without needing to make changes** — no scaling, no launching instances, no reconfiguring routing, no calls to a control plane.

**The canonical example.** A service needs 6 instances to serve peak load and runs in 3 zones.

- **Not statically stable:** 2 instances per zone (6 total). If a zone fails, the service must launch 3 new instances in the remaining zones — which requires the compute control plane to be healthy, capacity to be available, and time to boot and warm up, all during an event that may be impairing exactly those things.
- **Statically stable:** 3 instances per zone (9 total). If a zone fails, the remaining 6 instances already carry peak load. Nothing needs to change.

The same principle applies to data: a statically stable system has already replicated what it needs into each zone, already has credentials and configuration cached locally, and already holds the connections it will need.

**Constant work.** A related Builders' Library idea (*Reliability, constant work, and a good cup of coffee*) is to design systems that **do the same amount of work regardless of whether things are changing or failing**. For example, a configuration distributor that pushes the **entire** configuration every interval (rather than only deltas) does the same work in steady state as during a mass change, so a sudden burst of changes cannot overload it. Systems that do more work during failures — reconnecting everything, recomputing everything, re-syncing everything — are metastable failure candidates (Concept 10).

**Design checklist for static stability:**

- Pre-provision capacity for the loss of the largest failure domain you intend to survive (Concept 43).
- Cache configuration, secrets, and routing data locally, with **last-known-good** semantics, so a control-plane outage does not stop the data plane (Concept 51).
- Avoid dependencies on control-plane APIs in the request path (resource creation, IAM policy evaluation that is not cached, service discovery lookups without caching).
- Prefer push-based, full-state distribution for critical configuration.
- Make failover a **routing** change, not a **provisioning** change.

**The interview-grade sentence:** *"I want the system to be statically stable: if a zone fails, the remaining zones already have enough capacity, credentials, and config to carry the load without launching anything or calling a control plane. Recovery that requires provisioning during an outage is recovery that often won't work."*

---

## Concept 41 — Health checks: liveness, readiness, startup — and failing open

Health checks decide which instances receive traffic and which get restarted. Designed well, they remove broken instances. Designed badly, they remove healthy ones and start cascades (Concept 9).

**Three different questions** (Kubernetes names them, and the distinction applies everywhere):

| Probe | Question | Failure action | Should check |
|---|---|---|---|
| **Liveness** | "Is this process irrecoverably broken?" | **Restart** the container | Only the process itself: event loop responsive, not deadlocked. **Never dependencies** |
| **Readiness** | "Should this instance receive traffic right now?" | **Stop routing** traffic to it (no restart) | Warm-up complete, local resources OK, possibly critical local dependencies |
| **Startup** | "Has the app finished starting?" | Delay liveness/readiness until it has | Initialization complete (migrations checked, caches primed) |

**Why liveness must not check dependencies.** If the liveness probe calls the database and the database has a 30-second blip, every instance fails liveness and Kubernetes **restarts the entire fleet** at once — converting a 30-second dependency blip into a multi-minute total outage with a cold-start storm. Restarting your process does not fix the database.

**Shallow vs deep checks.**

- **Shallow** checks verify the process can respond (a trivial endpoint). They are cheap and safe but blind to gray failures (Concept 4).
- **Deep** checks verify dependencies and real functionality. They catch more, but they are expensive (health-check traffic × instances × frequency can be significant load on a database), and they make many instances fail simultaneously when a shared dependency blips.

Amazon's Builders' Library article *Implementing health checks* recommends combining them carefully: use deep checks for **local** resources (disk full, a corrupted local cache, a missing certificate) that genuinely make *this* instance bad; treat **shared** dependency failures differently, because removing every instance for a shared-dependency failure helps nobody.

**Fail open.** The critical safety property: **if every instance appears unhealthy, route to all of them anyway.** An all-unhealthy state usually means the health check is wrong or a shared dependency is failing — and in either case, removing all capacity guarantees a total outage, while continuing to route might serve some requests. AWS load balancers and Route 53 behave this way, and Azure Front Door documents similar behaviour: if every origin in an origin group fails its probes, it treats them all as healthy and continues distributing traffic. When you write your own discovery or routing logic, implement the same rule — and **cap the fraction of instances that can be removed at once**.

**ASP.NET Core health checks** (`Microsoft.Extensions.Diagnostics.HealthChecks`):

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddCheck<LocalDiskSpaceCheck>("disk", tags: ["ready"])
    .AddCheck<WarmupCompleteCheck>("warmup", tags: ["ready", "startup"]);
    // Dependency checks (e.g., AspNetCore.HealthChecks.SqlServer) go under a separate
    // "deps" tag for dashboards and alerting — not for liveness.

app.MapHealthChecks("/health/live",    new() { Predicate = r => r.Tags.Contains("live") });
app.MapHealthChecks("/health/ready",   new() { Predicate = r => r.Tags.Contains("ready") });
app.MapHealthChecks("/health/startup", new() { Predicate = r => r.Tags.Contains("startup") });
```

Aspire's service-defaults template maps `/health` (all checks) and `/alive` (live-tagged checks) by default in development — the same separation.

**Health checks as a health model.** At the service level, "healthy / degraded / unhealthy" should be computed from user-facing signals (SLIs), not from component probes. Azure's Well-Architected guidance calls this the **health model** (Concept 57): a layered view in which a dependency failure makes a flow *degraded* rather than *unhealthy* if the flow can degrade.

**The interview-grade sentence:** *"Liveness checks only the process — never dependencies — or a database blip restarts the whole fleet. Readiness gates traffic on local warm-up and local resources. Dependency health goes into dashboards and the health model, not into restart decisions. And routing fails open: if everything looks unhealthy, keep sending traffic."*

---

## Concept 42 — Crash-only design, supervision, and self-healing

**Crash-only software** (Candea and Fox, HotOS 2003) argues that if a component's only way to stop is to crash, and its only way to start is to recover, then **recovery is exercised on every restart** and is therefore reliable. There is no separate, rarely tested "clean shutdown" path whose correctness you depend on.

The practical design principles that follow:

- **Keep durable state outside the process** (databases, queues, blob storage) and make the process disposable.
- **Make startup idempotent and fast.** Startup must tolerate half-finished work from the previous incarnation (leases, locks, in-flight messages — Modules 9 and 11).
- **Prefer crashing to limping.** On an unexpected invariant violation or unrecoverable corruption, exit and let the supervisor restart you rather than continuing in an unknown state. (`Environment.FailFast` exists for the truly unrecoverable case.)
- **Use timeouts and leases on every external resource**, so a crashed holder's resources are released automatically.

**Supervision and self-healing.** The platform — Kubernetes, Azure Container Apps, App Service, Service Fabric, a process manager — restarts failed processes and replaces failed nodes. Erlang/OTP's supervision trees are the classic model: a supervisor restarts children according to a strategy, with a **maximum restart intensity** after which it escalates rather than looping.

**Bound the self-healing.** Automatic remediation is itself a source of cascades:

- **Restart storms:** a crash-looping deployment consumes capacity and hammers dependencies at startup. Kubernetes uses exponential backoff (`CrashLoopBackOff`) for exactly this reason.
- **Mass replacement:** an autoscaler or node-repair system that replaces many nodes at once because of a false signal (a health check bug, a monitoring outage) can take down a healthy fleet. **Rate-limit remediation** (at most *k*% of capacity per interval), and stop automation when too many things look unhealthy (the same logic as failing open).
- **Remediation that depends on the failing thing:** self-healing that needs the control plane to be healthy won't work during a control-plane incident (Concept 51).

**Graceful shutdown still matters** in the planned case (deployments, scale-in): stop accepting new work, finish or hand off in-flight work within a bounded time, release leases. In .NET, `IHostApplicationLifetime` and `HostOptions.ShutdownTimeout` (30 seconds by default in current .NET) govern this; align it with the orchestrator's termination grace period (Kubernetes' default is also 30 seconds) and with load balancer draining (Concept 61).

**The interview-grade sentence:** *"I design processes to be crash-only: state lives outside, startup is idempotent recovery, and on corruption we exit rather than limp. The platform supervises and restarts — but I rate-limit automated remediation, because a health-check bug plus unbounded self-healing can take down a healthy fleet."*

---

# Part G — Redundancy and failover

## Concept 43 — Redundancy vocabulary and capacity headroom for failure

**Redundancy notation:**

| Notation | Meaning | Survives |
|---|---|---|
| **N** | Exactly enough capacity for peak load | Nothing |
| **N+1** | One spare unit beyond peak need | One unit failure (not during maintenance) |
| **N+2** | Two spare units | One failure during planned maintenance, or two failures |
| **2N** | A full duplicate of the capacity | Loss of an entire side (e.g., one of two datacenters) |
| **2N+1** | Full duplicate plus a spare | Loss of a side plus one more failure |

The N+2 argument is worth making explicitly: **at any time, some capacity is out for planned maintenance** (patching, deployment), so N+1 leaves you with zero spare during maintenance windows — which is when failures are most likely.

**The zone-loss headroom formula.** If load is spread evenly across **Z** zones and you want to survive the loss of one zone with no scaling (static stability, Concept 40), each zone must be able to carry `1/(Z−1)` of the total load, so the fleet must be provisioned at `Z/(Z−1)` of peak:

| Zones | Provisioned capacity (× peak) | Max steady-state utilization |
|---|---|---|
| 2 | 2.0× | 50% |
| 3 | 1.5× | ~67% |
| 4 | 1.33× | 75% |
| 6 | 1.2× | ~83% |

This is a strong argument for **three zones rather than two**: the overhead drops from 100% to 50%. The same arithmetic applies to regions in active-active (Concept 45): two regions each serving half the traffic must each be able to serve all of it.

**Why "we'll autoscale during the failure" is weak:**

- Scaling takes minutes (provisioning, boot, JIT warm-up, cache warm-up), and the load arrives instantly.
- Scaling depends on the control plane and on available capacity in the surviving zones — which is under pressure because **everyone else is scaling there too** during a zone failure.
- Autoscaling reacts to load that has already caused latency.

Autoscaling is still valuable for *demand* changes; for *failure* capacity, pre-provision the critical tier and let autoscaling handle the rest.

**Redundancy is not only compute.** Apply the same thinking to: database replicas (and their capacity to take primary load), cache nodes, queue brokers, network paths (dual ExpressRoute circuits, redundant VPN tunnels), DNS providers, and people (on-call depth).

**The interview-grade sentence:** *"To survive losing one of three zones without scaling, I need 1.5× peak capacity, so I run the critical tier at no more than about two-thirds utilization. With two zones it's 2×, which is one reason I prefer three. I don't count on autoscaling during the failure, because everyone else is trying to scale in the same place."*

---

## Concept 44 — Active-passive: cold, warm, hot standby

In **active-passive** (also primary-standby, primary-secondary), one site or instance serves traffic; the others stand by and take over on failure. At the data layer, this is leader-follower replication (Module 8) plus a failover procedure (Module 9's leader election and fencing).

**The standby temperature spectrum:**

| Standby | State of the passive site | Failover time | Cost |
|---|---|---|---|
| **Cold** | Infrastructure defined (IaC) but not deployed; data restored from backups | Hours | Lowest |
| **Warm** | Infrastructure deployed at reduced scale; data continuously replicated | Minutes to tens of minutes (scale-up required) | Moderate |
| **Hot** | Fully deployed at full scale; data replicated; not serving user traffic | Seconds to minutes | Near 2× |

**RTO decomposition.** The recovery time for active-passive is the sum of several steps, and each needs a number:

```
RTO = detect + decide + (scale up) + promote data + redirect traffic + warm up
```

- **Detect:** how long until you know the primary is really down (not flapping) — often 1–5 minutes to avoid false positives.
- **Decide:** automatic or human. Human decisions take 10–30 minutes at 3 a.m.
- **Scale up:** only for warm standby — and dependent on the control plane (Concept 51).
- **Promote data:** replica promotion, failover group switch, DNS listener change.
- **Redirect traffic:** DNS TTLs, client connection pools holding old addresses (`PooledConnectionLifetime`, Concept 12), global router reconfiguration.
- **Warm up:** caches, JIT, connection pools — a cold standby under full load is a metastable-failure candidate (Concept 10).

**RPO.** With asynchronous replication, RPO equals the replication lag at the moment of failure — which is largest precisely when the primary is under stress. With synchronous replication, RPO is zero, but every write pays the latency to the standby (practical within a region or across zones, rarely across regions).

**Strengths of active-passive:**

- **Simple data model** — one writer, no conflicts, strong consistency is achievable.
- **Cheaper** at the warm and cold end.
- **Fits most relational databases** and most existing applications without changes.

**Weaknesses:**

- **The passive side is untested under real load** unless you deliberately exercise it. The failover path is exactly the "rarely exercised mode" of Concept 28.
- **Idle capacity** (for hot standby) that delivers nothing day-to-day.
- **Failover is an event** with a decision, a procedure, and risks (split brain, data loss window, failback).

**The interview-grade sentence:** *"Active-passive keeps one writer and a standby, which keeps the data model simple. I state the RTO as a sum — detect, decide, promote, redirect, warm up — and the RPO as the async replication lag, and I'm explicit that the standby is untested unless we fail over to it on a schedule."*

---

## Concept 45 — Active-active: what it actually requires

In **active-active**, two or more sites serve production traffic simultaneously. If one fails, the others absorb its share. At the stateless tier this is easy — it is just load balancing across regions. **The hard part is always the data.**

**The data strategies for active-active, from simplest to hardest:**

| Strategy | How it works | Trade-off |
|---|---|---|
| **Read-local, write-central** | All regions serve reads from local replicas; writes go to a single primary region | Simple; writes pay cross-region latency and have a single region dependency. Technically "active-active for reads, active-passive for writes" |
| **Partitioned ownership (homing)** | Each user/tenant/entity is owned by one region; writes go to the owner; regions replicate for failover | No conflicts; failover moves ownership; cross-partition operations are harder |
| **Multi-leader with conflict resolution** | Every region accepts writes; async replication; conflicts resolved by LWW, CRDTs, or custom merge (Modules 8, 9) | Low-latency writes everywhere; conflicts are a product problem; LWW silently loses data |
| **Globally consistent store** | A consensus-based distributed database (Spanner-class, or Cosmos DB with strong consistency where supported) | No conflicts; every write pays cross-region consensus latency (PACELC, Module 7) |

**Requirements beyond data:**

1. **Capacity:** each region must be able to absorb the traffic of any region that fails (Concept 43). Two regions at 50% each must be sized for 100% each.
2. **Routing:** a global load balancer (Azure Front Door, Traffic Manager, DNS with health checks, anycast) that can shift traffic away from a failing region quickly, plus **session and affinity handling** for any state that is region-local.
3. **Idempotency everywhere:** clients retried into another region may repeat operations (Concept 20).
4. **Consistency contracts:** users routed to a different region may not see their latest writes (read-your-writes, Module 7). Sticky routing, session tokens, or consistency tokens mitigate it.
5. **Global uniqueness:** IDs generated in multiple regions must not collide (UUIDv7, region-prefixed sequences).
6. **Independent regions:** no hidden cross-region synchronous dependencies (Concept 53).

**The big advantage of active-active: the failover path is always tested.** Both regions carry real traffic every minute, so you know their capacity, configuration, and code work. Removing a region from rotation is a routing change that you exercise routinely (for deployments, for maintenance), not an emergency procedure.

**The big risk: correlated overload.** When one region fails, its traffic arrives at the others instantly. If they lack headroom, you lose all regions in sequence (load redistribution, Concept 9). Active-active without failover capacity is a cascade waiting for a trigger.

**The interview-grade sentence:** *"Active-active is easy for stateless compute and hard for data, so the first question is the write model: central writes, homed partitions, multi-leader with a conflict strategy, or a consensus store with the latency cost. Then each region needs capacity for the failed region's traffic — otherwise failover is just a cascade."*

---

## Concept 46 — Active-active vs active-passive: making the decision

This is one of the most common architect-round questions, and the wrong answer is a preference. The right answer is a decision from explicit inputs.

| Input | Pushes toward active-passive | Pushes toward active-active |
|---|---|---|
| **RTO** | Minutes to hours acceptable | Seconds required |
| **RPO** | Some loss tolerable, or sync replication available within region | Near-zero across regions (needs consensus store or careful design) |
| **Data model** | Relational, cross-entity transactions, strong invariants | Partitionable by user/tenant, commutative or mergeable updates |
| **Conflict tolerance** | Conflicts unacceptable (money, inventory) | Conflicts rare or resolvable (profiles, carts, likes) |
| **Latency needs** | Users mostly in one geography | Global users needing local writes |
| **Cost** | Budget-constrained (warm/pilot light is cheap) | Budget for ~2× capacity |
| **Team maturity** | Small team, limited operational depth | Strong SRE practice, automated routing and testing |
| **Regulatory** | Data residency pinned to one region | Residency satisfied by regional homing |

**Hybrids are normal, and naming one is a strong answer:**

- **Active-active stateless tier + active-passive data tier.** Both regions serve traffic; all writes go to the primary database region; failover promotes the secondary. Most practical multi-region .NET/Azure systems look like this (for example, App Service or Container Apps in two regions, Azure SQL with a failover group).
- **Active-active by partition.** Each tenant is homed to one region (active-passive *per tenant*), but every region is active for its own tenants. Cells (Concept 33) often take this shape.
- **Per-partition failover.** Azure Cosmos DB's per-partition automatic failover (generally available since June 2026) keeps a single write region per account but fails over **individual partitions** to another region when they are affected, rather than the whole account — Microsoft states a P99 target of about 3 minutes for partition failover. It is a managed version of "active-passive, but at a much smaller failure granularity."
- **Active-active reads, active-passive writes** — Cosmos DB multi-region with a single write region, or Azure SQL geo-replicas serving read-only traffic.

**The senior move is to ask what the business actually needs.** Many teams asking for active-active really need a 15-minute RTO, which a well-tested warm standby delivers at a fraction of the cost and complexity. Concept 66 returns to this.

**The interview-grade sentence:** *"I'd pick from the RTO/RPO, the data model's tolerance for conflicts, cost, and the team's operational maturity. For most transactional systems I'd propose active-active compute with an active-passive data tier and a rehearsed failover — and go full active-active only for flows that can be partitioned or merged."*

---

## Concept 47 — Failover mechanics: detection, decision, fencing, redirection, failback

A failover is a **distributed systems procedure** with several ways to go wrong. Walk it step by step.

**1. Detection — without flapping.**
- Use **multiple independent signals** (synthetic probes from several locations, real user error rates, platform health) and require them to agree over a **window** (e.g., sustained for 2–3 minutes).
- Distinguish "the region is down" from "my monitoring can't reach the region" (the observer's own failure — gray failure again).
- Prefer **user-facing SLIs** over component probes.

**2. Decision — automatic or human.**
- **Within a region (zone failures, instance failures): automatic.** The platform does it; the blast radius of a wrong decision is small.
- **Across regions: often human-approved, or automated with conservative thresholds.** A regional failover has a data-loss window (async RPO), a failback cost, and a real chance of being triggered by a false positive. Many organizations deliberately keep a human in the loop for regional database failover while automating traffic routing. Azure SQL failover groups, for instance, support Microsoft-managed failover with a grace period (at least an hour) but recommend customer-managed failover so you control the decision.
- Whatever you choose, **the decision owner, criteria, and runbook must be written down in advance.**

**3. Fencing the old primary.**
- The old primary must not continue accepting writes after the new one is promoted, or you get **split brain** — two writers diverging (Module 9's fencing tokens, epochs, and lease expiry).
- Mechanisms: managed services handle it (failover groups, Cosmos DB, Service Bus geo-replication promotion); for self-managed systems, STONITH ("shoot the other node in the head"), revoking network access, or epoch numbers checked by the storage layer.

**4. Promotion.**
- Promote the replica; accept that asynchronously replicated writes in flight are lost (RPO) or must be reconciled later.
- Record the **exact point** of promotion (LSN, timestamp) so lost writes can be identified from the old primary's log after recovery.

**5. Redirection.**
- DNS changes are limited by **TTLs** and by clients that cache resolutions longer than the TTL (including .NET processes holding pooled connections — Concept 12).
- Global routers (Front Door) redirect faster than DNS.
- Connection strings pointing at **listener endpoints** (failover group read/write listeners, the Service Bus namespace hostname under geo-replication) avoid application configuration changes.

**6. Warm-up and load.**
- The promoted side receives full load instantly. Cold caches, cold JIT, and cold connection pools make it slow; slow makes it time out; timeouts cause retries (Concept 10). Ramp traffic if the router allows it; pre-warm hot standbys continuously.

**7. Failback.**
- Failing back is a **second failover** with its own risks: the original primary must be fully resynchronized, any divergent writes reconciled, and traffic moved back in a controlled window.
- Many incidents happen during failback because it is rushed after the adrenaline of the original event. Some teams simply do not fail back — the new primary becomes the primary.

**The interview-grade sentence:** *"Failover is a procedure: detect with multiple signals over a window, decide with a pre-agreed owner, fence the old primary so there's no split brain, promote and record the cut-over point, redirect traffic with DNS TTLs and pooled connections in mind, ramp load onto a warm target — and treat failback as a second, planned failover."*

---

## Concept 48 — Availability zones vs regions

**Availability zones** are physically separate datacenter locations within a region, with independent power, cooling, and networking, connected by low-latency links (Azure describes inter-zone round-trip latency on the order of a couple of milliseconds). **Regions** are geographically distant sets of zones, typically hundreds of kilometres apart, with round-trip latencies of tens of milliseconds (Module 5).

That latency difference drives the entire design distinction:

| Aspect | Across zones | Across regions |
|---|---|---|
| Replication | **Synchronous is practical** (RPO = 0) | Usually **asynchronous** (RPO > 0) |
| Failover | Usually **automatic**, platform-managed, seconds | Often a **decision**, minutes to hours |
| Protects against | Datacenter-level failures | Regional disasters, regional control-plane failures, some software incidents |
| Cost | Modest (often 0–30% premium; inter-zone data transfer) | High (duplicate stack, cross-region egress, operational complexity) |
| Complexity | Mostly configuration | Architecture |

**Azure's two deployment models for zones:**

- **Zonal (pinned):** a resource is placed in a specific zone (a VM in zone 1). You achieve resilience by deploying multiple zonal resources and managing replication and failover yourself.
- **Zone-redundant:** the platform spreads the resource across zones and handles replication and failover (zone-redundant storage, Azure SQL zone redundancy, zone-redundant App Service plans, Cosmos DB with zone redundancy, Service Bus Premium in zone-enabled regions).

**The strong default for most production workloads is: zone-redundant within one region**, which handles the most common infrastructure failures automatically at modest cost. Multi-region is a separate decision justified by specific requirements (Concept 66).

**Region pairs.** Azure historically organized regions into **pairs** (for platform update sequencing and some geo-replication defaults like GRS). Newer regions may not have a pair, and Microsoft's reliability guidance now emphasizes choosing secondary regions based on your requirements (latency, data residency, service availability, capacity) rather than assuming the pair. Knowing that pairs exist *and* that they are not a design requirement is the current, accurate position.

**What zones do not protect against:** regional control-plane failures, regional networking failures, bad deployments rolled to all zones at once, and global services (Concept 52). The AWS us-east-1 incident of October 2025 affected a whole region despite zone redundancy, because the failing components were regional services.

**The interview-grade sentence:** *"Zones are close enough for synchronous replication and automatic failover, so zone redundancy is my default for anything production. Regions need async replication and usually a failover decision, so multi-region is a separate, requirement-driven choice — and zones don't protect me from regional control-plane problems or bad global changes."*

---

## Concept 49 — The DR strategy ladder, RPO, and RTO

**RPO (Recovery Point Objective):** the maximum acceptable amount of data loss, measured in time. **RTO (Recovery Time Objective):** the maximum acceptable time to restore service. Module 12 introduced both for backups; here they drive the whole-system DR design.

AWS's whitepaper *Disaster Recovery of Workloads on AWS* names four strategies that map onto any cloud:

| Strategy | What runs in the recovery region | Typical RPO | Typical RTO | Relative cost |
|---|---|---|---|---|
| **Backup & restore** | Nothing; backups are copied there | Hours (backup frequency) | Hours to a day | $ |
| **Pilot light** | Data replicated continuously; core infrastructure provisioned but compute off or minimal | Seconds to minutes | Tens of minutes to hours | $$ |
| **Warm standby** | Scaled-down but fully functional copy, serving no (or test) traffic | Seconds to minutes | Minutes | $$$ |
| **Multi-site active-active** | Full production in each region | Near zero to seconds | Near zero | $$$$ |

**How to choose:**

1. **Do a business impact analysis per workload** (not per company). Ask what an hour of downtime costs and what losing 5 minutes of data costs. The answers differ enormously between the checkout flow, the admin portal, and the analytics pipeline.
2. **Tier the workloads** — e.g., Tier 0 (revenue-critical, minutes), Tier 1 (important, hours), Tier 2 (can wait a day) — and assign a strategy per tier.
3. **Include data corruption and ransomware** in the scenarios. Replication propagates corruption; only point-in-time backups (ideally immutable, in a separate security boundary) recover from it (Module 12, Concept 54).
4. **Account for dependencies.** A Tier 0 workload that depends on a Tier 2 identity system has a Tier 2 RTO.
5. **Measure, don't assume.** The RTO that matters is the one measured in the last drill (Concept 55).

**Pilot light and warm standby depend on the control plane during recovery** (to start or scale compute). That is acceptable for many scenarios, but for the regional-control-plane failure scenario it is a real risk — which is one reason Tier 0 workloads drift toward hot standby or active-active.

**The interview-grade sentence:** *"I tier workloads from a business impact analysis and give each tier a DR strategy — backup-and-restore, pilot light, warm standby, or active-active — with RPO and RTO stated up front, corruption and ransomware included in the scenarios, and the RTO taken from the last drill rather than the design doc."*

---

## Concept 50 — Multi-region data: the replication mode is the RPO

Every multi-region design decision eventually reduces to two data questions:

1. **Is replication synchronous or asynchronous?** That decides the **RPO** and the **write latency**.
2. **How many regions accept writes?** That decides the **conflict model**.

| Configuration | RPO on regional loss | Write latency | Conflicts |
|---|---|---|---|
| Single write region, async replicas | Replication lag (seconds typically; more under stress) | Local | None |
| Single write region, sync replica in another region | Zero | + cross-region RTT on every write | None |
| Multi-write, async | Unreplicated writes in the failed region may be lost or delayed | Local | Yes — must be resolved |
| Consensus across regions (3+ regions) | Zero | + quorum RTT | None (serialized) |

**Platform realities (Azure):**

- **Azure SQL Database** active geo-replication and failover groups replicate **asynchronously**; failover groups provide stable read-write and read-only listener endpoints so applications do not change connection strings. Zone-redundant configurations give synchronous durability within a region.
- **Azure Cosmos DB** offers single-write-region accounts (with service-managed failover, and per-partition automatic failover) and multi-region-write accounts with conflict resolution (last-writer-wins on a configurable path by default, or a custom stored-procedure policy). Its consistency levels (Module 7) interact with multi-region: strong consistency constrains region distances and is not available with multi-region writes.
- **Azure Storage** GRS/GZRS replicate **asynchronously** to the secondary region; Microsoft does not provide a guaranteed RPO for that replication, and customer-managed failover is available so you can decide when to fail over.
- **Service Bus Premium geo-replication** (generally available since December 2025) replicates both metadata and messages to a secondary region, with synchronous or asynchronous replication modes you choose, and supports promoting the secondary under the same namespace hostname. The older **Geo-Disaster Recovery** feature replicates **metadata only** — messages in the failed region are not available after failover — and the two features cannot be combined.
- **Event Hubs** has an equivalent geo-replication capability for its higher tiers; Kafka-based systems use MirrorMaker-style replication with offset translation (Module 11).
- **Azure Managed Redis** supports active geo-replication with CRDT-based conflict resolution (Module 10).

**Two traps that distinguish experienced candidates:**

- **Replication lag grows exactly when you need it to be small.** Under heavy write load or a degrading primary, async lag increases, so the RPO at the moment of failure is worse than the steady-state metric. Monitor lag and alert on it as an RPO SLI.
- **Every data store in the flow must fail over consistently.** If orders are in SQL (failed over, RPO 5 s), order events are in Service Bus (failed over, different RPO), and the search index is rebuilt from events, the regions may disagree about which orders exist. Design for reconciliation: idempotent consumers, the outbox pattern (Module 11), and a replay capability.

**The interview-grade sentence:** *"In a multi-region design the replication mode is the RPO and the number of write regions is the conflict model. On Azure most geo-replication is asynchronous, so I state the lag as the RPO, alert on it, and design reconciliation for the case where SQL, Service Bus, and the search index fail over at slightly different points."*

---

## Concept 51 — Control plane vs data plane: don't depend on the control plane to recover

**The data plane** is the part of a service that does the work: serving HTTP requests, reading and writing data, forwarding packets. **The control plane** is the part that manages resources: creating, configuring, scaling, deleting (in Azure, largely Azure Resource Manager and each service's management APIs).

The distinction matters because **control planes are typically more complex, change more often, and are less available than data planes**, and cloud providers design data planes to keep working when control planes fail. AWS's October 2025 incident illustrated it cleanly: instances that were already running continued to operate while new launches failed (Concept 8). Azure's October 2025 Front Door incident also froze configuration changes during mitigation — the control plane was deliberately locked while the data plane was being repaired.

**Design rules that follow:**

1. **Failover must be a data-plane operation where possible.** Shifting traffic via a pre-configured router with health probes, promoting a replica through a data-plane API, or letting clients switch endpoints is safer than "run a script that creates resources in the other region."
2. **Pre-provision what recovery needs** (static stability, Concept 40). Warm standby that must scale up during a regional control-plane failure may not scale.
3. **Cache control-plane results in the data plane.** Service discovery results, configuration, feature flags, authorization policies, and secrets should be cached locally with last-known-good semantics, so an outage of the config service or Key Vault does not stop request processing. (Key Vault reference caching in App Service and the Azure App Configuration provider's caching are examples.)
4. **Don't put control-plane calls in the request path.** A request handler that calls the management API (to look up a resource, check a quota, create a queue) inherits the control plane's lower availability.
5. **Keep your own control plane out of the hot path too.** Your deployment system, feature-flag service, and admin APIs are control planes. The data plane should continue if they are down.
6. **Know which recovery actions are control-plane operations** — scaling out, creating replicas, changing DNS records, modifying Front Door routes, rotating keys — and prefer designs where none are required during the event.

**The interview-grade sentence:** *"I assume the control plane is less available than the data plane during an incident, so recovery shouldn't require creating or reconfiguring resources. Failover is a routing change on pre-provisioned capacity, and config, secrets, and discovery are cached locally with last-known-good so the request path keeps working without them."*

---

## Concept 52 — Global routing and the ingress single point of failure

Every multi-region system has a **global entry point**: DNS (with a traffic-routing service like Azure Traffic Manager), an anycast edge (Azure Front Door, a CDN), or client-side logic. That entry point is, by construction, **shared by all regions** — the one component your regional redundancy does not protect.

**Options and their failure characteristics:**

| Mechanism | How it routes | Failover speed | Notes |
|---|---|---|---|
| **DNS-based (Traffic Manager, Route 53)** | Returns different IPs per health and policy | Bounded by TTL and client caching (often minutes) | Simple, no data path involvement |
| **Anycast edge / L7 proxy (Front Door)** | Single anycast address; edge routes to healthy origins | Seconds | Adds WAF, caching, TLS termination — and a shared data-path dependency |
| **Client-side** | Mobile/desktop app knows multiple endpoints | As fast as the client logic | Only for clients you control |
| **Regional endpoints exposed directly** | Users or partners can reach a region directly | Manual | A useful emergency path |

**The October 2025 Azure Front Door incident** made this concrete for many organizations: applications that were healthy in every region were unreachable because the shared global edge in front of them was failing, and customers could not reconfigure it during mitigation.

**Microsoft's guidance** (*Global routing redundancy for mission-critical web applications*, plus the Front Door high-availability implementation guide) describes an **alternate ingress path**: Traffic Manager in front, normally sending all traffic to Front Door, with a secondary path (Application Gateway with WAF in each region, or an alternate CDN) that you switch to — typically manually — if Front Door is unavailable. The same guidance is candid that this is **complex and costly and that most workloads do not need it**: WAF rules, certificates, caching behaviour, and origin security must be maintained on both paths, and your origins must accept traffic from both.

**How to reason about it in an interview:**

1. **Name the global entry point as a shared dependency** — this alone is a strong signal.
2. **Estimate its availability contribution** (Concept 5): a 99.99% global router caps a 99.99% system.
3. **Decide whether an alternate path is justified** from the business impact of a multi-hour global edge outage. For most, "we accept it and communicate" is the rational answer; for some (airlines, payments, emergency services), a pre-built, tested alternate path is worth the cost.
4. **Keep an emergency path** at minimum: documented regional endpoints, lowered DNS TTLs on the apex record, and a runbook — cheap measures that give options during a global incident.

**The interview-grade sentence:** *"The global router is the one component every region shares, so it caps my availability. For most systems I'd accept that and keep cheap escape hatches — regional endpoints, short DNS TTLs, a runbook — and only build a full alternate ingress like Traffic Manager plus Application Gateway when the business impact of a global edge outage justifies maintaining two WAF and certificate paths."*

---

## Concept 53 — Hidden shared dependencies

Two regions that look independent on the architecture diagram often share critical dependencies that nobody drew. These are where "multi-region" designs fail in practice.

**The checklist:**

| Hidden dependency | How it bites |
|---|---|
| **DNS** | Your zone is hosted in one provider; a resolver problem makes every region unreachable |
| **Identity provider** | Token issuance (Entra ID or your own IdP) is required for every request; token *validation* should be local (cached signing keys) |
| **Secrets and certificates** | Key Vault in one region; certificates expiring on the same day in every region |
| **Configuration and feature flags** | One global App Configuration store or flag service; a bad value pushed everywhere at once (Concept 8) |
| **Container registry / artifact store** | Scale-out in the DR region needs to pull images from a registry in the failed region (use geo-replicated registries) |
| **CI/CD pipeline** | Can't deploy a fix if the pipeline runs in the affected region or depends on the failing service |
| **Monitoring and alerting** | The telemetry backend is in the failed region; you are blind during the incident. Google's June 2025 status page initially depended on affected infrastructure |
| **Status page and incident tooling** | Communication fails when customers need it most |
| **Third-party APIs** | Payment, email, SMS, maps providers hosted in the same cloud region as you |
| **Global singletons in your own code** | A "global" scheduler, lock service, or ID generator hosted in one region |
| **Shared quotas and limits** | Subscription-level quotas, Cosmos RU/s limits, API rate limits consumed by all regions |
| **Time** | Certificate expiry, license expiry, leap seconds, DST — synchronized across all regions by definition |
| **People and access** | The on-call engineer's access (VPN, bastion, identity) depends on the failing region |

**Techniques to find them:**

- **Dependency mapping from telemetry** (distributed traces show real call graphs, including the ones nobody documented — Module 28).
- **Region-isolation game days:** block the primary region at the network level and see what breaks in the secondary (Concept 55).
- **Startup dependency audits:** list everything a fresh instance calls before it becomes ready.
- **Expiry inventories** for certificates, secrets, domains, and licenses with alerting well ahead of expiry.

**The interview-grade sentence:** *"Most failed multi-region designs die on dependencies nobody drew: DNS, identity, Key Vault, the container registry, the pipeline, the monitoring stack, and certificates that expire everywhere on the same day. I find them with traces and with a game day that cuts off the primary region entirely."*

---

# Part H — Verifying and operating reliability

## Concept 54 — Safe change: progressive exposure is a reliability pattern

Because change is the leading trigger (Concept 8), the deployment system is part of the reliability architecture, not a separate DevOps concern.

**Progressive exposure.** Roll every change — code, configuration, feature flags, infrastructure, data like ML models and policy files — through widening rings:

1. **Pre-production** with production-like traffic (shadow or replayed).
2. **Canary**: a small slice of production (one instance, one cell, a small percentage, internal users).
3. **Early region(s) / low-traffic cells**.
4. **Broader waves**, never more than one zone or region of a redundant pair at a time.
5. **Global**.

Microsoft describes its own **Safe Deployment Practices** for Azure along these lines — staged rollout through canary and pilot regions, then waves of regions with bake time, avoiding simultaneous rollout to paired regions — and Amazon's Builders' Library article *Automating safe, hands-off deployments* describes the equivalent pipeline in detail.

**Bake time.** Wait between waves long enough for slow-burning problems (memory leaks, daily jobs, cache expiries, certificate refresh) to surface. Minutes catch crashes; hours catch leaks.

**Automated health gates and rollback.** Each wave compares key SLIs (error rate, latency, saturation, business metrics like checkout conversion) against the baseline and **rolls back automatically** on regression. Rollback must be faster and safer than roll-forward.

**Rollback safety.** Amazon's *Ensuring rollback safety during deployments* makes the point that a change is only safe if the **previous version can still run** after the new version has written data. The standard technique is the **two-phase (expand/contract) change** you already know from schema migrations (Module 12, Concept 52): first deploy code that can *read* the new format, then deploy code that *writes* it.

**Feature flags — off by default.** Deploy code dark and enable it progressively and independently of deployment. Google's June 2025 remediation was to require feature flags on changes to critical binaries. Two cautions: the flag system is a global control plane (Concept 51) — cache flags locally with last-known-good; and old flags are technical debt — remove them.

**Configuration is code.** Configuration changes get the same review, validation, staged rollout, and rollback as code. Consumers validate configuration and keep running on **last-known-good** if a new version is invalid (Cloudflare's November 2025 lesson).

**Kill switches.** For every risky new path, a pre-built, tested way to disable it quickly — without a deployment.

**Change freezes and error budgets.** An SLO error budget (Module 28) gives an objective rule: when the budget is exhausted, slow or stop risky changes and spend effort on reliability.

**The interview-grade sentence:** *"Since changes cause most outages, I treat deployment as a reliability mechanism: every change — code, config, flags, data files — goes through canary and waves with bake time and automated rollback on SLI regression, new paths ship dark behind flags, and consumers keep last-known-good config instead of crashing."*

---

## Concept 55 — Chaos engineering, fault injection, game days, and DR drills

**Chaos engineering** is the discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions. The *Principles of Chaos Engineering* (from Netflix's practice) define the method:

1. **Define steady state** as a measurable output (orders per minute, successful logins), not internal metrics.
2. **Hypothesize** that steady state continues in both the control and experimental groups.
3. **Introduce real-world events**: instance termination, latency, dependency errors, zone loss, clock skew, disk full, certificate expiry.
4. **Try to disprove the hypothesis** by comparing groups.
5. **Minimize blast radius**, automate experiments, and — as maturity grows — run them in production.

**Fault injection at different levels:**

| Level | Tools | Good for |
|---|---|---|
| **In-process** | Polly's chaos strategies (`AddChaosFault`, `AddChaosLatency`, `AddChaosOutcome`, `AddChaosBehavior`), formerly Simmy | Testing your resilience pipeline's configuration with controlled injection rates |
| **Network / proxy** | Toxiproxy, service mesh fault injection (Istio, Linkerd), WireMock.Net fault responses | Latency, resets, partial responses between real components |
| **Platform** | Azure Chaos Studio (service-direct faults such as zone-down for VM scale sets, Cosmos DB failover, NSG rules; agent-based faults such as CPU, memory, network latency; AKS faults via Chaos Mesh) | Realistic infrastructure failures in Azure |
| **Kubernetes** | Chaos Mesh, LitmusChaos | Pod kill, network partitions, I/O faults |
| **Load** | Azure Load Testing, k6, NBomber | Overload behaviour, the metastable threshold, cold-start capacity |

```csharp
// Polly chaos strategies appended innermost in a custom HTTP pipeline,
// so the retry and breaker above them see the injected faults.
var chaosEnabled = builder.Configuration.GetValue<bool>("Chaos:Enabled");

builder.Services.AddHttpClient<CatalogClient>()
    .AddResilienceHandler("catalog", pipeline =>
    {
        pipeline
            .AddTimeout(TimeSpan.FromSeconds(5))                                   // total
            .AddRetry(new HttpRetryStrategyOptions { MaxRetryAttempts = 2, UseJitter = true })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions())
            .AddTimeout(TimeSpan.FromSeconds(1))                                   // per attempt
            .AddChaosLatency(new ChaosLatencyStrategyOptions
            {
                Enabled = chaosEnabled, InjectionRate = 0.05, Latency = TimeSpan.FromSeconds(3)
            })
            .AddChaosFault(new ChaosFaultStrategyOptions
            {
                Enabled = chaosEnabled, InjectionRate = 0.02,
                FaultGenerator = static _ =>
                    new ValueTask<Exception?>(new HttpRequestException("chaos"))
            });
    });
```

Chaos strategies go **innermost** in the same pipeline as the resilience strategies (so the retry, breaker, and timeouts react to the injected faults), and are gated by configuration so they can be enabled per environment or for a fraction of traffic. The injected 3-second latency is longer than the 1-second attempt timeout, so this experiment verifies that timeouts, retries, and the breaker actually engage.

**Game days.** A scheduled exercise where a team deliberately causes a failure scenario — a dependency outage, a zone loss, a region isolation — with observers, a hypothesis, and a debrief. Game days test people and runbooks as much as software: Can on-call detect it? Is the runbook correct? Does anyone have the access they need?

**DR drills.** Actually fail over to the secondary region on a schedule (quarterly is common for Tier 0), measure the real RTO and RPO, and fail back. Many organizations run production in the "secondary" region for a period to prove it is genuinely equivalent. **An untested DR plan has an unknown RTO and an unknown probability of working** — which is the same as not having one, only with more paperwork.

**The interview-grade sentence:** *"Resilience mechanisms are hypotheses until tested. I inject faults in-process with Polly's chaos strategies, at the network layer with a proxy, and at the platform layer with Azure Chaos Studio; I run game days that test the runbook and the people; and I do real regional failovers on a schedule so the RTO is a measurement."*

---

## Concept 56 — Observability for resilience mechanisms

Resilience mechanisms **hide failures by design**. A retry that succeeds on the second attempt produces a success — and a dependency that now fails 30% of first attempts looks healthy on your error dashboard until the day the retries are exhausted. You must observe the mechanisms themselves.

**What to measure:**

| Signal | Why |
|---|---|
| **Retry rate** (retries ÷ requests, per dependency) | Rising retries are an early warning of dependency degradation; also your amplification factor |
| **Attempts per successful request** | Hidden failure rate |
| **Breaker state and transitions** | An open breaker on a critical dependency is an incident |
| **Bulkhead / limiter rejections** | Either overload or a mis-sized bulkhead |
| **Timeout counts** (attempt vs total) | Fail-slow dependencies |
| **Hedges fired and hedge win rate** | Hedging cost and benefit |
| **Load-shedding rate by priority** | Who is being refused |
| **Fallback / degraded-mode activations** | How often users get the reduced experience |
| **Replication lag** | Your live RPO (Concept 50) |
| **Failover and recovery durations** | Your measured RTO |
| **Queue age** | Goodput risk (Concept 10) |

**In .NET**, Polly v8 emits **metrics** (via the `Polly` meter — resilience events such as retries, breaker transitions, and timeouts, plus attempt and pipeline durations) and **logs** through `Microsoft.Extensions.Logging` when you use the DI integration; `Microsoft.Extensions.Http.Resilience` adds HTTP-specific enrichment. Add the meter to your OpenTelemetry configuration (`metrics.AddMeter("Polly")`) and the events flow to Azure Monitor / Application Insights or any OTLP backend (Module 11 introduced OpenTelemetry; Module 28 goes deep).

**Alerting principles:**

- **Alert on user-facing symptoms** (SLO burn rate) for paging; use mechanism metrics for diagnosis and early warning tickets.
- **Alert on sustained open breakers for critical dependencies** and on retry-rate anomalies.
- **Distinguish "degraded" from "down"** in dashboards and status pages (Concept 57).

**Tracing.** Each retry and hedge should appear as a separate child span with its attempt number, so a slow request can be explained ("two timeouts at 2 s, success on attempt 3").

**The interview-grade sentence:** *"Retries hide failures, so I monitor the mechanisms: retry ratio per dependency, breaker transitions, limiter rejections, timeouts, hedges, and fallback activations — through Polly's OpenTelemetry metrics — and I page on SLO burn, with the mechanism metrics as early warning."*

---

## Concept 57 — Failure mode analysis and the health model

**Failure Mode and Effects Analysis (FMEA)** is a structured way to find failure modes before production does. Azure's Well-Architected Framework makes it a core reliability practice (RE:03). The method:

1. **Identify the critical flows** (user and system flows, ranked by business impact — RE:02).
2. **Decompose each flow into components and dependencies**, including the hidden ones (Concept 53).
3. **For each dependency, list failure modes**: unavailable, slow, erroring intermittently, returning wrong data, throttling, returning stale data, partially failing (one partition, one region).
4. **For each failure mode, assess** likelihood, impact on the flow (full outage, degraded, none), detection (how would we know, and how fast?), and mitigation (existing and planned).
5. **Classify each dependency as critical or non-critical** for the flow (Concept 6), and define the degraded behaviour for non-critical ones.
6. **Record it as a living artefact** and revisit it when the architecture changes and after every incident.

A compact FMEA row looks like:

| Flow | Dependency | Failure mode | Impact | Detection | Mitigation |
|---|---|---|---|---|---|
| Checkout | Recommendations API | Slow (p99 > 2 s) | Page latency, thread exhaustion | Attempt timeout rate, p99 alert | 300 ms timeout, bulkhead 50, breaker, hide widget |
| Checkout | Payments provider | 5xx for 10 min | Orders cannot complete | Error rate, breaker open | Accept order as pending, authorize asynchronously with idempotency key, notify |
| Checkout | Azure SQL primary | Regional outage | Full outage of writes | Failover group health, SLO burn | Failover group, runbook, RTO 15 min tested quarterly |

**The health model.** A health model (RE:04 and RE:10 in Azure's framework, and a central idea in its mission-critical guidance) defines what **healthy**, **degraded**, and **unhealthy** mean for each component and flow, and how component states **roll up** into flow and application states. Its value:

- A failed non-critical dependency makes a flow **degraded**, not unhealthy, so the system does not page for the wrong reason and does not fail over for the wrong reason.
- It gives failover automation and humans a **shared definition of "down"**.
- It connects directly to SLOs: the flow's health is measured by its SLIs.

**The interview-grade sentence:** *"Before building I'd run a failure mode analysis per critical flow: each dependency, each way it can fail — down, slow, wrong, throttled, partial — with impact, detection, and mitigation. The output classifies dependencies as critical or degradable and feeds a health model that says what healthy, degraded, and unhealthy mean for each flow."*

---

# Part I — The .NET and Azure surface

## Concept 58 — Polly v8: the model

Polly is the .NET resilience library and a .NET Foundation project; **v8** (the current major version, at 8.7.x as of mid-2026) replaced the v7 "policy" API with a new core. Module 25 goes deep; here is the architecture-level model.

**Core concepts:**

| Concept | Meaning |
|---|---|
| **`ResiliencePipeline` / `ResiliencePipeline<T>`** | An immutable, thread-safe composition of strategies; execute callbacks through it |
| **`ResiliencePipelineBuilder`** | Adds strategies in order — the first added is the outermost |
| **Strategies** | Retry, circuit breaker, timeout, hedging, fallback, rate limiter (Polly.RateLimiting), and chaos strategies |
| **`ResilienceContext`** | Per-execution context with cancellation, properties, and operation key |
| **`ResiliencePipelineRegistry<TKey>`** | Named, cached pipelines; supports dynamic reload of options |
| **DI integration** (`Polly.Extensions`) | `services.AddResiliencePipeline("name", builder => ...)`, telemetry, options binding |
| **Packages** | `Polly.Core` (strategies), `Polly.Extensions` (DI, telemetry), `Polly.RateLimiting`, `Polly.Testing`; `Polly` retains the v7 API for compatibility |

**Key v8 design choices worth mentioning:**

- **Allocation-conscious, `ValueTask`-based execution** — pipelines are cheap to invoke on hot paths.
- **Strategies are built once and reused** — create pipelines at startup (or via the registry), not per call. Breaker state lives in the pipeline instance, so **a pipeline per call means a breaker that never trips.**
- **Unified options with validation** (data annotations).
- **Built-in telemetry** (Concept 56) and **chaos strategies** (Concept 55, integrated from Simmy in v8.3).
- **`CircuitBreakerStateProvider` and `CircuitBreakerManualControl`** to read breaker state (e.g., into a health check) and to isolate or close circuits operationally.
- **Dynamic reloads** of options via the registry and `IOptionsMonitor`, so timeouts and thresholds can be tuned without redeploying.

```csharp
builder.Services.AddResiliencePipeline<string, HttpResponseMessage>("pricing", (pipeline, context) =>
{
    pipeline
        .AddTimeout(TimeSpan.FromSeconds(3))                         // total budget
        .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
        {
            MaxRetryAttempts = 2,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromMilliseconds(200),
            MaxDelay = TimeSpan.FromSeconds(1),
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .Handle<TimeoutRejectedException>()
                .HandleResult(r => (int)r.StatusCode is 502 or 503 or 504)
        })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 20,
            BreakDuration = TimeSpan.FromSeconds(5),
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .Handle<TimeoutRejectedException>()
                .HandleResult(r => (int)r.StatusCode >= 500)
        })
        .AddTimeout(TimeSpan.FromMilliseconds(800));                  // per attempt
});

// Usage: resolve once, execute many times
var pipeline = provider.GetRequiredService<ResiliencePipelineProvider<string>>()
                       .GetPipeline<HttpResponseMessage>("pricing");
var response = await pipeline.ExecuteAsync(
    static async (state, ct) => await state.Client.GetAsync($"/prices/{state.Sku}", ct),
    (Client: httpClient, Sku: sku), cancellationToken);
```

Note the retry does not handle `BrokenCircuitException`, so an open breaker fails the call immediately rather than consuming retry attempts (Concept 29). The `static` lambda with explicit state avoids a closure allocation per call — a v8 idiom.

**The interview-grade sentence:** *"Polly v8 composes strategies into an immutable pipeline — first added is outermost — built once and resolved from the registry, because breaker state lives in the pipeline instance. It's ValueTask-based, emits OpenTelemetry metrics, supports dynamic option reloads, and has chaos strategies built in."*

---

## Concept 59 — `Microsoft.Extensions.Http.Resilience`: the standard and hedging handlers

`Microsoft.Extensions.Http.Resilience` (part of the `dotnet/extensions` repository, versioned alongside .NET — 10.x currently) builds on Polly v8 and integrates with `IHttpClientFactory`. It is what Aspire's service-defaults template enables for every `HttpClient` via `ConfigureHttpClientDefaults(http => http.AddStandardResilienceHandler())`.

**The standard resilience handler** chains five strategies, outermost to innermost, with these defaults:

| Order | Strategy | Default |
|---|---|---|
| 1 | Rate limiter (concurrency bulkhead) | 1,000 concurrent permits, no queue |
| 2 | Total request timeout | 30 seconds |
| 3 | Retry | 3 retries, exponential backoff with jitter, 2-second base delay; handles `HttpRequestException`, timeout rejections, 408, 429, and 5xx; honours `Retry-After` |
| 4 | Circuit breaker | 10% failure ratio, minimum throughput 100, 30-second sampling window, 5-second break |
| 5 | Attempt timeout | 10 seconds |

**Configure it deliberately** — the defaults are generic and fairly generous:

```csharp
builder.Services.AddHttpClient<PaymentsClient>(c => c.BaseAddress = new("https://payments.internal"))
    .AddStandardResilienceHandler(o =>
    {
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(4);
        o.AttemptTimeout.Timeout      = TimeSpan.FromSeconds(1);
        o.Retry.MaxRetryAttempts      = 1;
        o.Retry.Delay                 = TimeSpan.FromMilliseconds(200);
        o.Retry.DisableForUnsafeHttpMethods();       // no automatic retries for POST/PATCH/PUT/DELETE/CONNECT
        o.CircuitBreaker.FailureRatio      = 0.5;
        o.CircuitBreaker.MinimumThroughput = 20;
        o.CircuitBreaker.SamplingDuration  = TimeSpan.FromSeconds(10); // must be ≥ 2 × attempt timeout
    });
```

(If your POST endpoints accept idempotency keys, you can re-enable retries for them explicitly — the point is that it is a decision, not a default.)

**The standard hedging handler** (`AddStandardHedgingHandler()`) replaces retry with hedging and applies the rate limiter, circuit breaker, and attempt timeout **per endpoint**, selected by URL authority by default (`SelectPipelineByAuthority()`), so hedges go only to healthy endpoints. Combined with a routing strategy (ordered or weighted groups of endpoints), it can hedge a request to a second region or a second instance of a service. Hedging defaults are conservative, and — as with retries — **only hedge idempotent requests** (Concept 38).

**Overriding defaults set globally.** If `ConfigureHttpClientDefaults` adds the standard handler to every client (as Aspire does), a specific client that needs a different pipeline must remove the default rather than stack a second one on top. `RemoveAllResilienceHandlers()` exists for this (currently marked experimental, diagnostic `EXTEXP0001`); stacking two resilience handlers is the double-retry trap (Concept 21).

**Custom pipelines.** `AddResilienceHandler("name", pipeline => ...)` gives you full control using the same Polly builder, with HTTP-aware options (`HttpRetryStrategyOptions`, `HttpCircuitBreakerStrategyOptions`) whose `ShouldHandle` predicates already understand transient HTTP outcomes.

**The interview-grade sentence:** *"The standard handler is rate limiter, 30-second total timeout, three jittered retries, a 10%-over-30-seconds breaker, and a 10-second attempt timeout. I tune it per client — much tighter timeouts, one retry, and retries disabled for unsafe methods unless the endpoint takes idempotency keys — and I remove the global default rather than stacking a second handler."*

---

## Concept 60 — Resilience you already have, and the double-retry trap

Many .NET clients ship with retry logic. Before adding Polly anywhere, inventory what is already there.

| Client | Built-in behaviour | Notes |
|---|---|---|
| **Azure SDK clients** (`Azure.Core`: Storage, Key Vault, Service Bus admin, etc.) | Retries with exponential backoff via `ClientOptions.Retry` (`MaxRetries`, `Delay`, `MaxDelay`, `Mode`, `NetworkTimeout`); honours `Retry-After` | Tune here instead of wrapping with Polly; defaults are a few retries with sub-second base delay |
| **Azure Service Bus / Event Hubs clients** | `ServiceBusRetryOptions` / `EventHubsRetryOptions` (`MaxRetries`, `Delay`, `MaxDelay`, `TryTimeout`, `Mode`) | Throttling (`ServiceBusy`) handled with backoff; the processor also redelivers per `MaxDeliveryCount` (Module 11) |
| **Cosmos DB .NET SDK v3** | Automatic retries on 429 (default up to 9 attempts / 30 s cumulative), on transient network errors, and cross-region retries per `ApplicationPreferredRegions`; optional **cross-region hedging availability strategy** and a **partition-level circuit breaker** | Do not wrap Cosmos calls in retry policies that retry 429s again; tune `CosmosClientOptions` |
| **EF Core** (SQL Server, Azure SQL) | `EnableRetryOnFailure()` execution strategy — default 6 retries, up to 30 s delay, curated transient error list | With a user-initiated transaction, wrap the whole unit in `strategy.ExecuteAsync(...)` (Module 12, Concept 60) |
| **SqlClient** | Configurable retry logic (`SqlRetryLogicBaseProvider`) for connections and commands, opt-in; connection resiliency (`ConnectRetryCount`) for idle connection recovery | Don't enable SqlClient retries under EF Core's execution strategy — pick one |
| **Azure Managed Redis / StackExchange.Redis** | Reconnect logic, `ConnectRetry`, backoff policies | Command-level retry is your decision (and must consider idempotency) |
| **gRPC for .NET** | Configurable client retry policy (`RetryPolicy` in channel options) | Honours server pushback |
| **`IHttpClientFactory` + standard handler** | Concept 59 | Often enabled globally by Aspire service defaults |

**How the trap assembles itself:** a team adds Aspire's service defaults (standard handler on every `HttpClient`), uses the Azure SDK (its own retries — and if they pass an `HttpClient` from the factory into the SDK transport, the standard handler too), wraps the SDK call in a Polly retry "to be safe," and the upstream caller also retries. The worst-case multiplier is now the product of all of them.

**Rules:**

1. **One retry owner per dependency** (Concept 21). Prefer the SDK's native retry when it understands the service's semantics.
2. **Outer layers add only timeouts, bulkheads, and possibly a breaker** — not more retries.
3. **Document the owner** in the code where the client is registered.
4. **Test the amplification**: inject a persistent failure and count attempts at the dependency (Exercise 2).

**The interview-grade sentence:** *"Before adding Polly I inventory the retries that already exist — Azure SDK retry options, the Cosmos SDK's 429 and cross-region retries, EF Core's execution strategy, Aspire's standard handler — pick one owner per dependency, and let outer layers add only timeouts and limits."*

---

## Concept 61 — Server-side resilience in ASP.NET Core

Most of this module has been about being a good *client*. A resilient service also protects *itself*.

**Rate limiting and load shedding middleware** (`Microsoft.AspNetCore.RateLimiting`, built on `System.Threading.RateLimiting`):

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Per-client quota (rate limiting): token bucket per API key
    options.AddPolicy("per-client", ctx =>
        RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: ctx.Request.Headers["X-Api-Key"].ToString(),
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100, TokensPerPeriod = 50,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                QueueLimit = 0, AutoReplenishment = true
            }));

    // Server-wide protection (load shedding): cap in-flight work, newest first
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(_ =>
        RateLimitPartition.GetConcurrencyLimiter("global", _ => new ConcurrencyLimiterOptions
        {
            PermitLimit = 500,
            QueueLimit = 50,
            QueueProcessingOrder = QueueProcessingOrder.NewestFirst
        }));

    options.OnRejected = (context, _) =>
    {
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();
        return ValueTask.CompletedTask;
    };
});

app.UseRateLimiter();
app.MapPost("/orders", CreateOrder).RequireRateLimiting("per-client");
```

Global concurrency rejection is load shedding, so a 503 is arguably the more accurate status than 429 for it; many teams set the status per policy. For priority-aware shedding, partition by a criticality header or route and give critical partitions separate, larger limits.

**Request timeouts.** `builder.Services.AddRequestTimeouts()` with `app.UseRequestTimeouts()` and `[RequestTimeout]` / `.WithRequestTimeout(...)` bound server-side work and trigger `HttpContext.RequestAborted` — pass that token everywhere (Concept 12).

**Kestrel limits** (`KestrelServerOptions.Limits`): `MaxConcurrentConnections`, `MaxConcurrentUpgradedConnections`, `MaxRequestBodySize`, `MinRequestBodyDataRate` / `MinResponseDataRate` (defends against slow-loris clients), `KeepAliveTimeout`, and `RequestHeadersTimeout`. These are the outermost bulkhead of the process.

**Health checks** — Concept 41.

**Graceful shutdown and draining:**

- On `SIGTERM`, the host stops accepting new requests and lets in-flight requests finish within `HostOptions.ShutdownTimeout` (30 seconds by default in current .NET).
- Make **readiness fail first** (so the load balancer stops sending new traffic), wait for the load balancer's deregistration delay, then stop. In Kubernetes, a short `preStop` delay achieves this ordering because endpoint removal and `SIGTERM` happen concurrently.
- `BackgroundService` implementations must observe the stopping token and checkpoint work; message processors should stop receiving and complete or abandon in-flight messages cleanly (Module 11).

**Thread-pool health.** Sync-over-async in request paths causes ThreadPool starvation that looks exactly like a slow dependency (Concept 9); Module 15 covers diagnosis (`dotnet-counters` ThreadPool queue length) and cures.

**The interview-grade sentence:** *"Server side, I combine per-client token-bucket limits with a global concurrency limiter that queues newest-first and returns Retry-After, request timeouts that flow into RequestAborted, Kestrel limits against slow clients, and a shutdown sequence that fails readiness first so the load balancer drains us before we stop."*

---

## Concept 62 — The Azure reliability surface, service by service

This concept is a map, not a feature list; for each service, know its **in-region HA mechanism**, its **cross-region DR mechanism**, and **what failover looks like to your code**. Verify SLAs and feature availability per region in the current documentation — they change.

| Service | In-region HA | Cross-region | What your code sees |
|---|---|---|---|
| **App Service / Container Apps / AKS** | Multiple instances; zone redundancy (zone-redundant plans/environments; AKS node pools across zones with topology spread and PodDisruptionBudgets) | Deploy per region behind Front Door / Traffic Manager | Nothing if stateless; session state must be external |
| **Azure Functions** | Zone-redundant hosting plans (e.g., Flex Consumption, Premium in supported regions) | Per-region deployment; trigger sources must be replicated too | Duplicate triggers after failover → idempotency |
| **Azure SQL Database** | Built-in replicas; zone redundancy; Business Critical uses local replicas | Active geo-replication; **failover groups** with read-write and read-only listeners | Transient errors during failover (retry via execution strategy); connection redirection |
| **Azure Cosmos DB** | Replicas within the region; zone redundancy | Multi-region replication; service-managed failover; **per-partition automatic failover** (GA June 2026); multi-region writes | SDK handles region routing via `ApplicationPreferredRegions`; optional hedging availability strategy; conflicts with multi-write |
| **Azure Storage** | LRS / ZRS | GRS / GZRS / RA-GRS / RA-GZRS; customer-managed failover | Read from secondary endpoint (RA-*) when primary is unavailable; async RPO |
| **Service Bus** | Availability zones; Premium messaging units | **Geo-replication** (Premium, GA December 2025, metadata + data) or Geo-DR (metadata only) | Same namespace hostname after geo-replication promotion; re-resolve DNS |
| **Event Hubs** | Availability zones | Geo-replication (higher tiers) or Geo-DR (metadata) | Offsets and checkpoints must be considered after failover |
| **Azure Managed Redis** | Zone redundancy with high availability | Active geo-replication | Treat the cache as rebuildable; plan for cold-cache load (Module 10) |
| **Key Vault** | Zone-redundant within region | Microsoft-managed replication to a paired region for many regions (read-only during failover) | Cache secrets; tolerate read-only periods |
| **Azure Front Door** | Global anycast edge, multiple POPs | Global by design | Origin health probes; the edge itself is a shared dependency (Concept 52) |
| **Traffic Manager** | Global DNS service | Global by design | DNS TTL governs failover speed |
| **Azure Load Balancer / Application Gateway** | Zone-redundant frontends | Regional; cross-region Load Balancer tier exists for L4 global | Health probes decide backend membership |
| **Container Registry** | Zone redundancy (Premium) | Geo-replication (Premium) | Image pulls succeed in the DR region |

**Composite availability in practice.** Azure publishes per-service SLAs (for example, VMs deployed across two or more zones carry a higher SLA than a single VM, and multi-region Cosmos DB accounts carry higher SLAs than single-region ones). Use them for composite calculations (Concept 5), remembering that SLAs are commitments with credits, not predictions.

**Azure Service Health and Resource Health** tell you about platform incidents and per-resource health; route Service Health alerts to on-call, because "is it us or Azure?" is the first question in every incident.

**The interview-grade sentence:** *"For each Azure service I know three things: how it survives a zone failure, how it replicates across regions, and what my code sees during failover — for example SQL failover groups give listener endpoints and transient errors, Cosmos routes by preferred regions and can now fail over per partition, and Service Bus geo-replication keeps the namespace hostname."*

---

## Concept 63 — The Azure Well-Architected Reliability pillar and mission-critical guidance

Azure's Well-Architected Framework organizes reliability into **design principles** (design for business requirements, for resilience, for recovery, for operations, and keep it simple) and a **design review checklist** of ten recommendations. Using its vocabulary in an Azure-focused architecture interview signals fluency:

| Code | Recommendation (paraphrased) | Where in this module |
|---|---|---|
| **RE:01** | Design for simplicity and efficiency; avoid unnecessary complexity | Concept 66 |
| **RE:02** | Identify and rate user and system flows by criticality | Concepts 6, 57 |
| **RE:03** | Perform failure mode analysis on dependencies and flows | Concept 57 |
| **RE:04** | Define reliability and recovery targets; build a health model | Concepts 5, 49, 57 |
| **RE:05** | Add redundancy at different levels according to targets | Concepts 43–48 |
| **RE:06** | Implement timely, reliable scaling | Module 6; Concept 43 |
| **RE:07** | Build self-preservation and self-healing; handle transient faults with design patterns | Parts B–F |
| **RE:08** | Test resiliency and availability, including chaos and fault injection | Concept 55 |
| **RE:09** | Implement BCDR plans, tested regularly | Concepts 49–53 |
| **RE:10** | Monitor with a health model; alert on what matters | Concepts 56, 57 |

**Mission-critical guidance.** The Azure Well-Architected **mission-critical** workload guidance (and its reference implementations) describes an opinionated architecture for very high availability: **active-active multi-region, stateless scale units (deployment stamps) that are deployed and replaced as a whole**, a global routing layer, globally distributed data (Cosmos DB with multi-region replication), a layered **health model** that drives routing decisions, zero-downtime blue-green deployments of entire stamps, and continuous validation with load and chaos testing. Its value for interviews is as a coherent example of everything in Parts E–H assembled into one design — and as a reference for how much operational investment "mission-critical" actually demands.

**Azure design patterns to name** (from the Architecture Center catalogue): Retry, Circuit Breaker, Bulkhead, Throttling, Rate Limiting, Queue-Based Load Leveling, Priority Queue, Health Endpoint Monitoring, Compensating Transaction, Leader Election, Deployment Stamps, Geode, and Competing Consumers.

**The interview-grade sentence:** *"In Azure terms I'd walk the reliability checklist: rate the flows, do failure mode analysis, set targets and a health model, add redundancy to match, handle transient faults with the standard patterns, test with chaos, rehearse BCDR, and monitor through the health model. The mission-critical reference architecture is what that looks like fully assembled — and how much it costs."*

---

# Part J — Judgment

## Concept 64 — Anti-patterns worth naming on sight

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Retries at every layer** | Multiplicative amplification (Concept 18) | One retry owner per call path; "don't retry" signals |
| **Retries without jitter** | Synchronized waves | Full or decorrelated jitter |
| **Retry until success / infinite retries** | Turns an outage into a permanent overload; holds message locks | Bounded attempts, budgets, dead-letter |
| **Retrying non-idempotent operations** | Duplicate charges, orders, emails | Idempotency keys; disable unsafe-method retries |
| **Retrying permanent errors** | Wasted load and delayed alerts | Classify first |
| **No timeouts / `HttpClient`'s 100 s default** | Slow dependency exhausts the caller | Per-attempt and total timeouts from measured percentiles |
| **Downstream timeouts longer than upstream ones** | Abandoned work (metastable sustaining effect) | Deadline propagation |
| **Ignoring `Retry-After`** | Throttling gets worse | Honour it; add jitter on top |
| **Breaker counting 4xx or caller cancellations** | Self-inflicted outages | Count dependency failures and timeouts only |
| **One breaker over a sharded dependency** | Partial failure becomes total | Scope breakers to the failing unit; outlier ejection |
| **Creating a Polly pipeline per call** | Breaker state is lost; allocation churn | Build once; registry |
| **Stacking resilience handlers / SDK + Polly retries** | Double retries | Inventory and choose one owner |
| **Unbounded queues** | Latency and memory bombs; zero goodput | Bounded queues with age limits; shed load |
| **Liveness probes that check dependencies** | Fleet-wide restarts on a dependency blip | Liveness checks the process only |
| **Health checks that never fail open** | All capacity removed on a false positive | Fail open; cap removal rate |
| **Fallback to an untested path** | Fails exactly when needed | Degrade by doing less; exercise fallbacks continuously |
| **Failover that requires provisioning** | Control plane may be unavailable | Static stability; pre-provisioned capacity |
| **Active-active without failover headroom** | Cascading regional failure | Size each region for the failed region's load |
| **Multi-leader writes with default LWW on money** | Silent data loss | Homing, single writer, or domain-specific merge |
| **"Two regions, therefore four nines"** | Ignores correlation and shared dependencies | Model shared failure domains; find hidden dependencies |
| **Global config push without staging** | Global outage in seconds | Stage config like code; last-known-good |
| **DR plan never tested** | Unknown RTO | Scheduled drills; measured RTO |
| **Singleton `HttpClient` without `PooledConnectionLifetime`** | DNS failover never takes effect | Set the lifetime or use `IHttpClientFactory` |
| **Mechanisms without telemetry** | Retries hide degradation until they don't | Monitor retry ratio, breaker state, rejections |
| **Hedging everything** | Doubles load under stress; duplicates writes | Hedge idempotent, latency-critical reads with a budget |

---

## Concept 65 — The reliability review: a checklist you can narrate

In a design interview, after the high-level design, walk this list. It is ordered so that it tells a story from a single call up to the whole system.

**Per dependency (for each box your service calls):**

1. **Criticality:** hard, soft, or asynchronous? What is the degraded behaviour?
2. **Timeouts:** connect, per-attempt (from measured p99/p99.9), total (from the caller's budget); deadline propagated?
3. **Retries:** which errors, how many, backoff and jitter, which layer owns them, budget, idempotency guaranteed?
4. **Breaker / outlier handling:** scope, thresholds, what counts, what happens when open?
5. **Bulkhead:** concurrency limit and pool, sized from peak RPS × p99?
6. **Throttling contract:** honour `Retry-After`; do we have quota headroom?

**Per service:**

7. **Overload protection:** per-client rate limits, global concurrency limit, priority-based shedding, bounded queues.
8. **Health:** liveness (process only), readiness (local), startup; fail-open routing.
9. **Lifecycle:** graceful shutdown and draining; idempotent startup.
10. **Capacity:** headroom to lose one zone without scaling; tested cold-start capacity.

**Per system:**

11. **Failure domains:** zone redundancy; cells or stamps if the blast radius needs bounding; shuffle sharding for noisy tenants.
12. **Multi-region (if justified):** active-active or active-passive per tier; data replication mode (RPO); conflict model; failover decision owner; hidden shared dependencies; global ingress risk.
13. **DR:** tiers, RPO/RTO, backups for corruption, measured restore and failover times.
14. **Change safety:** progressive rollout, bake time, automated rollback, flags, config staging.
15. **Verification:** chaos experiments, load tests past capacity, game days, DR drills.
16. **Observability:** SLOs and burn-rate alerts; mechanism metrics; health model.

You will not have time to cover all sixteen in a 45-minute round. **Pick the three that matter most for the system in front of you and say why** — that prioritization is itself the senior signal.

---

## Concept 66 — When not to

Reliability engineering has a cost curve that bends sharply upward. Knowing where to stop is as important as knowing the patterns.

**Don't go multi-region active-active when:**

- The SLO is 99.9% or 99.95% — zone redundancy in one region plus a tested backup-and-restore or pilot-light DR usually meets it at a fraction of the cost.
- The data model is transactional and conflict-intolerant, and the team is small. The operational burden will cause more outages than it prevents.
- The real requirement is an RTO of 15–60 minutes, which a rehearsed warm standby delivers.

**Don't build alternate global ingress** unless a multi-hour edge outage is a business-critical event; Microsoft's own guidance says most workloads don't need it (Concept 52).

**Don't put a circuit breaker on every call.** In-process calls, calls to a highly available managed service whose SDK already handles failover, and low-volume calls where a breaker cannot gather statistics gain little from one — a timeout and a bulkhead usually suffice.

**Don't add in-process retries in a queue consumer** that already has broker redelivery with backoff, beyond a very small immediate retry for blips (Module 11, Concept 13).

**Don't hedge** operations that are not latency-critical, not idempotent, or run against a single shared bottleneck.

**Don't build cells** for a product with a handful of tenants or with heavy cross-tenant interactions.

**Don't add fallbacks** you won't test.

**Don't chase nines your dependencies can't support** (Concept 6) — change the dependencies or the target.

**The meta-rule:** *every reliability mechanism is also a new failure mode* — a new mode, a new config, a new code path, a new dependency. Azure's RE:01 ("design for simplicity") is not a platitude; it is the recognition that complexity is itself a leading cause of unreliability. The best reliability designs are usually **the simplest design that meets the stated targets, with the targets stated explicitly.**

**The interview-grade sentence:** *"Before adding a mechanism, I ask which failure it addresses, how often that failure happens, and what new failure mode the mechanism introduces. For a 99.95% target I'd do zone redundancy, disciplined timeouts and retries, and a rehearsed warm-standby DR — not active-active in three regions."*

---

# Putting it together

## Worked example 1 — "Checkout calls Payments, Inventory, Pricing, and Recommendations. Design the resilience layer."

Narrate it in this order:

1. **Classify the dependencies first** (Concept 6). Pricing and Inventory are hard for placing an order; Payments is hard for *completing* it but can be made asynchronous; Recommendations is soft. Say out loud that this classification is agreed with product, because it defines what "degraded" means.
2. **Budget the request.** The checkout API has a 1.5 s p99 SLO. Pricing and Inventory run in parallel with a shared 600 ms window; payment authorization gets up to 2 s but only on the "place order" call; recommendations get 250 ms and are rendered separately (Concept 13).
3. **Timeouts from data.** Pricing p99.9 is 180 ms → attempt timeout 250 ms, total 600 ms. Inventory p99.9 is 220 ms → attempt 300 ms, total 600 ms. Recommendations → attempt 200 ms, total 250 ms, no retry (Concepts 11–12).
4. **Retries.** Pricing and Inventory reads: one retry with jittered exponential backoff (base 50 ms), only on transient errors, only if the remaining budget fits another attempt. The upstream BFF does **not** retry — this service is the retry owner (Concepts 15–18). A retry budget caps retries at ~10% of traffic (Concept 19).
5. **Payments is a write.** No blind retries. Client-generated idempotency key per order, stored with the order; the payment provider's own idempotency key derived from it; after a timeout, query the payment status rather than re-sending (Concepts 14, 20). If the provider is failing, **accept the order as `PendingPayment`** and authorize asynchronously via the outbox (Module 11) — the payment outage becomes a backlog, not lost revenue.
6. **Breakers.** Recommendations: breaker with 50% failure ratio, short break, hide the widget when open. Pricing: breaker scoped per pricing *region endpoint*, serving last-known prices from cache with a timestamp when open, if the business accepts it. Inventory: breaker per warehouse partition, so one warehouse's database failing doesn't block others (Concepts 24–27).
7. **Bulkheads.** Separate `HttpClient`s and concurrency limits per dependency: Recommendations capped at 50 concurrent, so a slow recommendation service can never take threads or sockets from checkout (Concepts 30–31).
8. **Self-protection.** Global concurrency limiter with newest-first queueing, per-client token buckets, and priority: "place order" requests are never shed before "view cart" (Concepts 35–36, 61).
9. **Verification.** Polly chaos strategies in staging inject 3 s latency into Recommendations and 503s into Pricing; the test asserts checkout p99 stays under SLO and that Pricing's attempt count does not exceed the budget (Concept 55).
10. **Observability.** Retry ratio per dependency, breaker transitions, limiter rejections, degraded-mode activations — alert on sustained open Pricing/Inventory breakers (Concept 56).

## Worked example 2 — "We need 99.99% for our SaaS on Azure. Active-active or active-passive?"

1. **Challenge the number first.** 99.99% is 4.3 minutes a month. Ask which *flows* need it — probably login, core read paths, and order placement — not the admin portal or reporting. Tier the workloads (Concept 49).
2. **Do the arithmetic.** Critical path: Front Door → Container Apps → Azure SQL → Service Bus. Each must be ≥ 99.99% or redundant; the global edge alone caps the system at its own SLA (Concepts 5–6, 52).
3. **Zone redundancy is the baseline** for everything: zone-redundant Container Apps environment, Azure SQL with zone redundancy, Service Bus Premium in a zone-enabled region, headroom for losing one of three zones (1.5× capacity) (Concepts 43, 48).
4. **Is one region enough?** 99.99% with human-driven regional recovery is not achievable if regional incidents happen at all — a single multi-hour regional event exhausts years of budget. So a second region is justified for the Tier 0 flows.
5. **Choose the data model before the topology.** Orders and billing are transactional and conflict-intolerant → single write region. Profile and preferences data can tolerate LWW. So: **active-active compute, active-passive data** (Concepts 45–46).
6. **Concrete design.** Front Door routes to both regions; reads are served locally from geo-replicas (with read-your-writes handled by routing post-write reads to the primary for a few seconds); writes go to the primary region's Azure SQL failover-group listener; Service Bus geo-replication with promotion; Cosmos DB for session/profile data with multi-region replication and per-partition automatic failover (Concepts 50, 62).
7. **Failover.** Traffic shift is automatic (Front Door health probes). Database failover is customer-managed and decided by the incident commander against written criteria, with a target RTO of 10 minutes and an RPO equal to measured geo-replication lag (seconds). Fencing is handled by the failover group; `PooledConnectionLifetime` is set so clients re-resolve listeners (Concepts 12, 47).
8. **Static stability.** The secondary region runs at full capacity for the stateless tier (it serves traffic anyway) — no scale-up needed during failover (Concept 40).
9. **Hidden dependencies.** Geo-replicated container registry, Key Vault access from both regions with local secret caching, App Configuration replicas, monitoring in a third region, certificates tracked centrally (Concept 53).
10. **Prove it.** Quarterly regional failover drill with measured RTO; monthly Chaos Studio zone-down experiments; load test at 2× peak on a single region (Concept 55).
11. **Say what you are not doing and why.** No multi-region writes for orders (conflict risk), no alternate global ingress (cost vs likelihood — revisit if contractual penalties justify it), no third region (not needed for the target) (Concept 66).

## Worked example 3 — "A dependency slowed down and the whole platform went down for three hours. Walk me through what happened and what you'd change."

This is a metastable-failure question in disguise (Concepts 9–10).

1. **Reconstruct the timeline as a feedback loop.** The profile service's database hit a slow query plan; profile p99 went from 40 ms to 4 s. Callers had `HttpClient`'s default 100 s timeout, so in-flight requests piled up (Little's Law), connection pools and threads were exhausted, and the API gateway started timing out.
2. **Name the amplifiers.** The gateway retried 3×; the BFF retried 3×; the profile client retried 3× → up to 27 attempts per user request against the struggling database (Concept 18). No jitter, so retries arrived in waves (Concept 17).
3. **Name the health-check spiral.** Liveness probes checked the database; Kubernetes restarted API pods fleet-wide; restarts cold-started JIT and caches, adding load (Concept 41).
4. **Explain why fixing the trigger didn't help.** The query plan was fixed after 20 minutes, but the retry volume and cold caches kept the database saturated — the system was in a metastable state. Recovery came only when the team shed 70% of traffic at the edge and restarted services in waves (Concept 10).
5. **The changes, in priority order:**
   - Attempt timeouts from measured percentiles (profile: 150 ms) and total timeouts inside the user budget (Concepts 11–12).
   - One retry owner (the profile client), jittered, with a retry budget; gateway and BFF stop retrying; profile returns a "don't retry" signal when exhausted (Concepts 17–19).
   - A bulkhead around profile calls and a breaker counting timeouts; profile made a **soft** dependency with a cached/anonymous fallback for most pages (Concepts 6, 27, 31).
   - Liveness checks the process only (Concept 41).
   - Global concurrency limiter with newest-first queueing and deadline checks at dequeue (Concepts 13, 36).
   - A runbook entry: "under saturation, shed first, disable retries via flag, restart in waves" (Concept 10).
   - A load test that removes the profile cache and makes the database slow, run before each major release (Concept 55).
6. **Close with the principle:** the root cause was a slow query; the *outage* was caused by the resilience configuration.

---

## Common questions and what a strong answer contains

**"How do you make a service resilient?"** Refuse the pattern list. Start with dependency classification and budgets (Concepts 6, 13), then timeouts → retries → breakers → bulkheads per dependency, then self-protection, then redundancy — and verification.

**"How do you choose a timeout?"** Pick an acceptable false-timeout rate, read the corresponding percentile of the dependency's latency as seen by the caller, add a margin, and check it fits the caller's budget (Concept 11). Mention connect vs attempt vs total and `HttpClient`'s 100 s default (Concept 12).

**"Explain exponential backoff and jitter."** Capped exponential spreads retries in time; jitter desynchronizes clients; full or decorrelated jitter per the AWS simulation; apply jitter to all synchronized work (Concepts 16–17).

**"When shouldn't you retry?"** Permanent errors, non-idempotent operations after ambiguous failures, when the remaining deadline can't fit another attempt, when a lower layer already retried, and when the budget is exhausted (Concepts 14–20).

**"What is retry amplification?"** attempts^depth; the numbers; single-layer retries; "don't retry" signals; budgets (Concept 18).

**"How does a circuit breaker work?"** Three states; failure ratio over a window with minimum throughput; what counts; half-open probing; what happens when open — and then the critique (Concepts 23–28).

**"Circuit breaker or retry?"** Both, for different failures — and in a specific order (Concept 29). Then note retry budgets as the alternative to breakers for multi-instance dependencies (Concept 28).

**"What is a bulkhead?"** Partitioned resources; semaphore-based in async .NET; sized from peak RPS × p99; architectural bulkheads up to cells (Concepts 30–33).

**"How do you handle a noisy neighbour?"** Per-tenant rate limits, per-tenant queues or sessions, partitioned bulkheads, shuffle sharding, cells for the biggest tenants (Concepts 32–34).

**"What happens when your service is overloaded?"** Goodput vs throughput; shed early, cheaply, by priority; bounded LIFO queues; 503 with Retry-After; adaptive concurrency (Concepts 10, 35–37).

**"Rate limiting vs load shedding?"** Per-client quota vs per-server capacity; rate limiting rejects when idle, shedding never does (Concept 37).

**"What are hedged requests?"** The tail-at-scale arithmetic, hedging after ~p95, idempotency, budget, Cosmos cross-region hedging and Polly hedging (Concept 38).

**"Liveness vs readiness?"** Process vs traffic; never check dependencies in liveness; fail open (Concept 41).

**"Active-active vs active-passive?"** Decide from RTO/RPO, conflict tolerance, cost, and maturity; the data layer is the hard part; hybrids are normal (Concepts 44–46). This is the highest-leverage question in the module.

**"How would you design multi-region on Azure?"** Zone redundancy first; then the data replication mode as the RPO; failover groups / Cosmos / Service Bus geo-replication; Front Door as a shared dependency; hidden dependencies; drills (Concepts 48–53, 62).

**"What's the difference between HA and DR?"** Automatic handling of expected component failures vs a planned response to disasters, with RPO/RTO; replication is not backup (Concepts 1, 49).

**"What's the difference between RPO and RTO?"** Data loss window vs downtime window; the DR ladder; tiering; measured, not assumed (Concept 49).

**"How do you know your failover works?"** Because you do it on a schedule and measure it (Concept 55).

**"Tell me about a recent large cloud outage and what you learned."** Pick one from Concept 8 and extract the pattern: fast global propagation of change, unvalidated input, missing flags, metastable recovery, control-plane dependency.

**"How do you do this in .NET?"** Polly v8 pipelines built once; `AddStandardResilienceHandler` tuned per client; one retry owner given Azure SDK / EF Core / Cosmos built-ins; ASP.NET Core rate limiting, request timeouts, health checks; OpenTelemetry for Polly (Concepts 58–61).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Lists patterns (retry, breaker, bulkhead) as a checklist | Starts from dependency criticality and latency budgets, then applies patterns per dependency |
| "The service was down" | Distinguishes availability, durability, resilience, correctness; says *which* requests failed and how |
| Treats slow as fine and dead as bad | Treats fail-slow as the dangerous case; timeouts count as breaker failures |
| Trusts health checks | Names gray failure and measures health from the caller's side |
| "Two regions, so four nines" | Does series/parallel arithmetic *and* names the correlated failure domains |
| Ignores dependency availability | Applies the extra-nine rule and makes dependencies degradable |
| "We retry three times" | Classifies errors, jitters, budgets, picks one layer, requires idempotency |
| Default `HttpClient` timeout | Sets connect/attempt/total timeouts from percentiles; propagates deadlines |
| Retries POST blindly | Idempotency keys, stored atomically; `DisableForUnsafeHttpMethods` otherwise |
| Wraps Azure SDK calls in Polly retries | Knows the SDK already retries; one owner per dependency |
| Breaker per whole dependency | Scopes breakers to the failure unit; knows the partial-failure critique |
| Breaker counts 404s and cancellations | Counts dependency failures and timeouts only; 429 slows down instead |
| Arbitrary composition order | Rate limit → total timeout → retry → breaker → attempt timeout, with reasons |
| Thread-pool bulkheads in async .NET | Concurrency limiters, per-client pools, sized from RPS × p99 |
| "We'll autoscale if a zone fails" | Pre-provisions Z/(Z−1) capacity; static stability |
| Unbounded queues "for resilience" | Bounded queues, deadline checks at dequeue, LIFO/CoDel under overload |
| Liveness probe checks the database | Liveness is process-only; readiness is local; routing fails open |
| Fallback to another system | Degrades by doing less; exercises any fallback continuously |
| "Active-active is better" | Decides from RTO/RPO, conflict tolerance, cost, and maturity; proposes a hybrid |
| Hand-waves failover | Detect, decide, fence, promote, redirect, warm up, fail back — each with a number |
| Ignores DNS caching and pooled connections | Sets `PooledConnectionLifetime`; uses listener endpoints |
| Failover that provisions resources | Keeps failover a data-plane routing change |
| Forgets DNS, identity, Key Vault, registry, monitoring | Hunts hidden shared dependencies with traces and region-isolation game days |
| Treats config pushes as low-risk | Stages config like code; last-known-good; flags off by default |
| "We have a DR plan" | Quotes the RTO measured in the last drill |
| No telemetry on resilience mechanisms | Tracks retry ratio, breaker state, rejections, hedges, fallbacks |
| Unaware of recent incidents | Can explain AWS Oct 2025, Azure Front Door Oct 2025, Cloudflare Nov 2025, Google Cloud Jun 2025 as patterns |
| Adds every mechanism everywhere | States which failure each mechanism addresses and the new failure mode it adds |

---

## Practice exercises

**Exercise 1 — Measure retry amplification.** Build three tiny ASP.NET Core services in a chain (A → B → C) with the standard resilience handler on every `HttpClient`. Make C return 503 for all requests. Send 100 requests to A and count requests arriving at C. Then fix it: one retry owner, `x-should-retry: false` propagation, a retry budget. Count again. **This is the highest-value exercise in the module** — it converts Concept 18 from arithmetic into a number you measured.

**Exercise 2 — Find the double retry.** Register an Azure Blob Storage client (or use Azurite locally) and wrap an operation in a Polly retry. Use a fault-injecting proxy (Toxiproxy) to reset connections. Count attempts at the proxy. Then remove the outer retry, tune `BlobClientOptions.Retry`, and count again. Write down which layer owns retries and why.

**Exercise 3 — Jitter simulation.** Write a small C# simulation: 1,000 clients fail at t=0 and retry against a server with capacity 100 requests per 100 ms. Compare no backoff, exponential without jitter, full jitter, and decorrelated jitter on total attempts and completion time. Chart the arrival rate over time. Compare with the AWS blog's results.

**Exercise 4 — Break the breaker.** Configure a circuit breaker over a dependency with 10 partitions, then make one partition fail 100%. Observe the breaker opening and rejecting the 90% healthy traffic. Rescope it per partition and repeat. Then implement a token-bucket retry budget instead and compare behaviour at 10%, 50%, and 100% failure rates.

**Exercise 5 — Bulkhead under a slow dependency.** A service calls two dependencies; make one respond in 10 s. Load test without a bulkhead and watch latency on the healthy path rise (and ThreadPool / connection metrics with `dotnet-counters`). Add `AddConcurrencyLimiter` per dependency and a 300 ms attempt timeout; re-measure. Record the healthy path's p99 before and after.

**Exercise 6 — Goodput collapse.** Build an endpoint with a fixed processing capacity and an unbounded FIFO queue. Drive load past capacity with clients that time out at 1 s, and plot throughput vs goodput. Then add a bounded queue, a deadline check at dequeue, and `QueueProcessingOrder.NewestFirst`. Plot again. This makes Concepts 10 and 36 visible.

**Exercise 7 — The liveness-probe spiral.** In a local Kubernetes cluster (kind or k3d), deploy a service whose liveness probe checks PostgreSQL. Pause PostgreSQL for 45 seconds and watch every pod restart. Fix the probes (Concept 41) and repeat.

**Exercise 8 — DNS failover and pooled connections.** Point a singleton `HttpClient` at a hostname you control, switch the DNS record mid-test, and observe that the client keeps using the old IP. Set `PooledConnectionLifetime` (or use `IHttpClientFactory`) and repeat. This is a two-hour exercise that prevents a very real failover bug.

**Exercise 9 — A chaos experiment with a hypothesis.** Write a one-page experiment: steady-state metric, hypothesis, fault (Polly chaos latency on a soft dependency, or Azure Chaos Studio zone-down on a scale set), blast radius, abort conditions. Run it in a test environment, record results, and write the debrief.

**Exercise 10 — The architect write-up (one page).** For a system you know: list each critical flow and its dependencies classified as hard/soft/async; each dependency's timeout, retry owner, breaker, and bulkhead; the failure domains the system survives and the ones it doesn't; the RPO/RTO per tier and when they were last measured; the hidden shared dependencies; and the three changes that would most improve reliability per unit of cost. This is very close to a real architect take-home.

---
## Free resources

### Papers and primary sources

| Resource | What it covers | Why read it |
|---|---|---|
| [Metastable Failures in Distributed Systems](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf) — Bronson, Aghayev, Charapko & Zhu, HotOS 2021 | Triggers, sustaining effects, vulnerable states | **The theory behind Concept 10.** Short and very readable |
| [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang) — Huang et al., OSDI 2022 | Empirical study of metastable incidents across major providers | Evidence that retries and load are the dominant sustaining effects |
| [Gray Failure: The Achilles' Heel of Cloud-Scale Systems](https://www.microsoft.com/en-us/research/publication/gray-failure-achilles-heel-cloud-scale-systems/) — Huang et al. (Microsoft Research), HotOS 2017 | Differential observability | The source for Concept 4 |
| [The Tail at Scale](https://research.google/pubs/pub40801/) — Dean & Barroso, CACM 2013 | Fan-out tail latency, hedged and tied requests | **The source for Concept 38** |
| [The Calculus of Service Availability](https://queue.acm.org/detail.cfm?id=3096459) — Treynor, Dahlin, Rau & Beyer, ACM Queue 2017 | Dependency availability math, the extra-nine rule | The source for Concept 6 |
| [Fail at Scale](https://queue.acm.org/detail.cfm?id=2839461) — Ben Maurer (Facebook), ACM Queue 2015 | Adaptive LIFO, CoDel for request queues, concurrency control | The source for Concept 36 |
| [Controlling Queue Delay](https://queue.acm.org/detail.cfm?id=2209336) — Nichols & Jacobson, ACM Queue 2012 | The CoDel algorithm | Where the queue-delay idea comes from |
| [Simple Testing Can Prevent Most Critical Failures](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/yuan) — Yuan et al., OSDI 2014 | Catastrophic failures caused by mishandled, already-detected errors | The argument in Concept 2 for testing error paths |
| [Crash-Only Software](https://www.usenix.org/conference/hotos-ix/crash-only-software) — Candea & Fox, HotOS 2003 | Recovery as the only start path | The source for Concept 42 |
| [Availability in Globally Distributed Storage Systems](https://research.google/pubs/pub36737/) — Ford et al., OSDI 2010 | Correlated failures in Google's storage fleet | Why independence assumptions overestimate availability (Concept 7) |
| [Fail-Slow at Scale](https://www.usenix.org/conference/fast18/presentation/gunawi) — Gunawi et al., FAST 2018 | Hardware that degrades rather than fails | The evidence behind Concept 3 |
| [An Analysis of Network-Partitioning Failures in Cloud Systems](https://www.usenix.org/conference/osdi18/presentation/alquraan) — Alquraan et al., OSDI 2018 | How real systems mishandle partitions | Pairs with Modules 7 and 9 |
| [Why Do Computers Stop and What Can Be Done About It?](https://www.hpl.hp.com/techreports/tandem/TR-85.7.pdf) — Jim Gray, Tandem 1985 | Process pairs, fail-fast modules, transactions for availability | The classic that started the field; still relevant |
| [How Complex Systems Fail](https://how.complexsystems.fail/) — Richard Cook | Eighteen short observations on failure in complex systems | Essential mindset for incident and "root cause" discussions |
| [Chaos Engineering](https://arxiv.org/abs/1702.05843) — Basiri et al. (Netflix), IEEE Software 2016 | The discipline and its principles | The source for Concept 55 |

### Engineering articles and books (free online)

| Resource | What it covers |
|---|---|
| [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — Marc Brooker, Amazon Builders' Library | **The single best article for Parts B and C.** Timeout selection, single-layer retries, token buckets, jitter |
| [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — AWS Architecture Blog | The jitter simulation behind Concept 17 |
| [Fixing retries with token buckets and circuit breakers](https://brooker.co.za/blog/2022/02/28/retries.html) — Marc Brooker | Retry budgets vs breakers, with simulation (Concepts 19, 28) |
| [Metastability and Distributed Systems](https://brooker.co.za/blog/2021/05/24/metastable.html) — Marc Brooker | An accessible introduction to Concept 10; the rest of [his blog](https://brooker.co.za/blog/) is excellent on retries and breakers |
| [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) — Amazon Builders' Library | Client request tokens and semantically equivalent responses (Concept 20) |
| [Avoiding fallback in distributed systems](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/) — Amazon Builders' Library | **The counter-intuitive argument in Concept 39** |
| [Static stability using Availability Zones](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/) — Amazon Builders' Library | The source for Concept 40 |
| [Reliability, constant work, and a good cup of coffee](https://aws.amazon.com/builders-library/reliability-and-constant-work/) — Amazon Builders' Library | Constant work as a design principle |
| [Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) — Amazon Builders' Library | Goodput, cheap rejection, prioritization (Concept 35) |
| [Workload isolation using shuffle-sharding](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/) — Amazon Builders' Library | The source for Concept 34 |
| [Implementing health checks](https://aws.amazon.com/builders-library/implementing-health-checks/) — Amazon Builders' Library | Shallow vs deep checks, failing open (Concept 41) |
| [Avoiding insurmountable queue backlogs](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/) — Amazon Builders' Library | Queues, LIFO, and backlog strategies (Concept 36, Module 11) |
| [Minimizing correlated failures in distributed systems](https://aws.amazon.com/builders-library/minimizing-correlated-failures-in-distributed-systems/) — Amazon Builders' Library | Failure domains and decorrelation (Concept 7) |
| [Automating safe, hands-off deployments](https://aws.amazon.com/builders-library/automating-safe-hands-off-deployments/) · [Ensuring rollback safety during deployments](https://aws.amazon.com/builders-library/ensuring-rollback-safety-during-deployments/) — Amazon Builders' Library | Progressive delivery and two-phase changes (Concept 54) |
| [Reducing the scope of impact with cell-based architecture](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.html) — AWS whitepaper | The deepest free treatment of Concept 33 |
| [Disaster Recovery of Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html) — AWS whitepaper | The DR strategy ladder (Concept 49), applicable to any cloud |
| [AWS SDK retry behavior](https://docs.aws.amazon.com/sdkref/latest/guide/feature-retry-behavior.html) | A production token-bucket retry implementation (Concept 19) |
| [Google SRE book](https://sre.google/sre-book/table-of-contents/) — especially [Handling Overload](https://sre.google/sre-book/handling-overload/), [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/), [Embracing Risk](https://sre.google/sre-book/embracing-risk/), and the [Availability Table](https://sre.google/sre-book/availability-table/) | **Core reading for Parts A and E.** Criticality, adaptive throttling, retry limits, cascade mechanics |
| [The SRE Workbook](https://sre.google/workbook/table-of-contents/) — especially [Managing Load](https://sre.google/workbook/managing-load/) and [Canarying Releases](https://sre.google/workbook/canarying-releases/) | Practical load management and progressive delivery |
| [Building Secure and Reliable Systems](https://sre.google/books/building-secure-reliable-systems/) — Google (free) | Design for recovery, resilience, and change safety |
| [Principles of Chaos Engineering](https://principlesofchaos.org/) | The five principles in Concept 55 |
| [Martin Fowler: CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html) | The canonical short explanation of Concept 23 |
| [Netflix: Fault Tolerance in a High Volume, Distributed System](https://netflixtechblog.com/fault-tolerance-in-a-high-volume-distributed-system-91ab4faae74a) and the [Hystrix "How it Works" wiki](https://github.com/Netflix/Hystrix/wiki/How-it-Works) | The origin of mainstream bulkheads and breakers (Hystrix is in maintenance; the ideas are current) |
| [Netflix concurrency-limits](https://github.com/Netflix/concurrency-limits) | Adaptive concurrency algorithms (Concept 36) |
| [Resilience4j documentation](https://resilience4j.readme.io/docs/circuitbreaker) | Count- vs time-based windows, slow-call rate, bulkhead variants (Concepts 24, 31) |
| [Envoy: circuit breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking) · [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier) | Proxy-level limits, retry budgets, and ejection (Concept 28) |
| [gRPC deadlines](https://grpc.io/docs/guides/deadlines/) · [gRPC client retry design (A6)](https://github.com/grpc/proposal/blob/master/A6-client-retries.md) | Deadline propagation and server pushback (Concepts 13, 15) |
| [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) · [IETF Idempotency-Key header draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) | Concept 20 from the practitioner and the standard |
| [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) · [RFC 6585 (429)](https://www.rfc-editor.org/rfc/rfc6585) | Idempotent methods, status codes, `Retry-After` (Concepts 15, 22) |
| [Good Retry, Bad Retry: An Incident Story](https://medium.com/yandex/good-retry-bad-retry-an-incident-story-648072d3cee6) — Yandex | Retry amplification in a real incident, with simulations |
| [Slack's migration to a cellular architecture](https://slack.engineering/slacks-migration-to-a-cellular-architecture/) | Cells and AZ draining in practice |
| [danluu/post-mortems](https://github.com/danluu/post-mortems) | A large, categorized collection of public postmortems — excellent interview material |

### Recent incident reports (primary sources)

| Resource | Lesson |
|---|---|
| [AWS post-event summary: us-east-1, October 2025](https://aws.amazon.com/message/101925/) | DNS automation race → lease-manager congestive collapse; control plane vs data plane (Concepts 8, 10, 51) |
| [Cloudflare outage on November 18, 2025](https://blog.cloudflare.com/18-november-2025-outage/) | Globally propagated feature file exceeds a hard limit; validate config, keep last-known-good (Concepts 8, 54) |
| [Azure status history](https://azure.status.microsoft/en-us/status/history/) (Front Door, October 29, 2025, tracking ID YKYN-BWZ) | Global config change bypasses safety checks; global ingress as a shared dependency (Concepts 8, 52) |
| [Google Cloud Service Health — incident history](https://status.cloud.google.com/summary) (June 12, 2025, "Multiple GCP products are experiencing Service issues") | No feature flag, no error handling, no randomized backoff on restart (Concepts 8, 17) |

### Azure and .NET documentation

| Resource | What it covers |
|---|---|
| [Azure Well-Architected Framework — Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/) · [Design review checklist](https://learn.microsoft.com/en-us/azure/well-architected/reliability/checklist) | **The vocabulary of Concept 63** (RE:01–RE:10) |
| WAF reliability guides: [Failure mode analysis](https://learn.microsoft.com/en-us/azure/well-architected/reliability/failure-mode-analysis) · [Redundancy](https://learn.microsoft.com/en-us/azure/well-architected/reliability/redundancy) · [Self-preservation](https://learn.microsoft.com/en-us/azure/well-architected/reliability/self-preservation) · [Transient faults](https://learn.microsoft.com/en-us/azure/well-architected/reliability/handle-transient-faults) · [Testing strategy](https://learn.microsoft.com/en-us/azure/well-architected/reliability/testing-strategy) · [Disaster recovery](https://learn.microsoft.com/en-us/azure/well-architected/reliability/disaster-recovery) · [Multi-region design](https://learn.microsoft.com/en-us/azure/well-architected/reliability/highly-available-multi-region-design) | Each checklist item in depth |
| [Mission-critical workloads](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/) | The fully assembled active-active reference (Concept 63) |
| Azure patterns: [Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) · [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) · [Bulkhead](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) · [Throttling](https://learn.microsoft.com/en-us/azure/architecture/patterns/throttling) · [Rate Limiting](https://learn.microsoft.com/en-us/azure/architecture/patterns/rate-limiting-pattern) · [Queue-Based Load Leveling](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) · [Priority Queue](https://learn.microsoft.com/en-us/azure/architecture/patterns/priority-queue) · [Health Endpoint Monitoring](https://learn.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring) · [Deployment Stamps](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) · [Geode](https://learn.microsoft.com/en-us/azure/architecture/patterns/geodes) · [Compensating Transaction](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction) | The named patterns of this module, as Microsoft names them |
| [Transient fault handling best practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transient-faults) | Retry guidance across Azure services |
| [Azure reliability documentation](https://learn.microsoft.com/en-us/azure/reliability/) · [Availability zones overview](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview) · [Regions overview](https://learn.microsoft.com/en-us/azure/reliability/regions-overview) | Per-service reliability guides, zone support, region pairs (Concepts 48, 62) |
| [Control plane and data plane](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/control-plane-and-data-plane) | Concept 51 in Azure terms |
| [Global routing redundancy for mission-critical web apps](https://learn.microsoft.com/en-us/azure/architecture/guide/networking/global-web-applications/overview) · [Front Door high availability guide](https://learn.microsoft.com/en-us/azure/frontdoor/high-availability) | Alternate ingress design and its costs (Concept 52) |
| [Azure SQL failover groups](https://learn.microsoft.com/en-us/azure/azure-sql/database/failover-group-sql-db) | Listeners, failover policy, grace period (Concepts 47, 50) |
| [Cosmos DB .NET SDK performance tips](https://learn.microsoft.com/en-us/azure/cosmos-db/performance-tips-dotnet-sdk-v3) · [Per-partition automatic failover](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-configure-per-partition-automatic-failover) · [PPAF GA announcement](https://devblogs.microsoft.com/cosmosdb/announcing-the-general-availability-of-per-partition-automatic-failover-for-azure-cosmos-db-nosql/) | Availability strategy (hedging), partition-level circuit breaker, PPAF (Concepts 38, 46, 62) |
| [Service Bus geo-replication](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-geo-replication) | Metadata + data replication and promotion (Concept 50) |
| [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy) · [Storage disaster recovery and failover](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance) | LRS/ZRS/GRS/GZRS and customer-managed failover |
| [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/) · [Azure Load Testing](https://learn.microsoft.com/en-us/azure/load-testing/) | Platform fault injection and load testing (Concept 55) |
| [Build resilient HTTP apps (.NET)](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience) · [Introduction to resilient app development](https://learn.microsoft.com/en-us/dotnet/core/resilience/) | **The official docs for Concepts 58–59**, including the standard handler defaults |
| [Circuit breaker policy fine-tuning best practice](https://devblogs.microsoft.com/dotnet/circuit-breaker-policy-finetuning-best-practice/) — .NET Blog | Threshold and break-duration guidance (Concept 24) |
| [Building resilient cloud services with .NET 8](https://devblogs.microsoft.com/dotnet/building-resilient-cloud-services-with-dotnet-8/) — .NET Blog | The design of `Microsoft.Extensions.Resilience` |
| [Polly documentation](https://www.pollydocs.org/) — [strategies](https://www.pollydocs.org/strategies/) · [circuit breaker](https://www.pollydocs.org/strategies/circuit-breaker.html) · [chaos engineering](https://www.pollydocs.org/chaos/) · [Polly on GitHub](https://github.com/App-vNext/Polly) | The v8 model, defaults, telemetry, and chaos strategies (Concept 58; Module 25) |
| [Microsoft Learn: Implement resiliency in a cloud-native ASP.NET Core microservice](https://learn.microsoft.com/en-us/training/modules/microservices-resiliency-aspnet-core/) | A free hands-on module |
| [HttpClient guidelines for .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines) | Lifetime, `PooledConnectionLifetime`, DNS behaviour (Concept 12) |
| [ASP.NET Core rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) · [Health checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) | Concepts 41 and 61 |
| [Announcing rate limiting for .NET](https://devblogs.microsoft.com/dotnet/announcing-rate-limiting-for-dotnet/) | The design of `System.Threading.RateLimiting` |
| [EF Core connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) · [SqlClient configurable retry logic](https://learn.microsoft.com/en-us/sql/connect/ado-net/configurable-retry-logic) | Built-in database retries (Concept 60) |
| [Azure SDK for .NET configuration samples (retry options)](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/core/Azure.Core/samples/Configuration.md) | `ClientOptions.Retry` and transport configuration (Concept 60) |
| [Aspire service defaults](https://aspire.dev/fundamentals/service-defaults) | Where the global standard handler comes from (Concepts 59–60) |
| [Kubernetes liveness, readiness, and startup probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) · [Pod disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) · [Topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) | Concepts 41–43 on AKS |
| [Toxiproxy](https://github.com/Shopify/toxiproxy) · [WireMock.Net](https://github.com/WireMock-Net/WireMock.Net) · [Chaos Mesh](https://chaos-mesh.org/) · [NBomber](https://nbomber.com/) · [k6](https://k6.io/) | The tooling for Exercises 1–9 |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Reliability vocabulary | Availability (fraction of requests), durability (data survives), resilience (degrade + recover), correctness |
| Fault/error/failure | Cause → bad state → visible failure; fault tolerance breaks the chain |
| Failure modes | Crash, omission, timing, Byzantine — fail-slow causes cascades |
| Gray failure | Healthy to monitors, broken for users; measure from the caller; eject outliers |
| Nines | 99.9% = 43 min/month; 99.99% = 4.3 min/month; 99.999% = 26 s/month |
| Availability math | Series multiplies; parallel multiplies unavailability; assumes independence; MTTR is the cheaper lever |
| Dependencies | Each critical dependency needs ~one more nine; or make it degradable/async |
| Correlated failure | Zones don't protect against config, code, operators, or time; stage changes |
| Outage causes | Change, propagated globally and fast; config as code; last-known-good; flags off by default |
| Cascades | Latency → exhaustion + retries + redistribution + health-check spirals |
| Metastable failure | Trigger + sustaining effect; goodput collapses; recover by cutting load below threshold |
| Timeouts | From p99/p99.9 (false-timeout rate) + margin; connect/attempt/total; `HttpClient` default 100 s |
| Deadlines | Propagate remaining budget; drop queued work past its deadline |
| Timeout meaning | "I don't know" — only retry idempotent operations |
| When to retry | Transient yes; throttled after `Retry-After`; permanent no; ambiguous only if idempotent |
| Backoff | Capped exponential; 1–2 retries in synchronous paths |
| Jitter | Full or decorrelated; jitter every synchronized thing |
| Amplification | attempts^depth (3 at 5 layers = 243); retry at one layer; "don't retry" signal |
| Retry budget | Token bucket: success deposits 0.1, retry costs 1 → retries ≤ ~10% under failure |
| Idempotency | Client key per operation, stored atomically with the effect, replay returns original result |
| Double retry | Azure SDK / Cosmos / EF Core already retry; one owner per dependency |
| Throttling | `Retry-After` wins; jitter on top; 429 doesn't trip the breaker |
| Circuit breaker | Closed/open/half-open; failure ratio over window with minimum throughput; count timeouts, not 4xx |
| Polly breaker defaults | 10% ratio, 100 min throughput, 30 s window, 5 s break — start nearer 50% |
| Breaker scope | Per failure unit (host, partition, tenant); coarse breakers turn partial into total failure |
| Breaker critique | Modal and hard to test; retry budgets, outlier ejection, and bulkheads as alternatives |
| Composition order | Rate limit → total timeout → retry → breaker → attempt timeout |
| Bulkhead | Bounded partition of a shared resource; in .NET a concurrency limiter, sized RPS × p99 × headroom |
| Architectural isolation | Per workload, priority, tenant; cells/stamps; shuffle sharding (C(100,5) ≈ 75M) |
| Overload | Shed early, cheaply, by criticality; 503 + Retry-After; adaptive limits; LIFO + CoDel |
| Four words | Rate limit (client quota), throttle (provider resource), shed (server capacity), backpressure (protocol) |
| Hedging | After ~p95; idempotent only; budgeted; 1,800 ms → 74 ms p99.9 with ~2% extra requests |
| Degradation | Do less (safe) vs redirect elsewhere (risky, rarely tested) |
| Static stability | No provisioning or control-plane calls needed to survive a failure |
| Health checks | Liveness = process only; readiness = local; fail open; rate-limit remediation |
| Headroom | Survive losing 1 of Z zones → Z/(Z−1) capacity: 2 zones 2×, 3 zones 1.5× |
| Active-passive | Simple data; RTO = detect + decide + promote + redirect + warm; RPO = async lag |
| Active-active | Data strategy first; capacity for failed region; always-tested failover path |
| The decision | RTO/RPO, conflict tolerance, cost, maturity; hybrid (active-active compute, active-passive data) is common |
| Failover | Multiple signals over a window; decision owner; fence; promote; redirect (TTL, pooled connections); warm; fail back carefully |
| Zones vs regions | Zones: sync + automatic; regions: async + decision; zone-redundant is the default |
| DR ladder | Backup & restore → pilot light → warm standby → active-active; tier per workload; measured RTO |
| Multi-region data | Replication mode = RPO; write regions = conflict model; lag grows under stress |
| Control plane | Recovery must not need it; cache config/secrets/discovery with last-known-good |
| Global ingress | Shared by all regions; alternate path is costly; keep cheap escape hatches |
| Hidden dependencies | DNS, identity, Key Vault, registry, pipeline, monitoring, certificates, quotas |
| Safe change | Canary → waves → bake → auto-rollback; two-phase changes; kill switches |
| Chaos | Steady state, hypothesis, real faults, small blast radius; Polly chaos, Toxiproxy, Chaos Studio; DR drills |
| Observability | Retry ratio, breaker transitions, rejections, hedges, fallbacks, replication lag, measured RTO |
| FMEA / health model | Per flow: dependency × failure mode × impact × detection × mitigation; healthy/degraded/unhealthy |
| Polly v8 | Immutable pipelines built once; first added = outermost; registry; telemetry; chaos |
| Standard handler | 1,000 concurrency, 30 s total, 3 jittered retries (2 s base), 10%/100/30 s/5 s breaker, 10 s attempt; tune; `DisableForUnsafeHttpMethods` |
| ASP.NET Core | Partitioned rate limiters, global concurrency limiter (NewestFirst), request timeouts, Kestrel limits, readiness-first shutdown |
| Azure recent state | Service Bus geo-replication GA (Dec 2025); Cosmos per-partition automatic failover GA (Jun 2026, ~3 min P99); Polly 8.7.x; `Microsoft.Extensions.Http.Resilience` 10.x |
| Recent incidents | Google Cloud Jun 2025, AWS us-east-1 Oct 2025, Azure Front Door Oct 2025, Cloudflare Nov 2025 |
| When not to | 99.95% rarely needs multi-region active-active; every mechanism is also a failure mode |

---

## Progress

Module 13 complete — and with it, **Phase 3 (Distributed Systems Theory) is complete.** The phase now covers **guarantees** (7), **mechanisms** (8), **agreement** (9), **weakening for latency** (10), **weakening for availability and decoupling** (11), **where the data lives** (12), and **what to do when things fail** (13).

This module closes several loops deliberately:

- **Retry amplification** — introduced in Module 6, named in Module 11, now quantified with budgets, single-layer ownership, and the composition order.
- **Overload** — Module 6's shedding and backpressure, Module 10's cold-cache death spiral, and Module 11's backlog are now one theory: metastability and goodput.
- **Active-active vs active-passive** — Module 8's replication topologies and Module 9's fencing now have a decision framework, failover mechanics, and a DR ladder.
- **RPO/RTO** — Module 12's backup discussion now extends to whole-system DR.

Threads left open on purpose:

- **Polly in depth** — pipeline registry and dynamic reload, custom strategies, testing with `Polly.Testing`, hedging with routing, and telemetry enrichment get their own treatment in **Module 25 (Resilience in .NET)**. Concepts 58–60 are the architect-level summary.
- **SLOs, SLIs, error budgets, burn-rate alerting, and distributed tracing** — Concepts 5 and 56 set these up; **Module 28 (Observability)** develops them.
- **ThreadPool starvation** — named in Concept 9 as a cascade carrier; diagnosed and cured in **Module 15 (Async/await & concurrency)**.
- **Compute choices for zone and region redundancy** — App Service vs AKS vs Container Apps vs Functions, including their HA and scaling characteristics, in **Module 26**.
- **Incremental resilience retrofits** — adding bulkheads and cells to an existing system without a rewrite is a brownfield problem for **Module 32 (Strangler fig and incremental migration)**.

Next in the curriculum: **Module 14 — CLR & memory internals** (stack vs heap, generational GC, Server vs Workstation GC, value vs reference types, boxing), which opens Phase 4 and moves from distributed systems theory to deep .NET mastery.
