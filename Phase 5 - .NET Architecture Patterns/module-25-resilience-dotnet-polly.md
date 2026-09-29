# Module 25 — Resilience in .NET: Polly v8, `Microsoft.Extensions.Resilience`, and Composition Without Multiplication
*Phase 5: .NET Architecture Patterns · Senior/Architect Interview Prep for .NET & C#*

> **Platform state verified on September 29, 2026:** Polly **8.8.0** (released September 14, 2026) is current; `Microsoft.Extensions.Http.Resilience` / `Microsoft.Extensions.Resilience` **10.10.0** (September 9, 2026); .NET 10 is the current LTS and .NET 11 is in preview; .NET 8 and .NET 9 both reach end of support on November 10, 2026. **Polly adopts the Open Source Maintenance Fee on November 16, 2026** — Concepts 67–68 cover what that is, what it isn't, and how to decide. Microsoft has not yet stated its position for the `Microsoft.Extensions.*.Resilience` packages that depend on Polly; treat anything in this module about that question as "open, verify before you decide."

## Orientation

Here is the sentence to carry through the whole module: **Polly doesn't make a system resilient; it makes resilience decisions executable. Every option in a pipeline is a number or a predicate that should have a source — a latency distribution, a failure classification, a budget — and the architect's job is to make sure each decision is made exactly once, at the layer that can safely act on it.**

Module 13 gave you the theory: timeouts chosen from percentiles, retries classified and budgeted, breakers scoped to failure units, bulkheads sized by Little's Law, hedging after the p95, and a composition order with a reason for every layer. It deliberately stopped at the architecture-level summary of Polly (Module 13, Concepts 58–60). This module turns the theory into .NET code you can defend line by line — and it answers the harder question that Modules 20, 23 and 24 left open on purpose: **where each piece lives** in an application that also has EF Core's execution strategy, a command pipeline with a concurrency-retry behavior, Azure SDK clients with their own retries, a message broker that redelivers, and Aspire service defaults that quietly add a resilience handler to every `HttpClient`.

Why this matters in an interview: resilience questions for senior .NET candidates have moved on. "What does a circuit breaker do?" is table stakes. Current loops ask things like *"Your service uses Aspire service defaults and the Azure Blob SDK — how many times is one failing call actually attempted?"*, *"Your breaker has never opened in production — why?"*, *"You added hedging and database CPU doubled — what happened?"*, *"What happens to breaker state when you hot-reload the options?"*, and — as of this year — *"Polly is asking companies for a maintenance fee; what do you recommend?"* Those are questions about **mechanics and ownership**, and this module is built for them.

This module has seven jobs:

1. **Build an exact mental model of Polly v8's execution engine** — pipeline, strategies, outcomes, predicates, context, generators, events — so that you can predict behavior rather than remember it.
2. **Teach each strategy precisely** — timeout, retry, circuit breaker, rate and concurrency limiter, hedging, fallback — including their defaults (several of which are wrong for a request path) and the traps each one sets.
3. **Make composition an engineering exercise** — order, budget arithmetic, validation, and a traced execution you can narrate on a whiteboard.
4. **Go deep on HttpClient** — `Microsoft.Extensions.Http.Resilience`, the standard and hedging handlers, the `HttpClient.Timeout` interaction, request replay, per-authority pipelines, static and gRPC clients, and Aspire's global defaults.
5. **Cover the operational surface** — the registry, complex keys, dynamic reload, telemetry and enrichment, testing, chaos, and custom strategies.
6. **Settle placement** — resilience belongs in adapters; one retry loop per failure kind; how EF Core's execution strategy, the command pipeline (Module 23), event-sourced decisions (Module 24), consumers, subscriptions, outbox relays and Azure SDKs each fit.
7. **Treat the library as a dependency with governance** — Polly's maintenance fee and the realistic options, alternatives, v7 migration, performance, anti-patterns, a review checklist, and when not to use any of it.

Seven framings to carry through:

1. **Built once, state in the instance.** A pipeline is an object with memory — breaker windows, limiter permits. Where and how often you create it decides what it remembers.
2. **Order is semantics.** If strategy A is outside strategy B, A sees B's outcome — including B's exceptions. Reordering two lines changes behavior, not style.
3. **Every number has a source.** A timeout without a percentile, a retry count without a budget, a breaker threshold without telemetry is a guess wearing a config key.
4. **A strategy only sees what passes through it.** Placement determines what a strategy counts, what it can cancel, and what it can repeat.
5. **One retry loop per failure kind,** owned by the level that can safely redo the work. Everything else adds only timeouts, limits, and signals.
6. **Cancellation is the currency.** Timeouts, total budgets, and hedging all work by cancelling a `CancellationToken`. Code that ignores the token silently defeats all three.
7. **Resilience code is production code.** It needs telemetry, tests, configuration governance, and a supply-chain decision — like any other dependency you would page someone about.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What Polly is and isn't | A composition engine for in-process resilience decisions; packages `Polly.Core`, `.Extensions`, `.RateLimiting`, `.Testing`; `Polly` is the v7 API |
| 2 | Pipelines and strategies | Immutable, thread-safe, built once; the first strategy added is the outermost |
| 3 | The callback contract | Everything becomes an `Outcome<T>`; exceptions and results are both outcomes; `ExecuteOutcomeAsync` avoids throwing |
| 4 | `ShouldHandle` | The classification function; the default handles every exception except cancellation — which is almost never what you want |
| 5 | `ResilienceContext` | Pooled per-execution state: token, operation key, typed properties; pass state, use static lambdas |
| 6 | Generators and events | Generators decide (delay, timeout, break duration); events observe (`OnRetry`, `OnOpened`) — never steer control flow from an event |
| 7 | Reactive vs proactive | Reactive strategies judge outcomes; proactive ones act before or during execution; each sees only what passes through it |
| 8 | How timeout works | Cooperative: links a token, cancels it, waits for the callback to notice, throws `TimeoutRejectedException`; 30 s default |
| 9 | Attempt vs total timeouts | Two timeout strategies in one pipeline; `TimeoutGenerator` can read the propagated deadline |
| 10 | The cancellation contract | Pass the pipeline's token everywhere; `WaitAsync` only abandons, it doesn't stop work |
| 11 | Retry mechanics and defaults | `MaxRetryAttempts` is in addition to the first call; defaults are 3 retries, 2 s constant, no jitter |
| 12 | Backoff and jitter math | ±25% jitter for constant/linear; decorrelated-jitter-v2 for exponential; `MaxDelay` doesn't cap a `DelayGenerator` |
| 13 | What to retry | A shared, per-dependency classifier: HTTP, gRPC, SQL, Azure SDK; never retry your own rejections by default |
| 14 | Server-directed delays | `Retry-After`, `x-ms-retry-after-ms`, `RetryAfter` on Polly's own exceptions; check the remaining budget first |
| 15 | Retry budgets | Not built in; implement with a token bucket in `ShouldHandle`, or use gRPC throttling / mesh budgets |
| 16 | Retry anti-patterns | Periodic work, infinite retries in request paths, control flow in `OnRetry`, one strategy across failure domains |
| 17 | Polly v8's breaker | Failure ratio over a sampled window, gated by minimum throughput; no consecutive-count breaker |
| 18 | Lazy transitions, half-open | State changes only when something executes; one probe per break; `BrokenCircuitException.RetryAfter` |
| 19 | Dynamic break durations | `BreakDurationGenerator` grows the break with repeated failure; cap it and jitter it |
| 20 | Operating a breaker | `CircuitBreakerStateProvider` for health; `CircuitBreakerManualControl` as a kill switch |
| 21 | Breaker scope in code | State lives in the pipeline instance; scope via registry keys or `SelectPipelineByAuthority` |
| 22 | Tuning a breaker | Start near 50% with a short break; window ≥ 2 × attempt timeout; tune from telemetry |
| 23 | `System.Threading.RateLimiting` | Leases, four algorithms, queue order, partitioned and chained limiters, `RetryAfter` metadata |
| 24 | Polly's limiter strategy | `AddConcurrencyLimiter` is the v8 bulkhead; 1,000 permits, no queue; `RateLimiterRejectedException` |
| 25 | Outbound quotas vs inbound protection | Polly limits what you send; ASP.NET Core middleware limits what you accept |
| 26 | Limiter placement and the per-process problem | Outermost; share one limiter per dependency; process-local limits divide by instance count |
| 27 | Polly's hedging | Typed only; one extra attempt after 2 s by default; reacts to slowness *and* failure |
| 28 | Hedging concurrency modes | Latency (delay > 0), parallel (0), fallback (negative), dynamic (`DelayGenerator`) |
| 29 | Hedging contexts and routing | Each attempt gets a cloned context; `ActionGenerator` can re-target; losers are cancelled and awaited |
| 30 | The standard hedging handler | Total timeout → hedging → per-endpoint limiter, breaker, attempt timeout; routing groups |
| 31 | Choosing and budgeting hedging | One hedging layer per call; gRPC and Cosmos have their own; hedge idempotent reads, budget it |
| 32 | The fallback strategy | Typed, outermost, runs after everything inside has given up; always observable |
| 33 | Fallbacks that do less | Defaults, stale cache, honest 503s; redirecting load is the risky kind |
| 34 | The canonical order in Polly | Fallback → limiter → total timeout → retry/hedge → breaker → attempt timeout → chaos → call |
| 35 | Tracing one execution | Walk a failing call through every layer with real numbers |
| 36 | Duplicates, typing, naming | Two retries for two failure classes; typed pipelines for results; name every strategy |
| 37 | Budget arithmetic | Total ≥ attempts × attempt timeout + delays; breaker window ≥ 2 × attempt timeout; validate at startup |
| 38 | Where the handler sits | A `DelegatingHandler` above the primary handler; each attempt is a real send with its own span |
| 39 | The standard handler precisely | Limiter 1,000 → total 30 s → 3 jittered exponential retries from 2 s → 10%/100/30 s/5 s breaker → 10 s attempt |
| 40 | Tuning it per dependency | Tighter timeouts, fewer retries, `DisableForUnsafeHttpMethods`, idempotency-key exceptions |
| 41 | `HttpClient.Timeout` vs pipeline timeouts | The effective budget is the minimum; `HttpClient.Timeout` cancels as if the caller gave up |
| 42 | Replaying requests | Bodies must be replayable; streaming uploads and non-idempotent calls need a decision |
| 43 | Custom handlers and global defaults | `AddResilienceHandler`, per-authority pipelines, `ConfigureHttpClientDefaults`, never stack |
| 44 | Static and gRPC clients | `ResilienceHandler` over `SocketsHttpHandler`; gRPC: pick the channel's retry or the handler's, not both |
| 45 | Registry and provider | Build once, resolve many; append-only; keyed services |
| 46 | Complex keys | Per-tenant or per-partition pipelines — and the unbounded-key memory trap |
| 47 | Dynamic reloads | Reload rebuilds the pipeline — breaker state and limiter permits start over |
| 48 | Options validation and governance | Validate at startup; resilience config is change-managed like code |
| 49 | Polly's telemetry | `Polly` meter; `resilience.polly.*` instruments; tags; logs; event IDs |
| 50 | Enrichment and severity | `MeteringEnricher`, `AddResilienceEnricher`, `SeverityProvider`; watch cardinality |
| 51 | From metrics to decisions | Retry ratio, attempts per success, breaker transitions, rejections, hedge win rate |
| 52 | What to test | Your configuration and your delegates — not Polly's internals |
| 53 | Composition tests | `Polly.Testing` descriptors assert order and options |
| 54 | Behavior tests | `FakeTimeProvider` plus a stub handler makes timing deterministic |
| 55 | Chaos strategies | Fault, latency, outcome, behavior; enabled by default at 0.1%; innermost; gated by generators |
| 56 | End-to-end tests | Real transports, fault proxies, and counting attempts at the dependency |
| 57 | Custom strategies | `ResilienceStrategy.ExecuteCore`, options, `AddStrategy`, telemetry — a deadline guard |
| 58 | Reusable classifiers | One transient-fault library per dependency type, shared by retry, breaker, and hedging |
| 59 | Resilience belongs in adapters | Handlers stay pure; the adapter owns transport resilience (closes Module 20) |
| 60 | The retry ownership map | Each failure kind has exactly one retry owner; everyone else adds limits and signals |
| 61 | EF Core and Polly | The execution strategy owns transient DB faults; never wrap it; commit ambiguity needs idempotency |
| 62 | The command pipeline | Concurrency retries outside the unit of work, fresh tracker per attempt, side effects re-run (closes Module 23) |
| 63 | Event-sourced decisions | Re-decide on `WrongExpectedVersion`, IDs outside the loop, bounded with jitter (closes Module 24) |
| 64 | Consumers, subscriptions, relays | Tiny in-process retry, then broker redelivery; breakers pause consumption; park poison |
| 65 | Azure SDK clients | Configure native retries; add outer timeouts only |
| 66 | Inbound vs outbound | Polly for outbound; ASP.NET Core for inbound limits, timeouts, and shedding |
| 67 | Polly's maintenance fee — the facts | $20/month from Nov 16, 2026 for orgs earning ≥ $20k from a product using Polly; source stays BSD-3 |
| 68 | The OSMF decision | Pay, pin, fork, or replace — decided on supply-chain risk and transitive exposure, not on $240 a year |
| 69 | Alternatives and complements | SDK-native, gRPC, service mesh, Dapr, hand-rolled — each wins somewhere |
| 70 | Migrating from v7 | `Policy` → pipeline; `PolicyWrap` → builder order; bulkhead → concurrency limiter; no pessimistic timeout |
| 71 | Performance | Near-zero cost relative to a network call; allocations matter only on hot in-process paths |
| 72 | Anti-patterns | The mistakes that make Polly the outage |
| 73 | The resilience code review | A checklist you can narrate for any `AddResilienceHandler` call |
| 74 | When not to | In-process calls, SDKs that already handle it, work a queue can own, and fallbacks you won't test |

---

# Part A — The Polly v8 execution model

## Concept 1 — What Polly is, what it isn't, and the package map

**Polly is an in-process composition engine.** You hand it a delegate — "call the pricing API", "run this query", "append these events" — and it executes that delegate through a stack of strategies that can time it out, repeat it, refuse it, race it, or replace its result. That is the whole idea. Everything else is detail about *which* decisions it can make and *what information* each decision gets.

It helps to be equally precise about what Polly is **not**:

- **Not a transport.** Polly cannot retry a TCP handshake or a TLS negotiation that happens below your delegate; it repeats whatever your delegate does, in full.
- **Not a queue or a scheduler.** A retry that waits 30 minutes holds memory, a context and often a lock the whole time. Durable, long-horizon retries belong to a broker or a workflow engine (Concept 64; Polly's own docs call "retry as a periodic scheduler" an anti-pattern).
- **Not a service mesh.** It sees one process's calls. It can't eject a bad replica from a load balancer pool or share a retry budget across 40 instances (Concept 69).
- **Not a substitute for idempotency.** It will happily repeat a non-idempotent `POST` if you tell it to (Concept 42).
- **Not "resilience" by itself.** A pipeline with default options is a set of guesses — several of them bad for a request path (Concept 11).

**The package map (Polly 8.8.0, September 2026):**

| Package | What it contains | When you reference it |
|---|---|---|
| `Polly.Core` | The v8 engine: `ResiliencePipeline`, builders, retry, circuit breaker, timeout, hedging, fallback, chaos strategies | Always, for v8 code |
| `Polly.Extensions` | DI integration (`AddResiliencePipeline`, registry wiring, keyed services), telemetry (`ConfigureTelemetry`, logs and metrics) | Any app using `IServiceCollection` |
| `Polly.RateLimiting` | The rate limiter and concurrency limiter strategies over `System.Threading.RateLimiting` | Bulkheads and client-side quotas |
| `Polly.Testing` | `GetPipelineDescriptor()` for composition tests | Test projects |
| `Polly` | The **legacy v7 API** (`Policy`, `PolicyWrap`), kept for compatibility, plus interop | Only while migrating (Concept 70) |

And the Microsoft layer on top:

| Package | Role |
|---|---|
| `Microsoft.Extensions.Resilience` | Telemetry enrichment for Polly (request metadata, exception summaries) — `AddResilienceEnricher()` |
| `Microsoft.Extensions.Http.Resilience` | `HttpClient` integration: `AddStandardResilienceHandler`, `AddStandardHedgingHandler`, `AddResilienceHandler`, HTTP-aware options and predicates |
| `Microsoft.Extensions.Http.Polly` | **Deprecated** v7-era integration (`AddTransientHttpErrorPolicy`, `AddPolicyHandler`) — migrate away |
| `Polly.Extensions.Http` | **Deprecated** — superseded by `Microsoft.Extensions.Http.Resilience` |

**A little history that interviewers like.** Polly v7's "policy" API served .NET for a decade but had duplicated sync/async surfaces, a confusing `PolicyWrap` ordering, bolt-on telemetry and allocation-heavy execution. Microsoft had built internal resilience libraries on top of Polly for its own large services; in 2023 the two efforts were merged into **Polly v8** (released September 2023), and Microsoft built `Microsoft.Extensions.Http.Resilience` on it. That's why the .NET resilience story you'll see in Microsoft's docs, Aspire, and eShop is Polly under the hood. Recent releases worth knowing: **8.3** folded the Simmy chaos library into core; **8.7.0** (June 2026) improved how the caller's cancellation token flows through the hedging and timeout strategies and refactored telemetry; **8.8.0** (September 2026) added `EnableReloadsWithMonitor()` for custom `IOptionsMonitor` sources.

**The interview-grade sentence:** *"Polly is an in-process composition engine: it wraps a delegate in strategies that can time it out, repeat it, refuse it, race it, or replace its result. It doesn't make anything idempotent, it doesn't see other processes, and it isn't a scheduler — so durable retries go to a broker and fleet-wide concerns go to the platform."*

---

## Concept 2 — Pipelines and strategies: immutable, built once, first-added is outermost

A **`ResiliencePipeline`** (or **`ResiliencePipeline<T>`** for a known result type) is an immutable, thread-safe object composed of **strategies**. You build it with a **`ResiliencePipelineBuilder`**, and the single most important rule is:

> **Strategies execute in the order they were added. The first strategy added is the outermost.**

Picture an onion. A call enters the first strategy, which calls the second, which calls the third, which finally calls your delegate. The result travels back out the same way, and each layer can act on it.

```
ExecuteAsync(callback)
 └─ Strategy 1 (added first)      sees everything below it, including their exceptions
     └─ Strategy 2
         └─ Strategy 3 (added last)
             └─ your callback
```

A concrete pipeline, built once:

```csharp
// Built once at startup and reused for every call.
// Circuit-breaker windows and limiter permits live INSIDE this object.
ResiliencePipeline<HttpResponseMessage> pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>
    {
        Name = "catalog",            // becomes the pipeline.name telemetry tag
        InstanceName = "eu-west"     // becomes pipeline.instance
    }
    .AddTimeout(TimeSpan.FromSeconds(3))                           // outermost: total budget
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 2,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        Delay = TimeSpan.FromMilliseconds(200),
        ShouldHandle = static args => TransientHttp.IsTransient(args.Outcome)   // Concept 13
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,
        MinimumThroughput = 20,
        SamplingDuration = TimeSpan.FromSeconds(10),
        BreakDuration = TimeSpan.FromSeconds(5),
        ShouldHandle = static args => TransientHttp.IsBreakerFailure(args.Outcome)
    })
    .AddTimeout(TimeSpan.FromMilliseconds(800))                   // innermost: per attempt
    .Build();

// Execute: the state-passing overload plus a static lambda avoids a closure allocation per call.
HttpResponseMessage response = await pipeline.ExecuteAsync(
    static async (state, ct) => await state.Http.GetAsync($"/items/{state.Id}", ct),
    (Http: httpClient, Id: itemId),
    cancellationToken);
```

Five properties of pipelines to internalize:

1. **Immutable and thread-safe.** One instance serves every concurrent call. There's no locking in your code.
2. **Stateful where the strategy is stateful.** The circuit breaker's health window and the rate limiter's permits live inside the pipeline instance. **Build a pipeline per call and your breaker never trips and your bulkhead never limits** — each call gets a fresh, empty window. This is the most common Polly bug in code reviews.
3. **Sync and async unified.** The same pipeline runs `Execute` and `ExecuteAsync`; v7's four interfaces (`ISyncPolicy`, `IAsyncPolicy`, and their generic twins) are gone.
4. **`ValueTask`-based and allocation-conscious.** The fast path allocates little or nothing (Concept 71).
5. **Composable.** `AddPipeline(otherPipeline)` nests a pipeline as a strategy, and `ResiliencePipeline.Empty` is the no-op pipeline — useful in tests (Concept 52) and as an explicit "no resilience here" decision.

**Generic vs non-generic.** `ResiliencePipeline` can execute callbacks of any result type but its reactive strategies are configured against `object` outcomes; `ResiliencePipeline<HttpResponseMessage>` knows the result type, so its strategies can classify *results* (a 503 is a value, not an exception). If failures show up as return values, use the typed pipeline (Concept 36).

**The interview-grade sentence:** *"A Polly pipeline is an immutable, thread-safe onion of strategies where the first one added is the outermost. I build it once — through DI or the registry — because breaker and limiter state live in the instance; a pipeline built per call has a breaker that can never open."*

---

## Concept 3 — The callback contract: everything becomes an `Outcome<T>`

Internally, Polly converts every execution into an **`Outcome<T>`**: either a **result** or an **exception**. Your delegate throws `HttpRequestException`? The pipeline catches it and creates an outcome carrying that exception. Your delegate returns an `HttpResponseMessage` with status 503? That's an outcome carrying a result. Reactive strategies (retry, breaker, hedging, fallback) then look at the outcome and decide.

Two consequences follow.

**First, failures can be values.** HTTP doesn't throw for a 503 — it returns a response. If the pipeline isn't typed (`ResiliencePipeline<HttpResponseMessage>`) and your predicate doesn't inspect results, a 503 is simply "success" to Polly. Conversely, some libraries throw for business outcomes (`EnsureSuccessStatusCode()` turns a 404 into an exception) — and then a default predicate that handles "any exception" will retry your 404s (Concept 4).

**Second, you can execute without exceptions.** `ExecuteOutcomeAsync` runs the pipeline and returns the outcome instead of throwing. It's the low-allocation, exception-free API — useful on hot paths and when an open circuit is an expected, frequent condition you'd rather branch on than catch:

```csharp
ResilienceContext context = ResilienceContextPool.Shared.Get(cancellationToken);
try
{
    Outcome<Price> outcome = await pricingPipeline.ExecuteOutcomeAsync(
        static async (ctx, state) =>
        {
            try
            {
                return Outcome.FromResult(await state.Client.GetPriceAsync(state.Sku, ctx.CancellationToken));
            }
            catch (Exception ex)
            {
                return Outcome.FromException<Price>(ex);       // the callback must convert, not throw
            }
        },
        context,
        (Client: pricingClient, Sku: sku));

    return outcome.Exception switch
    {
        null                        => PriceResult.Live(outcome.Result!),
        BrokenCircuitException bce  => PriceResult.Unavailable(bce.RetryAfter),   // expected, not exceptional
        _                           => PriceResult.Failed()
    };
}
finally
{
    ResilienceContextPool.Shared.Return(context);
}
```

Use it where it pays; for most business code, `ExecuteAsync` with a `try/catch` at the adapter boundary is clearer.

**What happens at the end.** When the outermost strategy returns an outcome carrying an exception, `ExecuteAsync` rethrows it (with its original stack). The exception your caller sees is therefore the *last* one — after all retries — or one of Polly's own (`TimeoutRejectedException`, `BrokenCircuitException`, `RateLimiterRejectedException`). Your adapter should translate these into your domain's failure vocabulary (Concept 59), not leak Polly types into handlers.

**The interview-grade sentence:** *"Polly turns every execution into an outcome — a result or an exception — and reactive strategies judge outcomes. So HTTP status codes only count if the pipeline is typed and the predicate looks at results, and on hot paths or for expected conditions like an open circuit I use ExecuteOutcomeAsync to avoid exceptions as control flow."*

---

## Concept 4 — `ShouldHandle`: the classification function

Every reactive strategy has a **`ShouldHandle`** predicate: given the outcome (and, for retry, the attempt number), return `true` if this strategy should act on it. This is where Module 13's classification (Concept 15: transient, throttled, permanent, ambiguous, overload) becomes code.

**The default is dangerous.** For retry, circuit breaker, hedging and fallback, the default `ShouldHandle` is **"any exception except `OperationCanceledException`"** — and it handles **no results**. That default is wrong in both directions:

- **It over-handles exceptions.** A default retry will retry `NullReferenceException`, `ArgumentException`, `JsonException`, `DbUpdateConcurrencyException`, your domain's `InsufficientFundsException` — and Polly's own `BrokenCircuitException` and `RateLimiterRejectedException`, so a retry outside an open breaker burns all its attempts in microseconds against the breaker. A default breaker counts every bug in your deserialization code as "the dependency is unhealthy."
- **It under-handles results.** A typed pipeline returning a 503 or 429 isn't retried, and the breaker never counts it.

**Two ways to write predicates.** `PredicateBuilder` is fluent and familiar from v7:

```csharp
ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
    .Handle<HttpRequestException>()
    .Handle<TimeoutRejectedException>()
    .HandleResult(r => r.StatusCode is HttpStatusCode.ServiceUnavailable or HttpStatusCode.GatewayTimeout)
```

A **switch expression** over the outcome is faster (no delegate chain), exhaustive by construction, and easier to review:

```csharp
public static class TransientHttp
{
    // Retry decision: transient, throttled, or our own attempt timeout.
    public static ValueTask<bool> IsTransient(Outcome<HttpResponseMessage> outcome) => outcome switch
    {
        { Exception: BrokenCircuitException }        => PredicateResult.False(), // open circuit: stop, don't spin
        { Exception: RateLimiterRejectedException }  => PredicateResult.False(), // our own bulkhead refused
        { Exception: HttpRequestException }          => PredicateResult.True(),
        { Exception: TimeoutRejectedException }      => PredicateResult.True(),  // the inner attempt timeout fired
        { Result.StatusCode: HttpStatusCode.RequestTimeout
                          or HttpStatusCode.TooManyRequests
                          or HttpStatusCode.BadGateway
                          or HttpStatusCode.ServiceUnavailable
                          or HttpStatusCode.GatewayTimeout } => PredicateResult.True(),
        _                                            => PredicateResult.False()  // 500, 4xx, success, anything else
    };

    // Breaker decision: dependency health. 5xx and timeouts count; 429 and 4xx do not.
    public static ValueTask<bool> IsBreakerFailure(Outcome<HttpResponseMessage> outcome) => outcome switch
    {
        { Exception: HttpRequestException or TimeoutRejectedException } => PredicateResult.True(),
        { Result: { } r } when (int)r.StatusCode >= 500                 => PredicateResult.True(),
        _                                                               => PredicateResult.False()
    };
}
```

Notice that retry and breaker get **different** predicates. That's deliberate and reflects Module 13: 429 is retried (after `Retry-After`) but is a capacity signal, not a health signal, so it shouldn't open the breaker (Module 13, Concepts 22 and 25); 500 is often a deterministic bug for that input, so it counts toward health but isn't retried.

`Microsoft.Extensions.Http.Resilience` ships **`HttpClientResiliencePredicates.IsTransient(outcome)`**, which encodes its default classification (`HttpRequestException`, `TimeoutRejectedException`, 408, 429, and all 5xx). It's a reasonable starting point to compose with, and it's what the standard handler uses (Concept 39).

**The interview-grade sentence:** *"Polly's default predicate handles every exception except cancellation and no results — so it retries bugs and ignores 503s. I write explicit, per-dependency predicates with switch expressions, and retry and breaker get different ones: 429 is retried after Retry-After but never counted by the breaker, 500 counts toward health but isn't retried."*

---

## Concept 5 — `ResilienceContext`: pooled per-execution state

Every execution carries a **`ResilienceContext`**. It holds:

- the **`CancellationToken`** the strategies link to and cancel,
- an **`OperationKey`** — a label for the call site that appears as the `operation.key` telemetry tag,
- **`ContinueOnCapturedContext`** — whether awaits inside Polly resume on a captured synchronization context (default `false`),
- **`Properties`** — a typed property bag keyed by **`ResiliencePropertyKey<T>`**.

Contexts are **pooled**: you rent one from `ResilienceContextPool.Shared` and return it. Simple `ExecuteAsync(callback, cancellationToken)` overloads rent and return for you; you only manage contexts yourself when you need properties or an operation key.

```csharp
public static class ResilienceKeys
{
    public static readonly ResiliencePropertyKey<string>   Tenant   = new("app.tenant");
    public static readonly ResiliencePropertyKey<DateTimeOffset> Deadline = new("app.deadline");
}

ResilienceContext context = ResilienceContextPool.Shared.Get("orders.get-by-id", cancellationToken);
try
{
    context.Properties.Set(ResilienceKeys.Tenant, tenantId);
    context.Properties.Set(ResilienceKeys.Deadline, deadline);

    return await pipeline.ExecuteAsync(
        static async (ctx, state) => await state.Repo.GetAsync(state.Id, ctx.CancellationToken),
        context,
        (Repo: repository, Id: orderId));
}
finally
{
    ResilienceContextPool.Shared.Return(context);
}
```

What context properties are for — each appears later in this module:

| Use | Concept |
|---|---|
| A propagated **deadline** read by a `TimeoutGenerator` or a custom guard | 9, 57 |
| A **tenant or partition key** read by a partitioned rate limiter | 26 |
| The **request message** that hedging clones and re-targets | 29 |
| **Sharing information between strategies** (the breaker publishing its break duration for the retry's delay generator) instead of capturing one strategy's state provider inside another | 18 |
| **Chaos targeting** — which user or tenant gets injected faults | 55 |

Three rules:

1. **Always use the token from the context (or the `ct` parameter) inside the callback**, never the outer token you captured. The timeout and hedging strategies cancel the *linked* token; code using the outer one can't be stopped (Concept 10).
2. **Don't hold a context after returning it to the pool**, and don't share one between concurrent executions. Hedging already clones the context for each attempt (Concept 29).
3. **Keep `OperationKey` low-cardinality.** It becomes a metric tag. `"orders.get-by-id"` is fine; `$"orders/{orderId}"` creates a time series per order (Concept 50).

**Static lambdas and state.** The overloads that take a `TState` let you write `static` lambdas that capture nothing, so no closure object is allocated per call. It's a small thing on a network call, a real thing on a hot in-process path, and it's the idiom you'll see in Polly's own docs (Concept 71).

**The interview-grade sentence:** *"ResilienceContext is the pooled per-execution carrier: the token strategies cancel, an operation key for telemetry, and typed properties. I use properties to pass deadlines, tenant keys and request messages into generators and limiters, keep the operation key low-cardinality, and always use the context's token inside the callback."*

---

## Concept 6 — Generators decide, events observe

Every strategy exposes two kinds of delegates, and confusing them produces fragile code.

**Generators compute a decision** from runtime information:

| Generator | Strategy | Decides |
|---|---|---|
| `DelayGenerator` | Retry, hedging | How long to wait before the next attempt |
| `TimeoutGenerator` | Timeout | How long this execution may take |
| `BreakDurationGenerator` | Circuit breaker | How long the circuit stays open |
| `ActionGenerator` | Hedging | What each hedged attempt actually does |
| `FallbackAction` | Fallback | The substitute outcome |
| `RateLimiter` delegate | Rate limiter | Which limiter (or partition) grants the lease |
| `EnabledGenerator`, `InjectionRateGenerator` | Chaos | Whether and how often to inject |

**Events notify you** that something happened: `OnRetry`, `OnTimeout`, `OnOpened`, `OnClosed`, `OnHalfOpened`, `OnHedging`, `OnFallback`, `OnRejected`.

The rule is: **generators decide, events observe.** Put decisions in generators and predicates; use events only for side effects that don't steer control flow — and often not even then, because Polly's built-in telemetry already logs and counts every event (Concept 49).

Three details about events that matter in production:

1. **They run on the execution path and are awaited.** A slow `OnRetry` handler (writing to a remote log store, say) slows every retried call; an exception thrown from an event handler fails the execution. Keep them cheap and non-throwing.
2. **Don't control flow from events.** Polly's docs call out cancelling a `CancellationTokenSource` from `OnRetry` to stop retrying on a particular exception as an anti-pattern — that condition belongs in `ShouldHandle`. Likewise, reading one strategy's `CircuitBreakerStateProvider` inside another strategy's `DelayGenerator` couples them invisibly; publish the information through the context instead (Concept 18).
3. **Some events are useful triggers for *other* systems** — `OnOpened` pausing a message processor (Concept 64), `OnOpened` flipping a feature flag to degraded mode. That's legitimate observation-driven side effect, as long as the pipeline's own behavior doesn't depend on it.

**The interview-grade sentence:** *"In Polly, generators decide — delays, timeouts, break durations, hedged actions — and events observe. Decisions go in predicates and generators; events stay cheap because they're awaited on the call path, and I don't hand-write logging in them because the built-in telemetry already records every event."*

---

## Concept 7 — Reactive vs proactive strategies, and what each one can see

Polly classifies strategies into two families, and the distinction explains most composition behavior:

| Family | Strategies | How they work | Their own exceptions |
|---|---|---|---|
| **Reactive** | Retry, circuit breaker, hedging, fallback | Inspect the **outcome** with `ShouldHandle` and act on it | `BrokenCircuitException`, `IsolatedCircuitException` |
| **Proactive** | Timeout, rate/concurrency limiter, chaos (fault, latency, behavior) | Act **regardless of outcome** — before or during execution — by cancelling or refusing | `TimeoutRejectedException`, `RateLimiterRejectedException`, injected faults |

**The "sees" principle.** A strategy sees exactly the outcomes produced *below* it:

- A retry **outside** an attempt timeout sees `TimeoutRejectedException` — so it must include it in `ShouldHandle` if slow attempts should be retried.
- A breaker **outside** an attempt timeout counts timeouts as failures — which is how Polly makes a breaker react to fail-slow dependencies (Module 13, Concept 3).
- A retry **outside** a breaker sees `BrokenCircuitException` — and must *not* handle it by default, or it burns every attempt against an open circuit in microseconds.
- A retry **outside** a rate limiter sees `RateLimiterRejectedException` — retrying your own bulkhead's refusal adds pressure to the thing the bulkhead protects.
- A total timeout **outside** a retry cancels the token the retry and its delays run on. The retry sees `OperationCanceledException`, which the default predicate doesn't handle, so it stops; the timeout strategy then converts the cancellation into `TimeoutRejectedException` for the caller.

And the converse: **a strategy can't see what happens above it.** An attempt timeout knows nothing about the total budget; a breaker inside a retry has no idea the retry exists. That's why information that crosses layers — the remaining deadline, the breaker's break duration — travels through the context (Concept 5).

**The interview-grade sentence:** *"Reactive strategies judge outcomes, proactive ones cancel or refuse regardless of outcome — and each strategy only sees outcomes from below it. So a retry outside an attempt timeout must handle TimeoutRejectedException, a breaker outside it counts slowness, and a retry outside a breaker or limiter must not handle their rejections."*

---

# Part B — Timeout

## Concept 8 — How the timeout strategy actually works

Module 13 (Concepts 11–13) covered *choosing* timeouts. Here is what Polly's **timeout strategy** does mechanically, because several production surprises follow directly from it.

When an execution enters the timeout strategy:

1. It creates a **linked `CancellationTokenSource`** from the incoming token (the caller's, or the one from an outer strategy) and schedules it to cancel after the timeout, using the builder's `TimeProvider`.
2. It invokes the inner pipeline and callback with the **linked** token.
3. If the callback completes first, the outcome passes through untouched.
4. If the timer fires first, it cancels the linked token — and then **waits for the callback to finish** reacting to that cancellation.
5. If the callback ended with an `OperationCanceledException` caused by *its* timer, the strategy throws **`TimeoutRejectedException`** (after invoking `OnTimeout`).
6. If instead the **caller's** original token was cancelled, the strategy lets the `OperationCanceledException` through unchanged — a caller giving up is not a timeout, and it isn't reported as one.

| Option | Default | Notes |
|---|---|---|
| `Timeout` | **30 seconds** | Fixed timeout |
| `TimeoutGenerator` | `null` | Per-execution timeout; overrides `Timeout`; **a value ≤ `TimeSpan.Zero` disables the strategy** for that execution |
| `OnTimeout` | `null` | Invoked just before `TimeoutRejectedException` is thrown |

Three consequences:

**The timeout is cooperative, so it's only as punctual as your code.** Step 4 is the one people miss: the strategy *waits* for the callback to observe cancellation. A callback that ignores the token isn't abandoned at the deadline — the timeout fires late, whenever the callback happens to finish. Polly's docs make the same point: if the callback doesn't respect the token, the timeout is "unnecessarily delayed" (Concept 10).

**There's no pessimistic timeout in v8.** Polly v7 had a "pessimistic" mode that walked away from a non-cancellable task and returned on time. It was removed because it left orphaned work running, could cause thread-pool starvation, and hid the real problem. If you truly need to stop *waiting* for uncancellable work, `Task.WaitAsync(timeout, ct)` does that explicitly — and makes the orphaned work visible in your code (Concept 10).

**Two different exceptions mean two different things.** `TimeoutRejectedException` means *our* budget ran out and the dependency may be slow — something to retry (if idempotent), count in the breaker, and alert on. `OperationCanceledException` from the caller's token means *the caller* left — something to never retry, never count against the dependency, and usually not even log as an error. Keeping those two apart is what makes breaker statistics honest (Module 13, Concept 25).

Also note the type: Polly throws **`TimeoutRejectedException`**, not `System.TimeoutException`. Predicates copied from v7-era code that look for `TimeoutException` silently stop matching.

**The interview-grade sentence:** *"Polly's timeout links a token, cancels it at the deadline, waits for the callback to notice, and then throws TimeoutRejectedException — while a caller cancellation passes through as OperationCanceledException. It's cooperative by design, so the timeout is only as good as the token plumbing below it."*

---

## Concept 9 — Attempt and total timeouts, and deadline-aware generators

Module 13 established that a resilient call needs at least two timers: a **per-attempt** timeout (inside the retry) and a **total** timeout (outside it) that keeps retries inside the caller's budget. In Polly they're simply two timeout strategies at different positions — name them so their telemetry is distinguishable:

```csharp
builder
    .AddTimeout(new TimeoutStrategyOptions { Name = "total",   Timeout = TimeSpan.FromSeconds(1.5) })
    .AddRetry(retryOptions)
    .AddCircuitBreaker(breakerOptions)
    .AddTimeout(new TimeoutStrategyOptions { Name = "attempt", Timeout = TimeSpan.FromMilliseconds(400) });
```

**Deadline-aware timeouts.** A fixed total timeout is right for an edge service. A service deeper in the call graph should use **the remaining deadline propagated from its caller** (Module 13, Concept 13) — so no work continues after the caller has left. A `TimeoutGenerator` can read that deadline from the context:

```csharp
services.AddResiliencePipeline<string, Quote>("quotes", (builder, context) =>
{
    TimeProvider time = context.ServiceProvider.GetService<TimeProvider>() ?? TimeProvider.System;
    TimeSpan ownBudget = TimeSpan.FromMilliseconds(900);
    TimeSpan responseMargin = TimeSpan.FromMilliseconds(50);

    builder
        .AddTimeout(new TimeoutStrategyOptions
        {
            Name = "total",
            TimeoutGenerator = args =>
            {
                if (!args.Context.Properties.TryGetValue(ResilienceKeys.Deadline, out DateTimeOffset deadline))
                    return ValueTask.FromResult(ownBudget);

                TimeSpan remaining = deadline - time.GetUtcNow() - responseMargin;

                // TRAP: a generator that returns a value <= TimeSpan.Zero DISABLES the timeout.
                // An expired deadline must never mean "no timeout" — clamp to a tiny positive value
                // here, and reject already-expired work up front with a guard strategy (Concept 57).
                TimeSpan effective = remaining < ownBudget ? remaining : ownBudget;
                return ValueTask.FromResult(effective > TimeSpan.Zero ? effective : TimeSpan.FromMilliseconds(1));
            }
        })
        .AddRetry(new RetryStrategyOptions<Quote> { /* Concept 11 */ })
        .AddTimeout(new TimeoutStrategyOptions { Name = "attempt", Timeout = TimeSpan.FromMilliseconds(300) });
});
```

Where does `Deadline` come from? From the inbound request: a small middleware reads a remaining-budget header (or, for gRPC, `ServerCallContext.Deadline`), converts it to an absolute `DateTimeOffset` using *your own* clock, and exposes it through a scoped accessor; the adapter copies it into the `ResilienceContext` before executing (Concept 5). Passing *remaining duration* between services and recomputing locally avoids cross-machine clock skew.

**The trap worth naming in an interview:** "returning zero from a timeout generator means no timeout." It's the kind of detail that turns an expired deadline into an unbounded call — exactly the abandoned work that sustains metastable failures (Module 13, Concept 10).

**The interview-grade sentence:** *"I use two named timeout strategies — total outside the retry, attempt inside it. For services deeper in the graph, the total timeout's generator reads the propagated deadline from the context, and I'm careful that an expired deadline never returns zero, because in Polly a non-positive generated timeout disables the strategy."*

---

## Concept 10 — The cancellation contract in real code

Every Polly strategy that stops work — total timeout, attempt timeout, hedging cancelling the losers, the caller's own cancellation — does it by cancelling a token. So the question for every line inside a callback is: **does this operation observe the token it was given?**

**Operations that honor it (when you pass it):**

| API | Notes |
|---|---|
| `HttpClient.SendAsync/GetAsync(..., ct)` | Cancels the request, aborting the connection if needed |
| EF Core `ToListAsync(ct)`, `SaveChangesAsync(ct)` | Cancels the command; the server is asked to abort |
| `SqlCommand.ExecuteReaderAsync(ct)` | Sends an attention signal; server work may still complete |
| Azure SDK methods (`..., CancellationToken`) | All async methods accept a token |
| `ServiceBusSender.SendMessageAsync(msg, ct)` | Cancels the send; the send may still have happened |
| `Channel<T>.Reader.ReadAsync(ct)`, `SemaphoreSlim.WaitAsync(ct)`, `Task.Delay(t, ct)` | Stop waiting promptly |

**Operations that don't:**

- Synchronous I/O and `.Result`/`.Wait()` (which also risk thread-pool starvation — Module 15).
- `Task.Run(() => SomeBlockingCall())` — the token passed to `Task.Run` only prevents *starting*.
- Libraries whose async methods have no token parameter.
- CPU-bound loops that never call `ct.ThrowIfCancellationRequested()`.
- Using the **outer** token you captured instead of the **inner** token Polly passed in — Polly's docs list this as the timeout strategy's main anti-pattern.

```csharp
// WRONG: the inner token is ignored; the timeout can't stop the call.
await pipeline.ExecuteAsync(async innerCt => await client.GetAsync(url, outerCt), outerCt);

// RIGHT
await pipeline.ExecuteAsync(static async (state, innerCt) => await state.Client.GetAsync(state.Url, innerCt),
                            (Client: client, Url: url), outerCt);
```

The .NET analyzer **CA2016** ("forward the `CancellationToken` parameter to methods that take one") catches many of these at compile time; treat it as an error in adapter projects.

**The escape hatch, and its price.** When you must call something uncancellable, `WaitAsync` stops *waiting* without stopping the *work*:

```csharp
Task<Report> work = legacyClient.BuildReportAsync(request);   // no token parameter
try
{
    return await work.WaitAsync(ct);                             // returns promptly on cancellation...
}
catch (OperationCanceledException)
{
    // ...but the work continues, holding a connection and memory. Observe its fault so it's not
    // lost, and cap how many of these orphans can exist at once (a concurrency limiter around it).
    _ = work.ContinueWith(static t => _ = t.Exception, TaskContinuationOptions.OnlyOnFaulted);
    throw;
}
```

This is the pessimistic timeout made explicit. Its cost — orphaned work consuming the resources the timeout was meant to free — is exactly why Polly removed the implicit version.

**Cancellation is not rollback.** Cancelling a command after the server received it is Module 13's "I don't know" outcome (Concept 14): the database may have committed, the message may have been sent. Timeouts and hedges therefore only make sense on operations that are idempotent or whose outcome you can query (Concept 42).

**The interview-grade sentence:** *"Every Polly timeout and hedge is only a token cancellation, so every async call in the callback must use the token Polly passes in — CA2016 enforces it. For uncancellable legacy calls I use WaitAsync explicitly and bound the orphaned work, and I remember that cancelling a call doesn't undo it."*

---

# Part C — Retry

## Concept 11 — Retry mechanics, and why the defaults are wrong for a request path

The retry strategy re-executes **everything inside it** — every inner strategy and your callback — when `ShouldHandle` returns `true`, up to a limit, waiting a computed delay between attempts. When it gives up, it returns the last outcome (rethrowing the last exception, or returning the last failed result).

| Option | Default | What it means |
|---|---|---|
| `MaxRetryAttempts` | **3** | Retries **in addition to** the first call — 3 means up to **4 attempts** |
| `BackoffType` | **`Constant`** | Constant, linear, or exponential (Concept 12) |
| `Delay` | **2 seconds** | Base delay |
| `MaxDelay` | `null` | Caps computed delays (but not generated ones) |
| `UseJitter` | **`false`** | Randomizes delays |
| `ShouldHandle` | Any exception except `OperationCanceledException` | Concept 4 |
| `DelayGenerator` | `null` | Per-attempt delay override |
| `OnRetry` | `null` | Invoked before each retry's delay |

Read those defaults with Module 13 in mind and each one is a problem in a synchronous request path:

- **Four attempts** is more than the "one or two retries" a request path can afford — and multiplied across layers it's the 4ᵈ amplification of Module 13, Concept 18.
- **Two seconds constant** adds up to six seconds of pure waiting to a request whose SLO is probably under a second.
- **No jitter** means clients that failed together retry together (Module 13, Concept 17).
- **The default predicate** retries bugs and ignores failed results (Concept 4).
- **No total timeout** exists unless you add one outside.

(The HTTP standard handler's *own* retry defaults differ — exponential with jitter — but also start from a 2-second base; Concept 39.)

A request-path retry with defensible numbers:

```csharp
var retry = new RetryStrategyOptions<HttpResponseMessage>
{
    Name = "retry",
    MaxRetryAttempts = 2,                               // 3 attempts total, at most
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,                                   // decorrelated jitter for exponential (Concept 12)
    Delay = TimeSpan.FromMilliseconds(100),
    MaxDelay = TimeSpan.FromSeconds(1),
    ShouldHandle = static args => TransientHttp.IsTransient(args.Outcome)
};
```

And the queue-consumer version for comparison — where time is cheap but the *broker* should own long waits (Concept 64):

```csharp
var consumerRetry = new RetryStrategyOptions
{
    MaxRetryAttempts = 2,                               // absorb blips; let redelivery handle outages
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,
    Delay = TimeSpan.FromMilliseconds(250),
    MaxDelay = TimeSpan.FromSeconds(2),
    ShouldHandle = static args => ServiceBusTransient.IsTransient(args.Outcome.Exception)
};
```

**Attempt numbering.** `AttemptNumber` in retry arguments is zero-based: attempt 0 is the original call, attempt 1 the first retry. Telemetry uses the same numbering (`attempt.number`), which makes "what fraction of successes needed attempt ≥ 1?" a straightforward query (Concept 51).

**The interview-grade sentence:** *"MaxRetryAttempts counts retries on top of the original call, and Polly's retry defaults — three retries, two seconds constant, no jitter, handle any exception — are wrong for a request path. I use one or two jittered exponential retries from about a hundred milliseconds, capped, with an explicit predicate and a total timeout outside."*

---

## Concept 12 — Polly's backoff and jitter math

The next delay is computed from `BackoffType`, `Delay`, `UseJitter`, `MaxDelay`, and optionally `DelayGenerator`. The documented behavior (an implementation detail Polly reserves the right to change, but stable across 8.x):

| BackoffType | Delay before retry *n* (n = 1, 2, 3…) | Example, `Delay` = 1 s |
|---|---|---|
| Constant | `Delay` | 1000, 1000, 1000, … |
| Linear | `Delay × n` | 1000, 2000, 3000, … |
| Exponential | `Delay × 2ⁿ⁻¹` | 1000, 2000, 4000, 8000, … |

**Jitter** behaves differently by type:

- **Constant and linear:** `UseJitter` adds a random value between **−25% and +25%** of the computed delay.
- **Exponential:** `UseJitter` switches to the **decorrelated jitter backoff V2** algorithm that originated in `Polly.Contrib.WaitAndRetry`. It produces delays that grow roughly exponentially on average but are well spread — and the *first* retry can be much shorter than `Delay` (Polly's docs show a 1-second base producing a first delay of about 0.4 s). It's designed to avoid the clustering that naive decorrelated jitter produces.

**`MaxDelay` caps computed delays — but not generated ones.** If a `DelayGenerator` returns a value, Polly uses it as-is, uncapped. If the generator returns `null` or a negative value, Polly falls back to the computed (and capped) delay. The practical trap: a `DelayGenerator` that honors a server's `Retry-After: 120` will wait two minutes regardless of `MaxDelay` — the budget check has to be yours (Concept 14).

**Latency envelope.** Before you ship a retry configuration, compute its worst case:

```
worst-case latency ≈ (MaxRetryAttempts + 1) × attempt timeout + Σ delays (upper bound with jitter)
```

For `MaxRetryAttempts = 2`, attempt timeout 400 ms, exponential from 100 ms: `3 × 400 + (100 + 200)` ≈ 1.5 s, before jitter spread. If the caller's budget is 1 s, the configuration is wrong on paper — and the total timeout will cut the last attempt short every time the dependency is slow (Concept 37).

**Two delay regimes? Two strategies.** Polly's docs discourage a `DelayGenerator` that switches from fast to slow retries at attempt N; the logic becomes a hidden state machine that's hard to test. Instead, add two retry strategies with their own triggers and delays:

```csharp
builder
    .AddRetry(new RetryStrategyOptions { Name = "slow-retry", MaxRetryAttempts = 2,
                                         Delay = TimeSpan.FromSeconds(30), ShouldHandle = ... })   // outer
    .AddRetry(new RetryStrategyOptions { Name = "quick-retry", MaxRetryAttempts = 3,
                                         Delay = TimeSpan.FromMilliseconds(200),
                                         BackoffType = DelayBackoffType.Exponential,
                                         UseJitter = true, ShouldHandle = ... });                  // inner
```

Be aware of what that multiplies to: (2 + 1) × (3 + 1) = 12 attempts. Nested retries are fine in a background job with a clear budget; in a request path they're the amplification problem wearing a new hat (Concept 36).

**The interview-grade sentence:** *"Polly computes constant, linear or exponential delays; jitter is plus-or-minus 25% for the first two and decorrelated-jitter-v2 for exponential. MaxDelay caps computed delays but not a DelayGenerator's, so if I honor Retry-After in a generator I check the remaining budget myself — and I compute the worst-case latency envelope before shipping any retry config."*

---

## Concept 13 — What to retry: classification per dependency type

A predicate is a statement about a dependency's failure semantics, so it belongs **with the dependency**, not scattered across call sites. Build a small classifier per dependency type and share it between retry, breaker and hedging (Concept 58). The table below is the starting point for each.

| Dependency | Retry (transient) | Don't retry | Notes |
|---|---|---|---|
| **HTTP** | `HttpRequestException`, `TimeoutRejectedException`, 408, 429 (after delay), 502, 503, 504 | 4xx (except 408/429), 500 by default, `OperationCanceledException` from the caller | Unsafe methods only with an idempotency key (Concept 42) |
| **gRPC** | `Unavailable`; `ResourceExhausted` after pushback | `InvalidArgument`, `NotFound`, `PermissionDenied`, `FailedPrecondition`, `Unauthenticated`; `Cancelled` | `DeadlineExceeded` is ambiguous — only if idempotent and budget remains; `Aborted` is a concurrency conflict for the *command* layer |
| **SQL Server / Azure SQL** | Leave it to EF Core's execution strategy or SqlClient's retry logic | — | Their curated transient error-number lists beat anything hand-written (Concept 61) |
| **Cosmos DB** | The SDK already retries 429s, transient network errors and cross-region failover | — | Outer retries re-multiply 429s (Concept 65) |
| **Service Bus / Event Hubs** | `ServiceBusException` / `EventHubsException` where **`IsTransient`** is true | Non-transient reasons (`MessageLockLost` is a *processing* concern, not a transport one) | The SDKs retry internally; consumers also have redelivery (Concept 64) |
| **Redis** | Connection and timeout exceptions — for idempotent commands | Non-idempotent writes (`INCR`, `LPUSH`) after an ambiguous timeout | StackExchange.Redis reconnects on its own |
| **Polly's own exceptions** | `TimeoutRejectedException` (from an inner attempt timeout) | `BrokenCircuitException`, `IsolatedCircuitException`, `RateLimiterRejectedException` | Retrying your own rejections defeats them |
| **Concurrency conflicts** | Not at the transport layer | `DbUpdateConcurrencyException`, `WrongExpectedVersion` | These need a fresh decision, not a resend (Concepts 62–63) |

**A .NET-specific nuance worth knowing: "the request never left."** Since .NET 8, `HttpRequestException` carries an **`HttpRequestError`** value. `NameResolutionError`, `ConnectionError` and `SecureConnectionError` mean the failure happened *before* the request was sent — DNS, TCP connect, or TLS handshake — so the server cannot have acted on it. That's the one situation where even a non-idempotent `POST` is safe to retry (Module 13, Concept 14):

```csharp
public static class TransientHttp
{
    // For unsafe methods WITHOUT an idempotency key: retry only if the request provably never left.
    public static ValueTask<bool> NeverSent(Outcome<HttpResponseMessage> outcome) => outcome switch
    {
        { Exception: HttpRequestException
            {
                HttpRequestError: HttpRequestError.NameResolutionError
                               or HttpRequestError.ConnectionError
                               or HttpRequestError.SecureConnectionError
            } } => PredicateResult.True(),
        _       => PredicateResult.False()      // anything after the bytes left is ambiguous
    };
}
```

**Never retry the caller's cancellation.** `OperationCanceledException` whose token is the caller's means nobody is waiting for the answer. Polly's defaults already exclude it; custom predicates written as `Handle<Exception>()` quietly include it.

**The interview-grade sentence:** *"Classification lives with the dependency: a small classifier per dependency type, shared by retry, breaker and hedging. SQL and Cosmos classification I leave to their own retry logic; I never retry Polly's own rejections or the caller's cancellation; and for a POST without an idempotency key I only retry when HttpRequestError says the request never left the machine."*

---

## Concept 14 — Server-directed delays: `Retry-After` and friends

Module 13 (Concept 22) established the rule: **a server's throttling signal overrides your backoff formula**, with jitter on top, and a wait longer than the remaining deadline means *fail fast*. Here's how that looks in Polly.

**HTTP `Retry-After` is handled for you** in `Microsoft.Extensions.Http.Resilience`: `HttpRetryStrategyOptions.ShouldRetryAfterHeader` is `true` by default, so the standard handler waits the server-specified time (seconds or HTTP date) instead of its computed delay.

**Other headers need a `DelayGenerator`.** Cosmos's REST API returns `x-ms-retry-after-ms`; several Azure AI endpoints return `retry-after-ms`; RFC 9331-style `RateLimit` headers exist on some APIs. And Polly's own rejection exceptions carry hints: **`BrokenCircuitException.RetryAfter`** (the remaining open time) and **`RateLimiterRejectedException.RetryAfter`** (from the limiter's lease metadata).

**The budget check goes in `ShouldHandle`, not in the generator.** A `DelayGenerator` can only choose a delay; it can't veto the retry. `ShouldHandle` sees the outcome *and* the context, so it can compare the server's requested wait with the remaining deadline and refuse:

```csharp
static TimeSpan? ServerDelay(HttpResponseMessage? r)
{
    if (r is null) return null;
    if (r.Headers.TryGetValues("retry-after-ms", out var ms) && double.TryParse(ms.First(), out var v))
        return TimeSpan.FromMilliseconds(v);
    if (r.Headers.RetryAfter?.Delta is { } delta) return delta;
    if (r.Headers.RetryAfter?.Date is { } date)   return date - DateTimeOffset.UtcNow;
    return null;
}

var retry = new RetryStrategyOptions<HttpResponseMessage>
{
    MaxRetryAttempts = 2,
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,
    Delay = TimeSpan.FromMilliseconds(100),

    ShouldHandle = async args =>
    {
        if (!await TransientHttp.IsTransient(args.Outcome)) return false;

        // Would the server's requested wait blow the caller's deadline? Then fail fast now.
        if (ServerDelay(args.Outcome.Result) is { } wait &&
            args.Context.Properties.TryGetValue(ResilienceKeys.Deadline, out DateTimeOffset deadline) &&
            DateTimeOffset.UtcNow + wait >= deadline)
        {
            return false;
        }
        return true;
    },

    DelayGenerator = static args =>
    {
        // Honor the server, plus up to 20% jitter so throttled clients don't return in lockstep.
        // Returning null falls back to the computed exponential delay.
        TimeSpan? wait = ServerDelay(args.Outcome.Result);
        return ValueTask.FromResult<TimeSpan?>(
            wait is { } w ? w + TimeSpan.FromMilliseconds(Random.Shared.NextDouble() * w.TotalMilliseconds * 0.2)
                          : null);
    }
};
```

(In production, read "now" from an injected `TimeProvider` rather than `DateTimeOffset.UtcNow`, so tests can control it — Concept 54.)

Two reminders from Module 13 that this code encodes: **jitter goes on top of the server's value**, and **429 is not a breaker failure** — throttling slows you down; it doesn't mean the dependency is down.

**The interview-grade sentence:** *"Retry-After is honored automatically by the HTTP resilience handler; other hints — retry-after-ms, x-ms-retry-after-ms, RetryAfter on Polly's own rejection exceptions — go through a DelayGenerator with jitter added. Because a generator can't veto a retry, the check 'does this wait exceed my deadline' goes in ShouldHandle."*

---

## Concept 15 — Retry budgets on top of Polly

Module 13 (Concept 19) argued for **retry budgets** — a token bucket in which successes deposit a fraction of a token and each retry spends one — because a fixed retry count multiplies load exactly when a dependency is failing hardest. Polly doesn't ship a retry budget, so you have four options:

| Option | Where the budget lives | Fits |
|---|---|---|
| **A token bucket consulted in `ShouldHandle`** | In-process, per dependency | Most .NET services — shown below |
| **A custom strategy** (Concept 57) | In-process | When you want the budget reusable and self-describing in telemetry |
| **gRPC retry throttling** | The gRPC channel's service config (the A6 retry design's `maxTokens`/`tokenRatio` token bucket) | gRPC clients using the channel's native retries |
| **Service mesh budgets** | Envoy/Istio retry budgets per upstream cluster | Fleet-wide policy without code |

The in-process version works because **`ShouldHandle` is evaluated for every outcome — successes included** — and it knows the attempt number, so it can avoid spending a token on an attempt that won't be retried anyway:

```csharp
// One budget per dependency per process (singleton), e.g. a keyed service "inventory".
public sealed class RetryBudget(double depositPerSuccess = 0.1, double maxTokens = 50)
{
    private double _tokens = maxTokens;
    private readonly Lock _gate = new();                            // System.Threading.Lock (.NET 9+)

    public void RecordSuccess() { lock (_gate) _tokens = Math.Min(maxTokens, _tokens + depositPerSuccess); }

    public bool TryAcquire()
    {
        lock (_gate)
        {
            if (_tokens < 1) return false;
            _tokens -= 1;
            return true;
        }
    }
}

services.AddHttpClient<InventoryClient>()
    .AddResilienceHandler("inventory", (builder, context) =>
    {
        RetryBudget budget = context.ServiceProvider.GetRequiredKeyedService<RetryBudget>("inventory");
        const int maxRetries = 2;

        builder
            .AddTimeout(TimeSpan.FromMilliseconds(1300))                  // total: 3 × 300 ms + ~300 ms delays fits
            .AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = maxRetries,
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                Delay = TimeSpan.FromMilliseconds(100),
                ShouldHandle = async args =>
                {
                    if (args.Outcome is { Exception: null, Result.IsSuccessStatusCode: true })
                    {
                        budget.RecordSuccess();                      // successes earn budget
                        return false;
                    }
                    if (!await TransientHttp.IsTransient(args.Outcome)) return false;
                    if (args.AttemptNumber >= maxRetries) return false;  // no retry would happen anyway
                    return budget.TryAcquire();                          // retries spend budget
                }
            })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions())
            .AddTimeout(TimeSpan.FromMilliseconds(300));
    });
```

(`context` here is the handler's `ResilienceHandlerContext`, which exposes the `ServiceProvider`; the budget is registered with `services.AddKeyedSingleton("inventory", new RetryBudget())`.)

Behavior: at low failure rates the bucket stays full and the client behaves like "two retries"; under a sustained outage, retries fall to roughly 10% of traffic (the deposit ratio), so the failing dependency sees about 1.1× load instead of 3×. Emit a metric when `TryAcquire` returns false — "retries refused by budget" is one of the most useful early-warning signals you can have (Concept 51).

**Scope it to shared fate.** One budget per dependency (or per endpoint), per process. One global budget lets a flaky optional dependency spend the retry capacity of a critical one.

**The interview-grade sentence:** *"Polly has no retry budget, so I add one: a token bucket per dependency consulted in ShouldHandle — successes deposit a tenth of a token, retries spend one, and I skip the withdrawal on the last attempt. Under a full outage that caps retry load around ten percent instead of tripling it; for gRPC I'd use the channel's retry throttling, and fleet-wide I'd use the mesh's budget."*

---

## Concept 16 — Retry anti-patterns specific to Polly

Module 13 (Concept 64) listed the architectural mistakes. These are the Polly-level ones you'll catch in code review — several come straight from Polly's own documentation:

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Retry as a periodic scheduler** (`ShouldHandle = _ => true`, `Delay = 24h`) | Holds memory for hours; not durable across restarts | A scheduler (Quartz.NET, Hangfire, a hosted service with `PeriodicTimer`) |
| **`MaxRetryAttempts = int.MaxValue` in a request path** | Retries until the caller's token ends — or forever | Bounded attempts and a total timeout; unbounded only in background loops with a cap on delay |
| **Default or `Handle<Exception>()` predicates** | Retries bugs, validation errors, caller cancellations | Explicit per-dependency classifiers (Concept 13) |
| **Retrying `BrokenCircuitException`** | Burns attempts against an open breaker in microseconds | Don't handle it — or wait at least `RetryAfter` |
| **Controlling flow from `OnRetry`** (cancelling a CTS to stop retrying) | Hidden control flow, hard to test | Put the condition in `ShouldHandle` |
| **One pipeline over several failure domains** (HTTP call *and* JSON deserialization) | A deserialization bug triggers network retries; timeouts mix | One pipeline per failure domain |
| **One `DelayGenerator` with two regimes** | Hidden state machine | Two retry strategies (Concept 12) |
| **Per-attempt setup outside the callback** (called once before `Execute` and again in `OnRetry`) | `OnRetry` doesn't run before the first attempt; easy to forget one call site | Put per-attempt work *inside* the callback |
| **Branching with `condition ? pipeline : ResiliencePipeline.Empty`** | Triggers scattered across code | Express the condition in `ShouldHandle` |
| **Retrying a unit of work in the same `DbContext`** | Stale tracked entities, failed transaction reused | Retry outside the unit of work with a fresh tracker (Concept 62) |
| **In-process retries in a message handler that also has redelivery** | Retries × deliveries; locks held during sleeps | Tiny in-process retry, broker owns the rest (Concept 64) |
| **Retry without a total timeout** | Latency = attempts × timeout + delays, unbounded by the caller's budget | Always an outer total timeout |
| **Wrapping an SDK call that already retries** | Double retry (Module 13, Concept 21) | Configure the SDK; add only an outer timeout (Concept 65) |

**The interview-grade sentence:** *"The Polly retry mistakes I look for in review are default predicates, retrying Polly's own rejections, control flow in OnRetry, one pipeline across failure domains, retry as a scheduler, and missing total timeouts — plus the architectural one: retrying something an SDK or a broker already retries."*

---

# Part D — Circuit breaker

## Concept 17 — Polly v8's breaker: ratio-only, sampled, gated by throughput

Module 13 (Concepts 23–28) covered breakers as a pattern. Polly v8 implements exactly one model — the former v7 "advanced" breaker:

> The circuit opens when, **within the sampling window**, the **ratio of handled failures to all executions** reaches `FailureRatio`, **provided at least `MinimumThroughput` executions** occurred in that window.

| Option | Default | Notes |
|---|---|---|
| `FailureRatio` | **0.1** | 10% of sampled executions failing opens the circuit |
| `MinimumThroughput` | **100** | Below this many executions in the window, the ratio is ignored |
| `SamplingDuration` | **30 s** | The rolling window over which the ratio is computed |
| `BreakDuration` | **5 s** | Fixed open time (ignored if a generator is set) |
| `BreakDurationGenerator` | `null` | Dynamic break duration (Concept 19) |
| `ShouldHandle` | Any exception except `OperationCanceledException` | What counts as a failure (Concept 4) |
| `StateProvider`, `ManualControl` | `null` | Observe and operate the circuit (Concept 20) |
| `OnOpened`, `OnClosed`, `OnHalfOpened` | `null` | Transition events |

**There is no consecutive-failure breaker in v8.** v7's basic "open after N consecutive exceptions" breaker is gone. If a low-traffic dependency needs that behavior, approximate it: `FailureRatio = 1.0` with a small `MinimumThroughput` (say 5) over a modest window opens only when *every* sampled call in the window failed and there were at least five of them.

**How reaction time really works.** The ratio is computed over a rolling window, so recent successes dilute new failures. At a steady load with the dependency failing 100% from time *t₀*, the failure ratio in a window of length *W* grows roughly linearly from 0 to 1 over *W* seconds, so:

```
time to open at total failure ≈ SamplingDuration × FailureRatio
```

| Window | Ratio | Time to open at 100% failure | At a steady 60% failure |
|---|---|---|---|
| 30 s | 0.1 (default) | ~3 s | ~5 s |
| 30 s | 0.5 | ~15 s | ~25 s |
| 10 s | 0.5 | ~5 s | ~8 s |
| 10 s | 1.0 | ~10 s | never |

That table is the reason for the tuning rule in Concept 22: **if you raise the ratio (as you usually should), shorten the window**, or your breaker reacts after the damage is done. And note the last column — a breaker with a 0.5 ratio *never* opens for a dependency failing 40% of requests, which is often exactly right (Module 13, Concept 28: a partial failure shouldn't become a total one).

**Minimum throughput gates the whole thing.** If fewer than `MinimumThroughput` executions happen in the window, the circuit can't open even at 100% failure. The default of 100 in 30 s means a dependency called fewer than ~3.3 times per second **has no functioning breaker at all** — a common reason for "our breaker has never opened." Set it relative to your *lowest* traffic period that matters (Concept 22).

**The interview-grade sentence:** *"Polly v8's breaker opens when the failure ratio over a rolling sampling window reaches the threshold, but only above a minimum throughput — there's no consecutive-count breaker any more. Time to open at total failure is roughly window times ratio, so a higher threshold needs a shorter window, and a minimum throughput set above real traffic means the breaker can never open."*

---

## Concept 18 — Lazy transitions, half-open, and what an open circuit throws

**The breaker is not an active object.** There's no background timer flipping states. Transitions happen **when an execution arrives**:

1. **Closed → Open** when an execution's outcome pushes the window over the threshold. That execution's own outcome is returned or thrown normally; subsequent executions are rejected.
2. **Open:** executions are rejected immediately with **`BrokenCircuitException`**, whose **`RetryAfter`** property tells you how long the circuit is expected to stay open.
3. **Open → Half-open** when the first execution arrives *after* the break duration has elapsed. That execution becomes the **probe** and goes through to the dependency; other executions arriving while the probe is in flight continue to be rejected (at most one trial per break period).
4. **Half-open → Closed** if the probe's outcome isn't handled as a failure; the sampling window starts fresh.
5. **Half-open → Open** if the probe fails, for another break duration (or a generated one).
6. **Isolated:** held open manually via `CircuitBreakerManualControl.IsolateAsync()`; executions get **`IsolatedCircuitException`** (a subclass of `BrokenCircuitException`) until `CloseAsync()`.

Three practical consequences:

**Don't guard execution with the state provider.** Polly's docs call out this anti-pattern explicitly:

```csharp
// WRONG: if nobody executes while the circuit is open, it can never go half-open and recover.
if (stateProvider.CircuitState is not (CircuitState.Open or CircuitState.Isolated))
{
    await pipeline.ExecuteAsync(...);
}
```

If you want to avoid exceptions for an open circuit, use `ExecuteOutcomeAsync` and branch on `outcome.Exception is BrokenCircuitException` (Concept 3) — the execution still reaches the breaker, so it can transition.

**On low-traffic dependencies, "Open" can be stale.** The state provider reports `Open` until a call arrives after the break duration. A dashboard showing "payments circuit open for 40 minutes" at 3 a.m. may just mean nobody called payments. Alert on `OnCircuitOpened` *events* and failure rates, not on the state value alone.

**The probe is a real user request.** In half-open, one caller's actual request tests the dependency. That's fine for idempotent reads; for expensive or non-idempotent operations, consider **out-of-band probing** — a background task hits the dependency's health endpoint and uses `ManualControl` to isolate or close the circuit (Concept 20) — so no user request is used as a canary.

**A retry outside a breaker should use `RetryAfter`, not burn attempts.** Polly's docs show the preferred way to link the two: rather than having the retry's `DelayGenerator` read the breaker's state provider (tight coupling), pass information through the context or the exception. The simplest correct version is: don't retry `BrokenCircuitException` at all in a request path; in a background job, retry it with the delay taken from `RetryAfter`:

```csharp
DelayGenerator = static args => new ValueTask<TimeSpan?>(
    args.Outcome.Exception is BrokenCircuitException { RetryAfter: { } wait } ? wait : null)
```

**The interview-grade sentence:** *"Polly's breaker transitions lazily — only when a call arrives — so the first call after the break duration becomes the single half-open probe, and guarding Execute with the state provider stops the circuit from ever recovering. An open circuit throws BrokenCircuitException with RetryAfter; manual isolation throws IsolatedCircuitException."*

---

## Concept 19 — Dynamic break durations

A fixed 5-second break recovers quickly from brief faults, but during a long outage it sends a probe every five seconds from *every instance* — a small but steady load on a dependency that's trying to come back. **`BreakDurationGenerator`** lets the break grow. Its arguments expose **`FailureRate`**, **`FailureCount`**, **`HalfOpenAttempts`** and the **`Context`**:

```csharp
BreakDurationGenerator = static args =>
{
    // 5 s, 10 s, 20 s, 40 s, then capped at 60 s — growing with consecutive failed probes —
    // plus ±20% jitter so a fleet of instances doesn't probe in lockstep.
    double seconds = Math.Min(60, 5 * Math.Pow(2, args.HalfOpenAttempts));
    double jitter  = 0.8 + Random.Shared.NextDouble() * 0.4;
    return new ValueTask<TimeSpan>(TimeSpan.FromSeconds(seconds * jitter));
}
```

(If both `BreakDuration` and `BreakDurationGenerator` are set, the generator wins.)

Design notes:

- **Cap it.** An uncapped exponential break turns a 10-minute outage into a 40-minute self-imposed one.
- **Jitter it.** Without jitter, 50 instances that opened together probe together — a synchronized wave (Module 13, Concept 17).
- **Keep the first break short.** Most faults a breaker sees are brief; long first breaks block healthy traffic after recovery. The .NET team's tuning guidance favors short breaks (around 5 seconds) for this reason (Concept 22).
- **Don't grow it on a single failure.** Growing only with `HalfOpenAttempts` means the breaker extends its break only after *probes* fail — evidence of a sustained outage, not noise.

**The interview-grade sentence:** *"I use a BreakDurationGenerator that starts at five seconds and grows with consecutive failed half-open probes, capped at a minute and jittered, so a long outage gets fewer probes from the fleet without a brief blip blocking healthy traffic for long."*

---

## Concept 20 — Operating a breaker: `StateProvider` and `ManualControl`

Two small objects turn a breaker from a black box into something operators can see and steer.

**`CircuitBreakerStateProvider`** exposes `CircuitState` (`Closed`, `Open`, `HalfOpen`, `Isolated`). A state provider is bound to **one** breaker instance. Use it for:

- a **dependency health check** that reports *Degraded* (never *Unhealthy* — Module 13, Concept 41: dependency failures don't belong in liveness),
- product decisions such as rendering a degraded UI proactively,
- dashboards — with the "lazy state" caveat from Concept 18.

**`CircuitBreakerManualControl`** exposes `IsolateAsync()` (hold the circuit open; executions throw `IsolatedCircuitException`) and `CloseAsync()` (reset to closed). One manual control can be **attached to several breakers**, so a single call can isolate every pipeline that talks to a dependency. Use it for:

- an **incident kill switch** — stop calling a dependency that's harming you *before* the statistics catch up,
- **planned maintenance** on a dependency,
- **out-of-band health probing** (Concept 18),
- pre-emptively isolating a region's endpoint during a failover.

Wiring them through DI for an `HttpClient`:

```csharp
builder.Services.AddKeyedSingleton("payments", new CircuitBreakerStateProvider());
builder.Services.AddKeyedSingleton("payments", new CircuitBreakerManualControl());

builder.Services.AddHttpClient<PaymentsClient>(c => c.BaseAddress = new("https://payments.internal"))
    .AddResilienceHandler("payments", (pipeline, context) =>
    {
        IServiceProvider sp = context.ServiceProvider;
        pipeline
            .AddTimeout(TimeSpan.FromSeconds(3))
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
            {
                FailureRatio = 0.5,
                MinimumThroughput = 20,
                SamplingDuration = TimeSpan.FromSeconds(10),
                BreakDuration = TimeSpan.FromSeconds(5),
                StateProvider = sp.GetRequiredKeyedService<CircuitBreakerStateProvider>("payments"),
                ManualControl = sp.GetRequiredKeyedService<CircuitBreakerManualControl>("payments")
            })
            .AddTimeout(TimeSpan.FromSeconds(1));
    });
// Note: a state provider binds to a single breaker. If this handler also used SelectPipelineByAuthority
// (one pipeline per host), you would need one state provider per authority instead.

public sealed class PaymentsCircuitHealthCheck(
    [FromKeyedServices("payments")] CircuitBreakerStateProvider circuit) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default) =>
        Task.FromResult(circuit.CircuitState switch
        {
            CircuitState.Closed   => HealthCheckResult.Healthy(),
            CircuitState.HalfOpen => HealthCheckResult.Degraded("payments: circuit probing"),
            _                     => HealthCheckResult.Degraded("payments: circuit open or isolated")
        });
}

// An operational kill switch — behind strong authorization, audited.
app.MapPost("/ops/circuits/payments/isolate",
        async ([FromKeyedServices("payments")] CircuitBreakerManualControl control) =>
        {
            await control.IsolateAsync();
            return Results.NoContent();
        })
   .RequireAuthorization("ops");
```

Register the health check under a **dependency** tag for dashboards and the health model — not under the liveness tag (Module 13, Concept 41). And remember that manual control is **per process**: an admin endpoint isolates the instance that received the request. For a fleet-wide switch, drive `IsolateAsync()` from a feature flag or configuration change that every instance observes.

**The interview-grade sentence:** *"I expose each critical breaker through a state provider — surfaced as a Degraded dependency health check, never liveness — and a manual control that acts as a kill switch. Manual control is per process, so a fleet-wide isolate comes from a flag every instance watches, not from one admin call."*

---

## Concept 21 — Breaker scope in code: pipeline instances and keys

Module 13 (Concept 26) said to scope a breaker to **the unit that fails together** — per host, per partition, per tenant — because a coarse breaker turns a partial failure into a total one. In Polly, **scope is simply "which pipeline instance handles this call,"** since breaker state lives in the instance. You choose scope by choosing how pipelines are keyed:

| Scope | How in .NET | Watch out for |
|---|---|---|
| Per dependency | One named/typed `HttpClient` with its own resilience handler | Multiple hosts behind one client share a breaker |
| Per host (authority) | `.SelectPipelineByAuthority()` on the resilience handler: one pipeline per scheme + host + port | Hosts that vary per tenant (subdomains) create unbounded pipelines |
| Per custom key (shard, region, tenant) | `.SelectPipelineBy(sp => request => key)` | Key cardinality = number of breakers kept in memory |
| Non-HTTP per key | Registry with complex keys (Concept 46) | The registry is append-only — keys are never evicted |

```csharp
// One breaker per shard: a failing shard opens only its own circuit.
builder.Services.AddHttpClient<ShardedCatalogClient>()
    .AddResilienceHandler("catalog", pipeline =>
    {
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions { FailureRatio = 0.5, MinimumThroughput = 20 })
                .AddTimeout(TimeSpan.FromMilliseconds(500));
    })
    .SelectPipelineBy(static _ => static request =>
        request.Headers.TryGetValues("x-shard", out var values) ? values.First() : "default");
```

The cardinality warning is the senior part. Each distinct key materializes and caches a full pipeline — breaker window, limiter, timeouts — for the life of the process. Per-shard (tens) is fine; per-region (a handful) is fine; per-tenant with 50,000 tenants is a memory leak with a resilience label on it. For large key spaces, bucket keys (hash tenants into 64 groups) or push per-tenant isolation to the edge with partitioned rate limiters instead (Concept 25).

**Breaker state stays per process.** Every instance keeps its own breakers and they converge quickly because they all see the same dependency. A shared breaker in Redis adds a critical dependency to the failure path and hides per-instance network differences (Module 13, Concept 26).

**The interview-grade sentence:** *"In Polly a breaker's scope is whichever pipeline instance handles the call, so I scope by keying pipelines — per client, per authority with SelectPipelineByAuthority, or per shard with SelectPipelineBy — and I watch key cardinality, because every key keeps a full pipeline in memory for the life of the process."*

---

## Concept 22 — Tuning a breaker from data

The defaults (10% over 30 s, minimum 100, 5 s break) are generic. A defensible configuration comes from four pieces of data.

**1. Normal failure-ratio noise.** Plot the per-window failure ratio for the dependency during healthy periods (Polly's telemetry gives you handled outcomes per attempt — Concept 51). If healthy windows routinely hit 3–5%, a 10% threshold is one bad deploy of *theirs* away from opening on noise. The .NET team's guidance is that, absent a threshold defined by the dependency's owners, **a failure ratio of 0.5 or higher** is a better starting point than 0.1.

**2. Reaction time you need.** From Concept 17, time to open ≈ window × ratio. For ratio 0.5 and a 5-second reaction target, the window is 10 s.

**3. Lowest meaningful traffic.** `MinimumThroughput` must be comfortably below the executions you'll see in one window during the quietest period you care about. At 5 RPS and a 10 s window (~50 calls), a minimum of 20 works; the default 100 never evaluates.

**4. The attempt timeout.** The breaker counts an attempt when it completes, so a window shorter than the attempt timeout can't see slow failures. `Microsoft.Extensions.Http.Resilience` enforces **`SamplingDuration ≥ 2 × attempt timeout`** in the standard handler's validation; apply the same rule to custom pipelines (Concept 37).

A worked configuration for a dependency at ~200 RPS normal, ~5 RPS at night, 800 ms attempt timeout:

```csharp
new HttpCircuitBreakerStrategyOptions
{
    FailureRatio      = 0.5,                        // well above healthy noise (~2%)
    SamplingDuration  = TimeSpan.FromSeconds(10),   // ≥ 2 × 800 ms; reacts in ~5 s at total failure
    MinimumThroughput = 20,                         // night traffic gives ~50 calls per window
    BreakDurationGenerator = /* 5 s growing to 60 s, jittered — Concept 19 */,
    ShouldHandle = static args => TransientHttp.IsBreakerFailure(args.Outcome)   // 5xx + timeouts; not 429/4xx
}
```

Then **validate against history**: replay the last few incidents' failure curves (or inject them with chaos — Concept 55) and check the breaker opens when you'd want it to and stays closed through the noisy-but-healthy periods. A breaker that never opened in a year is either protecting a perfect dependency or misconfigured — usually the latter.

**The interview-grade sentence:** *"I tune a breaker from four numbers: healthy failure-ratio noise, the reaction time I need, the lowest traffic per window, and the attempt timeout. That usually lands around a 50% ratio over a 10-second window, a minimum throughput below night-time volume, and a short jittered break — then I replay past incidents to check it would have opened."*

---

# Part E — Rate limiter and concurrency limiter

## Concept 23 — `System.Threading.RateLimiting` in five minutes

Polly doesn't implement its own limiters in v8; its rate limiter strategy is a thin layer over **`System.Threading.RateLimiting`** (the same primitives ASP.NET Core's rate-limiting middleware uses). Knowing the primitives is what makes the strategy predictable.

**The model: leases.** You ask a `RateLimiter` for permits and get a **`RateLimitLease`**:

```csharp
using RateLimitLease lease = await limiter.AcquireAsync(permitCount: 1, ct);
if (!lease.IsAcquired)
{
    TimeSpan? retryAfter = lease.TryGetMetadata(MetadataName.RetryAfter, out var ra) ? ra : null;
    // reject
}
// do the work; disposing the lease returns concurrency permits
```

- `AcquireAsync` may **wait in a queue** (if the limiter has one); `AttemptAcquire` never waits.
- **Disposing the lease matters** for the concurrency limiter — that's what releases the permit. For replenishing limiters it's harmless.
- Replenishing limiters attach **`RetryAfter` metadata** to failed leases; the concurrency limiter can't know when a permit will free up, so it doesn't.

**The four algorithms:**

| Limiter | Limits | Key options | Typical use |
|---|---|---|---|
| `ConcurrencyLimiter` | Operations **in flight** | `PermitLimit`, `QueueLimit`, `QueueProcessingOrder` | Bulkheads (Module 13, Concepts 30–31) |
| `TokenBucketRateLimiter` | Average **rate** with bursts | `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriod`, `QueueLimit` | Outbound quotas, per-client limits |
| `FixedWindowRateLimiter` | Requests **per window** | `PermitLimit`, `Window` | Simple quotas (bursty at window edges) |
| `SlidingWindowRateLimiter` | Requests per window, smoothed | `PermitLimit`, `Window`, `SegmentsPerWindow` | Quotas without edge bursts; v7's `RateLimit` equivalent |

**Queue order** is `OldestFirst` (FIFO) or `NewestFirst` (LIFO) — the latter implements the goodput-preserving discipline of Module 13, Concept 36.

**Partitioned and chained limiters.** `PartitionedRateLimiter.Create<TResource, TKey>(resource => RateLimitPartition.GetXxxLimiter(key, factory))` gives each key (tenant, API key, host) its own limiter; `PartitionedRateLimiter.CreateChained(a, b)` requires *both* to grant — "100 per minute **and** 10 per second."

**Statistics.** `limiter.GetStatistics()` returns available permits, queued count, and total successful/failed leases — cheap to export as gauges for sizing (Concept 51).

**The interview-grade sentence:** *"System.Threading.RateLimiting works in leases: a concurrency limiter caps in-flight work and releases on lease disposal, while token-bucket and window limiters cap rate and attach Retry-After metadata. Partitioned limiters give each key its own limiter and chained ones require all to agree — and Polly's limiter strategy is a thin layer over exactly these."*

---

## Concept 24 — The Polly rate limiter strategy, and the end of the Bulkhead policy

The strategy lives in the **`Polly.RateLimiting`** package (not `Polly.Core`). Three ways to add it:

```csharp
builder.AddConcurrencyLimiter(permitLimit: 50, queueLimit: 0);                   // the v8 bulkhead

builder.AddRateLimiter(new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
{
    TokenLimit = 20, TokensPerPeriod = 10, ReplenishmentPeriod = TimeSpan.FromSeconds(1),
    QueueLimit = 0, AutoReplenishment = true
}));                                                                              // an existing limiter instance

builder.AddRateLimiter(new RateLimiterStrategyOptions
{
    DefaultRateLimiterOptions = new ConcurrencyLimiterOptions { PermitLimit = 50, QueueLimit = 5 },
    OnRejected = static args => default                                           // observe rejections
});
```

| Option | Default |
|---|---|
| `RateLimiter` (delegate returning a lease) | `null` — if unset, a concurrency limiter is created from `DefaultRateLimiterOptions` |
| `DefaultRateLimiterOptions` | **`PermitLimit` = 1,000, `QueueLimit` = 0** |
| `OnRejected` | `null` |

On rejection the strategy throws **`RateLimiterRejectedException`**, which carries **`RetryAfter`** (when the lease had it) and **`TelemetrySource`** (which pipeline and strategy rejected — handy when a pipeline has both a quota limiter and a bulkhead). `OnRejected` fires first but, per Polly's docs, `RetryAfter` isn't available inside it.

**The Bulkhead policy is gone.** Polly v7's `Policy.BulkheadAsync(maxParallelization, maxQueuingActions)` maps directly to `AddConcurrencyLimiter(permitLimit, queueLimit)` — v8 doesn't expose a separate bulkhead because a bulkhead *is* a concurrency limiter. Likewise, v7's own `RateLimit` policy is replaced by the BCL limiters (a `SlidingWindowRateLimiter` is the closest equivalent).

**Disposal.** Limiters you create are disposable resources. For a pipeline built once at startup, it doesn't matter much; with **dynamic reloads** (Concept 47), every reload creates a new limiter and the old one must be disposed — Polly's docs show registering `context.OnPipelineDisposed(() => limiter.Dispose())` in the pipeline callback.

**The interview-grade sentence:** *"In v8 the bulkhead is AddConcurrencyLimiter — Polly's rate limiter strategy over System.Threading.RateLimiting, defaulting to 1,000 permits with no queue. Rejections throw RateLimiterRejectedException with RetryAfter and the telemetry source, and limiters I create get disposed through OnPipelineDisposed when pipelines reload."*

---

## Concept 25 — Outbound quotas vs inbound protection

Rate limiting shows up on both sides of a service, and each side has a different right tool.

**Outbound — Polly's job.** You're the client, and something downstream has a limit:

- **A provider quota**, e.g. a SaaS API allowing 100 requests per second per API key. Exceeding it gets you 429s, possibly penalty periods. A client-side token bucket keeps you under it *proactively*, so you don't spend requests learning the limit from 429s.
- **A dependency's capacity**, protected by a **concurrency limiter** (bulkhead) so one slow dependency can't absorb all your in-flight capacity.
- **Per-tenant fairness toward a shared quota** — a partitioned limiter keyed by tenant so one tenant's batch can't consume the whole API key's quota:

```csharp
// Each tenant gets 20 req/s toward the provider; the whole process also stays under 100 req/s.
var perTenant = PartitionedRateLimiter.Create<ResilienceContext, string>(ctx =>
    RateLimitPartition.GetTokenBucketLimiter(
        ctx.Properties.GetValue(ResilienceKeys.Tenant, "unknown"),
        _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 20, TokensPerPeriod = 20, ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            QueueLimit = 0, AutoReplenishment = true
        }));

var overall = PartitionedRateLimiter.Create<ResilienceContext, string>(_ =>
    RateLimitPartition.GetTokenBucketLimiter("all", _ => new TokenBucketRateLimiterOptions
    {
        TokenLimit = 100, TokensPerPeriod = 100, ReplenishmentPeriod = TimeSpan.FromSeconds(1),
        QueueLimit = 0, AutoReplenishment = true
    }));

var providerQuota = PartitionedRateLimiter.CreateChained(perTenant, overall);

builder.AddRateLimiter(new RateLimiterStrategyOptions
{
    Name = "provider-quota",
    RateLimiter = args => providerQuota.AcquireAsync(args.Context, 1, args.Context.CancellationToken)
});
```

**Inbound — ASP.NET Core's job.** When you're the server, use the **rate-limiting middleware** (`AddRateLimiter`/`UseRateLimiter`, Module 13, Concept 61): it integrates with endpoints and authorization, partitions by user or API key, sets status codes and `Retry-After`, and runs before your application code spends anything. Don't reinvent it with Polly pipelines in middleware.

**Inbound, but not HTTP.** Polly's limiter *is* a good fit for work your process pulls in: capping concurrent background jobs, concurrent calls into a native library, or concurrent work items taken from a channel — anywhere there's no framework-provided limiter. (Message processors usually have their own knob, such as Service Bus's `MaxConcurrentCalls`; use that first.)

**The interview-grade sentence:** *"Outbound, I use Polly limiters — a token bucket to stay under a provider's quota, partitioned per tenant and chained with a global limit, and a concurrency limiter as a bulkhead. Inbound HTTP belongs to ASP.NET Core's rate-limiting middleware; Polly's limiter is for inbound work that has no framework limiter, like background job concurrency."*

---

## Concept 26 — Limiter placement, sharing, and the per-process problem

**Placement depends on what you're limiting.** This is a subtlety most write-ups miss:

- A **concurrency limiter (bulkhead)** goes **outermost**. A call holds one permit for its whole life, *including its retries* — that's the correct meaning of "in-flight work to this dependency." A rejection here is cheap and isn't retried by this pipeline (Module 13, Concept 29).
- A **quota limiter** that models the *provider's* counting (every request that reaches them counts) belongs **inside the retry**, so **each attempt spends a token**. Put it outside and your retries bypass the quota — you'll hit the provider's 429s exactly when things are already going wrong.

```csharp
builder
    .AddConcurrencyLimiter(permitLimit: 64, queueLimit: 0)       // bulkhead: one permit per logical call
    .AddTimeout(TimeSpan.FromSeconds(2))                          // total
    .AddRetry(retryOptions)                                       // must NOT handle RateLimiterRejectedException
    .AddRateLimiter(providerQuotaLimiter)                         // quota: one token per attempt
    .AddCircuitBreaker(breakerOptions)
    .AddTimeout(TimeSpan.FromMilliseconds(500));                  // attempt
```

**Share one limiter per real constraint.** A quota belongs to the provider (or the API key), not to a code path. If three typed clients call the same provider, they must share one limiter instance — pass the same `RateLimiter`/`PartitionedRateLimiter` into each pipeline via `AddRateLimiter(limiter)`. The standard HTTP handler creates its own concurrency limiter **per pipeline instance** (per client, or per authority when you select by authority) — fine as a bulkhead, wrong as a shared quota.

**The per-process problem.** Every in-process limiter limits **one process**. Twelve instances with a 100 req/s limiter are a 1,200 req/s client. Options, in increasing cost:

1. **Divide statically**: limit = quota ÷ instance count, with headroom. Breaks when autoscaling changes the count.
2. **Let the provider's 429 + `Retry-After` be authoritative** and keep the local limiter as a smoothing layer; honor the server signal (Concept 14).
3. **Centralize egress**: route calls through a gateway (e.g., Azure API Management's rate-limit policies) or an egress proxy that enforces the quota once.
4. **A distributed limiter** (a Redis-backed token bucket) — accurate, but you've added a network round trip and a critical dependency to every call. Justified only when the quota is expensive to exceed.

Size bulkheads from Little's Law, as in Module 13 (Concept 31): peak RPS to the dependency × p99 latency × headroom. And export `GetStatistics()` so you can see how close you run.

**The interview-grade sentence:** *"A bulkhead goes outermost so a call holds one permit across its retries, but a provider-quota limiter goes inside the retry so every attempt spends a token. I share one limiter instance per real constraint, and I remember every in-process limiter is per instance — for a hard provider quota across a scaled-out fleet I either divide it, centralize egress through a gateway, or accept a distributed limiter's cost."*

---

# Part F — Hedging

## Concept 27 — Polly's hedging strategy: reacting to slowness *and* failure

Module 13 (Concept 38) explained why hedging works: at fan-out, a component's tail becomes the system's median, and sending a second copy of a slow request to another replica after about the p95 cuts the tail for a few percent of extra load. Polly's **hedging strategy** implements that — and a bit more.

| Option | Default | Notes |
|---|---|---|
| `MaxHedgedAttempts` | **1** (valid range 1–10) | Extra attempts **in addition to** the original |
| `Delay` | **2 seconds** | How long to wait for an attempt before launching the next; special values below |
| `ShouldHandle` | Any exception except `OperationCanceledException` | Which outcomes count as failed attempts |
| `ActionGenerator` | Re-runs the original callback | What each hedged attempt does (Concept 29) |
| `DelayGenerator` | `null` | Per-attempt delay; overrides `Delay` |
| `OnHedging` | `null` | Invoked before each hedged attempt |

Four things distinguish Polly's hedging from the textbook version:

1. **It's typed only.** There's `HedgingStrategyOptions<T>` and no non-generic variant, because the strategy must pick a winning *result*.
2. **It reacts to failure as well as latency.** In latency mode, if the first attempt *fails* (a handled outcome) before the delay, the next attempt starts immediately — hedging behaves like a zero-delay retry to the next target. That's why the standard hedging handler has no separate retry strategy (Concept 30).
3. **It waits for the losers.** When an acceptable result arrives, the strategy cancels the other in-flight attempts and **awaits their cancellation** before returning. Hedged callbacks that ignore their token delay the whole response — the cancellation contract again (Concept 10).
4. **The final failure is the primary's.** If every attempt fails, the caller receives the **original attempt's** failure, not the last one.

Polly's docs add two cautions worth repeating in an interview: **don't start background work** inside hedged callbacks (each attempt may start it), and remember that hedging buys latency with resources — "if low latency is not a critical requirement, the retry strategy may be more appropriate."

**The interview-grade sentence:** *"Polly's hedging is typed-only, launches one extra attempt after two seconds by default, and reacts to failures as well as slowness — a failed first attempt triggers the hedge immediately. It cancels and awaits the losers before returning, so hedged calls must honor cancellation, be idempotent, and never start background work."*

---

## Concept 28 — Hedging concurrency modes: latency, parallel, fallback, dynamic

The `Delay` value (or what `DelayGenerator` returns) selects one of four behaviors:

| Mode | Configure | Behavior | Use |
|---|---|---|---|
| **Latency** | `Delay > 0` | Start the primary; if it hasn't completed *or* has failed by `Delay`, start the next attempt; first acceptable result wins | The classic hedge: `Delay` ≈ p90–p95 |
| **Parallel** | `Delay = TimeSpan.Zero` | Start all `MaxHedgedAttempts + 1` attempts immediately; fastest acceptable result wins | Rarely justified; multiplies load by the attempt count |
| **Fallback** | `Delay < TimeSpan.Zero` (e.g., −1 ms) | Only **one attempt at a time**; the next starts only when the previous **fails** | Sequential failover across endpoints — "try region A, then B" without concurrency |
| **Dynamic** | `DelayGenerator` | Choose per attempt — e.g., first two in parallel, the rest sequential | Tiered strategies |

Fallback mode is worth singling out: it gives you **failover routing without concurrency** — each hedged attempt can go to a different endpoint (Concept 29), but you never have two in flight. That's often the right tool for *write* paths that must try a secondary but must not race it against the primary.

Dynamic mode example from Polly's docs, adapted:

```csharp
new HedgingStrategyOptions<HttpResponseMessage>
{
    MaxHedgedAttempts = 3,                             // up to 4 attempts in total
    DelayGenerator = static args => new ValueTask<TimeSpan>(args.AttemptNumber switch
    {
        0 or 1 => TimeSpan.Zero,                       // first two race in parallel
        _      => TimeSpan.FromMilliseconds(-1)        // then sequential failover
    })
}
```

And a practical use of dynamic mode: **switching hedging off under pressure** without redeploying. A `DelayGenerator` that reads a load signal (your own overload flag, CPU, or a "shed optional work" switch) and returns a negative delay turns latency hedging into sequential failover exactly when extra load would hurt most — Module 13's "stop hedging when the system is overloaded."

**The interview-grade sentence:** *"The delay picks the hedging mode: positive is latency hedging, zero is a parallel race, negative is sequential failover with one attempt in flight, and a DelayGenerator makes it dynamic — which I also use to degrade latency hedging into sequential failover when the system is under pressure."*

---

## Concept 29 — Hedging contexts, `ActionGenerator`, and routing to other replicas

Concurrent attempts sharing one `ResilienceContext` would race on its properties, so the hedging strategy keeps two kinds of context:

- the **primary context** — the one the strategy received — preserved for the whole execution;
- an **action context** per attempt — a **deep copy** of the primary with its **own cancellation token**, so each loser can be cancelled independently.

When an attempt wins, **its action context is merged back** into the primary (new and updated properties survive), and the others are cancelled and discarded. `ActionGenerator` and `OnHedging` receive both contexts and are never run concurrently with each other, so mutating them there is safe.

**`ActionGenerator` decides what a hedged attempt does.** By default it re-invokes the original callback with the action context. To hedge **to a different replica**, put the target into the action context and let the callback read it:

```csharp
static class Targets
{
    public static readonly Uri Primary   = new("https://pricing.westeurope.internal");
    public static readonly Uri Secondary = new("https://pricing.northeurope.internal");
    public static readonly ResiliencePropertyKey<Uri> Key = new("hedge.target");
}

ResiliencePipeline<HttpResponseMessage> hedged = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
    {
        MaxHedgedAttempts = 1,
        Delay = TimeSpan.FromMilliseconds(120),                    // ≈ p95 of the primary
        ShouldHandle = static args => TransientHttp.IsTransient(args.Outcome),
        ActionGenerator = static args =>
        {
            args.ActionContext.Properties.Set(Targets.Key, Targets.Secondary);   // hedge goes elsewhere
            return () => args.Callback(args.ActionContext);
        }
    })
    .Build();

ResilienceContext context = ResilienceContextPool.Shared.Get("pricing.get", cancellationToken);
try
{
    context.Properties.Set(Targets.Key, Targets.Primary);
    using HttpResponseMessage response = await hedged.ExecuteAsync(
        static async (ctx, state) =>
        {
            Uri target = ctx.Properties.GetValue(Targets.Key, Targets.Primary);
            // GetAsync builds a NEW HttpRequestMessage per attempt — concurrent attempts can't share one.
            return await state.Http.GetAsync(new Uri(target, $"/prices/{state.Sku}"), ctx.CancellationToken);
        },
        context,
        (Http: httpClient, Sku: sku));
    // ...
}
finally
{
    ResilienceContextPool.Shared.Return(context);
}
```

Three rules for hedged callbacks:

1. **Never share a request object across attempts.** An `HttpRequestMessage` can't be sent concurrently; build one per attempt (as above) or let the HTTP hedging handler clone it for you (Concept 30).
2. **Hedge to an independent replica.** A hedge to the same overloaded host or the same hot partition adds load and gains nothing (Module 13, Concept 38).
3. **Nothing inside the callback should outlive it** — no fire-and-forget tasks, no background timers — because the losers are cancelled and discarded.

**The interview-grade sentence:** *"Each hedged attempt runs with a deep-copied action context and its own token, and the winner's context is merged back. To hedge to another replica I set the target in the action context from the ActionGenerator, and I build a fresh request per attempt, because a request message can't be sent twice concurrently."*

---

## Concept 30 — The standard hedging handler for `HttpClient`

`AddStandardHedgingHandler()` from `Microsoft.Extensions.Http.Resilience` packages hedging for HTTP. Its pipeline, outermost to innermost:

| Order | Strategy | Default |
|---|---|---|
| 1 | Total request timeout | 30 s |
| 2 | Hedging | One hedged attempt (up to 10 allowed); 2 s delay; HTTP transient predicate |
| 3 | Rate limiter — **per endpoint** | 1,000 permits, no queue |
| 4 | Circuit breaker — **per endpoint** | 10% over 30 s, minimum 100, 5 s break |
| 5 | Attempt timeout — **per endpoint** | 10 s |

The key design idea: strategies 3–5 are a **pool of per-endpoint pipelines**, selected by URL authority by default, so **hedges are never sent to an endpoint whose breaker is open**. There's no retry strategy, because hedging already launches another attempt when one fails.

**Routing.** Without routing configuration, the handler hedges to the same URL (useful when a load balancer spreads attempts across instances). With routing, each attempt can target a different endpoint:

- **Ordered groups** — attempt *n* goes to group *n*; within a group, an endpoint is picked by weight. The number of groups bounds the number of attempts.
- **Weighted groups** — each attempt (or only the first, depending on `SelectionMode`) picks a group by weight — useful for A/B routing combined with hedging.

The handler rewrites the request's scheme, host and port per attempt and **snapshots the request** so attempts can run concurrently — which requires the request content to be replayable (Concept 42).

```csharp
builder.Services.AddHttpClient<PricingClient>()
    .AddStandardHedgingHandler(static routing =>
    {
        routing.ConfigureOrderedGroups(static o =>
        {
            o.Groups.Add(new UriEndpointGroup
            {
                Endpoints = { new() { Uri = new("https://pricing.westeurope.internal") } }
            });
            o.Groups.Add(new UriEndpointGroup
            {
                Endpoints = { new() { Uri = new("https://pricing.northeurope.internal") } }
            });
        });
    })
    .Configure(static o =>
    {
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(2);
        o.Hedging.MaxHedgedAttempts   = 1;
        o.Hedging.Delay               = TimeSpan.FromMilliseconds(150);    // ≈ p95 of the primary region

        o.Endpoint.Timeout.Timeout                 = TimeSpan.FromMilliseconds(800);
        o.Endpoint.CircuitBreaker.FailureRatio     = 0.5;
        o.Endpoint.CircuitBreaker.MinimumThroughput = 20;
        o.Endpoint.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(10);   // ≥ 2 × attempt timeout
    });
```

**When to choose it over the standard resilience handler:** idempotent, latency-critical reads with at least two independent endpoints (regions, replicas, or a load-balanced pool large enough that a second attempt lands elsewhere), where tail latency is a product requirement and a few percent of extra load is acceptable. Everything else: the standard resilience handler (Concept 39).

**The interview-grade sentence:** *"The standard hedging handler is total timeout, then hedging, then a per-endpoint limiter, breaker and attempt timeout selected by authority — so hedges never go to an endpoint whose circuit is open. With ordered groups each attempt goes to the next region; I set the hedge delay near the primary's p95 and keep it to idempotent reads."*

---

## Concept 31 — Choosing the hedging layer, and budgeting it

Hedging is available at several layers in a typical .NET-on-Azure system, and **only one should hedge a given call**:

| Layer | Mechanism | Notes |
|---|---|---|
| **Polly / HTTP handler** | `AddHedging`, `AddStandardHedgingHandler` | General purpose; routing across endpoints |
| **gRPC channel** | `HedgingPolicy` in the channel's service config (`MaxAttempts`, `HedgingDelay`, `NonFatalStatusCodes`) | Can't be combined with a gRPC retry policy on the same method; **`HedgingDelay` defaults to zero**, i.e. parallel |
| **Cosmos DB SDK** | Threshold-based cross-region **availability strategy** | Understands regions, sessions and consistency; hedges reads by default (Module 13, Concepts 38, 62) |
| **Service mesh** | Proxy-level hedging on per-try timeouts | Fleet-wide policy, no code |

The rule: **use the layer that understands the dependency best.** For Cosmos DB that's the SDK; for gRPC services pick either the channel's hedging or the HTTP handler — never both; for plain HTTP, the resilience handler.

**Budgeting hedges.** Polly doesn't cap hedges as a fraction of traffic. The budget comes from three places:

1. **The delay itself.** A delay at the p95 hedges roughly 5% of requests when healthy — the classic few-percent overhead.
2. **Failure-triggered hedges.** Because hedging also reacts to failures, during a partial outage the extra load is roughly the failure fraction. Keep `MaxHedgedAttempts` at 1 and let per-endpoint breakers stop hedging into a failing endpoint.
3. **An explicit off-switch under pressure** — the dynamic-mode trick from Concept 28: a `DelayGenerator` that switches to sequential failover when the system is overloaded.

Then measure: **hedges fired ÷ requests** and **hedge win rate** (how often the hedge, not the primary, produced the result). A hedge rate far above 5% means the delay is too low or the primary is degraded; a win rate near zero means the hedge isn't buying anything (Concept 51).

**The interview-grade sentence:** *"One layer hedges a given call — the Cosmos SDK for Cosmos, the channel or the handler for gRPC but not both, the resilience handler for HTTP. I budget hedging through the p95 delay, a single hedged attempt, per-endpoint breakers, and a switch to sequential mode under load — and I watch hedge rate and hedge win rate to know it's paying for itself."*

---

# Part G — Fallback

## Concept 32 — The fallback strategy

The **fallback strategy** replaces a failed outcome with a substitute. It's typed only (`FallbackStrategyOptions<T>`):

| Option | Default | Notes |
|---|---|---|
| `FallbackAction` | — (required) | Produces the substitute `Outcome<T>` |
| `ShouldHandle` | Any exception except `OperationCanceledException` | Which outcomes to replace |
| `OnFallback` | `null` | Invoked when the fallback runs — and reported as the `OnFallback` telemetry event |

**Placement: outermost.** Polly's docs show "fallback after retries" as the canonical position. Put a fallback **inside** a retry and the retry never sees a failure (the fallback turned it into success); put it inside a breaker and the breaker's statistics go blind. Outermost means: every other strategy has done its job and given up, and only then do you substitute.

**Handle what the inner strategies throw.** An outermost fallback will see `TimeoutRejectedException` (total or attempt timeout), `BrokenCircuitException`, `RateLimiterRejectedException`, and the dependency's own transient exceptions. Be explicit rather than relying on "any exception," so a bug in your mapping code isn't silently replaced by a default:

```csharp
builder
    .AddFallback(new FallbackStrategyOptions<Recommendations>
    {
        ShouldHandle = static args => args.Outcome.Exception switch
        {
            TimeoutRejectedException or BrokenCircuitException
                or RateLimiterRejectedException or HttpRequestException => PredicateResult.True(),
            _ => PredicateResult.False()        // bugs and caller cancellation surface as errors
        },
        FallbackAction = static _ => Outcome.FromResultAsValueTask(Recommendations.Popular with { Degraded = true })
    })
    .AddConcurrencyLimiter(50)
    .AddTimeout(TimeSpan.FromMilliseconds(250))                 // total
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<Recommendations> { FailureRatio = 0.5, MinimumThroughput = 20 })
    .AddTimeout(TimeSpan.FromMilliseconds(200));                // attempt (no retry: soft dependency)
```

**Make degradation visible.** The substitute should carry a flag (`Degraded = true`, an "as of" timestamp for stale data) so callers and UIs can be honest, and the `OnFallback` event gives you a "degraded responses" metric for free (Concept 49). A fallback nobody can see is Module 13's warning about fallbacks that hide failures until the fallback fails too.

**The interview-grade sentence:** *"Fallback goes outermost so retries and the breaker see real failures first. I list exactly which exceptions it replaces — timeouts, open circuit, limiter rejections, transport errors — so bugs still surface, and the substitute carries a degraded flag so the degradation is visible to callers and to telemetry."*

---

## Concept 33 — Fallbacks that degrade by doing less

Module 13 (Concept 39) drew the line: **degradation that does less work is low-risk; fallback that redirects work to another system is high-risk**, because the redirected path is rarely exercised and activates exactly when everything is stressed. The Polly fallbacks worth writing are almost all of the first kind:

| Fallback | Example | Notes |
|---|---|---|
| **Static default** | "Popular items" instead of personalized recommendations | Cheapest and safest |
| **Last-known-good** | Serve the last successful value with an "as of" timestamp | Needs a store written on success |
| **Honest failure result** | A domain result `PriceUnavailable(retryAfter)` instead of an exception | Often better done in the adapter than in the pipeline (Concept 59) |
| **Synthetic HTTP response** | Inside an HTTP pipeline, turn every final failure into a `503` response | Normalizes failures for a typed client; loses exception detail |
| **Queue for later** | Accept the command, publish to the outbox, confirm asynchronously | A design decision, not a Polly option (Module 11) |
| **Alternate endpoint** | Secondary region or provider | The risky kind — prefer hedging's fallback mode with a per-endpoint breaker (Concept 28), and exercise it continuously |

A last-known-good fallback, done carefully:

```csharp
public sealed class CatalogAdapter(
    ResiliencePipelineProvider<string> pipelines, IMemoryCache lastKnownGood, CatalogHttpClient client)
{
    private readonly ResiliencePipeline<Product> _pipeline = pipelines.GetPipeline<Product>("catalog");

    public async ValueTask<ProductView> GetAsync(string sku, CancellationToken ct)
    {
        try
        {
            Product product = await _pipeline.ExecuteAsync(
                static async (state, token) => await state.Client.GetProductAsync(state.Sku, token),
                (Client: client, Sku: sku), ct);

            lastKnownGood.Set($"lkg:{sku}", (product, DateTimeOffset.UtcNow), TimeSpan.FromHours(6));
            return ProductView.Live(product);
        }
        catch (Exception ex) when (ex is TimeoutRejectedException or BrokenCircuitException or HttpRequestException
                                   && lastKnownGood.TryGetValue($"lkg:{sku}", out (Product P, DateTimeOffset At) cached))
        {
            return ProductView.Stale(cached.P, asOf: cached.At);   // visible degradation, bounded staleness
        }
    }
}
```

Here the fallback lives in the **adapter** rather than as a Polly strategy, because it needs domain knowledge (the key, the staleness bound, the view type). That's the typical split: **Polly fallback for generic substitutes** (a static default, a synthetic 503), **adapter-level fallback for anything domain-aware**. Caching libraries can do the last-known-good part for you — FusionCache's "fail-safe" feature is built around exactly this — but the decision about acceptable staleness is still yours and the product's.

**The interview-grade sentence:** *"The fallbacks I write do less work: a static default, last-known-good data with an as-of timestamp, or an honest unavailable result. Generic substitutes can be a Polly fallback strategy; anything domain-aware lives in the adapter. Fallbacks that redirect load to another system I treat as a second production path that must be exercised continuously."*

---

# Part H — Composition

## Concept 34 — The canonical order, restated as a Polly pipeline

Module 13 (Concept 29) justified the order **rate limiter → total timeout → retry → breaker → attempt timeout**. With the full Polly toolbox, the complete canonical pipeline — outermost first — is:

```
Fallback                          optional; typed; substitutes only after everything below has given up
 └─ Concurrency limiter           the bulkhead; cheap rejection; one permit per LOGICAL call (incl. retries)
     └─ Total timeout             the caller's budget, covering attempts AND delays
         └─ Retry  —or—  Hedging  never both on the same call
             └─ Quota limiter     optional; per ATTEMPT, when modeling a provider's request quota
                 └─ Circuit breaker   sees every attempt, including attempt timeouts
                     └─ Attempt timeout   bounds each try
                         └─ Chaos          innermost; gated; injected faults are seen by everything above
                             └─ your callback
```

Why each position, compressed:

| Layer | Why here |
|---|---|
| **Fallback outermost** | It must see the *final* outcome — after retries, breaker rejections and timeouts — and it must not hide failures from the breaker or the retry (Concept 32) |
| **Bulkhead next** | Rejects before any resource is spent; the permit covers the whole logical call; rejections still degrade through the fallback |
| **Total timeout** | Bounds retries *and* their delays; when it fires, inner strategies see a cancellation and stop |
| **Retry or hedging** | Re-runs everything below; hedging replaces retry because it already relaunches on failure |
| **Quota limiter** | Every attempt costs a token, matching how the provider counts (Concept 26) |
| **Breaker** | Counts each attempt separately; an open circuit fails attempts fast (and the retry must not handle that) |
| **Attempt timeout** | Makes slowness a failure the breaker counts and the retry can retry |
| **Chaos innermost** | Subverts the call at the last moment so the real strategies above are exercised (Concept 55) |

The standard resilience handler is this pipeline without fallback, quota limiter and chaos; the standard hedging handler replaces retry with hedging and moves limiter, breaker and attempt timeout into **per-endpoint** pipelines.

**The interview-grade sentence:** *"My canonical Polly order is fallback, bulkhead, total timeout, retry or hedging, an optional per-attempt quota limiter, breaker, attempt timeout, then chaos innermost — so rejections are cheap, retries fit the budget, the breaker counts every attempt including slow ones, and injected faults exercise the real strategies."*

---

## Concept 35 — Tracing one execution through the pipeline

Being able to narrate an execution step by step, with numbers, is the fastest way to show an interviewer you understand composition. Take this pricing pipeline:

```
limiter 64 → total 1.5 s → retry (2, exponential 100 ms, jitter) → breaker (50% / 10 s / min 20 / break 5 s) → attempt 400 ms
```

and a dependency that has just started hanging.

**Call A — the one that trips the breaker (t = 0):**

1. **Limiter** grants a permit (40 calls in flight, limit 64).
2. **Total timeout** starts a 1.5 s timer and links a token.
3. **Retry**, attempt 0 → **breaker** (closed; window has 30 calls, 12 failures) → **attempt timeout** starts 400 ms → callback sends the request.
4. At **400 ms** the attempt timeout cancels its token; `HttpClient` aborts the request; the callback throws `OperationCanceledException`; the attempt timeout converts it to **`TimeoutRejectedException`**.
5. **Breaker** records a failure (13 of 31 — below 50%) and passes the exception up.
6. **Retry**: `TimeoutRejectedException` is handled → computes a jittered delay (~80 ms) → attempt 1.
7. Attempt 1 times out at ~880 ms → breaker records it: 16 failures of 34 (other concurrent calls are failing too) — still under 50%. Retry waits ~230 ms → attempt 2 at ~1.11 s.
8. Attempt 2 times out at ~1.51 s — but the **total timeout** fired at **1.5 s** first: it cancels the outer token, the attempt's linked token is cancelled with it, the in-flight request aborts. Inner strategies see `OperationCanceledException` (not handled by the breaker's or retry's predicates), and the total timeout throws `TimeoutRejectedException` to the caller.
9. The **limiter** releases the permit. Elapsed: ~1.5 s — exactly the budget, no more.

**Call B, a moment later:** attempt 0 times out; the breaker records the 18th failure of 36 — now **50%** with ≥ 20 samples → **the circuit opens**. Call B's own `TimeoutRejectedException` still propagates normally; retry waits and tries again → the breaker rejects immediately with **`BrokenCircuitException`** (`RetryAfter` ≈ 5 s) → the retry predicate does **not** handle it → it propagates at ~0.5 s. If a fallback is configured, the caller gets the degraded result.

**Calls C…Z during the break:** each is rejected by the breaker in microseconds; no threads or sockets are held; latency for the degraded path drops from 1.5 s to near zero. That drop — protecting the caller — is the breaker's first job.

**At t ≈ 5.5 s:** the next call arrives after the break → **half-open** → it becomes the probe; concurrent calls are still rejected. If the dependency has recovered, the probe succeeds and the circuit closes; if not, it reopens for another (possibly longer) break.

**A caller who gives up:** a user navigates away at 300 ms → the request-aborted token is cancelled → it flows down through the linked tokens → the callback throws `OperationCanceledException` → the attempt timeout sees that the *caller's* token was cancelled and passes the cancellation through unchanged → the breaker doesn't count it → the retry doesn't retry it → the caller's code sees a cancellation. Nothing about the dependency's health changed — which is exactly what the statistics should say.

**The interview-grade sentence:** *"If I trace a hanging dependency through the pipeline: the attempt timeout turns slowness into TimeoutRejectedException, the breaker counts it, the retry retries with jitter, the total timeout caps the call at the budget, and once the ratio crosses the threshold later calls fail in microseconds with BrokenCircuitException — which the retry deliberately doesn't handle. A caller's own cancellation passes through without touching the breaker's statistics."*

---

## Concept 36 — Multiple instances of the same strategy, typed pipelines, and naming

**Duplicates are allowed — and sometimes correct.** A pipeline can contain the same strategy type more than once:

| Duplicate | Why | Watch out |
|---|---|---|
| Two **timeouts** | Total and attempt (Concept 9) | Name them `"total"` and `"attempt"` |
| Two **limiters** | Bulkhead outside, quota inside (Concept 26) | Name them; `TelemetrySource` on the exception tells you which rejected |
| Two **retries** | Different failure classes with different delay regimes — e.g., an outer retry for throttling that honors `Retry-After`, an inner one for connection blips | **Attempts multiply**: (r₁+1)(r₂+1). Fine in background work with a budget; rarely fine in a request path |
| Retry **and** hedging | — | Don't: hedging already relaunches failed attempts |

**Typed vs untyped.** Use `ResiliencePipeline<T>` when **results** carry failure information (HTTP responses, gRPC-style result objects, your own `Result<T>` types); use the untyped `ResiliencePipeline` for operations whose failures are exceptions (a Service Bus send, an EF Core command). A typed builder can include an untyped pipeline with **`AddPipeline(sharedPipeline)`** — and because the *instance* is shared, so is its state. That's a feature when you want one bulkhead shared by several typed pipelines for the same dependency, and a bug when you didn't realize two call paths now share a breaker.

**Name everything.** Telemetry is only as useful as its tags:

- **Pipeline name** — the DI key (`AddResiliencePipeline("pricing", ...)`) or `builder.Name`; for HTTP handlers, derived from the client and handler names. Becomes `pipeline.name`.
- **Instance name** — `builder.InstanceName`, or produced by a key formatter for complex keys (Concept 46). Becomes `pipeline.instance`.
- **Strategy name** — `Name` on each options object (defaults like `"Retry"`, `"Timeout"`). Becomes `strategy.name` — and without it you can't tell the total timeout's events from the attempt timeout's.
- **Operation key** — per call site, on the context (Concept 5). Becomes `operation.key`.

**The interview-grade sentence:** *"A pipeline can hold two timeouts, two limiters, even two retries — but nested retries multiply attempts, and retry plus hedging on one call is always wrong. I use typed pipelines when results carry failures, share an untyped pipeline only when I want its state shared, and name every pipeline and strategy so telemetry can tell the total timeout from the attempt timeout."*

---

## Concept 37 — Budget arithmetic: validating a composition before production does

A resilience configuration is a set of numbers that must satisfy inequalities. Write them down and check them — ideally in code, at startup.

| Constraint | Why |
|---|---|
| **Attempt timeout ≥ dependency p99 (or p99.9) + margin** | Chosen false-timeout rate (Module 13, Concept 11) |
| **Total timeout ≥ (retries + 1) × attempt timeout + Σ delays** — or deliberately smaller | Otherwise the last retries can never complete; if truncation is intended ("retry only while time remains"), make that an explicit decision |
| **Total timeout ≤ caller's budget − response margin** | The total is the caller's deadline, not yours to extend |
| **Breaker `SamplingDuration` ≥ 2 × attempt timeout** | The window must be able to contain completed slow attempts |
| **Breaker `MinimumThroughput` < calls per window at lowest meaningful traffic** | Or the breaker can't evaluate (Concept 22) |
| **Bulkhead permits ≥ peak RPS × p99 latency × headroom** | Little's Law; otherwise the bulkhead causes its own rejections |
| **`HttpClient.Timeout` ≥ total timeout** | Otherwise `HttpClient.Timeout` silently becomes the budget (Concept 41) |
| **Downstream totals < upstream totals** | Deadlines shrink as you go down the graph |

`Microsoft.Extensions.Http.Resilience` validates two of these for the standard handler: the circuit breaker's **sampling duration must be at least twice the attempt timeout**, and the **total request timeout must exceed the attempt timeout**. For your own pipelines, encode the rest with an options validator:

```csharp
public sealed class DependencyBudget
{
    public TimeSpan Total { get; set; }
    public TimeSpan Attempt { get; set; }
    public int MaxRetries { get; set; }
    public TimeSpan BaseDelay { get; set; }
    public TimeSpan BreakerWindow { get; set; }
}

public sealed class DependencyBudgetValidator : IValidateOptions<DependencyBudget>
{
    public ValidateOptionsResult Validate(string? name, DependencyBudget b)
    {
        var failures = new List<string>();

        TimeSpan delays = TimeSpan.Zero;                                   // exponential, pre-jitter bound
        for (int i = 0; i < b.MaxRetries; i++) delays += b.BaseDelay * Math.Pow(2, i);
        TimeSpan worstCase = b.Attempt * (b.MaxRetries + 1) + delays;

        if (b.Total <= b.Attempt)
            failures.Add($"{name}: total timeout must exceed the attempt timeout.");
        if (worstCase > b.Total)
            failures.Add($"{name}: {b.MaxRetries} retries need up to {worstCase}, but the total is {b.Total}.");
        if (b.BreakerWindow < 2 * b.Attempt)
            failures.Add($"{name}: breaker window must be at least 2 × the attempt timeout.");

        return failures.Count == 0 ? ValidateOptionsResult.Success : ValidateOptionsResult.Fail(failures);
    }
}

builder.Services.AddOptions<DependencyBudget>("pricing")
    .Bind(builder.Configuration.GetSection("Resilience:Pricing"))
    .ValidateOnStart();                                                     // fail the deployment, not the request
builder.Services.AddSingleton<IValidateOptions<DependencyBudget>, DependencyBudgetValidator>();
```

`ValidateOnStart()` makes a bad configuration fail the **deployment** — ideally in the canary — instead of failing the first request after a config push (Module 13, Concept 8: configuration changes are a leading cause of outages).

**The interview-grade sentence:** *"A resilience config is a set of inequalities: attempt timeout above the p99, total covering attempts plus delays but within the caller's budget, breaker window at least twice the attempt timeout, minimum throughput below quiet-hour traffic, HttpClient.Timeout above the total. I encode them in an options validator with ValidateOnStart so a bad config fails the canary, not production."*

---

# Part I — HttpClient: `Microsoft.Extensions.Http.Resilience` in depth

## Concept 38 — Where the resilience handler sits in the `HttpClient` pipeline

`IHttpClientFactory` builds each client's handler chain from the handlers you register, **in registration order, outermost first**, ending in the primary handler (`SocketsHttpHandler`). The resilience extensions add a `DelegatingHandler` — **`ResilienceHandler`** — at the point where you call them. Therefore:

- **Handlers registered before** the resilience handler run **once per logical call**.
- **Handlers registered after** it run **once per attempt** — the retry re-executes everything below it.

That ordering is a design decision:

```csharp
builder.Services.AddTransient<CorrelationHandler>();
builder.Services.AddTransient<AccessTokenHandler>();

builder.Services.AddHttpClient<PaymentsClient>(c => c.BaseAddress = new("https://payments.internal"))
    .AddHttpMessageHandler<CorrelationHandler>()        // once per call: correlation, logical-call logging
    .AddResilienceHandler("payments", ConfigurePayments)// retries and hedges re-run everything below
    .AddHttpMessageHandler<AccessTokenHandler>();       // per attempt: a token that expired mid-retry is refreshed
```

| Concern | Per call (outside) or per attempt (inside)? |
|---|---|
| Correlation ID, logical-call span | Outside |
| Access-token acquisition | Inside — a retry after 401-due-to-expiry needs a fresh token |
| Per-attempt headers (`x-attempt-number`) | Inside |
| Request signing with a timestamp | Inside — the signature must be fresh per send |
| **Idempotency key** | **Not in a handler at all** — derive it from the business operation in the typed client or the caller, so a user's "try again" reuses it too (Concept 42) |

**Tracing falls out naturally.** Since .NET 8, `SocketsHttpHandler` emits the HTTP client span and metrics at the bottom of the chain, so **each attempt shows up as its own child span** in a distributed trace — two timeouts and a success look like three HTTP spans under your operation. Add your own `Activity` in the adapter for the logical call if you want a parent that spans the attempts.

**The interview-grade sentence:** *"The resilience handler is a DelegatingHandler in the factory chain, so handlers registered before it run once per call and handlers after it run per attempt — correlation outside, token acquisition and request signing inside. Each attempt reaches SocketsHttpHandler separately, so every retry is its own span in the trace."*

---

## Concept 39 — The standard resilience handler, precisely

`AddStandardResilienceHandler()` adds five strategies, outermost to innermost, with these defaults (as documented for the 10.x packages):

| Order | Strategy | Defaults |
|---|---|---|
| 1 | **Rate limiter** (concurrency) | 1,000 permits, queue 0 |
| 2 | **Total request timeout** | 30 s |
| 3 | **Retry** | 3 retries, **exponential**, **jitter on**, 2 s base delay; honors `Retry-After` |
| 4 | **Circuit breaker** | Failure ratio 10%, minimum throughput 100, sampling 30 s, break 5 s |
| 5 | **Attempt timeout** | 10 s |

Both the retry and the breaker handle the same outcomes by default: **`HttpRequestException`**, **`TimeoutRejectedException`**, **HTTP 408**, **HTTP 429**, and **HTTP 5xx**. And **retries apply to every HTTP method** unless you disable them.

Read critically, as an interviewer will expect:

- **30 s total / 10 s attempt** are generous "won't break anything" values, not engineered ones — most internal dependencies deserve sub-second attempts (Concept 40).
- **A 2 s base delay** means the first retry typically waits on the order of a second or more — long for a request path.
- **The breaker counts 429s.** Throttling opens the circuit and blocks requests the server would have accepted a moment later — contrary to Module 13's advice (Concept 22). Override the breaker's predicate for throttled dependencies.
- **All methods retried** includes non-idempotent `POST`s — the duplicate-charge bug (Concept 42).
- **Minimum throughput 100 in 30 s** means no functioning breaker below ~3.3 RPS (Concept 17).

**Configuring it.** Three equivalent styles:

```csharp
// 1. Inline
http.AddStandardResilienceHandler(o => o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(1));

// 2. Fluent Configure (also used by global defaults)
http.AddStandardResilienceHandler().Configure(o => o.CircuitBreaker.MinimumThroughput = 20);

// 3. Bound from configuration — the same HttpStandardResilienceOptions shape
http.AddStandardResilienceHandler(builder.Configuration.GetSection("Resilience:Catalog"));
```

The options object, `HttpStandardResilienceOptions`, has one property per strategy: `RateLimiter`, `TotalRequestTimeout`, `Retry`, `CircuitBreaker`, `AttemptTimeout`. Its validation enforces the two inequalities from Concept 37 when the pipeline is built.

**The interview-grade sentence:** *"The standard handler is a 1,000-permit concurrency limiter, a 30-second total, three jittered exponential retries from two seconds that honor Retry-After, a 10%-over-30-seconds breaker with a minimum of 100, and a 10-second attempt timeout — retrying and counting 408, 429, 5xx, transport errors and timeouts on every method. It's a safe default, not a tuned one: the timeouts are loose, the breaker counts 429s, and POSTs are retried."*

---

## Concept 40 — Tuning the standard handler per dependency

A tuned configuration for an internal payments API — sub-second p99, non-idempotent `POST`s that accept an `Idempotency-Key`, and a provider that throttles:

```csharp
builder.Services.AddHttpClient<PaymentsClient>(c =>
    {
        c.BaseAddress = new("https://payments.internal");
        c.Timeout = TimeSpan.FromSeconds(5);                         // backstop above the pipeline total (Concept 41)
    })
    .AddStandardResilienceHandler(o =>
    {
        o.RateLimiter.DefaultRateLimiterOptions.PermitLimit = 100;   // peak 250 RPS × 0.3 s p99 ≈ 75, plus headroom

        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(3);
        o.AttemptTimeout.Timeout      = TimeSpan.FromSeconds(1);     // p99.9 ≈ 700 ms + margin

        o.Retry.MaxRetryAttempts = 1;
        o.Retry.Delay            = TimeSpan.FromMilliseconds(200);
        // Every write this typed client sends carries an Idempotency-Key that the provider deduplicates on
        // (enforced inside PaymentsClient — Concept 42), so retrying POST is safe FOR THIS CLIENT.
        // A client whose writes are not keyed calls o.Retry.DisableForUnsafeHttpMethods() instead.
        o.Retry.ShouldHandle = static args => TransientHttp.IsTransient(args.Outcome);

        o.CircuitBreaker.FailureRatio      = 0.5;
        o.CircuitBreaker.MinimumThroughput = 20;
        o.CircuitBreaker.SamplingDuration  = TimeSpan.FromSeconds(10);            // ≥ 2 × attempt timeout
        o.CircuitBreaker.ShouldHandle = static args => TransientHttp.IsBreakerFailure(args.Outcome); // not 429
    });
```

Notes on the pieces:

- **Retry policy is decided per client, not per request.** A predicate can see the request only indirectly — for result outcomes through `args.Outcome.Result?.RequestMessage`, while exception outcomes (timeouts, connection failures) carry no request at all. So "may this call be repeated?" is simplest to answer at the client level: a client whose writes are *all* idempotency-keyed may retry every method; a client with unkeyed writes disables retries for them.
- **`DisableForUnsafeHttpMethods()` / `DisableFor(...)`** are the built-in tools for "no retries for POST/PATCH/PUT/DELETE/CONNECT" (they know the request method for you). They work by wrapping the retry predicate, so set any custom `ShouldHandle` first and call them afterward — and verify the result with a behavior test (Concept 54). If you want the narrower rule "retry unkeyed writes only when the request never left the machine," give those calls a dedicated client whose predicate is `TransientHttp.NeverSent` (Concept 13).
- **One retry** for a user-facing write with a 3 s budget; the remaining protection comes from the breaker, the bulkhead and a clear failure to the caller.
- **A backstop `HttpClient.Timeout`** above the pipeline's total so it never becomes the effective budget.

Put the numbers in configuration (bound options) rather than code once they're stable, and keep the reasons next to them (a comment, an ADR) — "attempt 1 s because p99.9 is 700 ms" is what lets the next engineer change it safely.

**The interview-grade sentence:** *"Per dependency I tighten the standard handler: attempt timeout from p99.9, a total inside the caller's budget, one retry for user-facing writes, and a breaker that ignores 429s with a window at least twice the attempt timeout. Whether writes may be retried is a per-client decision — all writes idempotency-keyed, or DisableForUnsafeHttpMethods — because exception outcomes don't carry the request."*

---

## Concept 41 — `HttpClient.Timeout` vs the pipeline's timeouts

`HttpClient.Timeout` (default **100 s**) wraps the **entire** `SendAsync` — every handler in the chain, including the resilience handler and all its attempts. When it fires, `HttpClient` cancels the token it passed into the chain. From the pipeline's point of view, that's **a cancellation from above** — indistinguishable from the caller giving up:

- the retry doesn't retry it,
- the breaker doesn't count it,
- the caller gets a `TaskCanceledException` (with a `TimeoutException` inner exception), not a `TimeoutRejectedException`.

So the **effective budget is `min(HttpClient.Timeout, TotalRequestTimeout)`**. If someone sets `HttpClient.Timeout = 2 s` "for safety" on a client whose pipeline has a 3 s total and two retries, the third attempt is killed by the client, silently, and the breaker never learns about it.

**Rule:** make the **pipeline's total timeout the governing budget** and set `HttpClient.Timeout` a little *above* it as a backstop (or to `Timeout.InfiniteTimeSpan` when the pipeline always has a total timeout).

**A subtlety about response bodies.** A handler's `SendAsync` completes when **response headers** arrive. With the default `HttpCompletionOption.ResponseContentRead`, `HttpClient` buffers the body **after** the handler chain has returned. That means:

- the resilience pipeline's **attempt and total timeouts don't cover the body download** — only `HttpClient.Timeout` and your cancellation token do;
- a connection reset **halfway through the body** happens outside the pipeline and **isn't retried**.

For small JSON responses this rarely matters. For large or slow bodies, decide deliberately: stream with `ResponseHeadersRead` and your own token and timeout, or execute an explicit pipeline in the typed client *around* the whole "send and deserialize" operation when a body-read failure should be retried — keeping in mind Polly's advice to use separate pipelines for separate failure domains (Concept 16).

**The interview-grade sentence:** *"HttpClient.Timeout wraps the whole handler chain, so when it fires the pipeline sees it as caller cancellation — no retry, no breaker count — and the effective budget becomes the smaller of the two. I make the pipeline's total timeout the governing budget and set HttpClient.Timeout just above it, and I remember the handler's timeouts end at response headers, not at the end of the body."*

---

## Concept 42 — Replaying requests: bodies, streams, and idempotency keys

A retry inside the handler chain **re-sends the same `HttpRequestMessage`** through the inner handlers. That's legal at the handler level, but only works if everything about the request can be sent twice:

| Request content | Replayable? |
|---|---|
| `StringContent`, `ByteArrayContent`, `FormUrlEncodedContent` | Yes — the bytes are in memory |
| `JsonContent` | Yes — it serializes the object again on each send |
| `StreamContent` over a seekable `MemoryStream`/`FileStream` | Usually — the stream must be rewound; verify in a test |
| `StreamContent` over a **non-seekable** stream (a network stream, a request body being proxied) | **No** — the second attempt fails or sends an empty body |
| Large uploads | Technically maybe, practically no — use a resumable protocol (e.g., block uploads through the Azure Blob SDK) |

**Hedging** is stricter still: concurrent attempts can't share one request, so the hedging handler **snapshots** the request (including content) and sends clones — the content must be bufferable.

**Headers added per attempt.** Inner handlers run once per attempt against the *same* message. A handler that does `request.Headers.Add("x-attempt", ...)` accumulates values across attempts; a signing handler that appends a signature header ends up with two. Inner handlers should **replace** headers (`Remove` then `Add`, or set typed headers like `Authorization`) rather than append.

**Idempotency keys belong to the operation, not the attempt.** Module 13 (Concept 20) covered the server side. On the client:

- Generate the key **once per logical business operation** — ideally derived from something stable, like the order ID and the operation name, or supplied by the caller — **above** the retry, so every attempt carries the same key.
- Put it in the typed client when you build the request, not in a delegating handler that generates a new GUID per call — otherwise a user who clicks "Pay" again after a timeout gets a new key and a second charge.
- Derive **downstream** keys deterministically from yours (Module 13, Concept 20), so a retry of your operation produces the same key at the payment provider.

```csharp
public sealed class PaymentsClient(HttpClient http)
{
    public async Task<PaymentResult> CaptureAsync(Guid orderId, Money amount, CancellationToken ct)
    {
        using var request = new HttpRequestMessage(HttpMethod.Post, "/payments")
        {
            Content = JsonContent.Create(new { orderId, amount })       // replayable content
        };
        // Stable per business operation: a user retry after a timeout reuses it too.
        request.Headers.Add("Idempotency-Key", $"capture:{orderId:N}");

        using HttpResponseMessage response = await http.SendAsync(request, ct);   // pipeline retries reuse the key
        // ...
    }
}
```

**The interview-grade sentence:** *"A handler-level retry re-sends the same request message, so the content must be replayable — buffered, JSON, or a rewindable stream — and inner handlers must replace headers, not append them. Idempotency keys are generated once per business operation in the typed client, derived from stable IDs, so every attempt and even a user's retry carries the same key."*

---

## Concept 43 — Custom handlers, per-authority pipelines, and global defaults

**`AddResilienceHandler(name, configure)`** gives you the full Polly builder for `HttpResponseMessage`, with HTTP-aware options types whose default predicates already understand transient HTTP outcomes: **`HttpRetryStrategyOptions`**, **`HttpCircuitBreakerStrategyOptions`**, **`HttpTimeoutStrategyOptions`**, **`HttpRateLimiterStrategyOptions`**. The `(builder, context)` overload exposes **`ResilienceHandlerContext`**: `ServiceProvider`, `GetOptions<T>(name)`, `EnableReloads<T>(name)` and `OnPipelineDisposed(...)` (Concept 47).

```csharp
builder.Services.AddHttpClient<RegionalCatalogClient>()
    .AddResilienceHandler("catalog", static (pipeline, context) =>
    {
        context.EnableReloads<HttpRetryStrategyOptions>("catalog-retry");
        HttpRetryStrategyOptions retry = context.GetOptions<HttpRetryStrategyOptions>("catalog-retry");

        pipeline
            .AddTimeout(TimeSpan.FromSeconds(2))
            .AddRetry(retry)
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions { FailureRatio = 0.5, MinimumThroughput = 20 })
            .AddTimeout(TimeSpan.FromMilliseconds(500));
    })
    .SelectPipelineByAuthority();   // one pipeline — and one breaker — per scheme+host+port
```

**Per-authority selection** (`SelectPipelineByAuthority()`, or `SelectPipelineBy(...)` for custom keys) is what you want when one client talks to several hosts — regional endpoints, a primary and a secondary — so a failing host opens only its own circuit (Concept 21). Polly's docs recommend it over hand-maintaining a dictionary of breakers keyed by URI.

**Global defaults.** `services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler())` adds the standard handler to **every** `HttpClient` created through the factory — which is exactly what Aspire's service-defaults project does. Three consequences to call out:

1. **Every client** gets retries on all methods, a 30 s total, and a 10 s attempt timeout unless you override them — including clients whose owners never thought about resilience.
2. **Library-created clients are included.** Libraries that obtain named clients from `IHttpClientFactory` (token management, OIDC validation, SDK integrations) get the global handler *and* often configure their own — the stacking problem.
3. **Overriding a client means removing, not adding.** Microsoft's guidance is explicit: add **one** resilience handler and don't stack them. To replace the default for a specific client, call **`RemoveAllResilienceHandlers()`** first, then add the one you want (it shipped as an experimental API — check whether your package version still requires suppressing the `EXTEXP0001` diagnostic):

```csharp
builder.Services.ConfigureHttpClientDefaults(static http => http.AddStandardResilienceHandler());

builder.Services.AddHttpClient<PricingClient>()
    .RemoveAllResilienceHandlers()          // drop the global standard handler for this client
    .AddStandardHedgingHandler();           // and use hedging instead
```

**The interview-grade sentence:** *"AddResilienceHandler gives me the full Polly builder with HTTP-aware options, reloadable options through the handler context, and per-authority pipelines so each host has its own breaker. Global defaults via ConfigureHttpClientDefaults — which is what Aspire does — apply to every factory client including library ones, so I override per client by removing the default handler first, never by stacking a second one."*

---

## Concept 44 — Static clients and gRPC clients

**Static or singleton `HttpClient`s** don't go through the factory, so there's no `AddResilienceHandler`. Build the chain yourself with **`ResilienceHandler`** over a `SocketsHttpHandler` — and remember the DNS rule from Module 13 (Concept 12): a long-lived client needs `PooledConnectionLifetime` or it never sees a DNS-based failover.

```csharp
public static class LegacyGateway
{
    // One client and one pipeline for the life of the process: breaker state persists, sockets are reused.
    public static readonly HttpClient Client = Create();

    private static HttpClient Create()
    {
        ResiliencePipeline<HttpResponseMessage> pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
            .AddTimeout(TimeSpan.FromSeconds(2))                                   // total
            .AddRetry(new HttpRetryStrategyOptions { MaxRetryAttempts = 2, Delay = TimeSpan.FromMilliseconds(100) })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions { FailureRatio = 0.5, MinimumThroughput = 20 })
            .AddTimeout(TimeSpan.FromMilliseconds(500))                            // attempt
            .Build();

        var handler = new ResilienceHandler(pipeline)
        {
            InnerHandler = new SocketsHttpHandler
            {
                PooledConnectionLifetime = TimeSpan.FromMinutes(2),   // re-resolve DNS so failover takes effect
                ConnectTimeout = TimeSpan.FromSeconds(2)
            }
        };

        return new HttpClient(handler) { Timeout = TimeSpan.FromSeconds(3) };     // backstop above the total
    }
}
```

**gRPC clients** raise a subtler question. `Grpc.Net.ClientFactory` lets you add the HTTP resilience handlers to a gRPC client (`AddGrpcClient<T>().AddStandardResilienceHandler()`, supported from `Grpc.Net.ClientFactory` 2.64.0; older versions throw at runtime). But the HTTP handler sees **HTTP semantics**, and gRPC reports most failures as a **`grpc-status`** value in response headers or trailers on an HTTP 200 — so the standard HTTP predicates **don't recognize most gRPC failures** (an `Unavailable` returned by the server looks like a successful HTTP exchange), and retrying streaming calls at the HTTP layer is unsound. gRPC's own retry understands status codes, **commit points** (no retry once response headers arrive or the retry buffer is exceeded), deadlines and server pushback.

So the defensible split for gRPC:

- **Status-aware retries or hedging** → the channel's `RetryPolicy` or `HedgingPolicy` in the service config (Concept 31 — never both on one method).
- **Deadlines** → per-call deadlines, propagated with `EnableCallContextPropagation()` for server-to-server calls.
- **Transport-level protection** (bulkhead, a breaker counting connection failures) → the HTTP handler if you want it, **with its retry disabled** so it doesn't stack with the channel's.

```csharp
builder.Services.AddGrpcClient<Inventory.InventoryClient>(o => o.Address = new("https://inventory.internal"))
    .ConfigureChannel(static o => o.ServiceConfig = new ServiceConfig
    {
        MethodConfigs =
        {
            new MethodConfig
            {
                Names = { MethodName.Default },
                RetryPolicy = new RetryPolicy
                {
                    MaxAttempts = 3,                                   // includes the original call
                    InitialBackoff = TimeSpan.FromMilliseconds(100),
                    MaxBackoff = TimeSpan.FromSeconds(1),
                    BackoffMultiplier = 2,
                    RetryableStatusCodes = { StatusCode.Unavailable }
                }
            }
        }
    })
    .EnableCallContextPropagation();                                   // flow deadline and cancellation downstream
```

(If Aspire's global defaults add the standard HTTP handler to this client, remove it or disable its retry — otherwise one `Unavailable` can be retried by the channel and a transport blip by the handler, and the attempt counts multiply.)

**The interview-grade sentence:** *"For a static HttpClient I compose ResilienceHandler over a SocketsHttpHandler with a pooled connection lifetime. For gRPC I let the channel's retry or hedging policy own status-aware retries, because the HTTP handler only sees HTTP 200 plus a grpc-status and can't respect gRPC's commit points — the HTTP handler, if present, keeps only transport-level protection with its retry off."*

---

# Part J — Registry, DI, and configuration

## Concept 45 — The registry and provider: build once, resolve many

Concept 2's rule — build once, because state lives in the instance — needs a mechanism in a real application. That mechanism is the **registry**:

- **`ResiliencePipelineRegistry<TKey>`** holds **builder callbacks** keyed by `TKey`, builds a pipeline **lazily on first request**, and **caches** it. It's thread-safe and **append-only** — you can't remove or replace a registered pipeline, which avoids races between readers and writers.
- **`ResiliencePipelineProvider<TKey>`** is the read-only view you inject into application code.

`services.AddResiliencePipeline(...)` (from `Polly.Extensions`) registers the builder callback, the registry, the provider, and options — and turns on telemetry automatically. Since Polly 8.3, pipelines are also available as **keyed services**:

```csharp
// Registration — composition root
builder.Services.AddResiliencePipeline<string, Quote>("quotes", static (pipeline, context) =>
{
    pipeline
        .AddTimeout(TimeSpan.FromSeconds(1))
        .AddRetry(new RetryStrategyOptions<Quote> { MaxRetryAttempts = 1, ShouldHandle = QuoteFaults.IsTransient })
        .AddTimeout(TimeSpan.FromMilliseconds(400));
});

// Consumption — option 1: the provider, resolved once in the constructor
public sealed class QuoteAdapter(ResiliencePipelineProvider<string> pipelines, QuoteGrpcClient client)
{
    private readonly ResiliencePipeline<Quote> _pipeline = pipelines.GetPipeline<Quote>("quotes");
    // ...
}

// Consumption — option 2: keyed injection
public sealed class QuoteAdapter2([FromKeyedServices("quotes")] ResiliencePipeline<Quote> pipeline) { /* ... */ }
```

Details worth knowing:

- **Keyed pipelines are registered as transient services, but they're resolved from the provider's cache** — you get the same pipeline instance every time, so breaker state is shared as intended. (Transient registration exists so complex keys can resolve different instances — Concept 46.)
- **`GetPipeline` throws if the key is unknown; `TryGetPipeline` doesn't.** Prefer resolving in constructors so a missing registration fails at startup, not mid-request.
- **Deferred registration** — `AddResiliencePipelines(ctx => ...)` — lets you register pipelines from configuration *after* the container is built (loop over a config section and call `ctx.AddResiliencePipeline(...)` for each). Those pipelines aren't available as keyed services.
- **Without DI**, create a `ResiliencePipelineRegistry<string>` yourself and use `TryAddBuilder` / `GetOrAddPipeline`.
- **Disposal**: when the container is disposed, the registry disposes its pipelines and runs any `OnPipelineDisposed` callbacks (Concept 47).
- **Anti-pattern from Polly's docs:** capturing `IServiceCollection` inside a pipeline callback and calling `BuildServiceProvider()` in `OnRetry` — use the `(builder, context)` overload's `context.ServiceProvider` instead.

**The interview-grade sentence:** *"The registry is what makes 'build once' real: it holds builder callbacks, builds each pipeline lazily on first use, caches it, and never replaces it except through a reload. I resolve pipelines in constructors — via the provider or keyed services, which return the cached instance — so a missing key fails at startup."*

---

## Concept 46 — Complex keys, and the unbounded-key trap

Sometimes you want **one pipeline definition, many instances** — a breaker per database shard, a limiter per partner API, a pipeline per region. The registry supports **complex keys** so the builder is chosen by part of the key while each full key gets its own cached instance:

```csharp
public readonly record struct ShardPipelineKey(string PipelineName, string Shard);

public sealed class ByPipelineName : IEqualityComparer<ShardPipelineKey>
{
    public bool Equals(ShardPipelineKey x, ShardPipelineKey y) => x.PipelineName == y.PipelineName;
    public int GetHashCode(ShardPipelineKey key) => key.PipelineName.GetHashCode(StringComparison.Ordinal);
}

builder.Services.AddResiliencePipelineRegistry<ShardPipelineKey>(o =>
{
    o.BuilderComparer       = new ByPipelineName();          // choose the BUILDER by pipeline name only
    o.BuilderNameFormatter  = static k => k.PipelineName;    // pipeline.name tag
    o.InstanceNameFormatter = static k => k.Shard;           // pipeline.instance tag
});

builder.Services.AddResiliencePipeline(new ShardPipelineKey("orders-db", string.Empty), static pipeline =>
{
    // Stateful strategies: each shard key gets its own instance, so one hot shard opens only its own circuit.
    pipeline.AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5, MinimumThroughput = 20 })
            .AddTimeout(TimeSpan.FromMilliseconds(300));
});

// At the call site:
ResiliencePipeline pipeline = provider.GetPipeline(new ShardPipelineKey("orders-db", shardId));
```

**The trap: the registry is append-only.** Every distinct key materializes a pipeline — breaker window, limiter, options — that lives **for the life of the process**, and every distinct instance name becomes a distinct telemetry series. Keys drawn from a small, known set (shards, regions, partners) are fine. Keys drawn from an unbounded set (tenant IDs, user IDs, URLs with IDs in them) are a slow memory leak plus a metrics-cardinality explosion.

Mitigations when the natural key is unbounded:

- **Bucket it**: `hash(tenantId) % 64` gives 64 breakers with bounded blast radius — shuffle sharding's idea applied to pipelines (Module 13, Concept 34).
- **Move per-tenant concerns to limiters, not breakers**: a `PartitionedRateLimiter` keyed by tenant manages its partitions and doesn't need a pipeline per tenant (Concept 25).
- **Push isolation to the edge**: per-tenant limits in the gateway or ASP.NET Core middleware.

**The interview-grade sentence:** *"Complex registry keys give one pipeline definition many instances — a breaker per shard, say — with the builder chosen by part of the key and the rest used as the instance name. But the registry is append-only, so I only key by small known sets; for tenants I bucket the key or use a partitioned limiter instead of a pipeline per tenant."*

---

## Concept 47 — Dynamic reloads, and what they do to breaker state

Polly can **rebuild a pipeline when its options change**, so you can tune timeouts and thresholds from configuration without redeploying. In a pipeline callback:

```csharp
builder.Services.Configure<RetryStrategyOptions>("inventory-retry",
    builder.Configuration.GetSection("Resilience:Inventory:Retry"));

builder.Services.AddResiliencePipeline("inventory", static (pipeline, context) =>
{
    context.EnableReloads<RetryStrategyOptions>("inventory-retry");            // rebuild when these options change
    RetryStrategyOptions retry = context.GetOptions<RetryStrategyOptions>("inventory-retry");

    var bulkhead = new ConcurrencyLimiter(new ConcurrencyLimiterOptions { PermitLimit = 64, QueueLimit = 0 });
    context.OnPipelineDisposed(() => bulkhead.Dispose());                       // old limiter disposed on reload

    pipeline
        .AddRateLimiter(bulkhead)
        .AddTimeout(TimeSpan.FromSeconds(1))
        .AddRetry(retry)
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5, MinimumThroughput = 20 })
        .AddTimeout(TimeSpan.FromMilliseconds(300));
});
```

The HTTP handlers expose the same through `ResilienceHandlerContext.EnableReloads<T>(name)` (Concept 43), and Polly 8.8 added **`EnableReloadsWithMonitor()`** for options that come from a custom `IOptionsMonitor` source.

What a reload actually does — each point is a production behavior to know:

1. **The callback re-runs and a new pipeline replaces the old one.** New executions use the new pipeline; the old one is discarded and its `OnPipelineDisposed` callbacks run (which is why limiters you create should be disposed there).
2. **All strategy state starts over.** The new breaker is **closed with an empty window**; the new concurrency limiter has **all permits free**. Reloading options *during an incident* therefore **closes an open circuit**, and for a moment effective concurrency can **exceed the bulkhead limit** (calls that started on the old pipeline are still running while new calls get fresh permits). Don't hot-tune a breaker in the middle of the outage it's protecting you from — or do it knowing exactly this.
3. **If the rebuild throws, the old pipeline stays — and reloading stops.** Per Polly's docs, an error during reload leaves the previous pipeline in place and disables further reloads. A single bad config push can silently freeze your resilience configuration until the next restart. Validate options before they reach the pipeline (Concept 48) and alert on reload failures in logs.

**The interview-grade sentence:** *"Dynamic reload rebuilds the pipeline from changed options and new calls use the new instance. But every stateful strategy starts over — a reload closes an open breaker and resets the bulkhead — and a reload that throws keeps the old pipeline and stops reloading altogether, so resilience config gets validated before it's pushed, like code."*

---

## Concept 48 — Options validation and configuration governance

**Polly validates its own options.** Every options class carries validation attributes (ranges on durations, ratios and counts), and the builder validates when the pipeline is built, throwing a `ValidationException` for invalid values. Your custom strategy options get the same treatment if you annotate them (Concept 57). The HTTP standard handler adds cross-field validation (Concept 37).

**But "when the pipeline is built" is usually "first use."** Registry pipelines build lazily; an invalid value surfaces on the first request after a deployment or config push — in production, under load. Pull the failure forward:

```csharp
// 1. Validate your own budget options at startup (Concept 37).
builder.Services.AddOptions<DependencyBudget>("pricing")
    .Bind(builder.Configuration.GetSection("Resilience:Pricing"))
    .ValidateOnStart();

// 2. Build the critical registry pipelines during startup so Polly's own validation runs before traffic.
public sealed class ResilienceWarmup(ResiliencePipelineProvider<string> pipelines) : IHostedService
{
    private static readonly string[] Critical = ["pricing", "quotes", "inventory"];

    public Task StartAsync(CancellationToken ct)
    {
        foreach (string key in Critical) _ = pipelines.GetPipeline(key);     // throws now, not on request #1
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

For HTTP resilience handlers, whose pipelines are built on the client's first request, cover the same ground with a **host-level integration test** that sends one request through each registered client (Concept 56), and with a startup/readiness check for the critical ones.

**Governance — treat resilience config as production change** (Module 13, Concept 54):

- **In source control and reviewed**, with the reason for each number next to it ("attempt 1 s because p99.9 is 700 ms").
- **Staged like code** — canary first — especially for reloadable options, which otherwise reach every instance at once.
- **Few environment overrides.** A timeout that differs silently between staging and production invalidates every test you ran in staging.
- **Kill switches as configuration** — breaker isolation (Concept 20), chaos enablement (Concept 55), hedging off-switches (Concept 28) — with an owner and a runbook.

**The interview-grade sentence:** *"Polly validates options when a pipeline is built, but that's usually the first request, so I pull validation forward: ValidateOnStart for my budget options, a startup warm-up that builds critical pipelines, and an integration test that exercises every HTTP client. Resilience config lives in source control with its reasons, and it's staged through canaries like code."*

---

# Part K — Observability

## Concept 49 — Polly's telemetry: meter, instruments, tags, logs

Module 13 (Concept 56) established *what* to observe — because resilience mechanisms hide failures by design. Polly provides most of it out of the box. Telemetry is enabled automatically for pipelines registered through `AddResiliencePipeline` and for the HTTP resilience handlers; for hand-built pipelines, call **`ConfigureTelemetry(loggerFactory)`** or `ConfigureTelemetry(telemetryOptions)` on the builder.

**Metrics** are emitted on the **`Polly`** meter:

| Instrument | Type | What it measures | Tags beyond the common ones |
|---|---|---|---|
| **`resilience.polly.strategy.events`** | Counter | Every resilience event | `event.name`, `event.severity`, `exception.type` |
| **`resilience.polly.strategy.attempt.duration`** | Histogram (ms) | Each attempt made by **retry** and **hedging** | `attempt.number` (0 = original), `attempt.handled` (was it a failure) |
| **`resilience.polly.pipeline.duration`** | Histogram (ms) | Whole-pipeline executions | `exception.type` |

Common tags: **`pipeline.name`**, **`pipeline.instance`**, **`strategy.name`**, **`operation.key`** (Concept 36).

**Event names** you'll query on: `OnRetry`, `OnTimeout`, `OnCircuitOpened`, `OnCircuitHalfOpened`, `OnCircuitClosed`, `OnRateLimiterRejected`, `OnHedging`, `OnFallback`, and the chaos events `Chaos.OnFault`, `Chaos.OnLatency`, `Chaos.OnOutcome`, `Chaos.OnBehavior`.

**Logs** go to the **`Polly`** logger category, with stable event IDs: **0** "resilience event occurred", **1** "pipeline executing" (Debug by default), **2** "pipeline executed" (Information by default), **3** "execution attempt".

Wiring it into OpenTelemetry:

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(static m => m
        .AddMeter("Polly")                                  // Polly's instruments
        .AddHttpClientInstrumentation()
        .AddAspNetCoreInstrumentation())
    .WithTracing(static t => t
        .AddHttpClientInstrumentation()                     // each attempt is its own HTTP span (Concept 38)
        .AddAspNetCoreInstrumentation());
```

**Traces.** Polly records metrics and logs, not its own spans. For HTTP, the per-attempt client spans from `System.Net.Http` make retries visible in a trace; for non-HTTP dependencies, create an `Activity` in the adapter around the logical call, and — if you want attempts as child spans — around the callback body.

**The interview-grade sentence:** *"Polly emits metrics on the 'Polly' meter — a strategy-events counter, a per-attempt duration histogram for retry and hedging with attempt number and handled tags, and a pipeline duration histogram — plus logs under the 'Polly' category. I add the meter to OpenTelemetry, rely on HTTP client spans to show each attempt in traces, and add my own activity for non-HTTP calls."*

---

## Concept 50 — Enrichment, severity, and `Microsoft.Extensions.Resilience`

Three knobs shape Polly's telemetry — configure them once, centrally, through `TelemetryOptions`:

```csharp
builder.Services.Configure<TelemetryOptions>(static o =>
{
    // 1. Severity: keep the signal, drop the noise.
    o.SeverityProvider = static args => args.Event.EventName switch
    {
        "PipelineExecuted" => ResilienceEventSeverity.Debug,     // one log line per call is too much in production
        "ExecutionAttempt" when args.Event.Severity == ResilienceEventSeverity.Information
                           => ResilienceEventSeverity.Debug,     // successful attempts
        _                  => args.Event.Severity                // retries, timeouts, breaker transitions stay loud
    };

    // 2. Enrichment: add bounded, useful dimensions.
    o.MeteringEnrichers.Add(new DependencyTierEnricher());
});

internal sealed class DependencyTierEnricher : MeteringEnricher
{
    public override void Enrich<TResult, TArgs>(in EnrichmentContext<TResult, TArgs> context)
    {
        // Bounded values only: "hard" | "soft" | "async" (Module 13, Concept 6) — never IDs.
        string tier = context.TelemetryEvent.Context.Properties.GetValue(ResilienceKeys.Tier, "unknown");
        context.Tags.Add(new("dependency.tier", tier));
    }
}
```

(`ResilienceKeys.Tier` is a `ResiliencePropertyKey<string>` your adapter sets on the context. Setting `ResilienceEventSeverity.None` suppresses an event entirely; a `TelemetryListener` can be added to `TelemetryListeners` for custom sinks.)

**`Microsoft.Extensions.Resilience`** adds Microsoft's enrichment on top of Polly:

- **`services.AddResilienceEnricher()`** registers an enricher that adds **request metadata** (dependency name, request name, route) and — if an `IExceptionSummarizer` from `Microsoft.Extensions.Diagnostics.ExceptionSummarization` is registered — **short, low-cardinality exception summaries** to Polly's metrics.
- **`ResilienceContext.SetRequestMetadata(...)`** / **`GetRequestMetadata()`** attach that metadata per execution.

**Cardinality is the constraint.** Every tag multiplies the number of time series. `operation.key` must be a call-site name, not a URL with IDs; `exception.type` is fine because it's bounded by your code; tenant IDs, order IDs and raw exception messages never become tags. Polly's own docs warn about unbounded tag values — heed it (Concept 5, Concept 46).

**The interview-grade sentence:** *"I configure Polly's telemetry centrally: a SeverityProvider that demotes per-call 'executed' and successful-attempt logs to Debug while keeping retries, timeouts and breaker transitions loud, and metering enrichers that add bounded dimensions like dependency tier. Microsoft.Extensions.Resilience adds request metadata and exception summaries — and nothing unbounded ever becomes a tag."*

---

## Concept 51 — From metrics to decisions: dashboards and alerts

Turning Polly's instruments into the Module 13 signals:

| Signal | How to compute it from Polly telemetry | What it tells you |
|---|---|---|
| **Retry ratio** | `strategy.events{event.name="OnRetry"}` ÷ `pipeline.duration` count, per pipeline | Early warning of dependency degradation; also your amplification factor |
| **Hidden failure rate** | Fraction of successful executions whose winning attempt had `attempt.number ≥ 1` | How much the retries are masking |
| **Attempt vs total timeouts** | `OnTimeout` split by `strategy.name` (`"attempt"` vs `"total"`) | Attempt timeouts = the dependency is slow; total timeouts = *users* saw failures |
| **Breaker transitions** | `OnCircuitOpened` / `OnCircuitClosed` counts | An open circuit on a hard dependency is an incident |
| **Bulkhead vs quota rejections** | `OnRateLimiterRejected` split by `strategy.name` | Mis-sized bulkhead, overload, or an exhausted provider quota |
| **Hedge rate and win rate** | `OnHedging` ÷ executions; share of successes won by `attempt.number ≥ 1` | Is hedging costing ~5% and actually winning? |
| **Degraded responses** | `OnFallback` ÷ executions | How often users get the reduced experience |
| **Time spent in resilience** | p99 of `pipeline.duration` minus p99 of `attempt.duration` | Delays and repeated attempts, as latency |
| **Retry-budget refusals** | Your own counter (Concept 15) | Budget exhaustion = sustained failure |

In PromQL (names as produced by a typical OpenTelemetry → Prometheus exporter, which turns dots into underscores and appends unit and `_total` suffixes — adjust for yours):

```promql
# Retry ratio per pipeline over 5 minutes
sum by (pipeline_name) (rate(resilience_polly_strategy_events_total{event_name="OnRetry"}[5m]))
/
sum by (pipeline_name) (rate(resilience_polly_pipeline_duration_milliseconds_count[5m]))

# Circuit openings in the last 10 minutes, per pipeline
sum by (pipeline_name) (increase(resilience_polly_strategy_events_total{event_name="OnCircuitOpened"}[10m]))
```

**Alerting policy** (Module 13, Concept 56): **page on user-facing SLO burn**; **page on a sustained open circuit for a hard dependency**; **ticket on anomalies** in retry ratio, hedge rate, and rejection rate. Mechanism metrics are for early warning and diagnosis — a retry ratio climbing from 0.5% to 8% over an hour is the dependency telling you about tonight's outage while your error rate still reads zero.

**The interview-grade sentence:** *"From Polly's metrics I build retry ratio, hidden failure rate from the attempt number of successful executions, attempt versus total timeouts by strategy name, breaker transitions, bulkhead versus quota rejections, hedge rate and win rate, and fallback rate. I page on SLO burn and on sustained open circuits for hard dependencies; the rest are early-warning tickets."*

---

# Part L — Testing and chaos

## Concept 52 — What to test (and what not to)

Polly's docs put it plainly: **don't test how the pipelines operate internally — test your settings and your delegates.** Retry works; you don't need a test proving it. What breaks in production is *your* configuration and *your* code:

| Test type | Tests | Tools | Concept |
|---|---|---|---|
| **Predicate unit tests** | Your classifiers: which outcomes retry, which count as breaker failures | Plain table-driven tests | 52 |
| **Composition tests** | Order and options of the pipeline you registered | `Polly.Testing` descriptors | 53 |
| **Behavior tests** | Attempts, delays, breaker opening, budgets — deterministically | `FakeTimeProvider`, stub handlers | 54 |
| **Host-level integration tests** | The *whole* handler chain as registered (including global defaults) against a scripted fake dependency | `WebApplicationFactory`, WireMock.Net | 56 |
| **Fault-injection experiments** | System behavior under real faults | Polly chaos, Toxiproxy, Azure Chaos Studio | 55 |

And for **application code that merely uses a pipeline**, don't run the real pipeline in unit tests at all: substitute **`ResiliencePipeline.Empty`** (Polly's docs show mocking `ResiliencePipelineProvider<string>` to return it), so the test is about the business logic and runs in milliseconds.

Predicates are pure functions — the cheapest, highest-value tests you'll write:

```csharp
public sealed class TransientHttpTests
{
    public static TheoryData<Outcome<HttpResponseMessage>, bool> Cases => new()
    {
        { Outcome.FromResult(new HttpResponseMessage(HttpStatusCode.ServiceUnavailable)), true  },
        { Outcome.FromResult(new HttpResponseMessage(HttpStatusCode.TooManyRequests)),    true  },
        { Outcome.FromResult(new HttpResponseMessage(HttpStatusCode.InternalServerError)),false },
        { Outcome.FromResult(new HttpResponseMessage(HttpStatusCode.NotFound)),           false },
        { Outcome.FromException<HttpResponseMessage>(new TimeoutRejectedException()),     true  },
        { Outcome.FromException<HttpResponseMessage>(new BrokenCircuitException()),       false },
        { Outcome.FromException<HttpResponseMessage>(new OperationCanceledException()),   false },
    };

    [Theory, MemberData(nameof(Cases))]
    public async Task Retry_classification(Outcome<HttpResponseMessage> outcome, bool expected) =>
        Assert.Equal(expected, await TransientHttp.IsTransient(outcome));
}
```

**The interview-grade sentence:** *"I don't test Polly; I test my configuration and my delegates: table-driven tests for predicates, descriptor tests for composition, deterministic behavior tests with fake time, a host-level test through the real handler chain, and fault injection. Business logic that uses a pipeline gets ResiliencePipeline.Empty in unit tests."*

---

## Concept 53 — Composition tests with `Polly.Testing`

`Polly.Testing` adds **`GetPipelineDescriptor()`**, which returns a `ResiliencePipelineDescriptor` describing the pipeline: its **`Strategies`** in order (each with its **`Options`** and **`StrategyInstance`**), the **`FirstStrategy`**, and whether it **`IsReloadable`**. That lets you assert the two things refactoring most often breaks — **order** and **key options** — against the pipeline actually registered in DI:

```csharp
[Fact]
public void Pricing_pipeline_has_the_agreed_shape()
{
    var services = new ServiceCollection();
    services.AddLogging();
    services.AddPricingResilience();                          // the same extension the app uses
    using ServiceProvider sp = services.BuildServiceProvider();

    ResiliencePipeline<HttpResponseMessage> pipeline = sp
        .GetRequiredService<ResiliencePipelineProvider<string>>()
        .GetPipeline<HttpResponseMessage>("pricing");

    ResiliencePipelineDescriptor d = pipeline.GetPipelineDescriptor();

    // Order is semantics: total timeout → retry → breaker → attempt timeout.
    Assert.Collection(d.Strategies,
        s => Assert.Equal(TimeSpan.FromSeconds(1.5), Assert.IsType<TimeoutStrategyOptions>(s.Options).Timeout),
        s => Assert.Equal(2, Assert.IsType<RetryStrategyOptions<HttpResponseMessage>>(s.Options).MaxRetryAttempts),
        s => Assert.True(Assert.IsType<CircuitBreakerStrategyOptions<HttpResponseMessage>>(s.Options).FailureRatio >= 0.5),
        s => Assert.Equal(TimeSpan.FromMilliseconds(400), Assert.IsType<TimeoutStrategyOptions>(s.Options).Timeout));
}
```

A descriptor test is cheap insurance against the classic regressions: someone "tidies up" the builder and moves the attempt timeout outside the breaker, or a merge brings back the default 30-second timeout. It's also executable documentation of the agreed design. For HTTP resilience handlers, whose pipelines live inside the handler, rely on behavior and host-level tests instead (Concepts 54, 56).

**The interview-grade sentence:** *"Polly.Testing's descriptor exposes a pipeline's strategies in order with their options, so I assert the registered pipeline's shape — total timeout, retry, breaker, attempt timeout, with the agreed numbers. It catches the reorder-in-a-refactor bug and doubles as executable documentation."*

---

## Concept 54 — Behavior tests with fake time and stub handlers

Behavior tests answer "does *this configuration* do what we think?" — how many attempts, which methods retry, when the breaker opens — **without real waiting**. Two ingredients:

1. **`FakeTimeProvider`** (package `Microsoft.Extensions.TimeProvider.Testing`) assigned to the builder's public **`TimeProvider`** property. Delays, timeouts and breaker windows then move only when the test calls `Advance(...)`. (Pipelines built through Polly's DI integration take their time provider from the container in current versions — register the fake as the container's `TimeProvider` in the test host, and verify on your version with one quick test.)
2. **A scripted stub handler** that returns a sequence of responses and counts calls.

```csharp
public sealed class ScriptedHandler(params HttpStatusCode[] script) : HttpMessageHandler
{
    private int _calls;
    public int Calls => Volatile.Read(ref _calls);

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        int i = Interlocked.Increment(ref _calls) - 1;
        return Task.FromResult(new HttpResponseMessage(script[Math.Min(i, script.Length - 1)]) { RequestMessage = request });
    }
}

public sealed class PricingRetryBehavior
{
    [Fact]
    public async Task Transient_503_is_retried_exactly_twice()
    {
        var time = new FakeTimeProvider();
        var stub = new ScriptedHandler(HttpStatusCode.ServiceUnavailable);   // always 503

        ResiliencePipeline<HttpResponseMessage> pipeline =
            new ResiliencePipelineBuilder<HttpResponseMessage> { TimeProvider = time }
                .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
                {
                    MaxRetryAttempts = 2,
                    Delay = TimeSpan.FromMilliseconds(100),
                    BackoffType = DelayBackoffType.Exponential,              // no jitter: deterministic
                    ShouldHandle = static args => TransientHttp.IsTransient(args.Outcome)
                })
                .Build();

        using var client = new HttpClient(new ResilienceHandler(pipeline) { InnerHandler = stub })
        {
            BaseAddress = new("https://pricing.test")
        };

        Task<HttpResponseMessage> call = client.GetAsync("/prices/42");
        await AdvanceUntilCompletedAsync(time, call, step: TimeSpan.FromMilliseconds(50));

        using HttpResponseMessage response = await call;
        Assert.Equal(HttpStatusCode.ServiceUnavailable, response.StatusCode);
        Assert.Equal(3, stub.Calls);                                         // original + 2 retries, never more
    }

    [Fact]
    public async Task Post_without_idempotency_key_is_not_retried() { /* same shape, assert stub.Calls == 1 */ }

    // Advance fake time in small steps, yielding so continuations can schedule their next timer.
    private static async Task AdvanceUntilCompletedAsync(FakeTimeProvider time, Task task, TimeSpan step)
    {
        for (int i = 0; i < 1_000 && !task.IsCompleted; i++)
        {
            time.Advance(step);
            await Task.Delay(1);                                              // real 1 ms: let continuations run
        }
    }
}
```

The same pattern tests the breaker (drive 20 failures, advance past the sampling window, assert `BrokenCircuitException`), timeouts (a stub that awaits `Task.Delay(Timeout.Infinite, ct)`), budgets (Concept 15), and `Retry-After` handling (a stub returning 429 with the header; assert the retry waited at least that long in fake time).

**The interview-grade sentence:** *"Behavior tests set a FakeTimeProvider on the builder and put a scripted stub handler under a ResilienceHandler, then advance fake time in small steps. That makes 'exactly three attempts on 503', 'POST without a key isn't retried', and 'the breaker opens after twenty failures' deterministic tests that run in milliseconds."*

---

## Concept 55 — Chaos strategies in Polly

Since 8.3, Polly includes the former Simmy library as **chaos strategies**:

| Strategy | Extension | Injects | Kind |
|---|---|---|---|
| **Fault** | `AddChaosFault` | An exception, instead of calling the callback | Proactive |
| **Latency** | `AddChaosLatency` | A delay *before* the call proceeds | Proactive |
| **Outcome** | `AddChaosOutcome` | A fake result (e.g., a 500 response) or exception | Reactive |
| **Behavior** | `AddChaosBehavior` | Arbitrary extra behavior before the call (e.g., restart a local Redis) | Proactive |

Common options (`ChaosStrategyOptions`):

| Option | Default | Notes |
|---|---|---|
| `InjectionRate` | **0.001** (0.1%) | Probability of injection per execution |
| `InjectionRateGenerator` | `null` | Per-execution rate; overrides `InjectionRate` |
| `Enabled` | **`true`** | Chaos strategies are **on as soon as they're added** |
| `EnabledGenerator` | `null` | Per-execution on/off; overrides `Enabled` |

**Read those defaults twice.** Adding a chaos strategy without a gate injects faults into 0.1% of executions *wherever that pipeline runs*, including production. Every chaos strategy in shared code must be gated by an `EnabledGenerator` or configuration.

**Placement: innermost**, so the real strategies above react to the injected faults — which is the point. When combining, Polly's docs suggest **fault before latency** (a faulted call shouldn't also wait).

**Targeting.** Polly's docs recommend encapsulating decisions in an `IChaosManager` and targeting by environment, tenant or user through the context — so production chaos can be limited to synthetic or internal tenants:

```csharp
public interface IChaosManager
{
    ValueTask<bool> IsEnabled(ResilienceContext context);
    ValueTask<double> InjectionRate(ResilienceContext context);
}

builder.Services.AddHttpClient<CatalogClient>()
    .AddResilienceHandler("catalog", static (pipeline, context) =>
    {
        IChaosManager chaos = context.ServiceProvider.GetRequiredService<IChaosManager>();

        pipeline
            .AddTimeout(TimeSpan.FromSeconds(1.5))
            .AddRetry(new HttpRetryStrategyOptions { MaxRetryAttempts = 2, UseJitter = true })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions { FailureRatio = 0.5, MinimumThroughput = 20 })
            .AddTimeout(TimeSpan.FromMilliseconds(400))
            // --- chaos: innermost, gated per execution ---
            .AddChaosFault(new ChaosFaultStrategyOptions
            {
                EnabledGenerator       = args => chaos.IsEnabled(args.Context),
                InjectionRateGenerator = args => chaos.InjectionRate(args.Context),
                FaultGenerator = new FaultGenerator().AddException<HttpRequestException>()
            })
            .AddChaosLatency(new ChaosLatencyStrategyOptions
            {
                EnabledGenerator       = args => chaos.IsEnabled(args.Context),
                InjectionRateGenerator = args => chaos.InjectionRate(args.Context),
                Latency = TimeSpan.FromSeconds(2)                  // > attempt timeout: exercises timeout + breaker
            })
            .AddChaosOutcome(new ChaosOutcomeStrategyOptions<HttpResponseMessage>
            {
                EnabledGenerator       = args => chaos.IsEnabled(args.Context),
                InjectionRateGenerator = args => chaos.InjectionRate(args.Context),
                OutcomeGenerator = new OutcomeGenerator<HttpResponseMessage>()
                    .AddResult(() => new HttpResponseMessage(HttpStatusCode.ServiceUnavailable))
            });
    });
```

(Polly 8.8 relaxed fault generation so a `FaultGenerator` delegate can return `null` to mean "no fault this time.") An in-process chaos run is still an experiment: define the steady-state metric, the hypothesis, the blast radius and the abort condition first (Module 13, Concept 55). In-process chaos tests your *pipeline*; network-level (Toxiproxy) and platform-level (Azure Chaos Studio) faults test everything else.

**The interview-grade sentence:** *"Polly's chaos strategies — fault, latency, outcome, behavior — go innermost so the real strategies react, and they're enabled by default at a 0.1% injection rate, so every one is gated with EnabledGenerator. I route decisions through a chaos manager so production experiments can target synthetic tenants only, and I treat each run as an experiment with a hypothesis and abort condition."*

---

## Concept 56 — End-to-end resilience tests with real transports

Unit and behavior tests check a pipeline in isolation. The failures that actually reach production are usually **emergent**: a global default handler stacked on a custom one, an SDK retrying underneath, three services each retrying twice. Only tests that run the **whole host as registered** catch those.

**Host-level test with a scripted fake dependency** — the application's real DI, real handler chain (Aspire defaults included), talking to WireMock.Net:

```csharp
public sealed class QuoteEndpointResilienceTests : IClassFixture<WebApplicationFactory<Program>>, IDisposable
{
    private readonly WireMockServer _pricing = WireMockServer.Start();
    private readonly WebApplicationFactory<Program> _factory;

    public QuoteEndpointResilienceTests(WebApplicationFactory<Program> factory) =>
        _factory = factory.WithWebHostBuilder(b =>
            b.UseSetting("Services:Pricing:BaseUrl", _pricing.Url));        // point the real client at the fake

    [Fact]
    public async Task A_failing_pricing_dependency_is_attempted_at_most_three_times_per_request()
    {
        _pricing.Given(Request.Create().WithPath("/prices/42").UsingGet())
                .RespondWith(Response.Create().WithStatusCode(503));

        HttpClient client = _factory.CreateClient();
        HttpResponseMessage response = await client.GetAsync("/api/quotes/42");

        Assert.Equal(HttpStatusCode.ServiceUnavailable, response.StatusCode); // honest degraded response
        Assert.Equal(3, _pricing.LogEntries.Count());                          // 1 + 2 retries: no hidden layers
    }

    public void Dispose() => _pricing.Stop();
}
```

That single assertion — **attempts counted at the dependency** — is the most valuable resilience test in a codebase, because it's the one that fails when someone adds a second retry layer anywhere in the stack (Module 13, Exercise 1).

Beyond that:

- **Toxiproxy** between the service and a real dependency (in containers) injects latency, bandwidth limits and connection resets at the TCP level — exercising connect timeouts, mid-body resets (Concept 41) and fail-slow behavior that HTTP stubs can't imitate.
- **Load tests with injected faults** (NBomber, k6, Azure Load Testing) find where goodput collapses (Module 13, Concept 10) — for example, removing a cache or slowing the database while at 1.5× peak.
- **Multi-service chains** in a test environment verify end-to-end amplification and "don't retry" signal propagation (Module 13, Concept 18).

**The interview-grade sentence:** *"The resilience test I value most runs the whole host as registered against a scripted fake dependency and counts attempts at the dependency — it's the test that breaks when someone adds a second retry layer anywhere. Toxiproxy covers TCP-level faults, and load tests with injected faults find where goodput collapses."*

---

# Part M — Extending Polly

## Concept 57 — Custom strategies: a deadline guard

When no built-in strategy fits, write one. Polly distinguishes **proactive** strategies (derive from `ResilienceStrategy`, work for any result type) and **reactive** ones (derive from `ResilienceStrategy<T>`, inspect outcomes). The recipe from Polly's extensibility docs has four parts: the **strategy** (internal), an **options** class deriving from `ResilienceStrategyOptions` (public, validated), **event arguments** (a struct ending in `Arguments` with a `Context` property), and a **builder extension** that calls `AddStrategy`.

A useful example that Concept 9 promised: a **deadline guard** that refuses work whose propagated deadline can't accommodate even one attempt — so expired requests are rejected in microseconds instead of starting work nobody will wait for (Module 13, Concept 13).

```csharp
public sealed class DeadlineExceededException(DateTimeOffset deadline)
    : Exception($"Deadline {deadline:O} leaves too little time to start the operation.");

public readonly struct DeadlineInsufficientArguments(ResilienceContext context, TimeSpan remaining)
{
    public ResilienceContext Context { get; } = context;
    public TimeSpan Remaining { get; } = remaining;
}

public sealed class DeadlineGuardOptions : ResilienceStrategyOptions
{
    public DeadlineGuardOptions() => Name = "DeadlineGuard";

    [Required]
    [Range(typeof(TimeSpan), "00:00:00.001", "00:10:00")]
    public TimeSpan? MinimumRemaining { get; set; }        // typically one attempt timeout
}

internal sealed class DeadlineGuardStrategy(
    TimeSpan minimumRemaining, TimeProvider time, ResilienceStrategyTelemetry telemetry) : ResilienceStrategy
{
    protected override ValueTask<Outcome<TResult>> ExecuteCore<TResult, TState>(
        Func<ResilienceContext, TState, ValueTask<Outcome<TResult>>> callback,
        ResilienceContext context,
        TState state)
    {
        if (context.Properties.TryGetValue(ResilienceKeys.Deadline, out DateTimeOffset deadline))
        {
            TimeSpan remaining = deadline - time.GetUtcNow();
            if (remaining < minimumRemaining)
            {
                telemetry.Report(new ResilienceEvent(ResilienceEventSeverity.Warning, "DeadlineInsufficient"),
                                 context, new DeadlineInsufficientArguments(context, remaining));
                return Outcome.FromExceptionAsValueTask<TResult>(new DeadlineExceededException(deadline));
            }
        }
        return callback(context, state);                    // enough time: proceed, untouched
    }
}

public static class DeadlineGuardBuilderExtensions
{
    public static TBuilder AddDeadlineGuard<TBuilder>(
        this TBuilder builder, DeadlineGuardOptions options, TimeProvider? timeProvider = null)
        where TBuilder : ResiliencePipelineBuilderBase =>
        builder.AddStrategy(
            context => new DeadlineGuardStrategy(
                options.MinimumRemaining!.Value, timeProvider ?? TimeProvider.System, context.Telemetry),
            options);                                        // AddStrategy validates the options
}

// Usage: outermost, before the bulkhead, so an expired request doesn't even take a permit.
pipeline
    .AddDeadlineGuard(new DeadlineGuardOptions { MinimumRemaining = TimeSpan.FromMilliseconds(300) })
    .AddConcurrencyLimiter(64)
    .AddTimeout(new TimeoutStrategyOptions { Name = "total", TimeoutGenerator = /* Concept 9 */ })
    .AddRetry(retryOptions)
    .AddCircuitBreaker(breakerOptions)
    .AddTimeout(TimeSpan.FromMilliseconds(300));
```

Because it reports through `ResilienceStrategyTelemetry`, the guard's refusals appear in the same `resilience.polly.strategy.events` counter as everything else (`event.name = "DeadlineInsufficient"`), and `AddStrategy` validates its options exactly like the built-ins.

Other strategies teams commonly write: a **retry-budget gate** (Concept 15) as a reusable strategy, a **"don't-retry" signal** strategy that converts a downstream `x-should-retry: false` into a non-retryable outcome, and **adaptive concurrency** limiters (Module 13, Concept 36). Keep them small, internal, options-driven, and covered by behavior tests.

**The interview-grade sentence:** *"A custom Polly strategy is an internal class overriding ExecuteCore, a validated public options type, an Arguments struct for its events, and an AddStrategy extension. My usual example is a deadline guard placed outermost that rejects requests whose propagated deadline can't fit one attempt, reporting through Polly's telemetry so it shows up with every other resilience event."*

---

## Concept 58 — Reusable classification libraries

Predicates encode **how a dependency fails**, so they deserve the same treatment as any other shared domain knowledge: one small, tested library per dependency type, used by retry, breaker, hedging and fallback alike.

```csharp
namespace Company.Resilience.Classification;

public static class ServiceBusFaults
{
    public static bool IsTransient(Exception? ex) => ex switch
    {
        ServiceBusException sb      => sb.IsTransient,                 // the SDK knows its own failure modes
        TimeoutRejectedException    => true,
        _                           => false
    };
}

public static class CosmosFaults
{
    // Outer layers should rarely retry Cosmos — the SDK already does (Concept 65).
    // This classifier exists for the breaker and for telemetry.
    public static bool IsHealthFailure(Exception? ex) => ex switch
    {
        CosmosException { StatusCode: HttpStatusCode.ServiceUnavailable
                                   or HttpStatusCode.RequestTimeout } => true,
        CosmosException { StatusCode: HttpStatusCode.TooManyRequests } => false,   // capacity, not health
        TimeoutRejectedException                                     => true,
        _                                                            => false
    };
}
```

Guidelines:

- **Pattern-match with `is`, don't compare `GetType()`.** A set of exception `Type`s checked with `Contains(ex.GetType())` (a pattern shown in Polly's own docs for grouping exceptions) ignores subclasses — `IsolatedCircuitException` isn't `typeof(BrokenCircuitException)`. Switch expressions handle inheritance correctly.
- **Separate "should retry" from "is unhealthy"** (Concept 4) — they differ for 429 and 500.
- **Prefer the SDK's own knowledge** — `ServiceBusException.IsTransient`, `EventHubsException.IsTransient`, EF Core's transient-error detection — over hand-maintained error-code lists.
- **Test them table-driven** (Concept 52), and version them with the dependency they describe.

**The interview-grade sentence:** *"Classification is shared knowledge about how a dependency fails, so it lives in one tested library per dependency type, used by retry, breaker and hedging. I pattern-match with switch expressions so subclasses are handled, keep 'should retry' separate from 'is unhealthy', and defer to SDK knowledge like ServiceBusException.IsTransient."*

---

# Part N — Placement: where resilience lives in a .NET architecture

## Concept 59 — Resilience belongs in adapters

Module 20 (Concept 44) set the rule: **resilience belongs in the adapter, not the handler.** A use case that contains a retry loop has leaked an infrastructure concern into the application core. With Polly, the rule becomes concrete:

- **The port** (`IPricingService`) speaks the domain's language: `GetPriceAsync(sku)` returns a price or a domain-meaningful failure.
- **The adapter** (`PricingHttpAdapter` in Infrastructure) owns the `HttpClient`, its resilience handler or pipeline, and — critically — **the translation** of Polly and transport exceptions into domain outcomes.
- **The composition root** owns the configuration (Concept 48).

```csharp
// Application layer — no Polly, no HTTP.
public interface IPricingService
{
    ValueTask<PriceLookup> GetPriceAsync(Sku sku, CancellationToken ct);
}

public abstract record PriceLookup
{
    public sealed record Found(Money Price) : PriceLookup;
    public sealed record Unavailable(TimeSpan? RetryAfter) : PriceLookup;   // a business-visible state
}

// Infrastructure — the adapter translates resilience outcomes into the port's vocabulary.
internal sealed class PricingHttpAdapter(PricingHttpClient client) : IPricingService
{
    public async ValueTask<PriceLookup> GetPriceAsync(Sku sku, CancellationToken ct)
    {
        try
        {
            return new PriceLookup.Found(await client.GetPriceAsync(sku, ct));   // resilience handler inside
        }
        catch (BrokenCircuitException ex)       { return new PriceLookup.Unavailable(ex.RetryAfter); }
        catch (RateLimiterRejectedException ex) { return new PriceLookup.Unavailable(ex.RetryAfter); }
        catch (TimeoutRejectedException)        { return new PriceLookup.Unavailable(null); }
        catch (HttpRequestException)            { return new PriceLookup.Unavailable(null); }
        // OperationCanceledException from the caller propagates: nobody is waiting for an answer.
    }
}
```

The handler then makes the **product decision** — show "price temporarily unavailable", fall back to a cached list price, or reject the order — without knowing whether that came from a breaker, a bulkhead or a timeout. That split keeps Polly swappable (Concept 68) and keeps business policy out of infrastructure.

**Three deliberate exceptions to "resilience lives in adapters":**

1. **Optimistic-concurrency retries** re-run the *decision*, so they belong at the application level — the command pipeline (Concept 62) or the event-sourced command handler (Concept 63).
2. **Idempotency** is an application concern (the command pipeline records outcomes by key — Module 23), which the adapters rely on.
3. **Degradation policy** — what to do when a dependency is unavailable — is a product decision made in the application layer from the adapter's honest signal.

**The interview-grade sentence:** *"Transport resilience lives in the adapter: it owns the client and pipeline and translates BrokenCircuitException, limiter rejections and timeouts into domain outcomes like 'price unavailable, retry after', so handlers make product decisions without seeing Polly. The exceptions are concurrency retries, which re-run the decision and belong in the command pipeline, and idempotency and degradation policy, which are application concerns."*

---

## Concept 60 — The retry ownership map: one loop per failure kind

Module 13 (Concept 21) said "one retry owner per dependency." In a real .NET application the sharper version is: **one retry loop per failure kind, owned by the level that can safely redo the work** — the rule Module 23 (Concept 64) stated in passing. Here is the map:

| Failure kind | Owner | Everyone else adds |
|---|---|---|
| Transport blip to an internal HTTP service | The adapter's HTTP resilience handler | Outer layers: timeouts and "don't retry" signals only |
| Transient database error (failover, throttling, deadlock victim) | **EF Core's execution strategy** (or SqlClient's retry logic — not both) | Nothing — no Polly retry around DB calls (Concept 61) |
| Optimistic concurrency conflict (`DbUpdateConcurrencyException`) | **Command pipeline** concurrency-retry behavior | The execution strategy deliberately doesn't retry it |
| Event-store `WrongExpectedVersion` | **Command handler's re-decide loop** | Transport retries of the append reuse event IDs (Concept 63) |
| Azure SDK transient/throttling | **The SDK's `RetryOptions`** | An outer total timeout (Concept 65) |
| Cosmos DB 429 / transient / regional | **The Cosmos SDK** | Nothing, or an outer timeout |
| gRPC `Unavailable` | **The channel's retry policy** | Transport-level protection without retries (Concept 44) |
| Message-processing failure, short blip | A tiny in-process retry in the consumer | — |
| Message-processing failure, lasting | **Broker redelivery / scheduled redelivery** | No long sleeps holding locks (Concept 64) |
| Poison message or event | **Dead-letter / park** | Alert; manual or automated repair |
| Multi-step process step failure | **Workflow engine** (Durable Functions, Temporal) | — |
| User-visible failure | **The user or client**, with the same idempotency key | Server returns `Retry-After` or "don't retry" |

**Check the multiplication for each failure kind separately.** A transport blip in a call from API → adapter → downstream service → its database touches the HTTP handler (2 retries) and the downstream's EF strategy (for DB faults) — different failure kinds, so they don't multiply for the same fault. But if the API's caller *also* retries, and the downstream service *also* has a resilience handler on its outbound calls, the same transport fault can be retried at three levels: (1+2)³ = 27 attempts. For each failure kind, the product of attempts along a request path should stay around 3–4.

```
Client ──(user retry, same key)──► Gateway ──► API ──► [adapter: HTTP handler, owns transport retries]
                                                         └─► Downstream service ──► [EF execution strategy, owns DB transient]
                                    [command pipeline: owns concurrency retries]     [downstream's own outbound handler: ⚠ same fault kind]
```

**The interview-grade sentence:** *"I map every failure kind to exactly one retry owner — the HTTP handler for transport blips, EF Core's execution strategy for transient DB faults, the command pipeline for concurrency conflicts, the SDKs for their own services, the broker for lasting processing failures, the user with the same idempotency key for anything visible — and I check the attempt product per failure kind along the whole request path."*

---

## Concept 61 — EF Core's execution strategy and Polly

**The execution strategy owns transient database faults.** `EnableRetryOnFailure()` installs `SqlServerRetryingExecutionStrategy` (other providers have equivalents), which retries using a curated list of transient SQL error numbers. Its defaults — **6 retries, up to 30 seconds between attempts** — suit background work more than a request path, so tune it:

```csharp
builder.Services.AddDbContext<OrderingDbContext>(o => o.UseSqlServer(connectionString, sql =>
{
    sql.EnableRetryOnFailure(
        maxRetryCount: 2,                                // request path: a couple of retries, not six
        maxRetryDelay: TimeSpan.FromSeconds(1),
        errorNumbersToAdd: null);
    sql.CommandTimeout(5);                               // seconds; the server-side work bound
}));
```

Four rules for combining it with anything else:

1. **Never wrap EF calls in a Polly retry for transient errors.** That's the double-retry trap (Module 13, Concept 21): 3 × 3 = 9 attempts against a database that's failing over. Polly around EF adds only what the strategy doesn't provide: an outer **timeout** (cooperative, via the token) and perhaps a **bulkhead** around expensive queries.
2. **User-initiated transactions must run inside the strategy.** With a retrying strategy, `BeginTransactionAsync()` outside `strategy.ExecuteAsync(...)` throws, because the strategy can't retry a transaction it doesn't control. The whole unit — begin, work, `SaveChanges`, commit — goes inside (Module 23, Concept 65).
3. **The strategy doesn't retry `DbUpdateConcurrencyException`** — by design, since re-sending the same stale update can't succeed. Concurrency conflicts need a *fresh decision* one level up (Concept 62).
4. **Commit ambiguity is real.** If the connection drops *during* commit, the strategy can't know whether the commit landed, so it re-runs the whole operation. EF Core offers two remedies: make the operation idempotent (an idempotency record with a unique constraint turns a duplicate into a detectable violation — Concept 62), or supply a **verification** delegate:

```csharp
IExecutionStrategy strategy = db.Database.CreateExecutionStrategy();

await strategy.ExecuteInTransactionAsync(
    (Db: db, Order: order),
    operation: static async (s, ct) =>
    {
        s.Db.Orders.Add(s.Order);
        await s.Db.SaveChangesAsync(acceptAllChangesOnSuccess: false, ct);   // don't accept until committed
    },
    verifySucceeded: static (s, ct) =>                                        // "did the commit actually land?"
        s.Db.Orders.AsNoTracking().AnyAsync(o => o.Id == s.Order.Id, ct),
    cancellationToken);

db.ChangeTracker.AcceptAllChanges();
```

**SqlClient's configurable retry logic** (`SqlRetryLogicBaseProvider`) is the alternative owner for code that uses ADO.NET or Dapper directly. Pick one owner per connection path — not both under EF Core.

**The interview-grade sentence:** *"EF Core's execution strategy owns transient database faults — I tune it down to a couple of retries for request paths and never wrap it in a Polly retry. Transactions go inside strategy.ExecuteAsync, concurrency conflicts are deliberately left to the command layer, and commit ambiguity is handled with an idempotency record or ExecuteInTransactionAsync's verification delegate."*

---

## Concept 62 — The command pipeline: concurrency retries done right

Module 23 (Concept 64) fixed the order of the command pipeline — **telemetry → authorization → validation → idempotency → concurrency retry → unit of work → handler** — and left two threads for this module: *how to compose the concurrency retry with EF Core's execution strategy without multiplying*, and *how to turn a commit-ambiguity re-run into the right response*. Here are both.

**The concurrency-retry behavior, with Polly:**

```csharp
// Registration: a small, fast, jittered retry for optimistic-concurrency conflicts only.
builder.Services.AddResiliencePipeline("command-concurrency", static b => b.AddRetry(new RetryStrategyOptions
{
    Name = "concurrency-retry",
    MaxRetryAttempts = 3,
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,
    Delay = TimeSpan.FromMilliseconds(20),                 // contention clears in milliseconds, not seconds
    ShouldHandle = new PredicateBuilder().Handle<DbUpdateConcurrencyException>()
}));

internal sealed class ConcurrencyRetryBehavior<TRequest, TResponse>(
    OrderingDbContext db,
    [FromKeyedServices("command-concurrency")] ResiliencePipeline pipeline)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        int attempt = 0;
        return await pipeline.ExecuteAsync(async token =>
        {
            if (attempt++ > 0)
                db.ChangeTracker.Clear();      // the previous attempt's view was stale: reload, re-decide
            return await next(token);          // unit of work (inside the execution strategy) + handler
        }, ct);
    }
}
```

**Why this doesn't multiply badly.** The concurrency retry handles only `DbUpdateConcurrencyException`; the execution strategy inside the unit-of-work behavior handles only transient SQL errors. They retry **different failure kinds**, so one fault triggers one loop. The worst case — a transient error *and* a conflict on the same command — is bounded by (3 + 1) × (2 + 1) = 12 short attempts, with the execution strategy tuned down as in Concept 61. That's acceptable; two loops for the *same* failure kind would not be.

**Every attempt re-runs the handler — including its side effects.** This is the part most teams miss. If the handler calls a payment API and then the commit hits a concurrency conflict, the retry calls the payment API **again**. Two acceptable designs:

- **Move external side effects to the outbox** (Module 11): the handler records intent in the same transaction; a relay performs the call after commit. The retry re-runs only local, transactional work.
- **Make the external call idempotent with a key derived from the command** (`$"capture:{command.OrderId}"`), so repeats are deduplicated downstream (Concept 42).

**Commit ambiguity → the right response.** Module 23's idempotency behavior stores the command's outcome by key inside the unit of work, protected by a unique constraint. If a commit's acknowledgment was lost and the execution strategy re-runs the attempt, the re-run's insert of the idempotency record hits the unique constraint. That exception is a plain `DbUpdateException` (not a concurrency exception), so the concurrency retry ignores it and it reaches the idempotency behavior — which should turn it into **the original answer**, not an error:

```csharp
// Inside the idempotency behavior (outside the retry — Module 23, Concept 64)
try
{
    return await next(ct);
}
catch (DbUpdateException ex) when (ex.InnerException is SqlException { Number: 2601 or 2627 } sql
                                   && sql.Message.Contains("IX_IdempotencyRecords_Key", StringComparison.Ordinal))
{
    // The first attempt DID commit; its acknowledgment was lost. Answer exactly as it did.
    db.ChangeTracker.Clear();
    IdempotencyRecord stored = await db.IdempotencyRecords.AsNoTracking()
        .SingleAsync(r => r.Key == request.IdempotencyKey, ct);
    return outcomeSerializer.Deserialize<TResponse>(stored.Outcome);
}
```

**When retries run out,** return a conflict result (HTTP 409) and emit a "hot aggregate" signal: a command that loses three optimistic races in 100 ms is telling you about contention, which is a modeling question (Module 22, Concept 69), not a retry-tuning one. And for commands carrying the version the **user** saw (`If-Match`), don't retry at all — return 412 so the user can review (Module 24, Concept 49).

**The interview-grade sentence:** *"The concurrency retry is a Polly pipeline in a command behavior that handles only DbUpdateConcurrencyException, sits outside the unit of work, and clears the change tracker so each attempt re-reads and re-decides; EF's execution strategy inside handles only transient faults, so the two never retry the same failure. Because each attempt re-runs the handler, external side effects go to the outbox or carry command-derived idempotency keys, and a unique-key violation on a re-run is turned back into the stored original outcome."*

---

## Concept 63 — Event-sourced decisions: the re-decide loop

Module 24 (Concept 49) described three responses to `WrongExpectedVersion`: **re-decide** automatically, **surface** to the user (412), or **merge** by event semantics — and left the resilience mechanics here. The mechanics have one subtlety: there are **two different loops**, and they must not be confused.

| Loop | Handles | Each attempt | Event IDs |
|---|---|---|---|
| **Outer: re-decide** | `WrongExpectedVersion` (someone else appended) | Re-reads the stream, folds, decides **again** — possibly differently | **New** decision → new events → new IDs |
| **Inner: append retry** | Transport failures while appending (timeouts, `Unavailable`) | Re-sends the **same** append | **Same** IDs, so the store deduplicates (Module 24, Concept 29) |

```csharp
ResiliencePipeline reDecide = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        Name = "re-decide",
        MaxRetryAttempts = 4,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        Delay = TimeSpan.FromMilliseconds(10),
        ShouldHandle = new PredicateBuilder().Handle<WrongExpectedVersionException>()
    })
    .Build();

await reDecide.ExecuteAsync(async ct =>
{
    (OrderState state, long version) = await store.LoadAsync<OrderState>(streamId, ct);   // fresh read every time

    if (state.HasProcessed(command.IdempotencyKey))       // our own earlier append may have landed (ambiguous timeout)
        return;                                            // ...then we're done — don't decide twice

    IReadOnlyList<object> events = OrderDecider.Decide(state, command);                   // pure decision
    EventData[] data = events
        .Select(e => serializer.ToEventData(e, Guid.CreateVersion7(), command.IdempotencyKey))  // IDs fixed for THIS decision
        .ToArray();

    await appendRetry.ExecuteAsync(                        // inner: transport retries reuse the same IDs
        static async (s, t) => await s.Store.AppendAsync(s.Stream, s.Version, s.Data, t),
        (Store: store, Stream: streamId, Version: version, Data: data), ct);
}, cancellationToken);
```

Design points, all from Module 24 made executable:

- **The retry re-runs the whole loop, never just the append.** An append with a stale expected version can only fail again.
- **Record the command's idempotency key in event metadata** and check it on re-read, so an append whose acknowledgment was lost (the inner loop's retry then sees `WrongExpectedVersion` against its *own* events) ends cleanly instead of deciding twice.
- **The inner loop must not handle `WrongExpectedVersion`**; that's the outer loop's job.
- **Bound it and jitter it** — three to five attempts. Exhaustion is a hot-stream signal (Module 24, Concept 32), surfaced as 409 with telemetry.
- **User-intent commands don't loop** — they carry the version the user saw and return 412 on conflict.

**The interview-grade sentence:** *"An event-sourced command has two loops: an outer re-decide loop on WrongExpectedVersion that re-reads, re-folds and decides again with new event IDs, and an inner transport retry that re-sends the same append with the same IDs. The command's idempotency key goes in event metadata and is checked on re-read, so a lost acknowledgment ends cleanly instead of producing a second decision."*

---

## Concept 64 — Message consumers, subscriptions, and outbox relays

Asynchronous processing changes the economics of retries: **the broker is a durable retry mechanism**, so in-process retries should only absorb blips (Module 11, Concept 13; Module 13, Concept 21).

**Service Bus consumers.** Know what the platform already does:

- The SDK retries its *own* operations (receive, complete, renew) per `ServiceBusRetryOptions` — exponential by default.
- If your handler throws, the processor **abandons** the message, and it's **redelivered immediately** — there's no built-in delay — with `DeliveryCount` incremented, until **`MaxDeliveryCount`** (default 10) sends it to the dead-letter queue.

So a consumer's resilience design is: a **tiny** Polly retry (one or two quick attempts) for blips, then **scheduled redelivery** for backoff — never a long in-process sleep that holds the message lock:

```csharp
processor.ProcessMessageAsync += async args =>
{
    try
    {
        await quickRetry.ExecuteAsync(                                  // ≤ 2 quick retries, ~100–500 ms total
            static async (s, ct) => await s.Handler.HandleAsync(s.Message, ct),
            (Handler: handler, Message: args.Message), args.CancellationToken);

        await args.CompleteMessageAsync(args.Message, args.CancellationToken);
    }
    catch (Exception ex) when (ServiceBusFaults.IsTransientDownstream(ex))
    {
        int attempt = args.Message.ApplicationProperties.TryGetValue("app-attempt", out object? a) ? (int)a : 0;
        if (attempt >= 5)
        {
            await args.DeadLetterMessageAsync(args.Message, "RetriesExhausted", ex.Message, args.CancellationToken);
            return;
        }

        // Backoff without holding the lock: schedule a copy for later, then complete the original.
        var retry = new ServiceBusMessage(args.Message)
        {
            ScheduledEnqueueTime = timeProvider.GetUtcNow() + Backoff.ExponentialWithJitter(attempt)
        };
        retry.ApplicationProperties["app-attempt"] = attempt + 1;
        await sender.SendMessageAsync(retry, args.CancellationToken);  // not atomic with Complete → at-least-once;
        await args.CompleteMessageAsync(args.Message, args.CancellationToken);   // the handler must be idempotent
    }
    // Anything else propagates: abandon → immediate redelivery → dead-letter at MaxDeliveryCount.
};
```

(Scheduling a copy breaks ordering; for **session**-ordered queues, use deferral or stop the session instead.) Messaging frameworks — MassTransit, NServiceBus, Wolverine — ship retry and delayed-redelivery policies built for exactly this; if you use one, configure *its* policies rather than adding Polly inside handlers.

**Pausing consumption when a dependency is down.** A breaker around the downstream dependency can drive the consumer: when it opens, stop pulling messages (the queue is the buffer — Module 11), then resume after the break. Trigger the stop **asynchronously** — signal a supervising hosted service from `OnOpened` — because awaiting `StopProcessingAsync()` from inside a message handler waits for the very handler that's calling it.

**Event-store subscriptions and projections** (Module 24) have an extra constraint: **order**. A projection can't skip event 1,001 and process 1,002, so:

- transient failures are retried **in place** (a bounded Polly retry with backoff; the subscription simply doesn't advance its checkpoint),
- a **poison event** (a permanent failure) is **parked** — record the stream position and the error, alert — and the projection either **stops** (strict, when correctness matters) or **skips and records** (lenient, when staleness is acceptable) — a product decision made in advance,
- handlers are idempotent by position, so a replay after a crash is harmless.

**Outbox relays** (Module 11) are background loops publishing committed outbox rows. Here, and only here, "retry until it works" is legitimate — the work is durable and the loop is not holding a user request — but it still needs **capped exponential backoff with jitter** and a **breaker around the broker** so an outage produces a quiet, slowly-probing relay rather than a tight loop of failures. Count attempts per row and park rows that fail permanently.

**The interview-grade sentence:** *"In consumers the broker is the durable retry: I keep one or two quick in-process retries for blips, then schedule redelivery with backoff instead of sleeping on a lock — because Service Bus abandon redelivers immediately — and dead-letter after a bounded count. Projections retry in place and park poison events because order matters, and outbox relays may retry indefinitely but with capped jittered backoff and a breaker around the broker."*

---

## Concept 65 — Azure SDK clients: configure, don't wrap

The Azure SDKs have resilience built in; the job is **configuration**, not decoration (Module 13, Concept 60).

| Client | Built-in behavior | Defaults to know |
|---|---|---|
| `Azure.Core`-based clients (Blob, Queue, Key Vault, App Configuration…) | Exponential retries, honor `Retry-After`, per-try network timeout | `RetryOptions`: 3 retries, 0.8 s base delay, 1 min max delay, exponential; `NetworkTimeout` 100 s |
| Service Bus / Event Hubs | Retries per `ServiceBusRetryOptions` / `EventHubsRetryOptions` | 3 retries, 0.8 s base, 1 min max, exponential; `TryTimeout` 1 min |
| Cosmos DB v3 SDK | Retries 429s, transient network errors, cross-region failover; optional hedging availability strategy | 429s: up to 9 retries, 30 s cumulative wait |

For a request path those defaults are too patient. Tune them per client, centrally:

```csharp
builder.Services.AddAzureClients(clients =>
{
    clients.AddBlobServiceClient(new Uri(builder.Configuration["Storage:BlobUri"]!))
        .ConfigureOptions(static o =>
        {
            o.Retry.MaxRetries     = 2;
            o.Retry.Delay          = TimeSpan.FromMilliseconds(200);
            o.Retry.MaxDelay       = TimeSpan.FromSeconds(2);
            o.Retry.NetworkTimeout = TimeSpan.FromSeconds(5);           // per try
        });

    clients.AddServiceBusClientWithNamespace(builder.Configuration["ServiceBus:Namespace"]!)   // e.g. "contoso.servicebus.windows.net"
        .ConfigureOptions(static o => o.RetryOptions = new ServiceBusRetryOptions
        {
            Mode = ServiceBusRetryMode.Exponential,
            MaxRetries = 3,
            TryTimeout = TimeSpan.FromSeconds(10)
        });

    clients.UseCredential(new DefaultAzureCredential());
});

builder.Services.AddSingleton(sp => new CosmosClient(
    builder.Configuration["Cosmos:Endpoint"], new DefaultAzureCredential(), new CosmosClientOptions
    {
        ApplicationPreferredRegions = ["West Europe", "North Europe"],
        MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(5),  // request path: don't wait 30 s on 429s
        MaxRetryAttemptsOnRateLimitedRequests = 5
    }));
```

What Polly may still add **around** an SDK call: an outer **total timeout** tied to the caller's budget (through the token), a **bulkhead** around expensive operations, and occasionally a **non-retrying breaker** for fast failure in a degradable flow. What it must not add: another retry.

Two stacking traps specific to Azure: plugging an `HttpClient` from `IHttpClientFactory` into an SDK's transport (`HttpClientTransport`) while global defaults add the standard resilience handler to every factory client — now the SDK retries *and* the handler retries; and wrapping a Cosmos call in a retry that handles 429s, re-multiplying what the SDK already did (Module 12, Concept 34: sustained 429s are a capacity signal, not a retry-tuning problem). If an SDK's classification truly needs customizing, Azure.Core lets you supply your own `RetryPolicy` subclass via `ClientOptions.RetryPolicy` — still one owner.

**The interview-grade sentence:** *"Azure SDK clients already retry with exponential backoff and honor Retry-After, so I tune their retry options per client — fewer retries and short per-try timeouts on request paths, a much shorter 429 wait for Cosmos — and add only an outer timeout or bulkhead with Polly. I watch for the stacking trap where a factory HttpClient with a global resilience handler is plugged into an SDK's transport."*

---

## Concept 66 — Inbound vs outbound: Polly is for the calls you make

A last placement rule, because it's often muddled: **Polly protects the calls your service makes; ASP.NET Core protects the calls your service receives.**

| Concern | Inbound (you are the server) | Outbound (you are the client) |
|---|---|---|
| Per-client limits | ASP.NET Core rate-limiting middleware (partitioned) | Polly partitioned rate limiter toward a provider (Concept 25) |
| Protecting capacity | Global concurrency limiter middleware, Kestrel limits | Polly concurrency limiter per dependency |
| Timeouts | `AddRequestTimeouts()` + `HttpContext.RequestAborted` | Polly attempt and total timeouts |
| Health | Health checks: liveness / readiness / startup | Breaker state as a Degraded dependency check |
| Signals | Send 429/503 with `Retry-After`, "don't retry" markers | Honor them (Concept 14) |

Module 13 (Concept 61) covered the inbound tools. The connection between the two sides is the **cancellation token**: the inbound request's `RequestAborted` token (bounded by request-timeout middleware) is what you pass into your adapters, so when the client leaves or the server-side timeout fires, every outbound pipeline sees the cancellation and stops. An outbound pipeline that isn't given the inbound token is working for nobody.

Polly on the inbound side is appropriate only where no framework mechanism exists — a bulkhead around a shared in-process resource (a native PDF renderer, a GPU model) used by many endpoints, or concurrency for background jobs (Concept 25). Using a Polly pipeline in middleware to "time out requests" is reinventing `AddRequestTimeouts()` badly.

**The interview-grade sentence:** *"Polly is for outbound calls; inbound limits, timeouts and shedding belong to ASP.NET Core's middleware and Kestrel. The two meet at the cancellation token — RequestAborted flows into every adapter, so when the client leaves or the request timeout fires, the outbound pipelines stop working for nobody."*

---

# Part O — Governance and judgment

## Concept 67 — Polly's Open Source Maintenance Fee: the facts

This is new in 2026, it affects a dependency that sits under much of the .NET ecosystem, and it's exactly the kind of topic an architect interview uses to test judgment. Get the facts straight first.

**What was announced.** On **July 14, 2026**, the Polly maintainers announced that Polly is adopting the **Open Source Maintenance Fee (OSMF)**. Payment requests begin **November 16, 2026**:

- **US $20 per month, per organization** — not per project; ten products using Polly still means one fee.
- It applies to **companies that earn at least US $20,000 from at least one product or project that uses Polly**. It's tied to **revenue, not price**: a free product that generates revenue (advertising, paid services, lead generation) counts.
- **Individuals, hobbyists, students, non-profits and organizations below the threshold owe nothing.**
- Payment is planned through **GitHub Sponsors**, with instructions to be published before November.

**What it is not**, per the announcement:

- **Not a license change.** The source remains under the **BSD-3-Clause** license; you can read, fork, build and contribute as before.
- **Not a support contract.** Paying buys no SLA, no guaranteed fixes, no priority support; it funds general upkeep — triage, releases, dependency updates, security response, signing certificates, CI.
- **Existing installations don't stop working.**

**How OSMF works in general.** The OSMF model (the same one used by other projects) attaches the fee to using the project's **maintained releases** rather than to the source code. Its consumer guidance says you pay for **direct dependencies only** — identified in .NET by the `PackageReference` items in your project files (including those imported via `Directory.Build.props`) — and explicitly **not for transitive** dependencies brought in by other packages. It also notes that `using` statements don't determine what's direct.

**The open question: Microsoft's packages.** `Microsoft.Extensions.Resilience` and `Microsoft.Extensions.Http.Resilience` reference Polly and **expose Polly types in their public APIs** (`ResiliencePipeline<T>`, `ResilienceContext`, `Outcome<T>`, the strategy options). An issue opened on **August 26, 2026** in `dotnet/extensions` (#7719) asks Microsoft to clarify whether it will license Polly, pin the version, build from source, fork, replace Polly, or redesign those APIs — and whether consumers who only reference Microsoft's packages remain covered by the transitive exemption. **As of September 29, 2026, no answer is visible on that issue.** Community commentary has also pointed out that the announcement's "$20,000 from a product" trigger differs from the OSMF template's wording; the maintainers have said they'll use the months before November to refine details.

**The community fork.** **Fences** (BrighterCommand) is a BSD-3 fork taken from **Polly 8.7.0** specifically to avoid the fee. It's **pre-release** (`9.0.0-alpha` packages), keeps the v8 API with packages renamed `Paramore.Fences.*` and the namespace `Paramore.Fences`, renames telemetry (`resilience.fences.*`), and states it may be retired if the OSMF decision is reversed. It can't help code that uses `Microsoft.Extensions.Http.Resilience`, which keeps Polly in the dependency graph regardless.

**Context.** This follows a pattern the curriculum has already met: MediatR's and MassTransit's moves to commercial licensing (Modules 11 and 23). The .NET ecosystem is actively working out how widely used open-source infrastructure gets funded, and an architect is expected to have a process for it — not a reflex.

**The interview-grade sentence:** *"From November 16, 2026, Polly asks organizations earning at least $20,000 from a product that uses it for $20 a month — one fee per organization, source still BSD-3, no support contract attached. Under OSMF you pay for direct dependencies, not transitive ones, but Microsoft's resilience packages expose Polly types and Microsoft hasn't yet said how it will handle that — so the facts I'd verify first are whether we reference Polly directly and what Microsoft decides."*

---

## Concept 68 — The OSMF decision as an architect

The fee itself is $240 a year. **The decision isn't about $240.** It's about four things an architect owns:

1. **Compliance exposure** — do we reference Polly directly, and are we above the threshold?
2. **Procurement friction** — some organizations find it harder to approve a $20/month sponsorship than a $20,000 license.
3. **Supply-chain health** — Polly sits on the hot path of almost every outbound call; a well-funded maintainer team is a *security* property (vulnerability response, signed releases, timely updates for new .NET versions).
4. **Optionality** — can we change course later if terms or Microsoft's position change?

**The options, honestly compared:**

| Option | Cost | Risk | When it's right |
|---|---|---|---|
| **Pay the fee** (if it applies) | $240/yr + a procurement step | Lowest; funds a security-critical dependency | The default for a qualifying organization that references Polly directly |
| **Reference only Microsoft's packages** (no direct `Polly.*` references) | Refactoring custom pipelines onto `Microsoft.Extensions.*` APIs | Relies on the transitive exemption and on Microsoft's pending decision; the public API still exposes Polly types | Only if it's a natural fit anyway — restructuring code to dodge a trivial fee is poor engineering |
| **Pin the last version released before the fee** | Zero cash | No fixes — including security fixes — unless you build and maintain them yourself; verify which terms attach to that exact version | A short-term bridge while a decision is made, with a date on it |
| **Build from source internally** (BSD-3 permits it) | An internal fork to maintain | You own security response and .NET-version compatibility | Organizations with a policy against the fee and the capacity to maintain it |
| **Adopt the Fences fork** | Package and namespace change | Pre-release, small maintainer base, may be retired; doesn't remove Polly if you use Microsoft's resilience packages | Teams already aligned with Brighter, or willing to take early-adopter risk |
| **Replace** (SDK-native resilience, gRPC policies, mesh, Dapr, hand-rolled) | High | Rewriting well-tested behavior; losing telemetry and tooling | Only when other reasons already point there (Concept 69) |

**A defensible recommendation for a typical company** — and the shape of the answer an interviewer wants:

1. **Inventory** — search `PackageReference` and `Directory.Packages.props` for `Polly*` across repositories; list which products use Polly directly and which only via Microsoft's packages or other libraries.
2. **Get the compliance reading** from legal/procurement on the direct-vs-transitive question — that's their call, informed by your inventory.
3. **If the fee applies, pay it** — it's the cheapest risk reduction available and it funds a dependency you page people about.
4. **Keep the architecture insulated regardless**: Polly lives behind adapters and the composition root (Concept 59), so a future change is a bounded refactor, not a rewrite. Don't wrap Polly in a home-grown abstraction layer "just in case" — the adapter boundary *is* the insulation.
5. **Track the open items** — `dotnet/extensions#7719`, the maintainers' final terms — and **record the decision in an ADR** (Module 31) with a review date.

**The interview-grade sentence:** *"I'd treat Polly's fee as a supply-chain decision, not a $240 one: inventory direct references, let legal decide the direct-versus-transitive question, pay if it applies because funding a security-critical dependency is cheap risk reduction, keep Polly behind adapters so we retain optionality, and record it in an ADR that watches Microsoft's pending answer. I wouldn't fork or restructure code just to avoid the fee."*

---

## Concept 69 — Alternatives and complements: SDKs, gRPC, mesh, Dapr, hand-rolled

Polly isn't the only place resilience can live, and a senior answer knows where each alternative wins — including the ones that *complement* rather than replace it.

| Option | Strengths | Weaknesses | Where it wins |
|---|---|---|---|
| **SDK-native** (Azure SDK, Cosmos, EF Core, StackExchange.Redis) | Understands the service's failure semantics, throttling headers, failover | Configuration per SDK; different knobs everywhere | Always first for that SDK's dependency (Concept 65) |
| **gRPC service config** | Status-aware retries and hedging, commit points, server pushback, retry throttling | gRPC only; static per channel | gRPC dependencies (Concept 44) |
| **Service mesh** (Istio, Linkerd, Envoy) | Fleet-wide policy without code; **outlier ejection**; retry budgets; mTLS and telemetry | Platform complexity; no application context (it can't know a `POST` has an idempotency key) | Kubernetes platforms with a platform team; "bad replica" failures (Module 13, Concept 28) |
| **Dapr resiliency policies** | Declarative timeouts, retries and circuit breakers for Dapr service invocation, components and actors | Requires Dapr sidecars; scoped to Dapr building blocks | Teams already on Dapr |
| **API gateway** (e.g., Azure API Management) | Central quotas and retries at the edge; one place to enforce a partner quota | Only for traffic through it; adds a hop | Egress to rate-limited partners (Concept 26) |
| **Hand-rolled** | No dependency | Reinvented bugs: jitter, cancellation, breaker state machines, telemetry | Libraries that must not take dependencies — and then only a minimal retry-with-timeout |

**Combining layers without stacking.** A mesh plus in-process resilience is common and fine — *if* each failure kind still has one retry owner. Typical splits: **the mesh ejects bad replicas and enforces connection limits; the application owns retries**, because only the application knows which operations are idempotent. Or the reverse, with retries in the mesh and **retries disabled in the app's handlers** (timeouts and breakers kept). Never both retrying the same calls.

**The interview-grade sentence:** *"SDK-native resilience comes first for its own service, gRPC's service config for gRPC, a mesh for fleet-wide policy and outlier ejection, Dapr policies if we're on Dapr, a gateway for shared partner quotas — and Polly for application-aware resilience that needs to know about idempotency and degradation. Layers can coexist as long as each failure kind has a single retry owner."*

---

## Concept 70 — Migrating from Polly v7 and `Microsoft.Extensions.Http.Polly`

Many codebases still carry v7 policies — and `Microsoft.Extensions.Http.Polly` is deprecated. The mapping:

| v7 | v8 | Notes |
|---|---|---|
| `Policy`, `IAsyncPolicy`, `ISyncPolicy` (+ generic) | `ResiliencePipeline` / `ResiliencePipeline<T>` | Sync and async unified |
| `Policy.WrapAsync(outer, inner)` | Builder order: first added = outermost | Same semantics, less confusion |
| `WaitAndRetryAsync(n, attempt => delay)` | `AddRetry` with `Delay`, `BackoffType`, `UseJitter`, or `DelayGenerator` | Delay arrays → a generator, or two strategies (Concept 12) |
| `CircuitBreakerAsync(exceptionsAllowedBeforeBreaking, …)` | **No equivalent** | Approximate with `FailureRatio = 1.0` and a small `MinimumThroughput` (Concept 17) |
| `AdvancedCircuitBreakerAsync(…)` | `AddCircuitBreaker` | The v8 model |
| `TimeoutAsync(…, TimeoutStrategy.Pessimistic)` | **No equivalent** | Make code cancellable, or `WaitAsync` explicitly (Concept 10) |
| `BulkheadAsync(maxParallelization, maxQueuing)` | `AddConcurrencyLimiter(permitLimit, queueLimit)` | `Polly.RateLimiting` |
| `RateLimitAsync(…)` | `AddRateLimiter(new SlidingWindowRateLimiter(…))` | BCL limiters |
| Cache policy | **None** | Use `HybridCache` / `IMemoryCache` (Module 10) |
| `FallbackAsync` | `AddFallback` | Typed only |
| `Context` | `ResilienceContext` + `ResiliencePropertyKey<T>` | Pooled; `CorrelationId` removed (use `Activity`) |
| `PolicyRegistry` | `ResiliencePipelineRegistry` / `Provider` | Append-only; reloads instead of `AddOrUpdate` |
| `Policy.NoOpAsync()` | `ResiliencePipeline.Empty` | |
| `ExecuteAndCaptureAsync` | `ExecuteOutcomeAsync` | Async only |
| `AddTransientHttpErrorPolicy`, `AddPolicyHandler` | `AddStandardResilienceHandler`, `AddResilienceHandler` | `Microsoft.Extensions.Http.Resilience` |

**The migration path Polly recommends:**

1. **Upgrade the `Polly` package to 8.x.** The v7 API still ships in it, so nothing breaks.
2. **Migrate one policy at a time.** For mixed code, `pipeline.AsAsyncPolicy()` / `AsSyncPolicy()` lets a new v8 pipeline participate in an old v7 `PolicyWrap`.
3. **Move HTTP clients** from `Microsoft.Extensions.Http.Polly` to `Microsoft.Extensions.Http.Resilience`.
4. **Switch the reference** from `Polly` to `Polly.Core` (+ `Polly.Extensions`, `Polly.RateLimiting`) once no v7 API remains.

**Behavioral changes to re-verify, not just re-compile:** consecutive-count breakers become ratio breakers (low-traffic dependencies may stop opening — Concept 17); pessimistic timeouts disappear (uncancellable code now times out late); predicate defaults may differ from what the old policy handled (Concept 4); and timeouts and delays should be re-derived from current latency data rather than transliterated. Put a descriptor test (Concept 53) and a behavior test (Concept 54) on each migrated pipeline.

**The interview-grade sentence:** *"Migrating from v7, policies become pipelines, PolicyWrap becomes builder order, bulkhead becomes a concurrency limiter, and Microsoft.Extensions.Http.Polly gives way to Http.Resilience; I go policy by policy with the interop methods. The traps are behavioral: no consecutive-count breaker, no pessimistic timeout, and different predicate defaults — so each migrated pipeline gets behavior tests, not just a successful compile."*

---

## Concept 71 — Performance: what Polly costs and when it matters

Polly v8 was redesigned around allocation-conscious, `ValueTask`-based execution; the repository publishes BenchmarkDotNet results for its strategies (`bench/`). On the happy path, the overhead of a pipeline is **negligible next to any network call** — you won't find Polly in a profile of an HTTP-bound service unless something is misused. The things that *do* show up:

| Cost | Cause | Fix |
|---|---|---|
| Allocations and a broken breaker | Building a pipeline **per call** | Build once via DI/registry (Concept 45) |
| Closure allocations on hot paths | Capturing lambdas | `static` lambdas with `TState` overloads (Concept 5) |
| Exception storms during outages | `BrokenCircuitException` thrown for every rejected call at high RPS | `ExecuteOutcomeAsync` on the hottest paths (Concept 3) |
| Log volume | "Pipeline executed" and successful "execution attempt" logs at Information for every call | `SeverityProvider` → Debug (Concept 50) |
| Metric cardinality | Unbounded `operation.key` or instance names | Bounded names only (Concepts 46, 50) |
| Wasted work | Resilience around **in-process** calls (cache lookups, pure functions) | Don't wrap them at all (Concept 74) |
| Unbounded pipelines | Per-tenant registry keys | Bucket keys (Concept 46) |

For high-RPS services, measure with BenchmarkDotNet and allocation profiling as Module 17 describes — and benchmark your *predicates* and *generators* too, since they run on every execution.

**The interview-grade sentence:** *"Polly v8's happy-path cost is negligible next to a network call; what shows up in profiles is misuse — pipelines built per call, closures on hot paths, exceptions thrown for every rejected call during an outage, per-call Information logs, and unbounded tags. The fixes are build once, static lambdas with state, ExecuteOutcomeAsync where it's hot, severity tuning, and not wrapping in-process calls at all."*

---

## Concept 72 — Anti-patterns catalogue

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Pipeline built per call** | Breaker never opens, limiter never limits, allocations | Registry / DI, build once |
| **Default `ShouldHandle`** | Retries bugs; ignores 5xx results | Explicit per-dependency classifiers |
| **Same predicate for retry and breaker** | 429s open the breaker; 500s retried | Separate "should retry" and "is unhealthy" |
| **Retry handles `BrokenCircuitException` / `RateLimiterRejectedException`** | Burns attempts against your own protection | Don't handle; honor `RetryAfter` only in background work |
| **Attempt timeout outside the breaker** | Slowness never trips the breaker | Breaker outside the attempt timeout |
| **No total timeout** | Retries escape the caller's budget | Total timeout outermost-but-one |
| **Callback ignores the inner token** | Timeouts and hedges are late or useless | Pass the pipeline's token; CA2016 |
| **Generated timeout ≤ 0 for expired deadlines** | Disables the timeout entirely | Clamp positive; deadline guard (Concept 57) |
| **`HttpClient.Timeout` below the pipeline total** | Silent budget cut; breaker blind | `HttpClient.Timeout` above the total |
| **Standard handler left at defaults** | 10 s attempts, POST retries, breaker counting 429 | Tune per dependency (Concept 40) |
| **Two resilience handlers on one client** | Double retries | One handler; `RemoveAllResilienceHandlers` to replace |
| **Global defaults applied blindly** (`ConfigureHttpClientDefaults`) | Library clients and POST-heavy clients inherit retries | Review every client the defaults touch |
| **Polly retry around EF Core or Azure SDK calls** | Double retry | Configure the built-in owner (Concepts 61, 65) |
| **Retrying non-idempotent writes** | Duplicate orders and charges | Idempotency keys; `NeverSent` rule (Concepts 13, 42) |
| **Non-replayable request content with retries** | Second attempt fails or sends an empty body | Buffered content; no retries for streams |
| **Retry and hedging on the same call** | Multiplicative load | One or the other |
| **Hedging non-idempotent or to the same replica** | Duplicates; no latency gain | Idempotent reads to independent replicas |
| **Fallback inside the retry or breaker** | Hides failures from both | Fallback outermost |
| **Guarding execution with the breaker's state provider** | Circuit never half-opens | `ExecuteOutcomeAsync` if you want no exceptions |
| **Unbounded registry keys** (per tenant, per URL) | Memory growth, cardinality explosion | Bucketed keys, partitioned limiters |
| **Hot-reloading options during an incident** | Resets breakers and bulkheads | Change deliberately; know the reset |
| **Chaos strategies without a gate** | 0.1% injected faults in production | `EnabledGenerator` everywhere |
| **Long in-process retries in message handlers** | Lock expiry, duplicate processing | Quick retries, then scheduled redelivery |
| **Retry as a scheduler** | Memory held for hours; not durable | A real scheduler |
| **Leaking Polly exceptions into handlers** | Business code coupled to infrastructure | Adapters translate (Concept 59) |
| **No telemetry names** | Can't tell attempt from total timeouts | Name pipelines and strategies |

---

## Concept 73 — The resilience code review

A checklist you can walk through, in order, for any `AddResilienceHandler`, `AddStandardResilienceHandler` or `AddResiliencePipeline` call — and narrate in an interview when asked to review a snippet:

**Ownership**
1. What failure kinds does this pipeline handle, and is it the **only retry owner** for them (SDK, EF, gRPC channel, mesh, broker, global defaults)?
2. Is the pipeline **built once** and resolved from DI or the registry?

**Composition**
3. Is the **order** canonical — fallback, bulkhead, total timeout, retry/hedging, quota, breaker, attempt timeout, chaos?
4. Are **predicates explicit** and different for retry and breaker? Are 429s kept out of the breaker? Is caller cancellation excluded everywhere?

**Numbers**
5. Is the attempt timeout from **measured** p99/p99.9? Does the total cover attempts plus delays and fit the **caller's budget**? Is `HttpClient.Timeout` above the total?
6. Retries: how many, what backoff, **jitter on**, which **methods**, **idempotency keys**, **replayable content**, a **budget**?
7. Breaker: **scope** (authority, key), ratio and window from data, window ≥ 2 × attempt timeout, minimum throughput below quiet-hour traffic, jittered growing break?
8. Limiters: bulkhead sized by Little's Law, **placement** (bulkhead outside, quota inside), **shared** where the constraint is shared, per-process arithmetic?
9. Hedging: idempotent, independent replicas, delay ≈ p95, one hedging layer, off-switch under load?
10. Fallback: outermost, explicit, visibly degraded?

**Operations**
11. **Names** for pipeline, strategies and operation keys; severity tuned; dashboards and alerts exist?
12. **Tests**: predicate tables, a descriptor test, behavior tests with fake time, and a host-level attempts-at-the-dependency test?
13. **Config**: validated at startup, staged like code, reload consequences understood?
14. **Chaos** strategies gated?
15. **Governance**: package versions current, the OSMF position recorded (Concept 68)?

You won't cover all fifteen in an interview. Pick the three that matter most for the snippet in front of you — usually **ownership, order, and numbers** — and say why.

---

## Concept 74 — When not to

**Don't wrap in-process calls.** Cache lookups, pure computations, in-memory repositories: there's no transient fault to handle, and a timeout can't stop CPU-bound code anyway.

**Don't add Polly retries where an owner already exists** — EF Core's execution strategy, Azure SDK clients, the Cosmos SDK, gRPC channel policies, a mesh with retries turned on, a broker with redelivery. Add an outer timeout if you need one; nothing else.

**Don't put a breaker on low-volume calls** that can never reach a meaningful minimum throughput — a timeout and a bulkhead give you fail-fast behavior without a mode you can't test.

**Don't hedge** calls that aren't latency-critical, aren't idempotent, or would hit the same bottleneck.

**Don't write fallbacks you won't exercise** — a fallback path that runs once a year is a second, untested system (Module 13, Concept 39).

**Don't retry writes without idempotency.** If you can't make the operation safe to repeat, the correct resilience for it is a clear failure plus a queryable outcome, not a retry.

**Don't use resilience to paper over bugs.** A retry that "fixes" a `NullReferenceException` one time in three is hiding a race condition.

**Don't build per-tenant pipelines**, custom strategies that duplicate built-ins, or abstractions over Polly "in case we switch" — the adapter boundary already gives you that.

**Don't accept global defaults unexamined.** Aspire's service defaults are a good starting point for a demo and a reasonable default for many internal read-heavy clients — but every client they touch should be looked at once, especially those issuing non-idempotent writes.

**The meta-rule, from Module 13:** *every resilience mechanism is also a new failure mode.* The best Polly configuration is the smallest one that meets the stated targets — with every number sourced, every owner named, and every behavior tested.

**The interview-grade sentence:** *"I don't wrap in-process calls, don't add retries where the SDK, EF Core, gRPC, the mesh or the broker already owns them, don't put breakers on calls too quiet to gather statistics, don't hedge or retry non-idempotent writes, and don't keep fallbacks I won't exercise. The goal is the smallest pipeline that meets the target, with every number sourced and every owner named."*

---

# Putting it together

## Worked example 1 — "Design the resilience for an Orders API built on Aspire."

*The Orders API calls Catalog (internal HTTP, reads), Payments (an external provider: `POST /captures`, 50 requests/second quota per API key, returns 429 with `Retry-After`), reads order views from Cosmos DB, writes orders to Azure SQL through EF Core, and publishes integration events to Service Bus through an outbox. The team uses Aspire service defaults.*

Narrate it in this order:

1. **Classify and assign owners first** (Concepts 59–60; Module 13, Concept 6). Catalog is *soft* for order placement (prices come from the command) but *hard* for the product page; Payments is *hard* for completion but can be made *asynchronous*; Cosmos views are *soft* (stale is acceptable briefly); SQL is *hard*; Service Bus is *asynchronous* behind the outbox. Owners: the HTTP handlers for Catalog and Payments transport faults; EF Core's execution strategy for SQL; the Cosmos and Service Bus SDKs for themselves; the command pipeline for concurrency; the outbox relay for publishing.

2. **Keep Aspire's global standard handler as the default — but review every client it touches** (Concept 43). For Catalog, replace it with a *tuned* standard handler (Concept 40): 350 ms attempt from a 250 ms p99.9, 1.5 s total (3 × 350 ms + ~300 ms of delays fits — Concept 37), two jittered retries from 100 ms, breaker 50% over 10 s with a minimum of 20 and a predicate that ignores 429.

3. **Payments gets its own pipeline, not the default** — `RemoveAllResilienceHandlers()` then a custom handler:
   - **Idempotency key** per capture, derived from the order ID, set in the typed client (Concept 42).
   - **Bulkhead outermost** (64 permits), **total 4 s**, **one retry** (keyed POSTs only), a **quota limiter inside the retry** at 12 requests/second per instance (50/s across four instances, with headroom) so retries spend quota (Concept 26), a **breaker that ignores 429**, a **1.5 s attempt timeout**.
   - `Retry-After` honored with a deadline check in `ShouldHandle` (Concept 14).

4. **Make Payments degradable instead of fragile.** When the pipeline gives up (open circuit, quota exhausted, timeout), the adapter returns `PaymentUnavailable`; the handler accepts the order as `PendingPayment` and records a "capture requested" row in the outbox; a background worker performs the capture with the **same idempotency key** when the provider recovers (Module 13, Concept 27). A payment outage becomes a backlog, not lost revenue.

5. **Cosmos and Service Bus: configure, don't wrap** (Concept 65). Cosmos with preferred regions and a 5 s cap on 429 waits for the request path; Service Bus with a 10 s try timeout. No Polly retries around either.

6. **SQL and concurrency** (Concepts 61–62): `EnableRetryOnFailure(2, 1 s)`; the command pipeline's concurrency-retry behavior (three jittered retries, change tracker cleared per attempt) outside the unit of work; the Payments call moved *out* of the transactional handler into the outbox so re-running the handler never re-charges.

7. **Telemetry** (Concepts 49–51): `AddMeter("Polly")`; strategies named `total`/`attempt`/`quota`/`bulkhead`; severity tuned; dashboards for retry ratio, breaker transitions, quota vs bulkhead rejections; a page on a sustained open Payments circuit.

8. **Tests** (Concepts 52–56): predicate tables; descriptor tests for each pipeline; a fake-time behavior test proving a `POST` without a key isn't retried; and a host-level test that fails Payments with 503 and asserts **exactly two** attempts at the fake provider. Chaos (gated to a synthetic tenant) in staging: 2 s latency on Catalog and 503s on Payments, hypothesis "order placement success stays above 99%."

A compact sketch of the registrations:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();                                   // Aspire: standard handler on every HttpClient

builder.Services.AddHttpClient<CatalogClient>(c => c.BaseAddress = new("https+http://catalog"))
    .RemoveAllResilienceHandlers()                              // replace the global default — never stack a second one
    .AddStandardResilienceHandler()
    .Configure(builder.Configuration.GetSection("Resilience:Catalog"));   // tuned numbers from Concept 40, bound from config

builder.Services.AddHttpClient<PaymentsClient>(c =>
    {
        c.BaseAddress = new(builder.Configuration["Payments:BaseUrl"]!);
        c.Timeout = TimeSpan.FromSeconds(6);                    // backstop above the 4 s total
    })
    .RemoveAllResilienceHandlers()                              // no global default for payments
    .AddResilienceHandler("payments", PaymentsResilience.Configure);

builder.Services.AddDbContext<OrderingDbContext>(o => o.UseSqlServer(
    builder.Configuration.GetConnectionString("orders"),
    sql => sql.EnableRetryOnFailure(maxRetryCount: 2, maxRetryDelay: TimeSpan.FromSeconds(1), errorNumbersToAdd: null)));

builder.Services.AddResiliencePipeline("command-concurrency", ConcurrencyResilience.Configure);   // Concept 62
builder.Services.AddOpenTelemetry().WithMetrics(m => m.AddMeter("Polly"));
```

Close by **saying what you're not doing and why**: no hedging (no latency requirement justifies extra load on Payments, and it's a write), no retries around Cosmos, SQL or Service Bus (their owners exist), no per-tenant pipelines (partitioned limiters would do if tenant fairness toward the provider quota becomes a requirement).

---

## Worked example 2 — "One failing call shows up as 48 requests at the dependency. Explain and fix."

*After a platform team rolled out Aspire service defaults, an Inventory outage caused Inventory to receive 48 requests for every user request, and it stayed overloaded for 20 minutes after the root cause was fixed.*

1. **Reconstruct the layers** (Concept 60). The BFF's client to Orders has the global standard handler: **4 attempts**. In Orders, a command handler wraps its Inventory call in a hand-written Polly retry with 2 retries: **3 attempts**. Orders' `InventoryClient` *also* has the global standard handler: **4 attempts**. For the same failure kind — Inventory unavailable — that's 4 × 3 × 4 = **48** attempts at Inventory per user request.

2. **Explain the 20 minutes** (Module 13, Concept 10). Once Inventory was fixed, the retry volume kept it saturated — a metastable state sustained by retries. The breakers didn't help: at the incident's traffic level, each client saw fewer than 100 calls per 30-second window, so the default `MinimumThroughput` of 100 meant **no breaker ever evaluated** (Concept 17).

3. **The fixes, in priority order:**
   - **Remove the hand-written retry from the handler** — resilience belongs in the adapter (Concept 59).
   - **Make Orders the single owner** of transport retries to Inventory: tune its handler to one retry, 300 ms attempts, a 1 s total (Concept 40).
   - **Stop the BFF from retrying** calls to Orders that Orders has already retried: Orders returns 503 with a "don't retry" marker when its own pipeline gives up, and the BFF's predicate honors it (Module 13, Concept 18) — or the BFF's handler for that client keeps timeouts and a breaker but no retries.
   - **Fix the breakers**: 50% ratio over 10 s with a minimum throughput of 10 (Concept 22), so they open at incident traffic.
   - **Add a retry budget** on Orders' Inventory client (Concept 15).

4. **Prove it** with the host-level test that counts attempts at a fake Inventory (Concept 56): the assertion "≤ 2 attempts per request" now lives in CI, where the next global-defaults rollout would fail it.

5. **The principle to close with:** *global defaults are a policy for every client, including the ones whose owners never thought about retries — every client they touch must be reviewed, and every failure kind must have one owner.*

---

## Worked example 3 — "Product-detail reads have p95 of 120 ms but p99 of 2 s. Would you hedge?"

1. **Find where the tail comes from first.** If one slow replica or GC pauses on the same instance cause it, hedging to *another replica* helps; if every replica is slow for the same keys (a hot partition), hedging adds load and gains nothing (Module 13, Concept 38). Check per-instance latency in the traces.

2. **Pick the layer** (Concept 31). If product details come from **Cosmos DB**, use the SDK's **cross-region availability strategy** — it understands regions and consistency. If from an **internal HTTP service** with replicas in two regions, use the **standard hedging handler** with ordered groups (Concept 30).

3. **Configure it:** one hedged attempt, delay ≈ **150 ms** (just above the p95), per-endpoint breakers (50% / 10 s / minimum 20), 800 ms per-endpoint attempt timeout, 2 s total. Reads only — `GET` is idempotent.

4. **Budget it:** at a 150 ms delay about 5% of requests hedge when healthy; during a partial outage more hedge, but per-endpoint breakers stop hedges into the failing region. Add the dynamic-mode switch so hedging falls back to sequential failover when the service is shedding load (Concept 28).

5. **Measure it:** hedge rate (~5% expected), hedge win rate (should be meaningful — otherwise the hedges aren't winning), p99 before and after, and downstream load. If the win rate is low, the tail isn't per-replica and hedging should come out.

---

## Worked example 4 — "Leadership read that Polly is 'going paid.' What do we do?"

An architect's answer is a short ADR (Module 31), not a hot take.

- **Context.** Polly adopts the Open Source Maintenance Fee on November 16, 2026: $20/month per organization earning ≥ $20,000 from a product using Polly; source remains BSD-3; not a support contract (Concept 67). Our services use Polly directly in *N* repositories (inventory attached) and indirectly through `Microsoft.Extensions.Http.Resilience` in *M* more. Microsoft's position on its resilience packages is pending (`dotnet/extensions#7719`).
- **Decision drivers.** Compliance, procurement effort, supply-chain health of a dependency on every outbound call, and optionality (Concept 68).
- **Options considered.** Pay; restrict to Microsoft's packages; pin; internal build from source; the Fences fork; replacement.
- **Decision.** If legal confirms we qualify for our direct references, pay via GitHub Sponsors. Keep Polly behind adapters (already true); no forks, no home-grown abstraction layer.
- **Consequences.** $240/year; one procurement item; no code change; security fixes continue to flow.
- **Review triggers.** Microsoft's answer on `dotnet/extensions#7719`; any change in the maintainers' terms; a policy decision against maintenance fees organization-wide.

---

## Common questions and what a strong answer contains

**"How does Polly v8 work?"** Immutable pipeline of strategies built once; first added is outermost; everything becomes an `Outcome<T>`; reactive strategies judge outcomes with `ShouldHandle`, proactive ones cancel or refuse; pooled `ResilienceContext` carries the token and properties (Concepts 2–7).

**"Why must a pipeline be built once?"** Breaker windows and limiter permits live in the instance; per-call pipelines never trip or limit — and allocate (Concepts 2, 45).

**"What's wrong with Polly's defaults?"** The retry default is three retries two seconds apart with no jitter; the default predicate handles all exceptions but no results; the breaker's minimum throughput of 100 disables it for quiet dependencies; the standard HTTP handler retries POSTs and counts 429 in the breaker (Concepts 4, 11, 17, 39).

**"How does the timeout strategy work?"** Links and cancels a token, waits for the callback to notice, throws `TimeoutRejectedException`; caller cancellation passes through unchanged; no pessimistic mode; a non-positive generated timeout disables it (Concepts 8–10).

**"How do you choose retry settings?"** One or two jittered exponential retries from ~100 ms in request paths, explicit per-dependency predicates, a total timeout, a budget, and the worst-case latency envelope computed up front (Concepts 11–15).

**"How does Polly's circuit breaker decide to open?"** Failure ratio over a rolling window above a minimum throughput; time to open ≈ window × ratio; lazy transitions; one half-open probe; `BrokenCircuitException.RetryAfter` (Concepts 17–18).

**"How would you tune a breaker?"** Healthy noise, required reaction time, lowest traffic per window, window ≥ 2 × attempt timeout; start near 50%, replay incidents (Concept 22).

**"How do you implement a bulkhead in .NET?"** `AddConcurrencyLimiter` — the v8 bulkhead — outermost, sized by Little's Law; a quota limiter goes inside the retry; limits are per process (Concepts 24–26).

**"What is hedging in Polly and when would you use it?"** Typed strategy that relaunches after a delay *or* on failure; latency/parallel/fallback/dynamic modes; losers cancelled and awaited; idempotent reads to independent replicas; one hedging layer (Concepts 27–31).

**"Where does fallback go?"** Outermost, explicit predicates, visible degradation; prefer fallbacks that do less (Concepts 32–33).

**"What order do you compose strategies in, and why?"** Fallback → bulkhead → total → retry/hedge → quota → breaker → attempt → chaos, with the reason for each (Concept 34) — then trace an execution (Concept 35).

**"What does `AddStandardResilienceHandler` do?"** The five strategies and their defaults, what they handle, and why you tune it per dependency (Concepts 39–40).

**"How do `HttpClient.Timeout` and the resilience handler interact?"** Effective budget is the minimum; `HttpClient.Timeout` looks like caller cancellation to the pipeline; handler timeouts end at headers (Concept 41).

**"Can you retry a POST?"** Only with an idempotency key the server deduplicates on, replayable content, or when `HttpRequestError` shows the request never left (Concepts 13, 42).

**"You use Aspire service defaults — what should you check?"** Every client gets retries on all methods; library clients included; override by removing, not stacking; count attempts in a host-level test (Concepts 43, 56).

**"How do Polly and EF Core's retries work together?"** They don't overlap: the execution strategy owns transient DB faults; Polly adds only timeouts or bulkheads; concurrency conflicts belong to the command layer (Concepts 61–62).

**"Where does the concurrency retry go in a MediatR pipeline?"** Outside the unit of work, inside idempotency, handling only concurrency exceptions, clearing the change tracker; side effects to the outbox (Concept 62).

**"How do you handle retries in a Service Bus consumer?"** Tiny in-process retry, then scheduled redelivery; abandon redelivers immediately; dead-letter after a bound; pause consumption via a breaker signal (Concept 64).

**"How do you test resilience?"** Predicate tables, descriptor tests, fake-time behavior tests, host-level attempt counting, gated chaos (Concepts 52–56).

**"What does dynamic reload do?"** Rebuilds the pipeline; state resets; a failed reload keeps the old pipeline and stops reloading (Concept 47).

**"What metrics do you watch?"** Retry ratio, hidden failure rate, attempt vs total timeouts, breaker transitions, rejections by limiter, hedge rate and win rate, fallback rate (Concepts 49–51).

**"What's the situation with Polly's licensing?"** OSMF facts, direct vs transitive, Microsoft's pending position, the options, and your recommendation (Concepts 67–68).

**"Polly or a service mesh?"** Both can coexist with one retry owner per failure kind; the mesh wins for outlier ejection and fleet-wide policy, Polly for idempotency- and degradation-aware decisions (Concept 69).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "We use Polly for resilience" | Names each strategy's job, its numbers, and their source |
| Builds a pipeline inside the method that uses it | Builds once via the registry; explains that state lives in the instance |
| Leaves `ShouldHandle` at default | Writes per-dependency classifiers; separate retry and breaker predicates |
| Copies Polly's retry defaults | One or two jittered exponential retries, total timeout, worst-case envelope computed |
| Retries `BrokenCircuitException` | Lets it propagate; uses `RetryAfter` only in background work |
| Puts the attempt timeout outside the breaker | Breaker outside the attempt timeout so slowness counts |
| Forgets the total timeout | Total timeout outside the retry, tied to the caller's budget or propagated deadline |
| Ignores the token inside callbacks | Uses the pipeline's token everywhere; CA2016 as an error |
| Assumes the breaker works at default settings | Checks minimum throughput against real traffic; knows time-to-open ≈ window × ratio |
| Guards `Execute` with the breaker's state | Uses `ExecuteOutcomeAsync`; knows transitions are lazy |
| Uses one limiter position for everything | Bulkhead outermost, quota inside the retry, shared per constraint |
| Treats in-process limits as global | Divides by instance count or centralizes egress |
| Hedges writes, or hedges to the same replica | Hedges idempotent reads to independent replicas, ~p95 delay, one layer, budgeted |
| Fallback inside the retry | Fallback outermost, explicit, visibly degraded |
| Leaves `AddStandardResilienceHandler` untuned | Tightens timeouts, disables unsafe-method retries, removes 429 from the breaker |
| Sets `HttpClient.Timeout` below the pipeline total | Makes the pipeline total the budget; `HttpClient.Timeout` as backstop |
| Retries requests with streaming bodies | Checks replayability; resumable protocols for uploads |
| Generates idempotency keys in a handler per call | Derives keys from the business operation, above the retry |
| Stacks a custom handler on Aspire's global one | `RemoveAllResilienceHandlers` then one handler; host-level attempt test |
| Adds the HTTP standard handler to gRPC clients as-is | Channel retry/hedging for status-aware retries; handler without retries |
| Wraps EF Core or Azure SDK calls in Polly retries | Tunes the built-in owner; adds only timeouts |
| Puts a retry loop in the handler/use case | Resilience in adapters; concurrency retries in the command pipeline |
| Retries the command without thinking about side effects | Outbox or command-derived idempotency keys for external calls |
| Confuses the re-decide loop with append retries | Two loops: new decision on conflict, same event IDs on transport retry |
| Sleeps in a message handler | Quick retry, then scheduled redelivery; dead-letter after a bound |
| Hot-reloads options mid-incident | Knows reload resets breakers and limiters, and that a failed reload stops reloading |
| Leaves chaos strategies ungated | `EnabledGenerator` everywhere; experiments with hypotheses |
| Tests that "retry retries" | Tests configuration, delegates, and attempts at the dependency |
| Uses unbounded registry keys or tags | Bounded keys and tags; partitioned limiters for tenants |
| "Polly is going paid, let's fork it" | Inventories direct references, lets legal decide, pays if it applies, keeps adapters as insulation, writes an ADR |

---

## Practice exercises

**Exercise 1 — Count the attempts (the most valuable one).** In a service you own, write the host-level test from Concept 56 for each outbound dependency: fail it with 503 and assert the number of attempts that reach it. Then add a second service in front with default settings and repeat. Record every layer you find.

**Exercise 2 — Trace your pipeline.** Take one real pipeline and write the Concept 35 trace for it with your actual numbers: a hanging dependency, a failing dependency, and a caller who cancels at 300 ms. Where does the total timeout fire? When does the breaker open? What exception does the caller see?

**Exercise 3 — Descriptor and behavior tests.** For the same pipeline, write a `Polly.Testing` descriptor test (order and key options) and two fake-time behavior tests: "503 is retried exactly N times" and "a POST without an idempotency key is not retried." Make them run in under 100 ms.

**Exercise 4 — Breaker reaction curves.** Simulate a dependency at 200 RPS and drive failure rates of 30%, 60% and 100% into breakers with (10%, 30 s), (50%, 30 s), (50%, 10 s) and (100%, 10 s, minimum 5). Measure time-to-open and compare with the window × ratio rule (Concept 17). Then drop traffic to 2 RPS and see which breakers still work.

**Exercise 5 — Retry budget.** Implement Concept 15's budget and chart the load multiplier on the dependency at 10%, 50% and 100% failure, compared with a fixed two-retry policy. Add a metric for refused retries.

**Exercise 6 — The `HttpClient.Timeout` trap.** Configure a pipeline with a 3 s total and two retries, set `HttpClient.Timeout` to 2 s, make the dependency hang, and observe: which exception reaches the caller, how many attempts happen, and whether the breaker records anything. Fix it and repeat.

**Exercise 7 — Replay failures.** Send a `POST` with `StreamContent` over a non-seekable stream through a retrying handler against a stub that fails the first attempt. Observe what the second attempt sends. Then switch to buffered content and add an inner handler that *appends* a header per attempt — observe the duplicates, then fix it.

**Exercise 8 — The command pipeline.** Implement Concept 62's concurrency-retry behavior. Write a test that forces a `DbUpdateConcurrencyException` on the first attempt and asserts (a) the aggregate is re-read, (b) the second decision is based on fresh state, and (c) an external side effect is recorded in the outbox exactly once. Then simulate a lost commit acknowledgment and assert the stored outcome is returned.

**Exercise 9 — Gated chaos.** Add fault, latency and outcome chaos strategies behind an `IChaosManager` that targets one synthetic tenant. Write a one-page experiment (steady state, hypothesis, blast radius, abort condition), run it in a test environment, and check the Polly metrics show what you predicted.

**Exercise 10 — The OSMF inventory and ADR.** Run `dotnet list package --include-transitive --format json` across your solutions and separate direct `Polly*` references from transitive ones (for example via `Microsoft.Extensions.Http.Resilience`). Write the ADR from Worked example 4 for your organization, including review triggers.

**Exercise 11 — v7 migration with parity.** Take a v7 `PolicyWrap` (or write one: wait-and-retry + consecutive-count breaker + pessimistic timeout), migrate it to v8, and write behavior tests that expose the three semantic differences from Concept 70. Decide, with numbers, how you'd re-tune each.

---

## Free resources

### Polly — official documentation and source

| Resource | What it covers |
|---|---|
| [Polly documentation](https://www.pollydocs.org/) | The v8 docs home |
| [Resilience pipelines](https://www.pollydocs.org/pipelines/index.html) · [Pipeline registry](https://www.pollydocs.org/pipelines/resilience-pipeline-registry.html) | Building, executing, caching, reloading (Concepts 2, 45–47) |
| [Resilience strategies overview](https://www.pollydocs.org/strategies/index.html) | Reactive vs proactive; fault handling with `ShouldHandle` (Concepts 4, 7) |
| [Retry](https://www.pollydocs.org/strategies/retry.html) | Defaults, delay calculation, jitter, anti-patterns (Concepts 11–16) |
| [Circuit breaker](https://www.pollydocs.org/strategies/circuit-breaker.html) | Defaults, state diagrams, manual control, anti-patterns (Concepts 17–22) |
| [Timeout](https://www.pollydocs.org/strategies/timeout.html) | Cooperative cancellation, generators, anti-patterns (Concepts 8–10) |
| [Rate limiter](https://www.pollydocs.org/strategies/rate-limiter.html) | Concurrency and rate limiters, partitioned and chained limiters, disposal (Concepts 23–26) |
| [Hedging](https://www.pollydocs.org/strategies/hedging.html) | Concurrency modes, contexts, action generator (Concepts 27–31) |
| [Fallback](https://www.pollydocs.org/strategies/fallback.html) | Fallback after retries and other patterns (Concepts 32–33) |
| [Resilience context](https://www.pollydocs.org/advanced/resilience-context.html) | Pooling, properties, operation keys (Concept 5) |
| [Dependency injection](https://www.pollydocs.org/advanced/dependency-injection.html) | Keyed services, complex keys, dynamic reloads, disposal (Concepts 45–47) |
| [Telemetry](https://www.pollydocs.org/advanced/telemetry.html) | Instruments, tags, logs, enrichment, severity (Concepts 49–50) |
| [Testing](https://www.pollydocs.org/advanced/testing.html) | `Polly.Testing` descriptors and mocking providers (Concepts 52–53) |
| [Performance](https://www.pollydocs.org/advanced/performance.html) | Low-allocation APIs and patterns (Concept 71) |
| [Chaos engineering](https://www.pollydocs.org/chaos/index.html) | Fault, latency, outcome, behavior strategies; selective injection (Concept 55) |
| [Extensibility](https://www.pollydocs.org/extensibility/index.html) · [Proactive strategy](https://www.pollydocs.org/extensibility/proactive-strategy.html) · [Reactive strategy](https://www.pollydocs.org/extensibility/reactive-strategy.html) | Writing custom strategies (Concept 57) |
| [Migration guide v7 → v8](https://www.pollydocs.org/migration-v8.html) | The complete mapping and interop (Concept 70) |
| [Polly on GitHub](https://github.com/App-vNext/Polly) · [Releases](https://github.com/App-vNext/Polly/releases) | Source, changelog (8.7.0 and 8.8.0 notes) |
| [Polly samples](https://github.com/App-vNext/Polly/tree/main/samples) · [Polly-Samples repository](https://github.com/App-vNext/Polly-Samples) | Runnable examples, including extensibility and DI |
| [Polly benchmarks](https://github.com/App-vNext/Polly/tree/main/bench) | BenchmarkDotNet results for strategies (Concept 71) |
| [Transient fault handling and proactive resilience engineering](https://github.com/App-vNext/Polly/wiki/Transient-fault-handling-and-proactive-resilience-engineering) | The role each strategy plays, from Polly's wiki |
| [Polly.Contrib.WaitAndRetry](https://github.com/Polly-Contrib/Polly.Contrib.WaitAndRetry) | Origin and rationale of the decorrelated-jitter-v2 algorithm (Concept 12) |
| [Polly v8 officially released](https://thepollyproject.org/2023/09/28/polly-v8-officially-released.html) | The design goals of the v8 rewrite |

### Microsoft .NET documentation and blogs

| Resource | What it covers |
|---|---|
| [Introduction to resilient app development](https://learn.microsoft.com/en-us/dotnet/core/resilience/) | `Microsoft.Extensions.Resilience`, pipelines in DI, enrichment |
| [Build resilient HTTP apps](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience) | **The reference for Part I**: standard and hedging handler defaults, routing, custom handlers, reloads, known issues |
| [HttpClient guidelines for .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines) | Lifetimes, `PooledConnectionLifetime`, resilience with static clients (Concept 44) |
| [IHttpClientFactory with .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory) | Handler chains and lifetimes (Concept 38) |
| [Building resilient cloud services with .NET 8](https://devblogs.microsoft.com/dotnet/building-resilient-cloud-services-with-dotnet-8/) | Why Microsoft built on Polly v8, and the design of the packages |
| [Circuit breaker policy fine-tuning best practice](https://devblogs.microsoft.com/dotnet/circuit-breaker-policy-finetuning-best-practice/) | Thresholds and break durations (Concept 22) |
| [Announcing rate limiting for .NET](https://devblogs.microsoft.com/dotnet/announcing-rate-limiting-for-dotnet/) · [`System.Threading.RateLimiting` API](https://learn.microsoft.com/en-us/dotnet/api/system.threading.ratelimiting) | The limiter primitives (Concept 23) |
| [ASP.NET Core rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) | Inbound limits, chained limiters (Concepts 25, 66) |
| [gRPC retries](https://learn.microsoft.com/en-us/aspnet/core/grpc/retries) · [gRPC deadlines and cancellation](https://learn.microsoft.com/en-us/aspnet/core/grpc/deadlines-cancellation) | Channel retry and hedging policies, commit points, deadline propagation (Concept 44) |
| [EF Core connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) | Execution strategies, transactions, commit-failure handling (Concept 61) |
| [TimeProvider overview](https://learn.microsoft.com/en-us/dotnet/standard/datetime/timeprovider-overview) · [Microsoft.Extensions.TimeProvider.Testing](https://www.nuget.org/packages/Microsoft.Extensions.TimeProvider.Testing) | Deterministic time in tests (Concept 54) |
| [CA2016: forward the CancellationToken](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2016) | The analyzer behind Concept 10 |
| [Metrics instrumentation best practices](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/metrics-instrumentation) | Tag cardinality (Concept 50) |
| [`Microsoft.Extensions.Http.Resilience` source](https://github.com/dotnet/extensions/tree/main/src/Libraries/Microsoft.Extensions.Http.Resilience) | Read the standard handlers and validators yourself |
| [Aspire service defaults](https://aspire.dev/fundamentals/service-defaults) | Where the global standard handler comes from (Concept 43) |
| [eShop reference application](https://github.com/dotnet/eShop) | Resilience handlers in a realistic multi-service .NET app |
| [Microsoft Learn: Implement resiliency in a cloud-native ASP.NET Core microservice](https://learn.microsoft.com/en-us/training/modules/microservices-resiliency-aspnet-core/) | A free hands-on module |

### Azure

| Resource | What it covers |
|---|---|
| [Transient fault handling best practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transient-faults) | Retry guidance across Azure |
| [Azure reliability documentation](https://learn.microsoft.com/en-us/azure/reliability/) | Per-service reliability guides, including how each service surfaces transient faults (Concept 65) |
| [Well-Architected: handle transient faults](https://learn.microsoft.com/en-us/azure/well-architected/reliability/handle-transient-faults) | The RE:07 recommendation in depth |
| [Azure.Core configuration samples (retry options)](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/core/Azure.Core/samples/Configuration.md) | `ClientOptions.Retry` and transport configuration |
| [Dependency injection with the Azure SDK for .NET](https://learn.microsoft.com/en-us/dotnet/azure/sdk/dependency-injection) | `AddAzureClients`, `ConfigureOptions`, defaults |
| [Service Bus: message transfers, locks and settlement](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement) · [Dead-letter queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues) | Abandon, redelivery, `MaxDeliveryCount` (Concept 64) |
| [Designing resilient applications with the Cosmos DB SDKs](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/conceptual-resilient-sdk-applications) · [Cosmos .NET SDK performance tips](https://learn.microsoft.com/en-us/azure/cosmos-db/performance-tips-dotnet-sdk-v3) | What the SDK already retries; availability strategy (Concepts 31, 65) |
| Azure patterns: [Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) · [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) · [Bulkhead](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) · [Rate Limiting](https://learn.microsoft.com/en-us/azure/architecture/patterns/rate-limiting-pattern) | The patterns as Microsoft names them |
| [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/) · [Azure Load Testing](https://learn.microsoft.com/en-us/azure/load-testing/) | Platform-level fault injection and load (Concepts 55–56) |

### Polly's maintenance fee and dependency governance

| Resource | What it covers |
|---|---|
| [Introducing the Open Source Maintenance Fee for Polly](https://thepollyproject.org/2026/07/14/polly-osmf-announcement.html) | **The primary source for Concept 67** |
| [Open Source Maintenance Fee](https://opensourcemaintenancefee.org) · [Which projects do you pay?](https://opensourcemaintenancefee.org/consumers/which/) · [FAQ](https://opensourcemaintenancefee.org/consumers/faq/) | The OSMF model; direct vs transitive dependencies |
| [Polly discussion: OSMF adoption](https://github.com/App-vNext/Polly/discussions/3183) | Community questions, including transitive use via Microsoft's packages |
| [dotnet/extensions #7719 — post-OSMF Polly strategy](https://github.com/dotnet/extensions/issues/7719) | The open question for `Microsoft.Extensions.*.Resilience` |
| [Fences (BrighterCommand)](https://github.com/BrighterCommand/Fences) | The community fork from Polly 8.7.0; migration notes and status |
| [Lee Conlin: "Polly is going paid. The fee isn't the problem."](https://leeconlin.co.uk/post/polly-going-paid/) | A practitioner's view of the transitive-dependency and procurement questions |

### Engineering reading behind the patterns

| Resource | What it covers |
|---|---|
| [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — Amazon Builders' Library | Single-layer retries, token buckets, jitter |
| [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — AWS Architecture Blog | The jitter simulation (Concept 12) |
| [Fixing retries with token buckets and circuit breakers](https://brooker.co.za/blog/2022/02/28/retries.html) — Marc Brooker | Retry budgets vs breakers (Concepts 15, 17) |
| [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) | Client request tokens (Concept 42) |
| [Avoiding fallback in distributed systems](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/) | Why redirecting fallbacks are risky (Concept 33) |
| [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/) · [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) | Adaptive throttling, retry limits, cascades |
| [The Tail at Scale](https://research.google/pubs/pub40801/) — Dean & Barroso | Why hedging works (Concept 27) |
| [gRPC client retry design (A6)](https://github.com/grpc/proposal/blob/master/A6-client-retries.md) | Retry, hedging, throttling and pushback in gRPC (Concepts 15, 44) |
| [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) | Idempotency keys in practice |
| [Martin Fowler: CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html) | The canonical short explanation |

### Community articles

| Resource | What it covers |
|---|---|
| [Polly v8 Resilience Pipelines: Migration and Ordering](https://milanjovanovic.tech/blog/polly-v8-resilience-pipelines) — Milan Jovanović | Ordering and v7 → v8 mapping |
| [Building Resilient Cloud Applications With .NET](https://milanjovanovic.tech/blog/building-resilient-cloud-applications-with-dotnet) — Milan Jovanović | `Microsoft.Extensions.Resilience` walkthrough |
| [Building Your First Resilience Pipeline](https://blog.nimblepros.com/blogs/building-your-first-resilience-pipeline/) — NimblePros | A step-by-step introduction series |

### Tools and alternatives

| Resource | Use |
|---|---|
| [WireMock.Net](https://github.com/WireMock-Net/WireMock.Net) | Scripted fake dependencies for host-level tests (Concept 56) |
| [Toxiproxy](https://github.com/Shopify/toxiproxy) | TCP-level latency, resets and bandwidth faults |
| [NBomber](https://nbomber.com/) · [k6](https://k6.io/) | Load tests with injected faults |
| [Dapr resiliency](https://docs.dapr.io/operations/resiliency/resiliency-overview/) | Declarative timeouts, retries and breakers for Dapr apps (Concept 69) |
| [Istio: network resilience and testing](https://istio.io/latest/docs/concepts/traffic-management/#network-resilience-and-testing) | Mesh-level timeouts, retries, circuit breaking, fault injection |
| [Envoy: circuit breaking and retry budgets](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking) | Proxy-level limits and budgets |
| [Resilience4j documentation](https://resilience4j.readme.io/docs) | The JVM counterpart — useful for comparing breaker window designs |
| [FusionCache](https://github.com/ZiggyCreatures/FusionCache) | Caching with a fail-safe (serve-stale) mechanism (Concept 33) |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Polly v8 in one sentence | Immutable pipeline of strategies, built once, first added is outermost |
| Packages | `Polly.Core`, `.Extensions`, `.RateLimiting`, `.Testing`; `Polly` = v7 API; `Microsoft.Extensions.Http.Resilience` on top |
| Why build once | Breaker windows and limiter permits live in the instance |
| Outcomes | Results and exceptions are both outcomes; typed pipelines see 5xx; `ExecuteOutcomeAsync` avoids throwing |
| Default `ShouldHandle` | All exceptions except cancellation, no results — replace it |
| Context | Pooled; token, operation key, typed properties; bounded operation keys |
| Generators vs events | Generators decide, events observe; events are awaited on the call path |
| Timeout mechanics | Cooperative; waits for the callback; `TimeoutRejectedException`; 30 s default; ≤ 0 generated = disabled |
| Retry defaults | 3 retries, 2 s constant, no jitter — too slow and synchronized for request paths |
| Jitter | ±25% constant/linear; decorrelated-jitter-v2 exponential; `MaxDelay` doesn't cap generators |
| POST retries | Only with idempotency key, replayable content, or `HttpRequestError` = never sent |
| `Retry-After` | Honored by the HTTP handler; budget check in `ShouldHandle`; jitter on top |
| Retry budget | Token bucket in `ShouldHandle`; gRPC throttling; mesh budgets |
| Breaker model | Ratio over window with minimum throughput; no consecutive-count breaker |
| Breaker reaction | Time to open ≈ window × ratio; minimum throughput 100 disables quiet breakers |
| Half-open | Lazy; first call after the break is the probe; don't guard with state |
| Breaker ops | State provider → Degraded health check; manual control → kill switch (per process) |
| Breaker scope | Pipeline instance = scope; `SelectPipelineByAuthority`; bounded keys |
| Bulkhead | `AddConcurrencyLimiter`, outermost, Little's Law sizing |
| Quota limiter | Inside the retry so each attempt spends a token; shared per constraint; per-process caveat |
| Hedging | Typed; 1 hedge after 2 s default; reacts to failure too; losers cancelled and awaited |
| Hedging modes | > 0 latency, 0 parallel, < 0 sequential failover, generator = dynamic |
| Standard hedging handler | Total → hedging → per-endpoint limiter/breaker/attempt; ordered/weighted routing |
| Fallback | Outermost, explicit, visibly degraded; prefer doing less |
| Canonical order | Fallback → bulkhead → total → retry/hedge → quota → breaker → attempt → chaos |
| Standard handler | 1,000 permits → 30 s → 3 exp. jittered from 2 s → 10%/100/30 s/5 s → 10 s; retries all methods; breaker counts 429 |
| `HttpClient.Timeout` | Effective budget = min; looks like caller cancellation; handler timeouts end at headers |
| Global defaults | Every factory client incl. libraries; remove, don't stack; count attempts in tests |
| gRPC | Channel retry/hedging for status codes; HTTP handler only transport-level, no retry |
| Registry | Lazy build, cached, append-only; keyed services return the cached instance |
| Reloads | Rebuild resets breaker and limiter; failed reload stops reloading |
| Telemetry | `Polly` meter: strategy events, attempt duration, pipeline duration; name everything |
| Testing | Predicate tables, descriptors, fake-time behavior, host-level attempt counts, gated chaos |
| Chaos | Enabled by default at 0.1% — gate everything; innermost |
| Custom strategy | `ResilienceStrategy.ExecuteCore` + options + `AddStrategy` + telemetry |
| Placement | Transport resilience in adapters; concurrency retries in the command pipeline |
| Ownership | One retry loop per failure kind; check attempt products per path |
| EF Core | Execution strategy owns transient DB faults; never wrap; transactions inside the strategy |
| Command pipeline | Concurrency retry outside UoW, clear tracker, side effects via outbox, unique-violation → stored outcome |
| Event sourcing | Outer re-decide (new IDs), inner append retry (same IDs), idempotency key in metadata |
| Consumers | Quick retry, scheduled redelivery, dead-letter; abandon redelivers immediately |
| Azure SDKs | Configure `RetryOptions`; outer timeout only |
| OSMF | $20/mo from Nov 16, 2026, orgs ≥ $20k revenue from a product using Polly; BSD-3 unchanged; direct deps only; Microsoft pending |
| OSMF decision | Inventory, legal decides, pay if it applies, adapters as insulation, ADR with review triggers |
| When not to | In-process calls, SDK-owned retries, quiet breakers, non-idempotent hedges, untested fallbacks |

---

## Closing — Phase 5 complete

This module closes **Phase 5 — .NET Architecture Patterns**. The phase moved from structure (Clean Architecture, Module 20) to boundaries (modular monolith vs microservices, Module 21), to the domain (DDD, Module 22), to the application's request flow (CQRS and the command pipeline, Module 23), to persistence as history (event sourcing, Module 24), and finally to **what happens at every boundary when something on the other side fails**.

Threads this module closes deliberately:

- **Module 13's "Polly in depth"** — registry and reloads, custom strategies, `Polly.Testing`, hedging with routing, and telemetry enrichment — is now covered (Parts J–M), along with the strategy-level mechanics behind Concepts 58–60 there.
- **Module 20's Concept 44** — resilience belongs in the adapter — now has an implementation and a translation layer (Concept 59).
- **Module 23's open threads** — composing the concurrency retry with EF Core's execution strategy without multiplying, and turning commit-ambiguity re-runs into the right response — are answered in Concepts 61–62.
- **Module 24's open threads** — the re-decide loop and resilience for subscriptions and publishers — are answered in Concepts 63–64.

Threads left open on purpose:

- **Compute-platform resilience** — Container Apps, AKS, App Service and Functions, including Dapr in Container Apps, KEDA-driven scaling of consumers, and zone-redundant hosting — is **Module 26**.
- **Service Bus, Event Grid/Event Hubs and Cosmos DB configuration in depth** — including partitioning, RU economics and consistency choices that interact with the SDK retry settings here — is **Module 27**.
- **Observability** — SLOs and burn-rate alerting built from the Polly metrics of Concept 51, and tracing across retries and hedges — is **Module 28**.
- **Security architecture** — token acquisition per attempt, managed identities for Azure SDK clients, and securing operational endpoints like the circuit kill switch — is **Module 29**.
- **ADRs** — the OSMF decision record of Worked example 4 — are **Module 31**.
- **Brownfield change** — migrating v7 policies and retrofitting bulkheads and budgets into an existing system — is **Module 32**.
- **Build-vs-buy and license risk in executive terms** — the OSMF decision as a cost and risk conversation — is **Module 33**.

Next in the curriculum: **Module 26 — Compute choices: App Service vs AKS vs Azure Functions / Container Apps, and when each wins** — opening Phase 6, Cloud & Platform Architecture.
