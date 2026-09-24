# Module 18 — ASP.NET Core Internals
*Phase 4: .NET & C# Technical Mastery · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **an ASP.NET Core application is four nested machines — a host that builds an object graph, a server that turns bytes into an `HttpContext`, a pipeline that is one composed function, and an endpoint that is one delegate — and almost every production incident in a .NET web service is caused by confusing the lifetime of one of those machines with the lifetime of another.**

That reframing matters because the naive picture of ASP.NET Core is a configuration exercise: call `AddThis`, call `UseThat`, put them in roughly the right order, and ship. A mid-level candidate can recite "singleton, scoped, transient" and "`UseAuthentication` before `UseAuthorization`." A senior candidate can tell you *why* the order is what it is (authorization needs the endpoint that routing selected, and the endpoint's metadata is what carries the policy), why their service leaked 4 GB in a week (a `DbContext` captured by a singleton, or transient `IDisposable`s tracked by the root provider), why a background job intermittently read a disposed `DbContext` (it captured a request scope in a closure), why their latency collapsed under load with CPU at 20% (ThreadPool starvation from one synchronous middleware, or Kestrel's `MaxConcurrentConnections` set from a copy-paste), why the response-modifying middleware silently stopped working (something upstream had already started the response), and why capturing `HttpContext` in a field produces data from a *different user's* request (the context is pooled and reused).

This module sits at the exact centre of the .NET half of the curriculum. Module 14 explained what the runtime does with memory; Module 15 explained what it does with time; Module 16 covered the language; Module 17 gave you measurement. This module is where all four land in the framework you will actually be asked about, and it picks up three debts explicitly: the **captive dependency problem** named in Module 14 (Concept 57) and Module 15 (Concepts 52, 67); the **startup-phase budget** from Module 17 (Concept 63); and the **AOT constraints** from Module 17 (Part H), which in a web app are mostly a statement about MVC, reflection, and the request-delegate generator.

It shows up in five places in an interview loop: the **deep technical round** ("what happens between the socket accept and your controller action?"), the **debugging round** ("our memory grows until the pod is OOM-killed — walk me through it"), the **code-review round** ("what's wrong with this middleware?"), the **design round** (composition root, multi-tenancy, timeouts and limits, graceful shutdown), and the **platform round** (Kestrel behind an ingress, HTTP/2 vs HTTP/3, cold start, AOT).

**Current platform state (verified September 2026).** ASP.NET Core 10 (November 2025) is the production-current LTS, supported to 14 November 2028. .NET 11 RC1 shipped on 8 September 2026 with a go-live licence, ahead of GA on 10 November 2026; it is STS, and C# 15 is its default language version. What changed that matters here:

- **ASP.NET Core 10** added **built-in validation for minimal APIs** (`builder.Services.AddValidation()`, source-generated and AOT-friendly, with `[ValidatableType]`, `DisableValidation()`, and error responses customizable through `IProblemDetailsService`); the validation APIs now live in the **`Microsoft.Extensions.Validation`** package and namespace so they work outside HTTP; **`TypedResults.ServerSentEvents`** with `SseItem<T>` for server push in both minimal APIs and controllers; and OpenAPI generation that defaults to **3.1** with YAML output.
- **ASP.NET Core 11** adds a **`[ShortCircuit]` attribute** (the attribute form of the existing `ShortCircuit()` convention, usable on MVC controllers and actions); **endpoint filters that run even when parameter binding fails**, so a filter can observe a 400 and substitute its own body; **asynchronous validation** end to end (`AsyncValidationAttribute`, `IAsyncValidatableObject`, run concurrently where possible); **built-in validation localization** by resource-name convention; `[ValidatableType]` and `[SkipValidation]` graduating from experimental; **OpenAPI 3.2 by default** (via `Microsoft.OpenApi` 3.3.1 — a breaking change), with `itemSchema` for SSE responses, HTTP `QUERY` as a known operation, and `[Obsolete]` mapped to `deprecated`; **C# 15 union types supported anywhere `System.Text.Json` is used**, including minimal API and MVC bodies, described in OpenAPI as `anyOf`; an **`IOutputCachePolicyProvider`** extension point; and **native OpenTelemetry tracing** — the framework now populates HTTP server semantic-convention attributes itself on the `Microsoft.AspNetCore` activity source, so `OpenTelemetry.Instrumentation.AspNetCore` is no longer required (suppressible with the `Microsoft.AspNetCore.Hosting.SuppressActivityOpenTelemetryData` switch).
- **Kestrel in .NET 11** replaced the throwing path in the HTTP/1.1 request parser with a result struct (success / incomplete / error). Under malformed-request load — port scanning, hostile traffic, broken clients — this removes the exception cost and improves throughput by roughly 20–40%, with no change for valid requests. HTTP logging middleware now pools its response-buffering streams.
- Carried forward and still current: **keyed services**, **`IExceptionHandler`**, **`CreateSlimBuilder`**, the **request delegate generator**, and the **named-pipes transport** (.NET 8); **`MapStaticAssets`** with build-time compression and fingerprinting (.NET 9); **output caching**, **rate limiting**, and **HTTP/3** (.NET 7). For Native AOT, the compatibility picture is unchanged in shape: **minimal APIs partially supported, MVC not supported**, gRPC supported, and EF Core's AOT/precompiled-query support still experimental.

This module has nine jobs:

1. **Make the request path mechanical.** Socket accept → transport → connection → protocol parse → `HttpContext` from a pool → pipeline → routing → endpoint → result → response flush → context reset. No magic anywhere on that line.
2. **Make hosting a design surface, not boilerplate** — configuration precedence, options validation at start, the builder/runtime boundary, hosted services, and graceful shutdown as a contract with your orchestrator.
3. **Explain Kestrel well enough to tune it** — its layering on `System.IO.Pipelines`, its protocol implementations, and its limits and timeouts as an availability control surface rather than trivia.
4. **Make middleware a compositional model you can reason about**, including the rules that are non-negotiable: ordering, the response-started boundary, and body wrapping.
5. **Make routing legible** — templates, precedence, the DFA matcher, and metadata as the extensibility mechanism that authorization, CORS, OpenAPI, rate limiting and caching all ride on.
6. **Diagnose DI properly.** Lifetimes defined by scope, captive dependencies, disposal, the transient-disposable leak, keyed services, `IHttpClientFactory`'s handler scopes, and validation that turns all of it into a startup failure instead of a 3 a.m. page.
7. **Compare minimal APIs and controllers honestly**, with the two filter pipelines, the two validation stories, the AOT difference, and the organizational answer for a large codebase.
8. **Survey the cross-cutting middleware** you are expected to know cold: auth, CORS, caching, rate limiting, compression, static assets, body handling, diagnostics.
9. **Raise it to architecture** — per-request cost model, composition root design, integration testing, multi-tenancy, cold start, and the anti-patterns reviewers look for.

Eight framings to carry through:

1. **Lifetime is the whole subject.** Process, host, connection, scope, request, response-started. Every hard bug in this module is a mismatch between two of those.
2. **The pipeline is a function, built once.** `app.Use(...)` runs at startup; the lambda runs per request. Confusing build time with request time explains most "why is my middleware a singleton?" questions.
3. **Order encodes dependencies, not taste.** Each pair in the canonical order exists because the later component consumes something the earlier one produced.
4. **The endpoint is chosen early and executed late.** That gap — between `UseRouting` and `UseEndpoints` — is where authorization, CORS, rate limiting and caching do their work, using metadata.
5. **Nothing per-request is yours to keep.** `HttpContext`, its headers, its buffers, its features, and the scope that owns your services all end when the response completes. Capture any of them and you have a bug that appears only under load.
6. **The container is a graph, not a bag.** Its shape is a design artifact: what is shared, what is per-request, what is created by hand, and what is validated at startup.
7. **Limits are availability features.** Body size, header size, connection counts, timeouts, queue depths, rate limiters. Defaults are chosen for general safety, not for your SLO.
8. **Startup cost is a product decision.** JIT vs R2R vs AOT, slim vs full builder, eager vs deferred initialization — the right answer differs per service and is measurable.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Four layers | Host → server → pipeline → endpoint; each has its own lifetime |
| 2 | `WebApplicationBuilder` | Config, logging, DI, Kestrel, and a deferred pipeline, all in one object |
| 3 | Configuration | Ordered providers, last wins; reload affects `IOptionsMonitor`, not `IOptions` |
| 4 | Options pattern | `IOptions` singleton, `IOptionsSnapshot` scoped, `IOptionsMonitor` live; validate at start |
| 5 | `Build()` | Freezes the container and splits builder time from request time |
| 6 | Slim and empty builders | Pay-for-play hosting for AOT, size, and startup |
| 7 | Startup phases | Image pull → runtime → host build → first request; measure each |
| 8 | Hosted services | `StartAsync` blocks startup; unobserved exceptions stop the host |
| 9 | Graceful shutdown | SIGTERM → stop accepting → drain → `ShutdownTimeout`; coordinate with the orchestrator |
| 10 | Auto-inserted middleware | `WebApplication` inserts routing, auth, endpoints under stated conditions |
| 11 | `IServer` + features | The feature collection is the server abstraction; program against it |
| 12 | Kestrel's layers | Transport, connection middleware, protocol, `HttpContext` |
| 13 | Pipelines and back-pressure | Pooled buffers, `PipeReader`/`PipeWriter`, flow control end to end |
| 14 | HTTP/1.1 | One request at a time per connection; keep-alive; .NET 11's non-throwing parser |
| 15 | HTTP/2 | Multiplexed streams, flow control, HPACK, `MaxStreamsPerConnection` |
| 16 | HTTP/3 | QUIC over UDP, no TCP head-of-line blocking, discovered via `alt-svc` |
| 17 | TLS in Kestrel | SNI callbacks, cert selection, handshake cost, resumption |
| 18 | Kestrel limits | Body size, header size, connection counts — availability controls |
| 19 | Timeouts and data rates | Keep-alive 130 s, headers 30 s, minimum data rates defeat slow clients |
| 20 | Other servers and proxies | HTTP.sys, IIS in/out of process, YARP, `UseForwardedHeaders` |
| 21 | Connection vs request | `RequestAborted` is a client signal, not a server timeout |
| 22 | `RequestDelegate` | The pipeline is `Func<HttpContext, Task>` composed in reverse at build time |
| 23 | Branching | `Use` vs `Run` vs `Map` vs `MapWhen` vs `UseWhen`, and where control returns |
| 24 | Middleware activation | Convention-based is a singleton; `IMiddleware` is resolved per request |
| 25 | Ordering | Each adjacency exists because of a dependency; know the canonical list |
| 26 | Short-circuiting | Terminal middleware, `MapShortCircuit`, `[ShortCircuit]` (.NET 11) |
| 27 | Response started | After the first flush, status and headers are frozen; use `OnStarting` |
| 28 | Body wrapping | Swap the stream or the `IHttpResponseBodyFeature`, and always restore it |
| 29 | Exception handling | `UseExceptionHandler` + `IExceptionHandler` + ProblemDetails; never leak internals |
| 30 | `HttpContext` is pooled | Never store it, never use it after the request; copy what you need |
| 31 | Pipeline cost | Per-request allocations and closures; measure with traces, not intuition |
| 32 | Endpoint routing | Match early, execute late; the gap is where policy runs |
| 33 | Templates and precedence | Literal > constrained > parameter > catch-all; `Order` breaks ties |
| 34 | The matcher | A DFA over path segments: cost scales with URL length, not route count |
| 35 | Metadata | The universal extension point: auth, CORS, caching, limits, OpenAPI |
| 36 | Route groups | `MapGroup` applies prefix, filters, and conventions to a whole family |
| 37 | Link generation | `LinkGenerator` is the inverse of matching and works outside a request |
| 38 | Ordering traps | Middleware before `UseRouting` has no endpoint; after `UseEndpoints`, nothing runs |
| 39 | Resolution engines | `ServiceDescriptor` graph compiled to IL/expressions; runtime resolver under AOT |
| 40 | Lifetimes | Singleton = root scope, scoped = per scope, transient = per resolution |
| 41 | The request scope | Created by the hosting layer, disposed after the response completes |
| 42 | Captive dependencies | A longer-lived service holding a shorter-lived one; caught by `ValidateScopes` |
| 43 | Disposal | The container disposes what it creates; instances you `new` are yours |
| 44 | Singleton concurrency | Shared state must be thread-safe; never capture per-request objects |
| 45 | `IHttpContextAccessor` | `AsyncLocal`-backed ambient context, with a real cost and real traps |
| 46 | Scopes from singletons | `IServiceScopeFactory.CreateAsyncScope()`, one scope per unit of work |
| 47 | Multiple registrations | Last wins for one, all for `IEnumerable<T>`; `TryAdd*` for libraries |
| 48 | Keyed services | Named variants without factory indirection (.NET 8+) |
| 49 | `ActivatorUtilities` | Construct non-services with injected dependencies; not everything is a service |
| 50 | `IHttpClientFactory` | Handler pooling, handler scopes ≠ request scope, `PooledConnectionLifetime` |
| 51 | DI performance | Resolution is cheap; graph size and reflective startup are not |
| 52 | Replacing the container | Justified only by features you actually need; you inherit its semantics |
| 53 | Minimal API mechanics | `RequestDelegateFactory` at runtime, or RDG interceptors at compile time |
| 54 | Parameter binding | Special types → attributes → services → route/query/header → body |
| 55 | Results | `TypedResults` for typed metadata; ProblemDetails; SSE and streaming |
| 56 | Endpoint filters | Nested `IEndpointFilter` chain inside the endpoint, after auth |
| 57 | Validation | `AddValidation()` source-generated (.NET 10); async validators (.NET 11) |
| 58 | MVC pipeline | Authorization → resource → binding → action → result filters, then result execution |
| 59 | `[ApiController]` | Automatic 400s, binding inference, ProblemDetails — conventions with costs |
| 60 | Minimal vs controllers | Startup, AOT, filters, conventions, team size — decide on evidence |
| 61 | Organizing minimal APIs | Endpoint modules + groups; vertical slices; avoid a 3,000-line `Program.cs` |
| 62 | OpenAPI | The contract surface; generated from metadata; 3.1 in .NET 10, 3.2 in .NET 11 |
| 63 | Authentication | Schemes and handlers; `HttpContext.User` is set by middleware, not routing |
| 64 | Authorization | Policies and requirements evaluated against endpoint metadata |
| 65 | CORS, HSTS, antiforgery | Browser-facing protections with order requirements |
| 66 | Caching layers | Output cache (server), response cache (headers), HybridCache (data) |
| 67 | Rate limiting | In-process partitioned limiters; queues and rejection are design choices |
| 68 | Compression and assets | `MapStaticAssets` for app assets; compression trade-offs and CRIME/BREACH |
| 69 | Bodies | `Stream` vs `PipeReader`, `EnableBuffering`, uploads, streaming responses |
| 70 | Built-in diagnostics | Meters, activity sources, HTTP logging, health checks — all free |
| 71 | Per-request cost model | Allocation, scope, serialization, logging: know the budget |
| 72 | Composition root design | Module registration, options validation, no assembly scanning at scale |
| 73 | Integration testing | `WebApplicationFactory` exercises the real pipeline; know its limits |
| 74 | Multi-tenancy | Resolve the tenant before the container needs it; scope everything to it |
| 75 | Cold start and AOT | Slim builder, RDG, JSON source gen, min replicas — measured per service |
| 76 | Anti-patterns | Service locator, God middleware, static context, sync-over-async, silent 500s |

---

# Part A — Hosting: from process start to a listening server

Everything in Parts B–G runs inside a host. This part is about what happens before the first request arrives — and it matters more than it looks, because the host is where lifetimes are decided, where configuration becomes typed objects, where failures should happen loudly, and where your startup budget is spent.

## Concept 1 — The four layers, and why the boundaries matter

An ASP.NET Core application is four nested machines:

| Layer | What it is | Lifetime | Key type |
|---|---|---|---|
| **Host** | Configuration, logging, DI container, hosted services, lifetime signals | Process | `IHost` / `WebApplication` |
| **Server** | Listens on sockets, speaks HTTP, produces an `HttpContext` per request | Process (connections come and go) | `IServer` (Kestrel, HTTP.sys, IIS) |
| **Pipeline** | An ordered chain of middleware, composed once into one delegate | Built once, invoked per request | `RequestDelegate` |
| **Endpoint** | The specific handler selected for this request | Per request | `Endpoint` + `RequestDelegate` |

The server calls the pipeline: `IServer.StartAsync(IHttpApplication<TContext> application, ...)` hands the server an application object it invokes once per request. The hosting layer's `HostingApplication` is what creates the `HttpContext`, creates the **DI scope**, starts the **activity and metrics** for the request, calls your pipeline, and then tears all of it down.

Three consequences you should be able to state:

1. **Your middleware never sees the socket.** It sees `HttpContext`, which is a facade over a feature collection the server filled in (Concept 11). That is why the same middleware works under Kestrel, HTTP.sys, IIS and `TestServer`.
2. **The DI scope is created and disposed by the hosting layer**, not by routing, not by MVC, and not by your code. Its lifetime is "one request," including the response body flush.
3. **The pipeline is built once.** `app.UseX()` runs at startup. Anything expensive you do in the body of a `Use(...)` lambda happens per request; anything you do outside it happens once. This single distinction resolves a large fraction of middleware confusion.

---

## Concept 2 — `WebApplicationBuilder`: what `CreateBuilder` actually wires up

`WebApplication.CreateBuilder(args)` is not a mystery object; it is a bundle of defaults that you should be able to enumerate, because interviewers ask "what do you get for free, and what would you change?"

- **Configuration**, in order (later sources override earlier ones): `appsettings.json` → `appsettings.{Environment}.json` → user secrets (Development only) → environment variables → command-line arguments.
- **Logging**: console, debug, `EventSource`, and on Windows `EventLog`, with `Logging` configuration section bound.
- **DI**: the built-in container, plus `IOptions`, and — in Development — `ValidateScopes` and `ValidateOnBuild` turned on via the default service-provider options.
- **Hosting**: Kestrel as `IServer`, content root from the current directory, `IHostEnvironment` from `ASPNETCORE_ENVIRONMENT`/`DOTNET_ENVIRONMENT`, host filtering, and forwarded headers **only** when `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`.
- **Exposed sub-builders**: `builder.Services`, `builder.Configuration`, `builder.Logging`, `builder.Environment`, `builder.WebHost` (for `IWebHostBuilder`-shaped APIs such as `ConfigureKestrel`), `builder.Host` (for `IHostBuilder`-shaped APIs), and `builder.Metrics`.

The important structural fact: `WebApplicationBuilder` **defers**. Calls to `builder.WebHost.ConfigureKestrel(...)` or `builder.Host.ConfigureServices(...)` are recorded and replayed during `Build()`. That is why some host-level APIs throw if you call them at the wrong time (`UseStartup`, for instance, is not supported on `WebApplicationBuilder`) — the builder has already committed to a shape.

A senior-level habit: treat `Program.cs` as a **manifest**. Someone reading the first sixty lines should be able to tell what this service is, what it depends on, and how it is protected. If they cannot, the composition root needs the treatment in Concept 72.

---

## Concept 3 — Configuration: ordered providers, and the boundary you must not cross

`IConfiguration` is a layered key-value store with case-insensitive keys, where `:` separates levels (and `__` does in environment variables, because most shells forbid `:`). Providers are consulted in registration order and **the last one to supply a key wins**.

Three things worth knowing precisely:

- **Arrays are keys.** `"Hosts:0"`, `"Hosts:1"`. An environment variable can override one element without replacing the array, which surprises people, and a later provider cannot *shorten* an array supplied by an earlier one.
- **Reload is per provider.** `reloadOnChange: true` on a JSON file triggers a reload token. That token is what `IOptionsMonitor` and `IConfiguration.GetReloadToken()` subscribe to. Environment variables are read once at startup; a change in the container's environment does not propagate.
- **Configuration is not a secret store.** Key Vault, App Configuration, and similar providers integrate as configuration sources (with their own refresh semantics), but the rule for interviews is simple: secrets do not live in `appsettings.json` or in the repository, and in Azure the modern answer is managed identity plus Key Vault references, not a connection string in an environment variable.

The architectural point: configuration should be **read once, validated, and converted into typed options**. Code that calls `IConfiguration["Foo:Bar"]` deep inside a service has given up on validation, discoverability, and testability. That is what Concept 4 is for.

---

## Concept 4 — The Options pattern, and validating at start

Three interfaces, three lifetimes, and the distinction is a favourite interview question:

| Interface | Lifetime | Recomputed when | Use for |
|---|---|---|---|
| `IOptions<T>` | Singleton | Never (computed once) | Settings fixed for the process; safe to inject anywhere |
| `IOptionsSnapshot<T>` | **Scoped** | Once per request/scope | Settings that may change and should be stable within a request |
| `IOptionsMonitor<T>` | Singleton | On every configuration reload, with change callbacks | Singletons and background services that must see updates |

Two traps: injecting `IOptionsSnapshot<T>` into a singleton is a **captive dependency** (Concept 42) and will fail scope validation; and reading `IOptionsMonitor<T>.CurrentValue` in a hot loop is not free — it can re-run the post-configure chain after a change, so read once per operation.

The senior move is **validation that fails at startup**:

```csharp
builder.Services
    .AddOptions<PaymentOptions>()
    .Bind(builder.Configuration.GetSection("Payments"))
    .ValidateDataAnnotations()
    .Validate(o => o.Timeout > TimeSpan.Zero, "Timeout must be positive")
    .ValidateOnStart();          // fail during host start, not on first request
```

`ValidateOnStart()` converts "the service starts, serves traffic, and 500s on the first payment" into "the pod never becomes ready and the rollout halts." That is the entire point: **a misconfiguration should be a deployment failure, not a runtime failure.** For AOT and trimming, `[OptionsValidator]` with the options validation source generator gives you the same guarantee without reflection.

---

## Concept 5 — `Build()`: the container freeze and the builder/runtime boundary

`builder.Build()` does several irreversible things:

1. Builds the final `IConfiguration`.
2. Builds the `ServiceProvider` from the `IServiceCollection` — after which **the collection is read-only**. Attempting `app.Services.AddSingleton(...)` after build throws. This is by design: the graph must be stable.
3. Runs container validation (in Development by default): `ValidateOnBuild` walks every registration and verifies it can be constructed; `ValidateScopes` arms the runtime check that a singleton never resolves a scoped service.
4. Creates the `WebApplication`, which is simultaneously an `IHost`, an `IApplicationBuilder`, and an `IEndpointRouteBuilder`.

Everything before `Build()` is **builder time** (registration, configuration, option binding). Everything after is **runtime** (middleware composition, endpoint mapping, running). Two rules follow:

- **Do not resolve services from the builder.** `builder.Services.BuildServiceProvider()` — often written to "just get the logger" — creates a *second* container. You now have two singletons of everything, two sets of disposables, and a memory leak. If you need a service during startup, resolve it from `app.Services` after `Build()`, in a scope: `using var scope = app.Services.CreateScope();`.
- **Migrations, seeding and warm-up go after `Build()`, before `Run()`**, in an explicit scope, and should be gated by environment or by a separate job in production (Concept 72).

---

## Concept 6 — `CreateSlimBuilder`, `CreateEmptyBuilder`, and pay-for-play hosting

Three entry points, in decreasing order of what they assume:

| Entry point | Includes | Use when |
|---|---|---|
| `CreateBuilder` | Everything in Concept 2 | Default for most services |
| `CreateSlimBuilder` | JSON config (`appsettings*.json`), environment variables, command line, console logging, Kestrel, routing, minimal APIs | AOT, small containers, fast startup |
| `CreateEmptyBuilder` | Essentially nothing; you add each piece | Ultra-minimal or highly specialised hosts |

`CreateSlimBuilder` deliberately omits: hosting startup assemblies and `UseStartup`; the debug, EventSource and EventLog logging providers; IIS integration; `UseStaticWebAssets`; and — importantly — **HTTPS is not configured by default**, because the AOT template assumes TLS terminates at an ingress. HTTP/3 must be opted into with `builder.WebHost.UseQuic()`.

The point is not that slim is better. The point is that **hosting features have a size and startup cost that is invisible under JIT and painful under AOT**, where every retained feature is bytes in the binary and work at startup. In an interview, the useful sentence is: *"We use `CreateSlimBuilder` for the AOT'd edge services where startup and image size are the binding constraints, and the full builder elsewhere; the difference was about X ms to first request and Y MB of image, measured."*

---

## Concept 7 — Startup phases and the startup budget

Module 17 (Concept 63) said: measure the phases before optimizing. In a web app the phases are:

1. **Image pull and container start** — frequently the largest term, and nothing to do with .NET.
2. **Runtime start** — the CLR, or a Native AOT binary's near-zero equivalent.
3. **Host build** — configuration providers, DI registration, `ValidateOnBuild`, assembly scanning if you do it.
4. **Server bind and pipeline build** — cheap, unless you map thousands of endpoints.
5. **First request** — JIT of the request path, request-delegate creation (if not source-generated), first JSON serializer metadata, first EF Core model build, first TLS handshake, first connection to every dependency.
6. **Readiness** — when you *tell* the orchestrator you are ready.

The diagnostic trick most candidates miss: **the first request is usually far more expensive than the tenth**, and the usual culprits are model building (EF Core), serializer metadata, and JIT. If cold start matters, the levers in order of typical impact are: smaller image, ReadyToRun or AOT, less reflection at startup, `EnableRequestDelegateGenerator`, `System.Text.Json` source generation, deferred initialization for things not needed to serve the first request, and a readiness-gated warm-up that pre-JITs the hot path before traffic arrives.

---

## Concept 8 — Hosted services and `BackgroundService`

`IHostedService` has two methods and one subtlety that bites nearly everyone.

- `StartAsync` is awaited **before the server starts accepting requests** (for services registered before the hosting service, which is the normal case). Work you do there delays readiness; long work there is a startup-time bug.
- `BackgroundService.ExecuteAsync` is invoked *from* `StartAsync`, and `StartAsync` returns at the first `await` that yields. So a `BackgroundService` whose `ExecuteAsync` begins with synchronous work blocks host startup; one that begins with `await Task.Yield()` or an immediately-awaiting call does not.
- **An unhandled exception in `ExecuteAsync` stops the host by default** (`BackgroundServiceExceptionBehavior.StopHost`, the default since .NET 6). This is usually what you want — a silently dead worker is worse — but it must be a deliberate decision, and your loop should catch, log, and back off around transient failures.
- `IHostedLifecycleService` (.NET 8+) adds `StartingAsync`/`StartedAsync`/`StoppingAsync`/`StoppedAsync` for finer ordering.
- Hosted services are **singletons**. They must create their own scopes for scoped dependencies (Concept 46).

Canonical shape for a worker:

```csharp
public sealed class OutboxPublisher(IServiceScopeFactory scopes, ILogger<OutboxPublisher> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();   // one scope per unit of work
                var publisher = scope.ServiceProvider.GetRequiredService<IOutboxProcessor>();
                await publisher.PublishBatchAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;                                               // shutdown, not an error
            }
            catch (Exception ex)
            {
                log.LogError(ex, "Outbox batch failed; will retry");  // don't kill the host on transients
            }
        }
    }
}
```

That is also the .NET implementation of the transactional outbox from Module 11 — the pattern and the hosting mechanism meet here.

---

## Concept 9 — Graceful shutdown as a contract with the orchestrator

Shutdown is where "it works on my machine" meets Kubernetes. The sequence:

1. The platform sends **SIGTERM** (or the console sends Ctrl-C, or IIS signals shutdown).
2. `IHostApplicationLifetime.ApplicationStopping` fires. **The server stops accepting new connections** and begins draining.
3. In-flight requests are given until `HostOptions.ShutdownTimeout` (default **30 seconds**) to complete; `IHostedService.StopAsync` is called on each hosted service in reverse registration order.
4. At the deadline, remaining work is cancelled and the process exits. The DI container is disposed, disposing singletons it created.

Two failure modes to be able to name:

- **The load balancer is still sending you traffic.** Removal from the rotation is eventually consistent. The standard fix is a **`preStop` sleep** (a few seconds) so the endpoint is deregistered before the process stops accepting, plus a readiness probe that starts failing immediately on `ApplicationStopping`.
- **Long requests exceed the shutdown timeout.** Either raise `ShutdownTimeout` above your longest legitimate request, or — better — stop having requests that outlive a deploy. Long work belongs in a background worker that checks the stopping token, not in a request.

```csharp
builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(45));
```

In an interview, tie this back to Module 13: **graceful shutdown is what makes a rolling deploy not look like a partial outage**, and it is a reliability feature with a testable behaviour (deploy under load, watch for 502s).

---

## Concept 10 — `IStartupFilter` and `WebApplication`'s auto-inserted middleware

Two mechanisms insert middleware you did not write into your pipeline.

**`IStartupFilter`** lets a library wrap the whole pipeline: it receives `Action<IApplicationBuilder>` and returns a new one, so it can add middleware before and after everything the app configured. This is how health checks, telemetry, and hosting-integration libraries insert themselves without requiring a `Use...` call. It is the right extension point when writing a platform library for internal teams; it is the wrong tool when a plain `Use...` call would do, because invisible middleware is hostile to the next reader.

**`WebApplication` auto-insertion.** When using minimal hosting, the following are added under stated conditions:

- `UseDeveloperExceptionPage` first, when the environment is Development.
- `UseRouting` second, **if you did not call it yourself** and endpoints are configured.
- `UseAuthentication` immediately after routing, if you did not call it and `IAuthenticationSchemeProvider` is registered (i.e. you called `AddAuthentication`).
- `UseAuthorization` next, under the equivalent condition for `IAuthorizationHandlerProvider`.
- `UseEndpoints` at the very end, if any endpoints are configured.
- **Your middleware goes between `UseRouting` and `UseEndpoints`.**

Two consequences that generate real bugs. First, if you need middleware to run **before route matching** — rewriting, tenant resolution from the host name, forwarded headers — you must call `app.UseRouting()` yourself and place that middleware before it. Second, if you add `UseCors`, you must also call `UseAuthentication` and `UseAuthorization` explicitly, because CORS has to run before them and adding it by hand disturbs the automatic placement.

This is the single most common "why doesn't my middleware run?" answer in modern ASP.NET Core, and being able to explain the auto-insertion rules precisely is a strong signal.

---

# Part B — The server: Kestrel and the feature collection

This part is what separates "I use ASP.NET Core" from "I understand ASP.NET Core." Most candidates can describe middleware; far fewer can describe what produces the `HttpContext` that middleware receives, or which knob prevents a slow-client attack.

## Concept 11 — `IServer`, `IFeatureCollection`, and why `HttpContext` is a facade

`HttpContext` is not a data object. It is a **facade over `IFeatureCollection`** — a type-keyed bag of interfaces the server populates:

| Feature | What it provides |
|---|---|
| `IHttpRequestFeature` | Method, path, query string, protocol, headers, body stream |
| `IHttpResponseFeature` | Status code, headers, `HasStarted`, `OnStarting`/`OnCompleted` |
| `IHttpResponseBodyFeature` | `Stream`, `PipeWriter`, `StartAsync`, `SendFileAsync`, `CompleteAsync` |
| `IHttpRequestLifetimeFeature` | `RequestAborted`, `Abort()` |
| `IHttpConnectionFeature` | Local/remote IP and port, connection id |
| `IHttpUpgradeFeature` / `IHttpWebSocketFeature` | Protocol upgrade, WebSockets |
| `IHttpResetFeature` / `IHttpExtendedConnectFeature` | HTTP/2 and HTTP/3 specifics |
| `ITlsConnectionFeature` | Client certificate, negotiated protocol |
| `IHttpMaxRequestBodySizeFeature` | Per-request body limit override |
| `IEndpointFeature` | The endpoint selected by routing |

Three reasons this design matters:

1. **Portability.** Kestrel, HTTP.sys, IIS and `TestServer` implement different subsets. Middleware asks `context.Features.Get<IHttpUpgradeFeature>()` and degrades gracefully when it is absent, instead of testing which server is running.
2. **Interception.** Because features are mutable, middleware can *replace* one — which is exactly how response buffering, compression and caching middleware work (Concept 28).
3. **Capability discovery.** "Can I send a file with zero copies?" is `IHttpResponseBodyFeature.SendFileAsync`. "Can I reset this HTTP/2 stream?" is `IHttpResetFeature`. This is how you write code that is fast when the server supports something and correct when it does not.

The feature collection is also **pooled and reset per request** along with the context (Concept 30).

---

## Concept 12 — Kestrel's layers: transport → connection → protocol → context

Kestrel is not one component. Reading it as four layers makes its configuration surface obvious:

1. **Transport.** An `IConnectionListenerFactory` produces connections. The default is the **Sockets transport** (`SocketTransportOptions`); alternatives are **named pipes** (.NET 8+, Windows, for local IPC), **Unix domain sockets**, and **QUIC** for HTTP/3. Transports deal in byte streams, not HTTP.
2. **Connection middleware.** A pipeline *per connection* (`ConnectionDelegate`), analogous to the HTTP middleware pipeline but one level down. TLS is implemented as connection middleware (`UseHttps`); so is HTTP/2 vs HTTP/1.1 selection via ALPN. You can write your own — connection-level logging, IP allow-lists, or a completely non-HTTP protocol on a port.
3. **Protocol.** `Http1Connection`, `Http2Connection`, `Http3Connection` parse frames or request lines into request state and drive the application.
4. **Application.** The protocol layer asks the hosting layer for an `HttpContext` (from a pool), invokes the pipeline, and then flushes and resets.

Understanding that TLS is connection middleware explains why `UseHttps` is configured per *listen endpoint*, and why SNI-based certificate selection is a callback on that middleware rather than a global setting.

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(8080);                                   // plaintext, behind ingress
    options.ListenAnyIP(8443, listen =>
    {
        listen.Protocols = HttpProtocols.Http1AndHttp2;
        listen.UseHttps(https =>
        {
            https.ServerCertificateSelector = (ctx, sni) => CertificateFor(sni);
        });
    });
});
```

---

## Concept 13 — Pipelines, pooled buffers, and back-pressure

Kestrel is built on **`System.IO.Pipelines`** (Module 17, Concept 40), and this is the mechanism behind several behaviours you can otherwise only memorize.

- Reads and writes go through `PipeReader`/`PipeWriter` over **pooled memory** (`MemoryPool<byte>`), not `byte[]` allocated per request. That is why a Kestrel service can serve tens of thousands of requests per second with a nearly flat gen-0 rate.
- `ReadOnlySequence<byte>` lets the parser handle a request line or header that straddles two buffer segments without copying, using `AdvanceTo(consumed, examined)` to say "I used this much, I looked at this much, wake me when there's more."
- **Back-pressure is real and end-to-end.** `MaxRequestBufferSize` (1 MB default) and `MaxResponseBufferSize` (64 KB default) bound how much Kestrel will hold. When your handler stops reading the request body, the pipe fills, the transport stops reading from the socket, and TCP flow control eventually slows the client. When you write faster than the client reads, `WriteAsync` stops completing immediately. **Your `await` is the back-pressure.** Code that fires writes without awaiting them defeats it and turns a slow client into unbounded server memory.
- Pooled buffers explain a rule: **anything you get from the request that is backed by a pooled buffer is invalid after you advance or after the request ends.** Copy what you need.

---

## Concept 14 — HTTP/1.1: keep-alive, pipelining, and the parse path

Per connection, HTTP/1.1 processes **one request at a time**. Concurrency comes from many connections; browsers open ~6 per origin, and server-to-server clients rely on a connection pool (`SocketsHttpHandler`, Concept 50).

Details worth having:

- **Keep-alive** is the default; `KeepAliveTimeout` defaults to **130 seconds**. An idle connection still costs a socket, a buffer and bookkeeping — which is why `MaxConcurrentConnections` matters more than it looks.
- **Pipelining** (sending several requests without waiting) is supported but effectively unused in practice; do not design for it.
- **Chunked transfer encoding** is how a response without a known length is framed. The moment you write more than the response buffer, Kestrel commits to chunked and the headers are gone (Concept 27).
- **.NET 11 changed the malformed-request path.** Previously every parse failure threw `BadHttpRequestException`; now the parser returns a result struct and only converts to an exception when the application needs one. Under port scanning or hostile traffic, this is a 20–40% throughput improvement. The senior observation: *exceptions on a hot path are a capacity problem, and an internet-facing server's "unusual" path is somebody's Tuesday.*

---

## Concept 15 — HTTP/2: multiplexing, flow control, and its limits

HTTP/2 multiplexes many **streams** over one TCP connection, with binary framing and **HPACK** header compression. What this changes for you:

- **One connection carries many concurrent requests**, so connection counts fall dramatically and per-connection limits become per-*client* limits. `Limits.Http2.MaxStreamsPerConnection` (default **100**) caps concurrency from a single peer; with gRPC or a chatty internal client this is a real throttle.
- **Flow control is per stream and per connection**: `InitialStreamWindowSize` and `InitialConnectionWindowSize`. For large uploads or downloads over high-latency links, these windows — not bandwidth — are often the limiter.
- **HPACK** uses a dynamic table (`HeaderTableSize`) shared across streams on the connection. Large, unique headers (tokens, correlation ids) compress poorly and the table costs memory per connection.
- **Head-of-line blocking moves, it does not vanish.** Application-level HOL is gone; TCP-level HOL remains, because one lost segment stalls every stream on that connection. That is the problem HTTP/3 solves.
- **Practical caution:** HTTP/2 without TLS (`h2c`) works between services you control but is rejected by browsers, and many proxies handle it inconsistently.

gRPC requires HTTP/2, which is the usual reason it shows up in a .NET service at all.

---

## Concept 16 — HTTP/3 and QUIC

HTTP/3 runs over **QUIC**, which is TCP-like reliability implemented over UDP with TLS 1.3 built into the handshake. In .NET it depends on **MsQuic** (bundled with the runtime on common platforms since .NET 8); where the platform requirements are not met, Kestrel disables HTTP/3 and falls back.

What actually changes:

- **No TCP head-of-line blocking** — a lost packet stalls only its own stream. This is the real win on lossy or mobile networks.
- **Faster connection establishment** (and 0-RTT resumption), which matters for short-lived clients.
- **Connection migration** across network changes (Wi-Fi to cellular) without re-handshaking.

Operational realities to state in an interview: HTTP/3 **requires HTTPS**; it is **discovered via the `alt-svc` header**, so the first request still uses HTTP/1.1 or HTTP/2; and because middleboxes still block or mishandle UDP/443, you enable it *alongside* the others:

```csharp
options.ListenAnyIP(443, listen =>
{
    listen.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
    listen.UseHttps();
});
```

And the architect's version: if your service sits behind an Azure/AWS ingress that terminates TLS, **HTTP/3 is the ingress's decision, not yours** — your Kestrel speaks whatever the proxy speaks on the inside, usually HTTP/1.1 or h2c.

---

## Concept 17 — TLS in Kestrel: selection, cost, and where to terminate

`UseHttps` is connection middleware wrapping the stream in `SslStream`. Configuration worth knowing:

- **Certificate selection**: a static certificate, a `ServerCertificateSelector` callback for **SNI** (host-based selection, the multi-tenant answer), or `HttpsConnectionAdapterOptions.OnAuthenticate` for full control.
- **Client certificates**: `ClientCertificateMode` (`NoCertificate`, `AllowCertificate`, `RequireCertificate`, `DelayCertificate`) plus `ITlsConnectionFeature` to read it. mTLS between services is configured here, not in middleware.
- **Protocols**: leave `SslProtocols` at the OS default unless a compliance requirement forces otherwise; hard-coding TLS versions ages badly.
- **Cost**: a full handshake is one or two round trips plus asymmetric crypto; session resumption and keep-alive amortize it. On a service with many short-lived clients, handshakes can be a measurable share of CPU — which is a real argument for terminating TLS at an ingress with hardware or a shared session cache.

The design question interviewers actually want answered: **where do you terminate TLS, and is the internal hop encrypted?** Common defensible answers are "TLS terminates at the ingress; internal traffic is plaintext within the cluster's network boundary," or "we run mTLS to the pod via a service mesh," or "Kestrel terminates directly because we have no proxy." What is *not* defensible is not knowing which one you have.

---

## Concept 18 — Kestrel limits as an availability control surface

These are not trivia; they are the settings that decide whether a bad client can take your service down. Defaults as of .NET 10/11:

| Limit | Default | What it protects against |
|---|---|---|
| `MaxConcurrentConnections` | **null (unlimited)** | Connection exhaustion; memory per connection |
| `MaxConcurrentUpgradedConnections` | null (unlimited) | WebSocket/upgrade fan-in |
| `MaxRequestBodySize` | **30,000,000 bytes (~28.6 MB)** | Memory and disk blowups from huge uploads |
| `MaxRequestBufferSize` | 1 MB | Unbounded buffering of unread request bodies |
| `MaxResponseBufferSize` | 64 KB | Unbounded buffering when the client reads slowly |
| `MaxRequestHeadersTotalSize` | 32 KB | Header-bomb attacks |
| `MaxRequestHeaderCount` | 100 | Same |
| `MaxRequestLineSize` | 8 KB | Absurd URLs |
| `Http2.MaxStreamsPerConnection` | 100 | One client monopolising a connection |

Two senior points. First, `MaxRequestBodySize` can be overridden **per request** through `IHttpMaxRequestBodySizeFeature` or per endpoint with `[RequestSizeLimit]` / `.WithMetadata(new RequestSizeLimitMetadata(...))` — so the upload endpoint gets a large limit and everything else keeps a small one, rather than raising the global default. Second, when running **out-of-process behind IIS**, IIS enforces its own body limit and Kestrel's is disabled — a classic "works locally, 404.13 in production" mismatch.

And the framing that makes this an architecture answer, not a settings dump: **these limits are the in-process half of Module 6's overload protection and Module 13's bulkheads.** They bound what a single peer can consume before your rate limiter or your load shedder ever sees a request.

---

## Concept 19 — Timeouts, minimum data rates, and slow-client attacks

A slow client is an attack vector: hold connections open, send one byte per second, exhaust the server's connection budget. Kestrel's defences:

| Setting | Default | Meaning |
|---|---|---|
| `KeepAliveTimeout` | 130 s | Idle connection is closed |
| `RequestHeadersTimeout` | 30 s | Time allowed to finish sending headers (Slowloris) |
| `MinRequestBodyDataRate` | 240 bytes/s, 5 s grace | Too-slow uploads are aborted |
| `MinResponseDataRate` | 240 bytes/s, 5 s grace | Too-slow readers are aborted |
| `Http2.KeepAlivePingDelay` / `KeepAlivePingTimeout` | — | Detects dead HTTP/2 peers |

Note that **none of these is a request-execution timeout**. ASP.NET Core does not, by default, kill a request because your handler took too long. For that you use the **request timeouts middleware** (`AddRequestTimeouts` / `WithRequestTimeout`, .NET 8+), which cancels `HttpContext.RequestAborted` after a budget and can return a configured status code. Pair it with per-dependency timeouts from Module 13 — a request timeout without dependency timeouts just means every slow call now fails *twice*.

Also know: these timeouts are **disabled when a debugger is attached**, which is why "it never times out locally" is not evidence.

---

## Concept 20 — Other servers, proxies, and the forwarded-headers trap

| Option | When it is the answer |
|---|---|
| **Kestrel** | Default everywhere, including edge-facing with TLS |
| **HTTP.sys** | Windows-only; needed for Windows Authentication at the server, port sharing, kernel-mode queueing/response caching |
| **IIS in-process** | Windows hosting; Kestrel-equivalent handler inside the IIS worker process; fastest IIS option |
| **IIS out-of-process** | ANCM reverse-proxies to a Kestrel child process; use when you need process isolation or specific IIS features |
| **YARP** | You need a *reverse proxy* written in .NET — routing, transforms, auth offload, gradual migration (Module 32's strangler fig) |

When anything sits in front of you, the client's IP, scheme and host arrive in headers, not on the connection. `UseForwardedHeaders` rewrites `HttpContext.Connection.RemoteIpAddress`, `Request.Scheme` and `Request.Host` from `X-Forwarded-For`/`Proto`/`Host`. The traps, in the order they bite:

1. **It must run first**, before anything that reads scheme or IP — including HTTPS redirection, authentication (redirect URIs!), and rate limiting partitioned by IP.
2. **`KnownProxies`/`KnownNetworks` default to loopback only.** In a container behind a cluster ingress, the proxy is not loopback, so the headers are silently ignored. You either register the proxy network or, if the ingress is the only path in and strips inbound forwarded headers, clear the known lists deliberately and document why.
3. **Forwarded headers are client-controllable** unless a trusted proxy overwrites them. Rate limiting or auditing on a spoofable IP is worse than not doing it.

Symptom to recognise instantly: **infinite redirect loops or "https" links rendered as "http"** behind a TLS-terminating proxy — that is forwarded headers not being applied.

---

## Concept 21 — Connection lifetime vs request lifetime, and cancellation semantics

`HttpContext.RequestAborted` is a `CancellationToken` that fires when **the client goes away** — closed connection, HTTP/2 stream reset, or an explicit `Abort()`. It is not a timeout and not a shutdown signal.

Rules that follow:

- **Pass `RequestAborted` down to every async call** (database, HTTP, queue). This is how a cancelled request stops consuming capacity instead of finishing work nobody will read. Under load, this is worth real money.
- **Do not treat cancellation as an error.** An `OperationCanceledException` when `RequestAborted.IsCancellationRequested` is a client disconnect; logging it as an error creates alert noise, and ASP.NET Core's own logging treats it as informational.
- **Do not use `RequestAborted` for work that must complete.** If a payment must be captured regardless, that work belongs in a durable mechanism (outbox, queue), not tied to the socket. This is the same lesson as Module 11's delivery semantics, in miniature.
- **Shutdown is a separate token**: `IHostApplicationLifetime.ApplicationStopping`. Background services use that one.
- **Connection-scoped state is a thing** (`IConnectionItemsFeature`, `ConnectionContext`), but it is rarely what you want in application code, and it is a common source of cross-request leakage when misused.

---

# Part C — The middleware pipeline

This is the part of ASP.NET Core everyone has *used* and few can *explain*. The explanations below are the ones that turn ordering rules into derivations.

## Concept 22 — `RequestDelegate`: the pipeline is a composed function

The entire model is one type:

```csharp
public delegate Task RequestDelegate(HttpContext context);
```

A middleware is a function that takes the *next* `RequestDelegate` and returns a new one. `IApplicationBuilder.Use` stores those functions; `Build()` composes them **in reverse**, starting from a terminal delegate (a 404) and wrapping outward, so the first registered middleware is the outermost:

```csharp
public RequestDelegate Build()
{
    RequestDelegate app = context =>
    {
        context.Response.StatusCode = StatusCodes.Status404NotFound;
        return Task.CompletedTask;
    };
    for (var i = _components.Count - 1; i >= 0; i--)
        app = _components[i](app);
    return app;
}
```

Three things fall straight out of this:

1. **There is no "middleware list" at runtime.** There is one delegate, holding closures. A profiler shows nested frames, not a loop.
2. **Every middleware sees the request on the way in and the response on the way out** — the code before `await next(context)` runs inbound, the code after runs outbound, in reverse order. That is why timing/logging middleware works and why "my code after `next` didn't run" means something downstream didn't return (or threw).
3. **The registration order is the execution order inbound, and the reverse outbound.** Ordering is not a convention; it is function composition.

---

## Concept 23 — `Use`, `Run`, `Map`, `MapWhen`, `UseWhen`: branch semantics

| API | Behaviour |
|---|---|
| `Use(async (ctx, next) => …)` | Normal middleware; call `next` to continue, don't to short-circuit |
| `Run(async ctx => …)` | **Terminal** — no `next`; ends the pipeline |
| `Map("/path", branch)` | Path-prefix branch: strips the prefix into `PathBase`, **never rejoins** the main pipeline |
| `MapWhen(predicate, branch)` | Predicate branch, **never rejoins** |
| `UseWhen(predicate, branch)` | Predicate branch that **rejoins** the main pipeline if the branch does not short-circuit |

The rejoin distinction is the one people get wrong. `UseWhen` is for "run extra middleware for these requests and then continue normally" (e.g. extra logging for `/admin`). `MapWhen` is for "these requests are handled entirely differently."

Two more rules for correctness:

- **Do not both call `next` and write the response.** If you write after `next` returns, the response has probably already started (Concept 27).
- **Always `await next`.** Returning the task is fine and slightly cheaper (`return next(context)`), but calling it without awaiting or returning it is a fire-and-forget bug: the framework will complete the request while your continuation is still running, against a pooled context.

In modern code, `Map` for path branching is largely superseded by endpoint routing (`MapGroup`, Part D). Reach for `Map` when you genuinely need a *separate pipeline* — for example, a health-check port or a static-file branch with different middleware.

---

## Concept 24 — Middleware activation: convention-based vs `IMiddleware`

Two ways to write a class-based middleware, with very different lifetimes.

**Convention-based** (the classic shape): constructor takes `RequestDelegate` plus any **singleton** dependencies; `InvokeAsync(HttpContext, …)` takes per-request dependencies as parameters.

```csharp
public sealed class CorrelationMiddleware(RequestDelegate next, ILogger<CorrelationMiddleware> log)
{
    public async Task InvokeAsync(HttpContext context, ITenantResolver tenants)   // scoped: inject here
    {
        // …
        await next(context);
    }
}
```

The class is **instantiated once, at pipeline build time, and reused for every request** — effectively a singleton. Injecting a scoped service into its *constructor* is a captive dependency: in Development, scope validation throws at startup; in Production without validation, every request shares the first request's `DbContext`, which is a data-corruption bug, not a performance bug. The rule: **constructor = singleton dependencies; `InvokeAsync` parameters = anything else.**

**Factory-based `IMiddleware`**: implement `IMiddleware.InvokeAsync(HttpContext, RequestDelegate)`, register it in DI with the lifetime you want, and add it with `app.UseMiddleware<T>()`. `IMiddlewareFactory` resolves it **per request** from the request scope, so constructor injection of scoped services is legal.

| | Convention-based | `IMiddleware` |
|---|---|---|
| Activation | Once, at build | Per request, from DI |
| Scoped deps | `InvokeAsync` parameters only | Constructor is fine |
| Allocation | Zero per request | One instance per request |
| Testability | Awkward (reflection-based invoke) | Straightforward |
| Strong typing | Duck-typed `InvokeAsync` | Interface-enforced |

Use convention-based by default (it is what the framework uses and it allocates nothing); use `IMiddleware` when the middleware has substantial scoped dependencies or you want it unit-testable.

---

## Concept 25 — Ordering is semantics: the canonical order and its reasons

Memorising the list is mid-level. Deriving it is senior. Here it is with the *reason* for each position:

1. **`UseForwardedHeaders`** — everything downstream that reads scheme/host/IP needs the corrected values.
2. **`UseExceptionHandler`** (or the developer exception page) — must be outermost among things that can handle failures, so it wraps everything below it.
3. **`UseHsts`** — adds a response header; must run for responses produced below.
4. **`UseHttpsRedirection`** — cheap redirect before doing any work.
5. **`UseStaticFiles` / `MapStaticAssets`** — short-circuits asset requests before auth and routing costs are paid. (Note the security consequence: static files are served *before* authorization, so files that need protection must not be served this way.)
6. **`UseRouting`** — selects the endpoint; everything after it can read `HttpContext.GetEndpoint()` and its metadata.
7. **`UseCors`** — needs the endpoint (per-endpoint CORS policies), must precede auth so preflights are not rejected with a 401.
8. **`UseAuthentication`** — populates `HttpContext.User`.
9. **`UseAuthorization`** — evaluates policies from endpoint metadata against `User`.
10. **`UseRateLimiter`, `UseRequestTimeouts`, `UseOutputCache`, `UseSession`, `UseAntiforgery`** — all need the endpoint, and mostly the user; place after authorization, before endpoints.
11. **`UseEndpoints`** (implicit) — executes the selected endpoint.

Every adjacency has a dependency behind it: **CORS before auth** because a browser preflight carries no credentials; **routing before authorization** because the policy lives on the endpoint; **exception handling outermost** because it can only catch what it wraps; **response-modifying middleware outermost** because by the time the inner ones write, it is too late (Concept 27).

The diagnostic form of this concept: when someone says "my middleware doesn't see the user," the answer is "it runs before `UseAuthentication`." When they say "my exception handler doesn't fire," the answer is usually "the exception came from something outside it, or the response had already started."

---

## Concept 26 — Short-circuiting: terminal middleware and the short-circuit endpoint APIs

Three levels of short-circuiting:

1. **Don't call `next`.** The simplest: a middleware that responds itself and returns. Everything below is skipped; everything above still runs on the way out.
2. **`app.MapShortCircuit(404, "robots.txt", "favicon.ico")`** (.NET 8+): the request is matched by routing and then **the rest of the middleware pipeline is skipped entirely** — no auth, no CORS, no session — returning the given status. This exists because noise requests should not pay for your security stack.
3. **`.ShortCircuit()` on an endpoint**, or in .NET 11 the **`[ShortCircuit]` attribute** (which also works on MVC controllers and actions): the endpoint *does* execute, but the middleware between routing and the endpoint is skipped.

```csharp
app.MapGet("/health/live", () => Results.Ok()).ShortCircuit();       // convention
app.MapGet("/robots.txt", () => Results.Text("User-agent: *"));      // or [ShortCircuit] in .NET 11
app.MapShortCircuit(404, "favicon.ico", "*.php");
```

The senior framing: **short-circuiting is a cost decision and a security decision at once.** It removes per-request work from endpoints that don't need it, and it removes middleware you may have assumed always runs. If your audit logging lives in middleware, a short-circuited endpoint is invisible to it — that is either exactly what you wanted or a compliance gap, and you should say which.

---

## Concept 27 — The response-started rule

Once the first byte of the response has been flushed to the client, the status code and headers are **on the wire and unchangeable**. `HttpResponse.HasStarted` tells you which side of that line you are on, and writing headers afterwards throws `InvalidOperationException`.

The response starts when: something writes more than the response buffer; something calls `Response.Body.FlushAsync()` or `StartAsync()`; a result writes the body; or the framework completes the response.

Two mechanisms exist for the "I need to do something at the right moment" case:

```csharp
context.Response.OnStarting(() =>                 // runs just before headers are sent
{
    context.Response.Headers["X-Request-Id"] = id;
    return Task.CompletedTask;
});

context.Response.OnCompleted(() =>                // runs after the response is finished
{
    metrics.Record(sw.Elapsed);
    return Task.CompletedTask;
});
```

Rules of thumb: **set headers inbound or in `OnStarting`, never after `await next(...)`**; keep both callbacks cheap and non-throwing (an exception in `OnStarting` is very hard to diagnose because the response is mid-flight); and remember that **your exception handler cannot help after the response has started** — the best it can do is abort the connection, which is why a half-written response shows up as a truncated body rather than a 500.

---

## Concept 28 — Wrapping the response body correctly

Middleware that must *see* or *modify* the response — compression, caching, sanitisation, logging bodies — has to interpose on the body. The naive version swaps `Response.Body` for a `MemoryStream`. The correct version does three more things:

1. **Replace `IHttpResponseBodyFeature`**, not just `Response.Body`, so that `SendFileAsync`, `PipeWriter`, `StartAsync` and `CompleteAsync` all go through your wrapper (`StreamResponseBodyFeature` exists for this).
2. **Restore the original feature in a `finally`**, always, even on exception.
3. **Bound the buffer.** An unbounded `MemoryStream` over a streaming endpoint is an out-of-memory bug waiting for a large download, and — per Module 14 — anything over 85 KB goes straight to the LOH.

```csharp
var original = context.Features.Get<IHttpResponseBodyFeature>()!;
using var buffer = new MemoryStream();
try
{
    context.Features.Set<IHttpResponseBodyFeature>(new StreamResponseBodyFeature(buffer));
    await next(context);
    buffer.Position = 0;
    await buffer.CopyToAsync(original.Stream, context.RequestAborted);
}
finally
{
    context.Features.Set(original);
}
```

Then the judgement: **most teams should not write this.** Use HTTP logging middleware (which pools its buffers, and in .NET 11 pools them better) for body logging, output caching for caching, and response compression for compression. Writing your own body-buffering middleware and running it on every request is one of the most common self-inflicted latency and memory regressions in .NET services — it converts streaming responses into buffered ones and defeats back-pressure (Concept 13).

---

## Concept 29 — Exception handling: middleware, `IExceptionHandler`, ProblemDetails

The production shape, in one block:

```csharp
builder.Services.AddProblemDetails();                        // RFC 9457 responses
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
builder.Services.AddExceptionHandler<FallbackExceptionHandler>();

app.UseExceptionHandler();                                    // outermost
```

`IExceptionHandler` (.NET 8+) lets you register **several handlers, tried in registration order**; each returns `true` if it handled the exception. This replaces the older pattern of one big lambda with a `switch`, and it composes: domain-validation exceptions map to 400, concurrency conflicts to 409, `TimeoutException` to 504, everything else falls through to a generic 500.

What a senior answer includes:

- **Never leak internals.** Stack traces, SQL, connection strings and internal type names must not reach a client. The developer exception page is Development-only for exactly this reason.
- **Return ProblemDetails** so clients get a consistent, machine-readable shape, with a `traceId` that matches your telemetry (`Activity.Current?.Id`). That single field turns "it failed" support tickets into one-query investigations.
- **Exceptions are not flow control.** Module 17 priced them in microseconds; .NET 11's Kestrel parser change is the framework making the same move. A validation failure is a return value, not a throw.
- **`UseExceptionHandler` re-executes the pipeline** for the error path when given a path; the `IExceptionHandler`/`ProblemDetails` route avoids that re-execution. Know which one you configured.
- **Log once.** The classic double-logging pattern (catch, log, rethrow, handler logs again) doubles your log bill and hides the real count.

---

## Concept 30 — `HttpContext` is pooled: the most expensive misunderstanding

`DefaultHttpContextFactory` and Kestrel **reuse** `HttpContext`, its `HttpRequest`, `HttpResponse`, feature collection and header dictionaries across requests on a connection. When a request ends, the object is reset and handed to the next one.

Therefore:

- **Never store `HttpContext` in a field, a static, a captured closure that outlives the request, or a cache.** A stored reference will later observe *someone else's* request — a correctness and security bug that only appears under concurrency, which is exactly the kind that survives QA.
- **Never touch `HttpContext` after the request completes.** Fire-and-forget work started in a request (`_ = DoLaterAsync(context)`) is the canonical form of this bug. Copy the values you need into a record first, or hand the work to a queue.
- **Do not use `HttpContext` from multiple threads concurrently.** It is not thread-safe; `Parallel.ForEach` over items that each read the context is a race.
- **Headers you read are backed by pooled storage.** If you keep one for later, `ToString()` it.

The interview-grade sentence: *"`HttpContext` is a pooled, request-scoped facade over server features; capturing it is a use-after-free in a garbage-collected language."*

---

## Concept 31 — What the pipeline costs, and how to measure it

Per request, a typical ASP.NET Core service pays for: the `HttpContext` rent (pooled, ~free), **one DI scope** (a `ServiceProviderEngineScope` plus a disposables list), one `Activity` if anything is listening, several metric records, the middleware closures, route matching, endpoint execution, model binding/serialization, and the response write.

How to find out what *yours* costs, in order:

1. **`dotnet-counters` / the built-in meters** for rate, duration and active requests — establish the baseline (Concept 70).
2. **`dotnet-trace` with `cpu-sampling` and `gc-verbose`** under load, opened in PerfView or Speedscope (Module 17, Part D). Middleware shows as nested frames; a fat frame at the top of the pipeline is a body-buffering or logging middleware.
3. **Remove one middleware at a time under identical load.** The pipeline is short; bisection is fast.
4. **Watch for the big four**: serialization, logging (especially structured logging of large objects, or `LogInformation` with interpolated strings instead of `[LoggerMessage]`), body buffering, and per-request reflection.

Two specific and common findings worth naming because interviewers recognise them: **`IHttpContextAccessor` registered and used everywhere** (Concept 45), and **a `Stopwatch`-and-log middleware** that duplicates what the framework's metrics already give you for free.

---

# Part D — Routing and endpoints

Routing is where ASP.NET Core stopped being "a pipeline with MVC bolted on" and became a framework with one unified extension model. If you understand endpoints and metadata, minimal APIs, controllers, gRPC, SignalR, health checks, Blazor and rate limiting all become the same thing wearing different hats.

## Concept 32 — Endpoint routing: match early, execute late

Two middleware, one decision:

- **`UseRouting`** matches the request against the endpoint data sources and sets `HttpContext.Features.Get<IEndpointFeature>().Endpoint`. It does **not** execute anything.
- **`UseEndpoints`** (implicit at the end of a `WebApplication` pipeline) executes the selected endpoint's `RequestDelegate`.

Between those two calls sits everything that needs to *know what will run* before it runs: authorization, CORS, rate limiting, output caching, request timeouts, antiforgery. That is the entire architectural reason for the split, and the reason ASP.NET Core 3.0 replaced the old `UseMvc` routing model.

```csharp
var endpoint = context.GetEndpoint();                  // null before UseRouting or if no match
var policy   = endpoint?.Metadata.GetMetadata<IAuthorizeData>();
var route    = endpoint?.Metadata.GetMetadata<IRouteDiagnosticsMetadata>();
```

Two facts worth stating: **a request that matches nothing has a null endpoint** and falls through to the terminal 404; and **`http.route` on your metrics comes from the matched endpoint**, which is why unmatched requests have no route label and why cardinality stays bounded (you get `/orders/{id}`, not `/orders/12345`).

---

## Concept 33 — Route templates, constraints, and precedence

A template is segments of literals, parameters (`{id}`), optional parameters (`{id?}`), defaults (`{page=1}`), catch-alls (`{*rest}` or `{**rest}` for round-trip-encoded), and constraints (`{id:int}`, `{code:length(6)}`, `{date:datetime}`, `{slug:regex(^[a-z-]+$)}`).

**Precedence** is deterministic and worth memorising, because ambiguity exceptions in production start here. More specific wins, segment by segment:

1. Literal segment (`/orders/active`)
2. Constrained parameter (`/orders/{id:int}`)
3. Unconstrained parameter (`/orders/{id}`)
4. Constrained catch-all
5. Unconstrained catch-all (`/{**rest}`)

Ties are broken by the endpoint's **`Order`** (lower runs first; `MapFallback` uses `int.MaxValue`). A genuine tie throws `AmbiguousMatchException` at request time — a bug that survives to production because it only fires on the colliding URL.

Two important clarifications:

- **Constraints are not validation.** `{id:int}` means "this route doesn't match otherwise," producing a 404, not a 400 with a useful message. Business validation belongs in the handler or the validation pipeline (Concept 57).
- **Custom constraints** implement `IRouteConstraint` and are registered in `RouteOptions.ConstraintMap`. Keep them cheap: they run during matching, potentially several times per request.

---

## Concept 34 — The matcher: a DFA, not a loop

ASP.NET Core does **not** test routes one by one. At startup it compiles all route templates into a **DFA** (`DfaMatcher`, built by `DfaMatcherBuilder`): a state machine over path segments, with jump tables from segment text to next state, plus candidate sets with precedence and policies (HTTP method, host, CORS) applied as `MatcherPolicy` implementations.

Consequences you can state confidently:

- **Matching cost scales with the URL's segment count, not with the number of routes.** A thousand endpoints match about as fast as ten. This is the answer to "does having many endpoints slow down routing?" — no, but it does cost startup time and memory to build the table.
- **HTTP method selection is a policy**, which is why a path that exists but with the wrong verb returns **405** rather than 404.
- **Route matching happens before any of your code**, so you cannot make matching depend on a database lookup. Dynamic routing needs `DynamicRouteValueTransformer` or a rewrite before `UseRouting`.
- Endpoint data sources are **live**: `EndpointDataSource` supports change tokens, which is how Blazor, gRPC reflection, and dynamically-mapped endpoints rebuild the matcher at runtime.

---

## Concept 35 — Endpoints and metadata: the real extensibility model

An `Endpoint` is three things: a `RequestDelegate`, a **metadata collection**, and a display name. `RouteEndpoint` adds the route pattern and order.

Metadata is the universal mechanism. Nearly every cross-cutting feature reads it:

| Metadata | Consumed by |
|---|---|
| `IAuthorizeData`, `AuthorizationPolicy`, `IAllowAnonymous` | Authorization middleware |
| `IEnableCorsAttribute` / `ICorsMetadata` | CORS middleware |
| `IRateLimiterPolicyMetadata` | Rate limiting middleware |
| `OutputCachePolicy` | Output caching middleware |
| `RequestTimeoutAttribute` | Request timeouts middleware |
| `IProducesResponseTypeMetadata`, `IAcceptsMetadata` | OpenAPI generation |
| `IShortCircuitMetadata` | Endpoint short-circuiting |
| `EndpointNameMetadata`, `RouteNameMetadata` | Link generation |
| `IAntiforgeryMetadata` | Antiforgery middleware |

You add metadata with `.WithMetadata(...)`, `.RequireAuthorization()`, `.RequireRateLimiting("api")`, `.CacheOutput()`, `.WithName("GetOrder")`, `.Produces<Order>(200)`, or attributes on a controller action or minimal API delegate.

The senior insight worth stating out loud: **this is how ASP.NET Core avoids a combinatorial explosion of middleware×framework integrations.** Authorization doesn't know what MVC is; it knows what `IAuthorizeData` is. When you write a cross-cutting feature of your own, the framework-idiomatic design is metadata on the endpoint plus one middleware that reads it — not a base controller, not a service locator, not a static.

---

## Concept 36 — Route groups and conventions

`MapGroup` (since .NET 7) is the compositional unit for endpoints:

```csharp
var orders = app.MapGroup("/api/v1/orders")
    .RequireAuthorization("orders:read")
    .RequireRateLimiting("api")
    .AddEndpointFilter<TenantScopeFilter>()
    .WithTags("Orders")
    .WithOpenApi();

orders.MapGet("/{id:guid}", GetOrder).WithName("GetOrder");
orders.MapPost("/", CreateOrder).RequireAuthorization("orders:write");
orders.MapGroup("/{id:guid}/lines").MapGet("/", GetLines);      // groups nest
```

Everything applied to the group is applied to every endpoint under it, groups nest, and the prefix composes. `AddEndpointFilterFactory` lets you build filters that inspect the handler's signature at startup and specialise — the fast way to write cross-cutting endpoint behaviour that costs nothing per request when it doesn't apply.

Conventions (`IEndpointConventionBuilder.Add(builder => …)`) run at **build time** over the endpoint builder, which is where library authors hook in. The distinction matters: a convention shapes the endpoint once; a filter runs per request.

---

## Concept 37 — Link generation: the inverse of matching

`LinkGenerator` turns route values back into URLs, and unlike the old `IUrlHelper` it works **outside an HTTP request** — in a background service generating an email link, for instance.

```csharp
var url = links.GetUriByName(httpContext, "GetOrder", new { id = order.Id });
// outside a request, supply scheme/host explicitly:
var abs = links.GetUriByName("GetOrder", new { id }, scheme: "https", host: new HostString("api.example.com"));
```

Two practical points: **name your endpoints** (`.WithName(...)`) if anything generates links to them, because generating by route values alone is fragile; and be careful with host/scheme outside a request — you must supply them, and getting them from configuration (not from a header) is the safe choice, since the alternative is host-header injection in password-reset emails, a well-known vulnerability class.

---

## Concept 38 — Routing-related ordering traps

The three that recur:

1. **Middleware before `UseRouting` has no endpoint.** `context.GetEndpoint()` returns null. If your middleware needs the route or its metadata, it must sit between `UseRouting` and `UseEndpoints` — which, with `WebApplication`, is where `app.Use(...)` puts it by default *unless* you called `UseRouting` yourself somewhere odd.
2. **Middleware added after endpoint mapping still runs before the endpoint.** Registration order for `Use` is independent of where `Map` calls appear in `Program.cs`; endpoints always execute last because `UseEndpoints` is appended at the end. Many people expect that `app.Use(...)` written after `app.MapGet(...)` won't run — it does.
3. **Authorization requires routing.** `UseAuthorization` without `UseRouting` before it throws at startup with a clear message, which is the framework protecting you from a silent security hole: with no endpoint, there is no policy metadata, so nothing would be enforced.

A fourth, subtler one: **`MapFallback` is not a 404 handler.** It is a lowest-priority endpoint (SPA hosting uses it). Requests that reach the terminal delegate because nothing matched are a different path, and if you want a formatted 404 body you need `UseStatusCodePages` or a fallback endpoint — not an exception handler.

---

# Part E — Dependency injection in depth

Module 14 and Module 15 both deferred the captive-dependency problem to this module. Here it is, along with the rest of the container's behaviour — because in a .NET service, most memory leaks, most "disposed object" exceptions and a surprising number of correctness bugs are DI lifetime bugs.

## Concept 39 — `ServiceDescriptor` and the resolution engines

A registration is a `ServiceDescriptor`: service type, one of {implementation type, factory delegate, singleton instance}, a lifetime, and (since .NET 8) an optional **key**. `IServiceCollection` is just `IList<ServiceDescriptor>`, which is why `Remove`, `Replace` and `TryAdd*` are simple list operations.

At `Build()`, `ServiceProvider` compiles a **call-site graph**: for each service type, a tree of constructor calls, factory invocations, and cached-instance lookups. Then it picks an engine:

| Engine | When | Behaviour |
|---|---|---|
| `DynamicServiceProviderEngine` | Default on platforms with dynamic code | Starts with the runtime resolver; after a couple of resolutions, compiles the call site (expression trees / IL) on a background thread |
| `ILEmitServiceProviderEngine` / `ExpressionsServiceProviderEngine` | Explicit modes | Compiled resolution |
| `RuntimeServiceProviderEngine` | **Native AOT / no dynamic code** | Interpreted call sites — still fast, slightly slower than compiled |

Two facts for interviews: **the first resolution of a type is the expensive one** (call-site construction, possibly compilation), which is part of first-request cost; and **the container is deliberately minimal** — no property injection, no auto-registration, no interception, no named registrations before keyed services. That minimalism is a feature: it is fast, predictable, and it makes container behaviour explainable, which is why most teams should keep it (Concept 52).

---

## Concept 40 — The three lifetimes, defined by scope rather than intent

Define them mechanically, not by vibes:

| Lifetime | Instance per | Cached in | Disposed when |
|---|---|---|---|
| **Singleton** | Container (root scope) | Root provider | Provider disposed (app shutdown) |
| **Scoped** | Scope | That scope | Scope disposed (request end) |
| **Transient** | Every resolution | Nowhere (but **tracked** if disposable) | The scope that created it is disposed |

The clarifications that separate answers:

- **A singleton resolved from a scope is still the root's instance.** Lifetime is about where the instance is cached, not where it was asked for.
- **Transient does not mean untracked.** If a transient implements `IDisposable`, the resolving scope holds a reference until disposal — the leak in Concept 43.
- **"Scoped" in ASP.NET Core means "per request"** only because the hosting layer creates one scope per request. In a worker, a scope is whatever you create (Concept 46). Saying "scoped means per request" without that qualifier is a small but audible imprecision.
- **Default choice:** stateless services → singleton (cheapest, no per-request allocation); anything holding per-request state or a unit of work (`DbContext`) → scoped; transient mostly for small, cheap, stateful helpers. If a service is stateless and you registered it scoped "to be safe," you are paying an allocation and a tracking entry per request for nothing.

---

## Concept 41 — The request scope: who creates it and when it ends

The hosting layer (`HostingApplication`) creates an `AsyncServiceScope` per request, exposes its `IServiceProvider` as `HttpContext.RequestServices`, invokes the pipeline, and then **disposes the scope after the response completes** — including the body flush.

Practical consequences:

- **`HttpContext.RequestServices` is the request's container.** Resolving from `app.Services` inside a request instead would give you the root scope — which means scoped services would be resolved as if they were singletons, and disposables would accumulate for the life of the process.
- **Work that outlives the response cannot use request services.** Fire-and-forget continuations will hit `ObjectDisposedException` on a `DbContext` "randomly" — random because it depends on whether the response finished first.
- **Streaming responses extend the scope**, which is good (your services stay alive while you stream) and a constraint (long SSE or download connections hold their entire scope's object graph for the duration — a real memory consideration at thousands of concurrent streams).

---

## Concept 42 — Captive dependencies: definition, failure mode, detection

**A captive dependency is a longer-lived service holding a reference to a shorter-lived one.** The classic: a singleton that injects a scoped `DbContext`. The container resolves the `DbContext` once, from whichever scope was active at first resolution, and the singleton keeps it forever.

The failure modes, in the order they appear in production:

1. **Stale data** — every request sees the first request's change-tracked entities.
2. **`ObjectDisposedException`** — the captured scope was disposed; the next request using the singleton throws.
3. **Concurrency corruption** — `DbContext` is not thread-safe, so concurrent requests through the singleton produce "A second operation was started on this context" errors or worse.
4. **A leak** — the scope's whole object graph is pinned by the singleton.

**Detection is a configuration setting, and it should be on everywhere you can afford it:**

```csharp
builder.Host.UseDefaultServiceProvider((context, options) =>
{
    options.ValidateScopes  = true;   // throw when a singleton resolves a scoped service
    options.ValidateOnBuild = true;   // verify every registration can be constructed, at startup
});
```

Both default to **true in Development** and **false elsewhere**. The senior recommendation: enable `ValidateOnBuild` in all environments (it is a startup-time cost that converts a runtime failure into a deployment failure) and enable `ValidateScopes` at least in your integration test suite, where it will catch the captive dependency your Development runs missed because nothing exercised that path.

Note the direction that is *fine*: a scoped service depending on a singleton, or a transient depending on anything. The rule is only violated downward in lifetime length.

---

## Concept 43 — Disposal: who disposes what, and the transient-disposable leak

Two rules, and the second one causes leaks:

1. **The container disposes what the container created.** Registered types and factory results implementing `IDisposable`/`IAsyncDisposable` are disposed with their owning scope (or the root provider for singletons).
2. **The container does *not* dispose instances you created yourself.** `services.AddSingleton(new ExampleService())` registers an instance the container did not construct; **the framework will not dispose it** and it is your responsibility. If you want container-managed disposal, register a factory instead: `services.AddSingleton<IExample>(sp => new ExampleService(...))`.

Now the leak. **Transient `IDisposable`s are tracked by the resolving scope.** Resolve a transient disposable from the **root** provider — from a singleton, or from `app.Services` in a background loop — and the reference is held until the process exits. This is one of the most common "slow memory growth with no obvious cause" findings in .NET services, and it looks exactly like the mid-life-crisis pattern from Module 14.

Guidance that avoids all of it:

- Avoid transient `IDisposable` registrations; make them scoped or singleton, or don't put them in DI at all.
- Never resolve services from the root provider at runtime — create a scope.
- For "I need a disposable per operation," inject a **factory** (`Func<T>`, or a dedicated `IThingFactory`) and `using` the result. Ownership is then explicit and local, which is the same ownership discipline Module 17 applied to pooled buffers.
- `ServiceProvider.DisposeAsync()` awaits each `IAsyncDisposable` with `ConfigureAwait(false)`; don't rely on disposal resuming on any particular context.

---

## Concept 44 — Singletons, concurrency, and captured request state

A singleton is shared by every concurrent request, so **every mutable field in it is shared mutable state**. That is fine for caches (use a concurrent collection), configuration snapshots (immutable), and connection multiplexers (designed for it). It is not fine for:

- `DbContext`, `HttpContext`, `ClaimsPrincipal`, the current tenant, the current correlation id, or anything else that describes "this request."
- Non-thread-safe collections used without synchronization — a `Dictionary<K,V>` written concurrently can corrupt into an infinite loop on read, which presents as a pegged CPU core with no obvious cause.
- Lazy initialization done naively: `if (_x is null) _x = Create();` runs `Create()` more than once under load. Use `Lazy<T>` with `ExecutionAndPublication`, or `LazyInitializer`, or initialize eagerly.

Module 15's rules apply unchanged: `lock` (or the `System.Threading.Lock` type in .NET 9+) around short critical sections, `Interlocked` for counters, `ConcurrentDictionary` with an eye on the factory running more than once, and never block on async inside a lock.

The reviewer's question that catches this: *"this singleton has a field — who writes it, and from how many threads?"*

---

## Concept 45 — `IHttpContextAccessor`: how it works, what it costs, what it breaks

`IHttpContextAccessor` is backed by an `AsyncLocal<HttpContextHolder>`. The hosting layer sets it at the start of the request; the holder is nulled when the request ends (so a captured accessor does not resurrect a pooled context — it observes null, which is the framework protecting you).

Costs and hazards:

- **Registering it makes `ExecutionContext` flow matter more.** `AsyncLocal` reads and writes are not free, and every async state machine transition carries the context. On a high-throughput service this is measurable — the ASP.NET Core team's own benchmarks treat it as a real cost, not a rounding error.
- **It is null outside a request.** Background services, `IHostedService` startup, and anything after the response completes see null. Code that assumes otherwise NREs in production and not in tests.
- **It is an ambient dependency — a service locator in a nicer coat.** A domain service that reads the current user via `IHttpContextAccessor` cannot be unit tested without a fake HTTP context, cannot be reused from a worker, and hides its real inputs.

The idiomatic alternative: **resolve the ambient facts once, at the edge, into a small scoped object.**

```csharp
public sealed record RequestContext(string TenantId, string UserId, string CorrelationId);

builder.Services.AddScoped<RequestContext>(sp =>
{
    var http = sp.GetRequiredService<IHttpContextAccessor>().HttpContext
               ?? throw new InvalidOperationException("Not in a request");
    return new RequestContext(
        http.User.FindFirstValue("tid")!,
        http.User.FindFirstValue(ClaimTypes.NameIdentifier)!,
        http.TraceIdentifier);
});
```

Now one place knows about HTTP, everything downstream takes a plain record, and the same services run in a worker by constructing the record differently. That is the shape senior reviewers look for.

---

## Concept 46 — Using scoped services from singletons and background services

The rule is: **do not inject a scope; create one, and make its boundary a unit of work.**

```csharp
public sealed class ReconciliationJob(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var scope = scopes.CreateAsyncScope();   // async-aware disposal
            var work = scope.ServiceProvider.GetRequiredService<IReconciler>();
            await work.RunAsync(ct);
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

Points that make this a good answer:

- **`CreateAsyncScope()` over `CreateScope()`** whenever anything in the scope might be `IAsyncDisposable` (`DbContext` is). `CreateScope()` with an `IAsyncDisposable`-only service throws on dispose.
- **One scope per unit of work, not one per service lifetime.** A scope that lives for the whole job is a scope that accumulates change-tracked entities forever — the EF Core version of the leak in Concept 43.
- **Never cache a resolved scoped service in a singleton field**, even from a correctly created scope. That is a captive dependency built by hand.
- `IServiceScopeFactory` is itself a **singleton**, which is why injecting it into a singleton is legal and injecting `IServiceProvider` and calling `CreateScope()` is equivalent but weaker (it also hands the class the ability to resolve anything — Concept 76).

---

## Concept 47 — Multiple registrations, `TryAdd*`, and decoration

Semantics you should know without checking:

- Registering the same service type twice: **`GetRequiredService<T>()` returns the last registration**; **`GetServices<T>()` / injecting `IEnumerable<T>` returns all, in registration order.**
- **`TryAdd*`** adds only if the service type is absent — the correct call for a library providing a default the app may override.
- **`TryAddEnumerable`** adds only if that *exact implementation type* is absent — the correct call for contributing one handler to a set without duplicating on repeated `AddX()` calls. Libraries that use plain `Add` here cause double-registration bugs when their `AddX()` is called twice.
- **`Replace`** and **`RemoveAll<T>()`** exist for tests and for genuinely overriding a framework registration.
- **Decoration** is not built in. Hand-rolled:

```csharp
services.AddScoped<OrderService>();                                  // the real one
services.AddScoped<IOrderService>(sp => new CachingOrderService(
    sp.GetRequiredService<OrderService>(),
    sp.GetRequiredService<IMemoryCache>()));
```

  Scrutor's `services.Decorate<IOrderService, CachingOrderService>()` does it generically. Use decoration for cross-cutting behaviour over a small number of services; when you find yourself decorating everything, you have reinvented interception, and a pipeline (endpoint filters, or a mediator pipeline in Module 23) is usually the cleaner model.

- **Open generics** are supported: `services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>))`. This is how `ILogger<T>` and `IOptions<T>` work, and it is cheap.

---

## Concept 48 — Keyed services

Since .NET 8 the built-in container supports **keyed registrations**, which removes the most common reason teams switched containers:

```csharp
builder.Services.AddKeyedSingleton<IBlobStore, AzureBlobStore>("primary");
builder.Services.AddKeyedSingleton<IBlobStore, S3BlobStore>("archive");

public sealed class ReportService(
    [FromKeyedServices("primary")] IBlobStore hot,
    [FromKeyedServices("archive")] IBlobStore cold) { }
```

Details: `GetRequiredKeyedService<T>(key)`, `KeyedService.AnyKey` to receive any keyed instance, `[FromKeyedServices]` works in minimal API handlers and controller constructors, and keyed and non-keyed registrations of the same type are independent.

Use them for genuinely interchangeable implementations selected by configuration or tenant. **Do not** use them as a general-purpose service locator (`GetKeyedService<IHandler>(commandName)` in a switch) — that is Concept 76's anti-pattern with a modern API.

---

## Concept 49 — `ActivatorUtilities`, factories, and the line around DI

Not everything belongs in the container. Entities, value objects, DTOs, messages and per-operation state should be constructed with `new`, because their lifetime is tied to a logical operation, not to a container scope. Putting them in DI is how "service" ends up meaning nothing.

When you need a type constructed with *some* dependencies injected and *some* arguments supplied:

```csharp
var importer = ActivatorUtilities.CreateInstance<CsvImporter>(serviceProvider, stream, options);
// or, for hot paths, cache the factory:
private static readonly ObjectFactory<CsvImporter> Factory =
    ActivatorUtilities.CreateFactory<CsvImporter>([typeof(Stream), typeof(ImportOptions)]);
```

`ActivatorUtilities` is what the framework itself uses for controllers, middleware, SignalR hubs, and Razor components. `CreateFactory` caches the constructor selection, which matters if you do this per request.

The design sentence: **DI is for wiring long-lived collaborators, not for managing data.** Reviewers notice the difference immediately.

---

## Concept 50 — `IHttpClientFactory`: the canonical lifetime lesson

This one is nearly guaranteed to come up, because it is where lifetime mistakes have famous consequences.

The two failure modes it exists to solve:

1. **`new HttpClient()` per request** → each disposal leaves sockets in `TIME_WAIT`; under load you exhaust ephemeral ports (`SocketException: Only one usage of each socket address…`).
2. **One static `HttpClient` forever** → the handler caches DNS at connection time; when the target's IP changes (failover, scale-out, blue/green), you keep talking to the old address.

`IHttpClientFactory` solves both by **pooling `HttpMessageHandler`s and rotating them** on `HandlerLifetime` (default **2 minutes**): the `HttpClient` is cheap and disposable, the handler underneath is shared and rotated.

```csharp
builder.Services.AddHttpClient<IPricingClient, PricingClient>(c =>
{
    c.BaseAddress = new Uri("https://pricing.internal/");
    c.Timeout = TimeSpan.FromSeconds(5);                       // always set one
})
.SetHandlerLifetime(TimeSpan.FromMinutes(2))
.AddStandardResilienceHandler();                               // Polly-based, Module 25
```

Four details that mark a senior answer:

- **Typed clients are registered transient**, and their handler comes from the pool. So the *client* is cheap; never cache a typed client in a singleton field expecting rotation to happen.
- **Handlers get their own DI scope, not the request scope.** A `DelegatingHandler` that injects a scoped service does **not** get the current request's instance — this surprises everyone the first time, and it is why correlation/auth handlers should read from `IHttpContextAccessor` (deliberately) or be given what they need per call, rather than injecting scoped services.
- **The modern alternative to handler rotation** is a single `SocketsHttpHandler` with `PooledConnectionLifetime` set (typically 2–5 minutes), which handles DNS refresh at the connection level. `IHttpClientFactory` remains valuable for named/typed configuration, logging, and the resilience pipeline, but you should know both answers.
- **Timeouts**: `HttpClient.Timeout` covers the whole request including the body; per-attempt timeouts belong in the resilience pipeline. An un-timed-out client is how one slow dependency consumes all your threads (Module 13's cascading failure, Module 15's starvation).

---

## Concept 51 — DI performance: what is cheap, what is not

Measured reality, so you can push back on folklore:

- **A resolution is nanoseconds** once the call site is compiled. Nobody's p99 is caused by `GetRequiredService`.
- **What costs** is graph *size* and *shape*: a controller whose constructor pulls 15 services, each pulling 5 more, allocates dozens of objects per request. With scoped lifetimes that is a per-request allocation bill, and with disposables it is a per-request tracking list. This is the real argument for smaller services and for singletons where state allows.
- **Startup cost** comes from `ValidateOnBuild` (proportional to registration count) and, much more, from **assembly scanning** (Scrutor, MediatR-style auto-registration, AutoMapper profiles). Scanning a few dozen assemblies with reflection is tens to hundreds of milliseconds and is invisible until someone complains about cold start. Explicit registration modules cost typing and save startup.
- **AOT** uses the runtime (non-compiled) engine, so resolution is somewhat slower, but there is no JIT and no call-site compilation — net startup is far better. Reflection-based registration is also a trimming hazard.
- The optimization that actually pays: **make the hot path's graph shallow**. An endpoint that needs one repository and one cache should not construct a `MediatR` pipeline of eight behaviours per request unless those behaviours earn it (Module 23 argues that case properly).

---

## Concept 52 — Replacing the container, and whether you should

Third-party containers (Autofac, Lamar, DryIoc, SimpleInjector) plug in through `IServiceProviderFactory<TBuilder>`:

```csharp
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
builder.Host.ConfigureContainer<ContainerBuilder>(b => b.RegisterModule<AppModule>());
```

Legitimate reasons: **property injection**, **interception/decoration as a first-class feature**, **assembly-scanning conventions**, **child containers**, **disposal or lifetime semantics** the built-in container doesn't model, or an existing large codebase already built on one.

Reasons that have expired: named registrations (keyed services, .NET 8) and "it's faster" (the built-in container is competitive and its resolution is not your bottleneck).

Costs to name: every library you use assumes `Microsoft.Extensions.DependencyInjection` semantics; you inherit a different disposal and scoping model; new team members must learn it; AOT and trimming support varies; and diagnosing a resolution failure now requires knowing two containers. The default recommendation for a new .NET service in 2026 is **stay on the built-in container**, and treat needing more as a signal to simplify the design first.

---

# Part F — Endpoint programming models: minimal APIs and MVC

The question "minimal APIs or controllers?" is asked in almost every .NET interview, and most answers are taste. This part gives you the mechanics so your answer can be evidence.

## Concept 53 — What a minimal API endpoint actually is

`app.MapGet("/orders/{id:guid}", (Guid id, IOrderService svc) => svc.GetAsync(id))` is not interpreted at request time. At **startup**, `RequestDelegateFactory` inspects the delegate's signature with reflection and **emits a `RequestDelegate`** using expression trees: read `id` from route values, parse it, resolve `IOrderService` from `RequestServices`, invoke, convert the result to a response. It also produces metadata (accepts, produces, parameter descriptions) for OpenAPI.

Two consequences:

- **Per request, there is no reflection and no model binder** — just the compiled delegate. This is why minimal APIs benchmark ahead of controllers; the gap is small in absolute terms for a real service, but the machinery is genuinely thinner.
- **Startup pays**, once per endpoint, unless you use the **Request Delegate Generator**.

The **RDG** is a source generator that produces the same delegates at **compile time**, using C# interceptors to rewrite your `MapGet` calls. It is enabled implicitly when publishing with `PublishAot` or `PublishTrimmed`, and can be turned on explicitly:

```xml
<PropertyGroup>
  <EnableRequestDelegateGenerator>true</EnableRequestDelegateGenerator>
</PropertyGroup>
```

Enabling it manually is a good practice even without AOT: it eliminates runtime code generation (a trimming and AOT hazard), shaves startup, and — usefully — **surfaces AOT incompatibilities as build diagnostics** (the `RDG###` warnings) long before you attempt a Native AOT publish.

---

## Concept 54 — Parameter binding: the rules, in order

Minimal API binding sources, in precedence order:

1. **Explicit attributes**: `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromForm]`, `[FromServices]`, `[FromKeyedServices("k")]`, `[AsParameters]`.
2. **Special types**, bound automatically: `HttpContext`, `HttpRequest`, `HttpResponse`, `ClaimsPrincipal`, `CancellationToken` (which is `RequestAborted`), `Stream`, `PipeReader`, `IFormFile`/`IFormFileCollection`, `IFormCollection`.
3. **A registered service**, when the type is known to the container (`IServiceProviderIsService`).
4. **`BindAsync`** — a static `public static ValueTask<T?> BindAsync(HttpContext, ParameterInfo)` on the type takes complete control.
5. **`TryParse`** — a static `TryParse(string, out T)` (or with `IFormatProvider`) makes the type bindable from **route or query**.
6. **Route values, then query string**, matched by parameter name for simple types.
7. **The body**, deserialized as JSON, for a complex type with none of the above. **At most one body parameter.**

Two patterns worth having in your pocket:

```csharp
// Strongly-typed ids bind from the URL without ceremony
public readonly record struct OrderId(Guid Value)
{
    public static bool TryParse(string? s, out OrderId id) =>
        Guid.TryParse(s, out var g) ? (id = new(g)) is var _ : (id = default) is var _ && false;
}

// Group many query parameters into one type
public readonly record struct PageQuery([FromQuery] int Page = 1, [FromQuery] int Size = 20);
app.MapGet("/orders", ([AsParameters] PageQuery q, IOrderService s) => s.ListAsync(q.Page, q.Size));
```

Binding failures produce a **400** (with `BadHttpRequestException` surfaced to the developer exception page in Development, controlled by `RouteHandlerOptions.ThrowOnBadRequest`). New in .NET 11: when an endpoint has filters, **the filter pipeline runs even if binding failed**, so a filter can see `StatusCode == 400` and substitute a consistent error body — previously impossible.

---

## Concept 55 — Results: `IResult`, `TypedResults`, ProblemDetails, and streaming

`Results.Ok(x)` returns `IResult`; `TypedResults.Ok(x)` returns `Ok<T>`, a concrete type. Prefer `TypedResults`, because the static type flows into OpenAPI metadata and into tests:

```csharp
static async Task<Results<Ok<OrderDto>, NotFound, ProblemHttpResult>> GetOrder(
    OrderId id, IOrderService svc, CancellationToken ct)
{
    var order = await svc.FindAsync(id, ct);
    return order is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(order.ToDto());
}
```

`Results<T1, T2, …>` is a union of possible results; the framework derives "this endpoint returns 200 or 404" for the OpenAPI document without `.Produces<T>(…)` annotations. In .NET 11, **C# 15 union types** are supported anywhere `System.Text.Json` is used and are described as `anyOf` in OpenAPI, which is a cleaner way to model "one of these payloads" than a nullable grab-bag DTO.

Other results worth knowing: `TypedResults.Problem(...)` and `ValidationProblem(...)` for RFC 9457 bodies; `TypedResults.Stream(...)` and `File(...)` for large payloads (which use `SendFileAsync` where the server supports it); and `TypedResults.ServerSentEvents(...)` (.NET 10) for push:

```csharp
app.MapGet("/orders/stream", (CancellationToken ct) =>
    TypedResults.ServerSentEvents(StreamOrdersAsync(ct)));

static async IAsyncEnumerable<SseItem<OrderDto>> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct) { /* … */ }
```

SSE is the right tool when you need server→client push over plain HTTP with automatic reconnect and no WebSocket infrastructure; SignalR remains the answer when you need bidirectional messaging, groups, and transport fallback. Note the streaming caveat from Concept 41: a long-lived SSE connection holds its request scope open.

---

## Concept 56 — Endpoint filters

`IEndpointFilter` is minimal APIs' cross-cutting mechanism, and it is *inside* the endpoint: it runs after routing, after authorization, and after parameter binding (or, in .NET 11, after binding failed).

```csharp
public sealed class IdempotencyFilter(IIdempotencyStore store) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var key = ctx.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (await store.TryGetAsync(key) is { } cached) return cached;   // short-circuit
        var result = await next(ctx);
        await store.SaveAsync(key, result);
        return result;
    }
}

orders.MapPost("/", CreateOrder).AddEndpointFilter<IdempotencyFilter>();
```

Points: filters nest like middleware (registration order = outermost first); `ctx.Arguments` gives you the bound parameters by index (`GetArgument<T>(i)`); returning without calling `next` short-circuits; and `AddEndpointFilterFactory` lets you inspect the handler's `MethodInfo` at **build time** and return a filter only when it applies — so an "audit if the handler takes a `Command`" filter costs nothing on endpoints that don't.

Versus middleware: middleware sees every request and runs before routing knows anything; filters see one endpoint's bound arguments. Versus MVC filters: same idea, different pipeline — an MVC action does **not** run endpoint filters, and a minimal API endpoint does **not** run MVC filters. Mixing the two models in one codebase means writing every cross-cutting concern twice, which is itself an argument for picking one (Concept 60).

---

## Concept 57 — Validation: built-in (.NET 10) and asynchronous (.NET 11)

Before .NET 10, minimal APIs had no built-in validation and every team wrote a filter or pulled in FluentValidation. Now:

```csharp
builder.Services.AddValidation();          // Microsoft.Extensions.Validation

app.MapPost("/reservations", (ReservationRequest request) => TypedResults.Ok(request));

public sealed record ReservationRequest(
    [property: Required, EmailAddress] string Email,
    [property: Range(1, 10)] int Guests);
```

Mechanics that matter:

- It is **source-generated and AOT-friendly** — no runtime reflection walk of your model graph.
- It validates **body, route, query, header and form** parameters, plus nested objects and collections, using `DataAnnotations`.
- `[ValidatableType]` forces generation for types the generator can't discover; `DisableValidation()` opts an endpoint out; both attributes left experimental status in .NET 11.
- Failures produce a `ValidationProblemDetails` 400 that you can shape through **`IProblemDetailsService`**.
- **.NET 11 adds async validation**: `AsyncValidationAttribute` and `IAsyncValidatableObject` (returning `IAsyncEnumerable<ValidationResult>`), run concurrently where possible, with DI available via `ValidationContext.GetRequiredService<T>()`. That makes "is this email already registered?" a first-class validation rule instead of a hand-rolled filter.
- **.NET 11 also localizes validation messages** built in: register `AddLocalization()` before `AddValidation()` and messages resolve by resource-name convention (`{DeclaringType}_{MemberName}_{AttributeType}_Error`, then less specific forms), or through a custom `IStringLocalizerFactory`.

The architectural caution, which is the part interviewers care about: **input validation is not domain validation.** `[Range(1, 10)]` on `Guests` says the payload is well-formed. "This room is available on that date" is an invariant that belongs in the domain (Module 22) and must be enforced where the transaction is, not at the edge — because between validation and commit, the world changes.

---

## Concept 58 — MVC's pipeline: model binding, filters, and result execution

When a request matches a controller endpoint, MVC runs its **own** pipeline inside the endpoint:

1. **Authorization filters** (largely superseded by the authorization middleware).
2. **Resource filters** — wrap everything below, including model binding. `IAsyncResourceFilter` is where MVC-level caching or short-circuiting belongs.
3. **Model binding and validation** — value providers (route, query, form, body), model binders, `ModelState`.
4. **Action filters** — run before and after the action method.
5. **The action** — the controller is created per request by `IControllerActivator` via `ActivatorUtilities`.
6. **Result filters** — run before and after the `IActionResult` executes.
7. **Result execution** — the formatter writes the body (`IOutputFormatter`; content negotiation via `Accept` unless `[Produces]` pins it).
8. **Exception filters** — catch exceptions from the action and action filters (not from result execution or later middleware).

Ordering across scopes: **global → controller → action** on the way in, reversed on the way out, with `IOrderedFilter.Order` overriding scope. Filters registered as types are resolved from DI when registered with `ServiceFilterAttribute`/`TypeFilterAttribute` or via `[FromServices]`-style constructor injection in a `IFilterFactory`; a plain attribute filter is constructed by the framework with only its constructor arguments, so it cannot take scoped dependencies — the attribute-filter equivalent of Concept 24.

What MVC gives you that minimal APIs don't: content negotiation, model binding of complex forms, `ModelState`, views and Razor Pages, `ApiExplorer` conventions, application parts and feature providers (plug-in controllers from other assemblies), and a mature filter ecosystem. What it costs: more per-request machinery, heavy reflection at startup, and **no Native AOT support**.

---

## Concept 59 — `[ApiController]` and its conventions

Applying `[ApiController]` to a controller opts into behaviours you should be able to list, because each is a convention that surprises someone eventually:

- **Automatic 400 on invalid `ModelState`** — before your action runs. Convenient, and occasionally wrong (you may want to accept partially-valid input); disable with `SuppressModelStateInvalidFilter`.
- **Binding source inference** — complex types bind from the body, simple types from route/query, `IFormFile` from form; no `[FromBody]` needed.
- **`ProblemDetails` for error status codes**, matching minimal APIs' default.
- **Attribute routing required** — conventional routes don't apply.
- **Multipart/form-data inference** for `IFormFile` parameters.

The related architectural point: `[ApiController]` conventions are configured through `ApiBehaviorOptions`, and the error shape should be **identical across minimal APIs and controllers** in the same service. Two different error contracts in one API is the kind of inconsistency that generates client bugs for years.

---

## Concept 60 — Minimal APIs vs controllers: the honest decision

| Dimension | Minimal APIs | Controllers |
|---|---|---|
| Per-request overhead | Slightly lower (no model binder, no filter pipeline unless added) | Slightly higher |
| Startup | Lower, and can be compile-time with RDG | Higher (reflection over controllers/actions) |
| **Native AOT** | **Partially supported** | **Not supported** |
| Cross-cutting | Endpoint filters + groups | Filters at three scopes, conventions, application parts |
| Model binding | Simple sources + `TryParse`/`BindAsync` | Full value-provider/binder model, complex forms |
| Validation | `AddValidation()` (.NET 10+) | `ModelState` + `[ApiController]` |
| Content negotiation | JSON by default; you opt in to more | Built-in via formatters |
| Views / Razor Pages | No | Yes |
| Organizational fit | Needs deliberate structure (Concept 61) | Structure imposed by convention |

The decision rule that holds up under questioning:

- **New JSON/HTTP API, especially if startup, footprint or AOT matter** → minimal APIs.
- **Server-rendered UI, complex form binding, content negotiation, or an existing large MVC codebase** → controllers. Migrating a working MVC app to minimal APIs for performance is almost never justified by the numbers.
- **Mixed is legal and sometimes right** (controllers for the admin UI, minimal APIs for the public API), but pay the tax consciously: two filter models, two validation stories, two sets of conventions.

And the meta-answer interviewers reward: *"Performance is rarely the deciding factor at our request rates — the deciding factors are AOT, team familiarity, and whether we need MVC's binding and view machinery. I'd pick one per service, not per endpoint."*

---

## Concept 61 — Organizing minimal APIs at scale

The commonest objection to minimal APIs is "`Program.cs` becomes 3,000 lines." That is a discipline problem with a standard solution: **endpoint modules plus groups.**

```csharp
public interface IEndpointModule
{
    static abstract void MapEndpoints(IEndpointRouteBuilder app);   // static abstract, C# 11+
}

public sealed class OrderEndpoints : IEndpointModule
{
    public static void MapEndpoints(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders").RequireAuthorization().WithTags("Orders");
        group.MapGet("/{id:guid}", GetOrder).WithName(nameof(GetOrder));
        group.MapPost("/", CreateOrder);
    }

    private static async Task<Results<Ok<OrderDto>, NotFound>> GetOrder(/* … */) => /* … */;
}
```

Register them **explicitly** (`OrderEndpoints.MapEndpoints(app);` — a list of one-liners), not by assembly scanning, for the startup reason in Concept 51 and the AOT reason in Concept 75. Handlers as `static` methods keep them testable as plain functions and avoid closure allocations.

This structure also composes naturally with **vertical slices** (Module 20/21 territory): a feature folder holds its endpoint, its request/response records, its handler, and its tests. The endpoint file is thin; the logic is a normal class you can unit test without any HTTP machinery. Libraries like FastEndpoints and Carter formalize the same idea; know they exist, and know that the built-in primitives are sufficient.

---

## Concept 62 — OpenAPI as the contract surface

`Microsoft.AspNetCore.OpenApi` generates the document from **endpoint metadata**, which is why `TypedResults`, `Results<…>`, `[ProducesResponseType]`, validation attributes and XML doc comments all improve it for free.

```csharp
builder.Services.AddOpenApi(options =>
{
    options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_1;   // pin if your tooling lags
});
app.MapOpenApi();                                             // /openapi/v1.json (and .yaml)
```

Current state: **.NET 10 defaults to OpenAPI 3.1** and supports YAML; **.NET 11 defaults to 3.2** via `Microsoft.OpenApi` 3.3.1 — a breaking change for custom transformers — and adds `itemSchema` for SSE streams, `[Obsolete]` → `deprecated`, HTTP `QUERY` operations, binary file-response schemas, multiple `Produces` per status code, and `anyOf` for C# 15 unions. Build-time document generation can now pick its environment with the `OpenApiGenerationEnvironment` MSBuild property.

Three architect-level points: **generate the document at build time and commit it**, so breaking changes appear in code review as a diff; **run a compatibility check in CI** against the previous version; and **do not ship Swagger UI to production** by default — serve the document, and use a viewer (Scalar, Redoc, or Swagger UI) in non-production or behind auth. This is where Module 30/31's "documenting decisions so they survive" meets the daily artifact.

---

# Part G — The cross-cutting middleware catalogue

You are not expected to have memorised every option. You are expected to know what each component does, where it sits, and what it costs.

## Concept 63 — Authentication: schemes and handlers

Authentication answers "who is this?" and produces `HttpContext.User`, a `ClaimsPrincipal`. The model:

- **Schemes** are named configurations, each bound to a **handler**: `"Bearer"` → `JwtBearerHandler`, `"Cookies"` → `CookieAuthenticationHandler`, `"OpenIdConnect"` → the interactive flow.
- `AddAuthentication(defaultScheme)` sets what `UseAuthentication` uses when no scheme is specified. Endpoints can override with `[Authorize(AuthenticationSchemes = "Bearer")]`.
- The handler's job is `AuthenticateAsync` (validate the credential, build the principal), plus `ChallengeAsync` (401 or redirect to login) and `ForbidAsync` (403 or access-denied page). **Knowing that challenge ≠ forbid** — unauthenticated versus authenticated-but-not-allowed — is a frequently-probed detail.
- `UseAuthentication` runs the default scheme and sets `User`. **Without it, `User` is an empty principal and every `[Authorize]` fails** — the second most common "why is everything 401" cause after a misconfigured audience.

For JWT bearer specifically, the things worth naming: signing key resolution via the OIDC discovery document (cached, refreshed), validation of issuer, audience, lifetime and signature, clock skew (default 5 minutes — reduce it deliberately), and the fact that **token validation is CPU work on every request** unless you cache. Module 29 covers the protocol side (OAuth2/OIDC, Entra ID) properly; here the point is where the work happens in the pipeline and what it costs.

---

## Concept 64 — Authorization: policies, requirements, and metadata

Authorization answers "may they do this?" and is evaluated by `UseAuthorization` against **endpoint metadata**:

```csharp
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("orders:write", p => p.RequireAuthenticatedUser()
                                      .RequireClaim("scope", "orders.write")
                                      .Requirements.Add(new SameTenantRequirement()));
    o.FallbackPolicy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build();
});
```

Concepts to have straight:

- **Policy = a set of requirements**; each requirement is satisfied by one or more `AuthorizationHandler<TRequirement>`. **Handlers are OR-ed per requirement; requirements are AND-ed.** All handlers run unless one calls `context.Fail()` (which is final).
- **Resource-based authorization** (`IAuthorizationService.AuthorizeAsync(user, resource, policy)`) is for decisions that need the entity — "may this user edit *this* document." It happens in the handler, not in middleware, because the entity isn't loaded yet at middleware time.
- **`FallbackPolicy` is the secure default**: it applies to every endpoint with no authorization metadata, turning "forgot to add `[Authorize]`" from a vulnerability into a 401. Pair it with explicit `.AllowAnonymous()` on the handful of public endpoints. This is a genuinely senior configuration choice and cheap to justify.
- **`DefaultPolicy`** applies when `[Authorize]` has no policy named; **`RequireAuthorization()`** on a group applies it to a family (Concept 36).
- .NET 11 makes `IAuthorizationRequirementData` attributes work consistently across minimal APIs, MVC, SignalR hubs and Blazor's `AuthorizeView`, via a shared `AuthorizationPolicy.CombineAsync` overload — previously they were enforced only on endpoints.

---

## Concept 65 — CORS, HTTPS/HSTS, host filtering, antiforgery

- **CORS** is a browser mechanism, enforced by the browser, advertised by your headers. `UseCors` must run **before** authentication (preflights are unauthenticated `OPTIONS` requests) and after routing if you use per-endpoint policies. `AllowAnyOrigin` with `AllowCredentials` is invalid and the framework will tell you so. The senior note: **CORS is not a security control for your API** — it protects *browser users* from other sites using their credentials; a non-browser client ignores it entirely.
- **`UseHttpsRedirection`** redirects HTTP to HTTPS (needs correct forwarded headers behind a proxy, Concept 20). **`UseHsts`** tells browsers never to try HTTP again — powerful and hard to undo, so don't enable it in Development and be deliberate about `includeSubDomains` and `preload`.
- **Host filtering** (`AddHostFiltering`) rejects requests with unexpected `Host` headers — mitigation for host-header injection when you are not behind a proxy that already pins it.
- **Antiforgery** protects cookie-authenticated, browser-submitted state changes. `UseAntiforgery` must run after auth and before endpoints. In .NET 11, **CSRF protection is applied by auto-injected middleware**, so an explicit `app.UseAntiforgery()` is no longer needed in Blazor Web Apps (and the template drops it). If your API is bearer-token authenticated with no cookies, antiforgery is not your threat model — say that rather than adding it reflexively.

---

## Concept 66 — Caching: output cache, response cache, and where HybridCache fits

Three different things that people conflate:

| | Output caching | Response caching | HybridCache |
|---|---|---|---|
| Caches | The generated response, **server-side** | Nothing — it emits/honours HTTP cache headers | Arbitrary data (objects) |
| Controlled by | Your policy | `Cache-Control` and friends | Your code |
| Works for authenticated requests | Yes, if you vary correctly | No (correctly refuses) | Yes |
| Invalidation | **Tag-based**, `EvictByTagAsync` | Expiry only | Key-based |
| Storage | In-memory or `IOutputCacheStore` (e.g. Redis) | Client/proxy | L1 memory + L2 distributed |

```csharp
builder.Services.AddOutputCache(o =>
    o.AddPolicy("catalog", b => b.Expire(TimeSpan.FromMinutes(5)).Tag("catalog").SetVaryByQuery("page")));

app.MapGet("/catalog", GetCatalog).CacheOutput("catalog");
// later, on a write:
await outputCache.EvictByTagAsync("catalog", ct);
```

.NET 11 adds **`IOutputCachePolicyProvider`**, so base policies and named-policy lookup can be resolved dynamically — the hook for per-tenant or configuration-driven caching rules.

Module 10 established the framing this all inherits: **a cache is an asynchronously-updated replica with weak consistency**, and the three questions are staleness tolerance, invalidation strategy, and what happens on a miss storm. Output caching's tag-based eviction is the strongest of the three answers because it gives you an explicit invalidation edge; response caching gives you almost none; **HybridCache** (Module 10's current default recommendation) handles the stampede problem with request coalescing and gives you the L1/L2 combination without hand-rolling it.

---

## Concept 67 — Rate limiting as in-process overload protection

`Microsoft.AspNetCore.RateLimiting` (since .NET 7) provides four algorithms — **fixed window, sliding window, token bucket, concurrency** — partitioned by whatever key you choose:

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddPolicy("per-tenant", ctx => RateLimitPartition.GetTokenBucketLimiter(
        ctx.User.FindFirstValue("tid") ?? "anon",
        _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100, TokensPerPeriod = 50,
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            QueueLimit = 0, QueueProcessingOrder = QueueProcessingOrder.OldestFirst
        }));
    o.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.Headers.RetryAfter = "1";
        await ctx.HttpContext.Response.WriteAsJsonAsync(new { error = "rate_limited" }, ct);
    };
});

app.UseRateLimiter();
orders.RequireRateLimiting("per-tenant");
```

The judgement points, which is what the interview is testing:

- **It is per instance.** With ten replicas, a "100 rps" limit is really 1,000 rps unless your limiter state is shared (Redis-backed, or enforced at the gateway). Say this before anyone asks.
- **`QueueLimit = 0` is usually right for interactive traffic.** A queue converts rejection into latency, and Module 6 established that a queue you cannot drain is a latency amplifier. Queue only where the caller genuinely prefers waiting.
- **Concurrency limiting is the bulkhead** (Module 13) in HTTP form and is often more useful than request-rate limiting, because it bounds *resource usage* rather than arrival rate.
- **Always return `Retry-After`.** Clients that don't know when to retry retry immediately, and you get a retry storm (Module 13 again).
- Rate limiting is **defence against accidental overload and noisy neighbours**, not against a determined attacker — that belongs at the edge (WAF/CDN/API gateway), before your compute is involved.

---

## Concept 68 — Compression and static assets

**Response compression** (`UseResponseCompression`, Brotli + Gzip) matters when you serve large text responses and TLS-terminating infrastructure isn't already doing it. Know three things: it is a CPU-for-bandwidth trade; **do not compress already-compressed types** (images, video, zip); and **do not enable it for HTTPS responses that mix secrets with attacker-controlled input** without thought — that is the CRIME/BREACH class of attack, which is why the middleware requires `EnableForHttps = true` as a conscious opt-in.

**Static assets**: `MapStaticAssets` (.NET 9+) is the modern replacement for `UseStaticFiles` for your *app's own* assets. At build/publish time it compresses each asset, computes a content hash, and emits a manifest; at runtime it serves them with fingerprinted URLs, correct ETags, and `Cache-Control: immutable`. `UseStaticFiles` remains the answer for files not known at build time (uploads, generated content) — and for those, remember it runs **before authorization**, so anything sensitive must be served through an authorized endpoint instead.

The architect's version: for a real production app, static assets usually belong on a **CDN**, and the middleware discussion is about the fallback path and about local development.

---

## Concept 69 — Reading and writing bodies

| Need | Use |
|---|---|
| Deserialize JSON | Let the framework bind it; or `ReadFromJsonAsync<T>()` |
| Large upload, streamed | `Request.Body` or `Request.BodyReader` (a `PipeReader`) |
| Read the body twice | `context.Request.EnableBuffering()` **before** anything reads it |
| Large file upload | Multipart streaming with `MultipartReader`, never `IFormFile` into memory |
| Large download | `TypedResults.File`/`Stream`, or `SendFileAsync` via the response body feature |
| Server push | SSE (`TypedResults.ServerSentEvents`) or SignalR/WebSockets |

The rules that prevent incidents:

- **The request body is a forward-only stream by default.** `EnableBuffering()` buffers to memory then to disk past a threshold; it is a debugging and signature-verification tool, not something to enable globally.
- **`IFormFile` buffers the whole file.** For anything large, stream with `MultipartReader` and write directly to your destination, and set a per-endpoint body size limit.
- **Never call synchronous `Read`/`Write` on the body.** `AllowSynchronousIO` is `false` by default in Kestrel precisely because sync-over-async on the request path is Module 15's ThreadPool starvation in its purest form. A library that demands it (older serializers, some PDF generators) should be wrapped: copy to a `MemoryStream`, hand that to the library.
- **Streaming responses defeat buffering middleware** (Concept 28) and keep the request scope alive (Concept 41). Both are usually fine; both must be known.

---

## Concept 70 — Built-in diagnostics: metrics, activities, logging, health

Modern ASP.NET Core emits a great deal for free, and a senior candidate should reach for it before writing custom instrumentation.

**Metrics** (`System.Diagnostics.Metrics`, exportable via OpenTelemetry):

| Meter | Key instruments |
|---|---|
| `Microsoft.AspNetCore.Hosting` | `http.server.request.duration` (histogram, with `http.route`, `http.request.method`, `http.response.status_code`, `error.type`), `http.server.active_requests` |
| `Microsoft.AspNetCore.Routing` | `aspnetcore.routing.match_attempts` |
| `Microsoft.AspNetCore.Server.Kestrel` | `kestrel.active_connections`, `kestrel.connection.duration`, `kestrel.rejected_connections`, `kestrel.queued_connections`, `kestrel.queued_requests`, TLS handshake metrics |
| `Microsoft.AspNetCore.RateLimiting` | lease acquisition, queued requests, rejections |
| `Microsoft.AspNetCore.Diagnostics` | `aspnetcore.diagnostics.exceptions` |
| `Microsoft.AspNetCore.Authorization` / `.Authentication` | attempts, challenges (.NET 10+) |

`http.server.request.duration` is a **histogram with the route as a dimension**, which is exactly what Module 17 said you need: mergeable percentiles, bounded cardinality, per-endpoint.

**Tracing**: the `Microsoft.AspNetCore` activity source has always produced a request activity; **in .NET 11 the framework itself populates the OpenTelemetry HTTP-server semantic attributes**, so `OpenTelemetry.Instrumentation.AspNetCore` is no longer required — subscribe to the source directly (`.AddSource("Microsoft.AspNetCore")`), and use the `Microsoft.AspNetCore.Hosting.SuppressActivityOpenTelemetryData` switch if you need the old behaviour.

**HTTP logging** (`UseHttpLogging`) logs configurable parts of requests and responses, with per-endpoint overrides via `[HttpLogging]`; .NET 11 pools its response-buffering streams. Treat it as a diagnostic you enable narrowly — logging bodies at volume is an expense and a PII hazard.

**Health checks**: `AddHealthChecks()` plus `MapHealthChecks("/health/live")` and `MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") })`. The distinction is the whole point: **liveness must not check dependencies** (a database outage that restarts every pod is a self-inflicted outage), **readiness may**. Combine with Concept 26's short-circuiting so probes don't pay for auth, and with Concept 9 so readiness fails the moment shutdown begins.

---

# Part H — The architect's view

## Concept 71 — The per-request cost model

Know roughly what a request costs in your service, because capacity planning (Module 17, Concept 62) starts here:

| Cost | Typical magnitude | Lever |
|---|---|---|
| Connection + TLS (amortized) | µs–ms | Keep-alive, HTTP/2, terminate at ingress |
| `HttpContext` + features | ~free (pooled) | — |
| DI scope + graph construction | Hundreds of ns to µs; allocations proportional to graph | Fewer/smaller services, singletons where stateless |
| Routing | ~hundreds of ns | — |
| Auth (JWT validation) | µs–ms | Cache keys; avoid per-request introspection calls |
| Model binding / JSON deserialize | µs–ms, allocation proportional to payload | Source-generated JSON, smaller payloads |
| **Application + I/O** | **ms–hundreds of ms** | This is where the time actually is |
| Serialization + response write | µs–ms | Streaming, UTF-8 end to end |
| Logging and metrics | µs, unless careless | `[LoggerMessage]`, sampling, bounded tag cardinality |

The lesson Module 17 taught and this table restates: **the framework is almost never your bottleneck; your dependencies are.** Candidates who want to "optimize ASP.NET Core" before they have looked at a trace are signalling inexperience. The framework costs you microseconds; a missing index costs you 300 ms.

---

## Concept 72 — Designing a composition root that survives ten teams

What a good `Program.cs` looks like at scale:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddTelemetry();                 // OTel: traces, metrics, logs
builder.AddPlatformDefaults();          // auth, health, problem details, rate limiting, resilience
builder.AddOrdersModule();              // one call per bounded context / feature module
builder.AddPaymentsModule();

var app = builder.Build();

app.UsePlatformPipeline();              // the ordered middleware, in exactly one place
app.MapOrdersEndpoints();
app.MapPaymentsEndpoints();
app.MapHealthEndpoints();

app.Run();
```

Principles behind it:

1. **One extension method per module**, owning that module's registrations, options binding and validation. Registration lives next to the code it registers.
2. **The middleware order lives in exactly one place**, with comments explaining the non-obvious adjacencies (Concept 25). Never let module code append middleware to the global pipeline; that is how ordering becomes emergent and untestable.
3. **Validate at startup** — options (`ValidateOnStart`), the container (`ValidateOnBuild`), and any connectivity that must exist for the service to be meaningful. Fail fast, fail in the rollout.
4. **Prefer explicit registration over assembly scanning** (Concept 51). If you keep scanning, restrict it to one assembly and measure it.
5. **Write an architecture test** that asserts your rules — no `IServiceProvider` in constructors outside the composition root, no `DbContext` outside the data layer, no module referencing another module's internals. NetArchTest or ArchUnitNET makes this a unit test rather than a convention nobody enforces.
6. **`Program.cs` is documentation.** If a new engineer cannot infer the service's shape from it in two minutes, restructure it.

---

## Concept 73 — Testing the pipeline: what `WebApplicationFactory` proves

`WebApplicationFactory<TEntryPoint>` boots your real `Program.cs` **in-process** with `TestServer` as the `IServer`, and gives you an `HttpClient` that talks directly to the pipeline — no sockets, no ports.

```csharp
public sealed class ApiFixture : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseSetting("Environment", "Testing");
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton<IPaymentGateway, FakeGateway>();
        });
    }
}
```

What it **does** prove: middleware ordering, routing, model binding, validation, auth policy evaluation, filters, serialization, and DI wiring — the integration seams where most real bugs live. This is the highest value-per-line testing available in ASP.NET Core, and a senior candidate should say so.

What it **does not** prove: Kestrel behaviour (no real sockets, no TLS, no HTTP/2 framing, no Kestrel limits or timeouts), proxy/forwarded-header behaviour, real network failures, or performance. For those you need the app running for real (Testcontainers for dependencies, a real Kestrel on a port).

Three practical notes: make `Program` accessible (a `public partial class Program { }` at the bottom of the file, or use the generated entry point); replace dependencies with `RemoveAll<T>()` + `Add...` rather than `Replace` guesswork; and **turn on `ValidateScopes` in the test host** so captive dependencies fail your test suite rather than production.

---

## Concept 74 — Multi-tenancy in the pipeline and the container

The standard shape, and the order it must happen in:

1. **Resolve the tenant as early as possible** — from the host name, a path segment, a header, or a claim. Host- or path-based resolution must run **before `UseRouting`** if the route itself depends on it; claim-based resolution must run **after `UseAuthentication`**. You often need both, in two places.
2. **Put the tenant in a scoped object**, not in an `AsyncLocal` and not on a static (Concept 45).
3. **Make tenant-dependent services scoped** and resolve their configuration from the tenant: connection string, storage container, feature flags, rate-limit partition, cache key prefix.
4. **Make the tenant part of every key**: cache keys, output-cache tags, rate-limit partitions, metric dimensions (carefully — tenant id is unbounded cardinality; use it as an exemplar or a log field, not a metric tag, unless the tenant count is small).
5. **Fail closed.** No tenant resolved → 400/404, never "use the default tenant."

The container question people get wrong: you do **not** need per-tenant containers. Per-tenant *options* (`IOptionsMonitor<T>` with a named option per tenant, or a scoped factory) plus scoped services that read the current tenant covers almost every real requirement, and it keeps one object graph. Reach for child containers only when tenants need genuinely different *implementations*, and price the complexity honestly.

---

## Concept 75 — Startup, cold start, and AOT for web applications

Bringing Module 17's Part H down to ASP.NET Core specifics.

What works with Native AOT today: **minimal APIs (partially)**, gRPC, JWT bearer auth, CORS, health checks, HTTP logging, localization, output caching, rate limiting, response caching/compression, rewrite, static files, WebSockets. What does **not**: **MVC**, Blazor Server, Session, SPA middleware, most non-JWT authentication. EF Core's AOT and precompiled-query support remains experimental — which is why **EF Core is the usual reason a service cannot go AOT**.

The checklist for an AOT-able web service:

1. `WebApplication.CreateSlimBuilder(args)`; add back only what you need.
2. `<PublishAot>true</PublishAot>`, `<InvariantGlobalization>true</InvariantGlobalization>` if you can.
3. `System.Text.Json` source generation, with every DTO on a `JsonSerializerContext` registered via `ConfigureHttpJsonOptions`.
4. RDG on (implicit under AOT), and fix every `RDG###` diagnostic.
5. Zero trim/AOT warnings with no unjustified suppressions — Module 17's equivalence guarantee.
6. Test the **native binary**, not the JIT build, in your integration suite.

And the decision framing, unchanged from Module 17: **AOT wins startup and footprint; JIT wins peak throughput** (tiered PGO can't devirtualize at runtime in an AOT binary). For a request-per-second-bound API that never scales to zero, R2R is usually the better trade; for a function, a CLI, a sidecar or anything scaled from zero, AOT is worth the constraints. Decide per service, with numbers, and write it down as an ADR (Module 31).

---

## Concept 76 — Anti-patterns that show up in architecture review

The list reviewers actually use, each with its tell:

| Anti-pattern | Tell | What to do instead |
|---|---|---|
| **Service locator** | `IServiceProvider` injected into a business class | Inject what you need; use `IServiceScopeFactory` only for scope boundaries |
| **Captive dependency** | Scoped service in a singleton's constructor | Scope validation on; resolve per operation |
| **Root-provider resolution** | `app.Services.GetRequiredService<T>()` in a loop | `CreateAsyncScope()` per unit of work |
| **`BuildServiceProvider()` in `ConfigureServices`** | Two containers, duplicated singletons | Post-`Build()` scope, or options/`IConfigureOptions` |
| **God middleware** | One `Use(...)` doing auth, logging, tenancy and caching | One concern per component; endpoint filters for endpoint-scoped concerns |
| **Ambient request state** | `static HttpContext Current`, `AsyncLocal` for the tenant | A scoped `RequestContext` record resolved at the edge |
| **Captured `HttpContext`** | Fields, closures, fire-and-forget | Copy values; queue the work |
| **Sync-over-async** | `.Result`, `.Wait()`, `GetAwaiter().GetResult()` on the request path | Async all the way (Module 15) |
| **Body-buffering middleware everywhere** | Custom logging middleware reading the response | HTTP logging middleware, narrowly scoped |
| **Swallowed exceptions → 200** | `catch { }` in a handler | Let it throw; map it in `IExceptionHandler` |
| **`ILogger.LogInformation($"…")`** | Interpolated strings in logs | `[LoggerMessage]` source-generated logging |
| **Validation in three places** | Attributes + filters + domain guards, inconsistent | One input-validation mechanism; invariants in the domain |
| **No timeouts anywhere** | `HttpClient` with default timeout, no request timeouts | Timeouts at every hop, plus a request budget |
| **Assembly scanning at startup** | 400 ms cold start nobody can explain | Explicit module registration |
| **Static files serving protected content** | `wwwroot/reports/*.pdf` | An authorized endpoint that streams the file |

Being able to name five of these with their failure modes — unprompted, in the context of reviewing someone else's code — is close to the definition of the senior signal this module is trying to build.

---

# Putting it together

## Worked example 1 — "Our pods get OOM-killed every 36 hours. Walk me through it."

**The method** (Module 17's loop, applied to a leak):

1. **Confirm the shape.** Is it a leak (monotonic growth, never recovered after gen-2) or a working-set issue (high but stable, or DATAS not releasing)? `dotnet-counters monitor System.Runtime` for gen-2 size and committed bytes. A saw-tooth that resets on collection is not a leak; a staircase is.
2. **Take two heap snapshots** an hour apart under steady load (`dotnet-gcdump collect`), and diff them. You are looking for a type whose count grows with uptime rather than with concurrency.
3. **Read the retention path**, not just the type. This is the step that answers the question. Four paths cover most ASP.NET Core leaks:

| Retention path | Diagnosis |
|---|---|
| Objects rooted by `ServiceProviderEngineScope` belonging to the **root** provider | Transient `IDisposable` resolved from the root, or a background loop resolving from `app.Services` (Concept 43) |
| A scoped service (e.g. `DbContext`) rooted by a **singleton** | Captive dependency (Concept 42) |
| Entities rooted by a `ChangeTracker` | A scope living far longer than one unit of work (Module 19's territory) |
| `MemoryCache` entries with no size limit or expiry | Unbounded cache — Module 10's mistake |

4. **Reproduce the finding as a failing check**, not just a fix: turn on `ValidateScopes` in the integration test host and watch the captive dependency become a test failure.
5. **Fix and verify** with the same measurement over the same window.

**The narration an interviewer wants**: *"Memory growth proportional to uptime rather than concurrency says retention, not pressure. I'd gcdump twice and diff. In ASP.NET Core the four usual retention paths are root-provider transients, captive scoped services, over-long scopes holding change-tracked entities, and unbounded in-memory caches. The fix is usually a lifetime change plus enabling scope validation so it can't come back."*

---

## Worked example 2 — "Review this middleware."

```csharp
public class AuditMiddleware
{
    private readonly RequestDelegate _next;
    private readonly AppDbContext _db;                    // (1)
    private static HttpContext _current;                  // (2)

    public AuditMiddleware(RequestDelegate next, AppDbContext db) { _next = next; _db = db; }

    public async Task InvokeAsync(HttpContext context)
    {
        _current = context;                               // (2)
        var body = new MemoryStream();                    // (3)
        context.Response.Body = body;

        _next(context);                                   // (4)

        context.Response.Headers["X-Audited"] = "true";    // (5)
        _db.Audits.Add(new Audit { Path = context.Request.Path });
        _db.SaveChanges();                                // (6)

        _ = Task.Run(async () => await Notify(context));   // (7)
    }
}
```

Seven defects, in the order a reviewer would call them out:

1. **Captive dependency.** Convention-based middleware is a singleton; `AppDbContext` is scoped. In Development this throws at startup; in Production it silently shares one context across all requests — stale reads, thread-safety violations, disposal errors. Move it to an `InvokeAsync` parameter (Concept 24).
2. **Static `HttpContext`.** The context is pooled and shared; a static reference is cross-request data leakage (Concept 30).
3. **Body swap without restoring the feature.** `Response.Body` alone bypasses `IHttpResponseBodyFeature`, so `SendFileAsync`/`PipeWriter` paths escape the wrapper; the original is never restored; the buffer is unbounded and lands on the LOH for any sizeable response (Concept 28). Also nothing ever copies the buffer back — **the client receives an empty body**.
4. **`_next(context)` is not awaited.** The request completes while the continuation is still running against a recycled context.
5. **Header set after `next`.** By then the response has almost certainly started; this throws (Concept 27). Use `OnStarting`.
6. **Synchronous `SaveChanges` on the request path.** Sync-over-async against a network resource — Module 15's ThreadPool starvation, and an audit write on the critical path of every request besides.
7. **Fire-and-forget using `HttpContext`.** `Task.Run` escapes the request; the context will be reset underneath it; exceptions are unobserved; nothing bounds the concurrency.

**The rewrite**: make it `IMiddleware` (or keep it convention-based and take dependencies in `InvokeAsync`), drop the body capture entirely in favour of HTTP logging middleware scoped to the endpoints that need it, set the header in `OnStarting`, copy the few values you need into a record and hand them to a `Channel<AuditRecord>` consumed by a `BackgroundService` that writes in batches with its own scope. That last move also turns N writes per request into one batch per interval.

---

## Worked example 3 — "Design the request pipeline for a multi-tenant B2B API."

Requirements: tenant per subdomain, OAuth2 bearer tokens, per-tenant rate limits, consistent error contract, full tracing, zero-downtime deploys, and one endpoint that accepts 500 MB uploads.

```csharp
var app = builder.Build();

app.UseForwardedHeaders();          // 1. behind an ingress; must be first
app.UseExceptionHandler();          // 2. wraps everything below; ProblemDetails
app.UseHsts();                      // 3. (skipped in Development)
app.UseHttpsRedirection();          // 4.

app.UseTenantResolution();          // 5. host-based → scoped TenantContext; BEFORE routing
app.UseRouting();                   // 6. endpoint + metadata now available

app.UseCors();                      // 7. before auth: preflights are unauthenticated
app.UseAuthentication();            // 8. sets HttpContext.User
app.UseAuthorization();             // 9. evaluates endpoint metadata; FallbackPolicy on

app.UseRateLimiter();               // 10. partitioned by tenant claim → needs User
app.UseRequestTimeouts();           // 11. request budget
app.UseOutputCache();               // 12. after auth so it can vary by tenant

app.MapOrdersEndpoints();
app.MapUploadEndpoints();
app.MapHealthChecks("/health/live").ShortCircuit();      // no auth, no rate limit
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });

app.Run();
```

Decisions to narrate:

- **Tenant resolution twice.** Host-based resolution runs before routing (the route doesn't depend on it, but rate limiting, logging enrichment and options do); the token's `tid` claim is validated against it after authentication, and a mismatch is a 403. Fail closed.
- **`FallbackPolicy` requires an authenticated user**, so a new endpoint is protected by default; the two health endpoints are explicitly anonymous and short-circuited.
- **Uploads** get `[RequestSizeLimit(500 * 1024 * 1024)]` (and a matching `RequestTimeout`), while the global `MaxRequestBodySize` stays at the default — per-endpoint, not global (Concept 18). The handler streams with `MultipartReader` straight to blob storage; no `IFormFile`.
- **Error contract**: `AddProblemDetails()` plus two `IExceptionHandler`s (domain-validation → 400, everything else → 500 with a `traceId`), identical for minimal API and any controller endpoints.
- **Deploys**: `preStop` sleep of 5 s, readiness fails on `ApplicationStopping`, `ShutdownTimeout` above the longest legitimate request, and every handler passes `RequestAborted` down (Concept 9, 21).
- **Observability**: subscribe to the `Microsoft.AspNetCore` activity source and the built-in meters; enrich logs with tenant and correlation id from the scoped `TenantContext`, **not** with an `IHttpContextAccessor` read deep in the stack.

---

## Worked example 4 — "A nightly job intermittently throws `ObjectDisposedException` on `DbContext`."

**The trace**: the job is a `BackgroundService` registered as a singleton. Someone "avoided the scope boilerplate" by injecting `IServiceProvider` and calling `GetRequiredService<AppDbContext>()` once in the constructor.

What actually happens: that resolution comes from the **root** provider. The `DbContext` becomes a de facto singleton — and because `AddDbContext` registers it scoped, resolving it from the root is only legal because `ValidateScopes` is off in Production. Then one of two things breaks it: something else disposes it, or two iterations overlap and the context is used concurrently. Intermittency comes from timing, which is why it never reproduces locally.

**The fix and the guardrails**:

```csharp
await using var scope = _scopes.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
```

one scope per iteration; `CreateAsyncScope` because `DbContext` is `IAsyncDisposable`; `ValidateScopes = true` in the test host so this class of bug fails CI; and an architecture test forbidding `IServiceProvider` in constructors outside the composition root.

**The generalisation to say out loud**: *"Every 'randomly disposed' or 'second operation started on this context' bug is the same bug — a scoped object escaping its scope. The three shapes are root resolution, constructor capture in a singleton, and fire-and-forget work that outlives the request."*

---

## Worked example 5 — "After moving to Kubernetes we get redirect loops, and rate limiting blocks everyone at once."

Both symptoms have one cause: the app cannot see the real client or the real scheme.

- The ingress terminates TLS and forwards over HTTP. Without forwarded headers applied, `Request.Scheme` is `http`, so `UseHttpsRedirection` issues a 301 to `https://`, the ingress terminates again and forwards `http` again — **a loop**.
- Every request appears to come from the ingress's pod IP, so an IP-partitioned rate limiter sees one client at the full aggregate rate and throttles everyone together.

**The fix, with its caveats:**

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownNetworks.Clear();                 // deliberate: the ingress is the only path in
    o.KnownProxies.Clear();                  // and it overwrites inbound X-Forwarded-*
    o.ForwardLimit = 1;
});

app.UseForwardedHeaders();                   // FIRST
```

Clearing the known lists is safe **only** if the ingress is the sole ingress path and strips client-supplied `X-Forwarded-*` headers. If it does not, a client can set its own `X-Forwarded-For`, and you have just made your rate limiter and your audit log trivially spoofable. The rigorous alternative is to register the ingress's pod CIDR in `KnownNetworks`.

**The extra move that shows seniority**: partition the rate limiter by **tenant or API key**, not by IP, whenever the caller is authenticated. IP is a weak identity behind NAT, proxies and mobile carriers, and it is attacker-influenceable. IP-partitioned limits belong at the edge, on unauthenticated traffic.

---

## Worked example 6 — "Should we move our API from controllers to minimal APIs?"

The answer that scores is a decision procedure, not a preference:

1. **What is the binding constraint?** If it is cold start or image size (a scale-to-zero function, a sidecar), minimal APIs plus AOT is a real lever — measure it, because the difference is tens of milliseconds and tens of megabytes, not a rewrite's worth on its own. If it is request throughput, the framework overhead is microseconds and your bottleneck is elsewhere (Concept 71).
2. **Do we need what MVC provides?** Views, complex form binding, content negotiation, `ModelState`, application parts. If yes, the question is closed.
3. **What does migration actually cost?** Every MVC filter must be rewritten as an endpoint filter or middleware; `ModelState` validation becomes `AddValidation()`; conventions become explicit groups; tests that use `ControllerBase` helpers change shape.
4. **What is the incremental path?** Both models coexist in one app, so a strangler-fig migration (Module 32) is available: new endpoints as minimal APIs, existing controllers untouched, shared auth/error/telemetry.
5. **Write the ADR** (Module 31) with the measurement, the constraint, and the trigger for revisiting.

The honest conclusion for most teams: **for new services, minimal APIs; for a healthy existing MVC API, leave it alone** unless AOT or startup is a stated requirement. Saying "leave it alone" with reasons is a stronger signal than proposing a migration.

---

## Common questions and what a strong answer contains

**"What happens between the socket accept and my controller action?"** Transport accepts the connection → connection middleware (TLS, protocol selection) → the protocol layer parses the request → the hosting layer rents an `HttpContext` from a pool, creates a DI scope, and starts the activity and metrics → the composed `RequestDelegate` runs → `UseRouting` selects an endpoint → cross-cutting middleware reads its metadata → `UseEndpoints` executes it → the response is written through pooled buffers → the scope is disposed and the context is reset (Concepts 1, 12, 41).

**"Explain the DI lifetimes."** Singleton is cached in the root scope, scoped in the current scope, transient created per resolution — and transients that are disposable are still *tracked* by the resolving scope. In ASP.NET Core the hosting layer creates one scope per request. The rule that matters is that a longer-lived service must not capture a shorter-lived one (Concepts 40, 42, 43).

**"What is a captive dependency and how do you prevent it?"** A singleton holding a scoped service; it produces stale data, disposal exceptions, concurrency errors and a leak. Prevented by `ValidateScopes` (on by default in Development, and worth enabling in tests), `ValidateOnBuild` in every environment, and `IServiceScopeFactory` for background work (Concept 42).

**"Why can't I inject a `DbContext` into middleware's constructor?"** Convention-based middleware is instantiated once at pipeline build time, so its constructor dependencies are effectively singletons. Take scoped dependencies as `InvokeAsync` parameters, or implement `IMiddleware` and let the factory resolve it per request (Concept 24).

**"Why must `UseAuthentication` come before `UseAuthorization`, and both after `UseRouting`?"** Authentication populates `HttpContext.User`; authorization evaluates policies against that user *and* against the endpoint's metadata, which only exists after routing has matched. Ordering encodes data dependencies (Concepts 25, 32, 64).

**"What does `UseRouting` actually do?"** It matches and sets the endpoint; it executes nothing. The gap between it and `UseEndpoints` is where metadata-driven middleware does its work (Concept 32).

**"How does route matching scale with the number of routes?"** It doesn't: templates compile to a DFA over path segments at startup, so matching cost tracks URL length, not route count. Many routes cost startup time and memory, not per-request latency (Concept 34).

**"Why shouldn't I store `HttpContext`?"** It's pooled and reset between requests, so a stored reference eventually observes a different request's data. Copy what you need into an immutable object (Concept 30).

**"What's wrong with `IHttpContextAccessor` everywhere?"** It's an `AsyncLocal` ambient dependency with a real cost, it's null outside a request, and it hides a class's true inputs. Resolve request facts once at the edge into a scoped record (Concept 45).

**"Singleton or scoped for this service?"** Stateless → singleton, and save the per-request allocation and tracking. Holds per-request state or a unit of work → scoped. Disposable and transient → usually a mistake (Concepts 40, 43).

**"How does `IHttpClientFactory` help?"** It pools and rotates `HttpMessageHandler`s, which fixes both socket exhaustion from per-request clients and DNS staleness from a forever-static client. Handlers live in their own DI scope, not the request scope. The modern alternative for the DNS half alone is `SocketsHttpHandler.PooledConnectionLifetime` (Concept 50).

**"When would you use `IMiddleware` over conventional middleware?"** When it has meaningful scoped dependencies or you want it unit-testable — at the cost of one allocation per request (Concept 24).

**"How do you handle errors in a production API?"** `UseExceptionHandler` outermost, `AddProblemDetails`, one or more `IExceptionHandler`s mapping exception types to status codes, a `traceId` in every body, no internal detail leaked, logged exactly once, and the developer exception page confined to Development (Concept 29).

**"How do you protect a service from a slow or hostile client?"** Kestrel's limits and timeouts (body/header sizes, connection counts, `RequestHeadersTimeout`, minimum data rates), request timeouts middleware for execution budgets, rate limiting for arrival rate, concurrency limiters as bulkheads — and an edge WAF/CDN for anything adversarial (Concepts 18, 19, 67).

**"Minimal APIs or controllers?"** Decide on AOT/startup requirements, on whether you need MVC's binding, views and content negotiation, and on team familiarity — not on benchmark deltas that are microseconds against millisecond dependencies. New JSON APIs default to minimal; existing healthy MVC apps stay (Concept 60).

**"How do you validate input?"** `AddValidation()` from .NET 10 (source-generated, AOT-friendly, covers nested types and collections), async validators from .NET 11 for rules that hit I/O, `ValidationProblemDetails` shaped through `IProblemDetailsService`, and the reminder that domain invariants belong with the transaction, not at the edge (Concept 57).

**"What does graceful shutdown look like in Kubernetes?"** SIGTERM → stop accepting → drain within `ShutdownTimeout` → hosted services stopped in reverse order → container disposed; plus a `preStop` delay and a readiness probe that fails immediately, because deregistration from the load balancer is eventually consistent (Concept 9).

**"We're behind an ingress and client IPs are wrong."** Forwarded headers, applied first, with `KnownProxies`/`KnownNetworks` configured for the actual proxy — and the caveat that these headers are client-controllable unless the proxy overwrites them (Concept 20).

**"What's new in ASP.NET Core recently?"** .NET 10: minimal API validation, `Microsoft.Extensions.Validation`, SSE via `TypedResults.ServerSentEvents`, OpenAPI 3.1 + YAML. .NET 11: `[ShortCircuit]`, filters that observe binding failures, async and localized validation, OpenAPI 3.2 by default, C# 15 unions through `System.Text.Json`, `IOutputCachePolicyProvider`, native OpenTelemetry attributes on `Microsoft.AspNetCore`, and a non-throwing Kestrel HTTP/1.1 parse path worth 20–40% under malformed load (Orientation).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Describes middleware as "a list that runs in order" | Describes composition of `RequestDelegate`s and derives inbound/outbound behaviour from it |
| Memorises the middleware order | Derives each adjacency from the data dependency behind it |
| "Scoped means per request" | "Scoped means per scope; the hosting layer makes one per request, and I make my own in workers" |
| Injects `DbContext` into middleware's constructor | Takes it as an `InvokeAsync` parameter, or uses `IMiddleware` |
| Leaves scope validation at defaults | `ValidateOnBuild` everywhere, `ValidateScopes` in tests; configuration failures become deploy failures |
| Registers `IHttpContextAccessor` and uses it deep in the stack | Resolves request facts once at the edge into a scoped record |
| Stores `HttpContext` or its headers | Copies values; knows the context is pooled |
| Writes custom body-buffering logging middleware | Uses HTTP logging middleware narrowly, and knows what buffering costs |
| Sets response headers after `await next` | Uses `OnStarting`, and knows the response-started rule |
| Catches and returns 200 | Throws, and maps exceptions in `IExceptionHandler` with ProblemDetails and a trace id |
| `new HttpClient()` per call, or one static forever | `IHttpClientFactory` or `PooledConnectionLifetime`, and knows why each failure mode exists |
| Raises the global `MaxRequestBodySize` for one upload endpoint | Per-endpoint limits; global default stays small |
| Rate limits by IP behind a proxy | Limits by tenant/API key; knows forwarded headers are spoofable |
| Believes rate limiting is global | Knows it's per instance and says so before being asked |
| Health check that pings the database for liveness | Liveness is process health; readiness checks dependencies |
| "Minimal APIs are faster, so we should migrate" | Names AOT, startup and MVC feature requirements as the real criteria, and measures |
| Assembly-scans everything at startup | Explicit module registration; knows scanning is a cold-start and trimming hazard |
| Treats `Program.cs` as boilerplate | Treats it as the composition root and the service's documentation |
| Tests only with unit tests | Uses `WebApplicationFactory` for the integration seams, and knows what `TestServer` cannot prove |
| Says "the framework is slow" | Produces a trace showing where the time is, and it is almost never the framework |

---

## Practice exercises

**Exercise 1 — Read the pipeline (45 min).** Take a service you know. Print the actual middleware order at startup (wrap `IApplicationBuilder.Use` in a logging decorator, or set a breakpoint on `Build`). For each component, write one sentence saying what it depends on from the component before it. Find at least one that is in the wrong place or is unnecessary.

**Exercise 2 — Build a captive dependency, then catch it (45 min).** Create a minimal API with a scoped `Counter` service injected into a singleton. Observe the behaviour with `ValidateScopes` off, then turn it on and watch the startup failure. Repeat with a transient `IDisposable` resolved from `app.Services` in a loop; use `dotnet-gcdump` to show the retention path from the root provider.

**Exercise 3 — Two middleware implementations (1 hr).** Write the same correlation-id middleware twice: convention-based and `IMiddleware`. Benchmark both under load with `[MemoryDiagnoser]`-style measurement of per-request allocation. Write down when the allocation is worth paying.

**Exercise 4 — Response wrapping done right (1.5 hrs).** Implement a middleware that appends a `Server-Timing` header using `OnStarting`, and a second that captures the response body with a **bounded** buffer and correct `IHttpResponseBodyFeature` replacement and restoration. Prove the second one breaks a streaming endpoint, then make it skip endpoints marked with metadata you define.

**Exercise 5 — Routing precedence (45 min).** Map `/orders/active`, `/orders/{id:int}`, `/orders/{id}`, and `/{**rest}` to different handlers. Predict which wins for six URLs, then verify. Add two endpoints that genuinely collide and observe `AmbiguousMatchException`. Then write an integration test that asserts your route table hasn't changed.

**Exercise 6 — Kestrel limits (1 hr).** Set `MaxConcurrentConnections`, `MaxRequestBodySize`, `RequestHeadersTimeout` and `MinRequestBodyDataRate` to small values. Write a client that (a) opens many idle connections, (b) sends headers one byte per second, (c) uploads slowly. Observe which limit fires and what the client sees. Then raise the body limit for one endpoint only.

**Exercise 7 — Graceful shutdown under load (1.5 hrs).** Run the service under constant-arrival-rate load (Module 17's open-model rule). Send SIGTERM. Count failed requests. Add a readiness probe that fails on `ApplicationStopping`, a `preStop` delay, and `RequestAborted` propagation, then repeat and compare. Target: zero 5xx during a rolling deploy.

**Exercise 8 — Minimal APIs at scale (2 hrs).** Refactor a 20-endpoint `Program.cs` into endpoint modules with groups, static handlers, and explicit registration. Add `EnableRequestDelegateGenerator` and fix every `RDG###` diagnostic. Measure cold start before and after, and measure again with `CreateSlimBuilder`.

**Exercise 9 — Error contract (1 hr).** Implement `AddProblemDetails` plus two `IExceptionHandler`s. Assert with integration tests that: a domain validation failure returns 400 with field-level details; an unexpected exception returns 500 with a `traceId` and no stack trace; and the `traceId` matches the `Activity` recorded in your telemetry.

**Exercise 10 — Integration tests that earn their keep (2 hrs).** Build a `WebApplicationFactory` fixture with `ValidateScopes` on, a fake external dependency, and a real database via Testcontainers. Write tests that would fail if: middleware order changed, the fallback authorization policy were removed, a route template changed, or a DTO's JSON shape changed. Time the suite; keep it under a minute.

**Exercise 11 — Multi-tenancy (2 hrs).** Implement host-based tenant resolution before `UseRouting`, claim validation after authentication, a scoped `TenantContext`, per-tenant options via `IOptionsMonitor` named options, and a tenant-partitioned rate limiter. Write a test proving that a token for tenant A cannot read tenant B's data even when the host says B.

**Exercise 12 — AOT a service (2–3 hrs).** Take a small JSON API. Move it to `CreateSlimBuilder`, add JSON source generation, publish with `PublishAot`, and reach zero trim/AOT warnings. Measure container start to first response, RSS under load, throughput at a fixed p99, and image size against the JIT and R2R builds. Write a one-page ADR recommending one of the three for this service.

---

## Free resources

### Primary sources — the framework itself

| Resource | What it covers | Why read it |
|---|---|---|
| [dotnet/aspnetcore](https://github.com/dotnet/aspnetcore) | The whole framework, source and issues | The definitive answer to "what does it actually do?"; issue threads often contain the design rationale |
| [src/Http](https://github.com/dotnet/aspnetcore/tree/main/src/Http) | `HttpContext`, features, `RequestDelegate`, minimal API machinery (`RequestDelegateFactory`) | **Read `DefaultHttpContext` and `RequestDelegateFactory` once**; Concepts 22, 30, 53 become obvious |
| [src/Http/Routing](https://github.com/dotnet/aspnetcore/tree/main/src/Http/Routing) | `DfaMatcher`, `DfaMatcherBuilder`, endpoint data sources, `MatcherPolicy` | Concepts 33–35 from the source |
| [src/Servers/Kestrel](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel) | Transports, connection middleware, HTTP/1.1, /2, /3 implementations | Part B; the HTTP/1.1 parser is a masterclass in span-based parsing |
| [src/Middleware](https://github.com/dotnet/aspnetcore/tree/main/src/Middleware) | Every built-in middleware, including output caching and rate limiting | The best available examples of correct middleware |
| [Microsoft.Extensions.DependencyInjection (runtime)](https://github.com/dotnet/runtime/tree/main/src/libraries/Microsoft.Extensions.DependencyInjection) | `ServiceProvider`, call sites, engines, scope implementation | Concepts 39–43 from the source |
| [ASP.NET Core Guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AspNetCoreGuidance.md) — David Fowler | Do/don't list for `HttpContext`, bodies, background work, sync-over-async | **The single densest free document for this module.** Read it end to end |
| [Async Guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md) — David Fowler | The async rules that Module 15 covered, in a server context | Companion to the above |
| [System.IO.Pipelines: high performance IO in .NET](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/) | The abstraction Kestrel is built on | Concept 13 |
| [aspnet/Benchmarks](https://github.com/aspnet/Benchmarks) | The team's own benchmark apps and infrastructure | How the framework measures itself |
| [TechEmpower benchmarks](https://www.techempower.com/benchmarks/) | Cross-framework results including ASP.NET Core | Useful for calibration, dangerous as a design input — read the caveats |

### Microsoft Learn — hosting, configuration, DI

| Resource | What it covers |
|---|---|
| [.NET Generic Host](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host) · [`WebApplication` and `WebApplicationBuilder`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/webapplication) | Concepts 2, 5 |
| [Middleware in minimal API apps](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/middleware) | **The auto-inserted middleware rules** (Concept 10) |
| [Configuration](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/) · [Options pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options) · [Options in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/options) | Concepts 3–4, including `ValidateOnStart` and the options source generator |
| [Dependency injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) · [DI in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) | Part E's official version |
| [**Dependency injection guidelines**](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines) | Captive dependencies, disposal, "services not created by the container" — Concepts 42–43 |
| [Third-party container integration](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/extensibility-third-party-container) | Concept 52 |
| [Background tasks with hosted services](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services) | Concepts 8–9, 46 |

### Microsoft Learn — servers and hosting environment

| Resource | What it covers |
|---|---|
| [Web server implementations](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/) | Kestrel vs HTTP.sys vs IIS (Concept 20) |
| [Kestrel overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel) · [Options](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/options) · [Endpoints](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/endpoints) | **Concepts 17–19: every limit and timeout with its default** |
| [`KestrelServerLimits` API reference](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.server.kestrel.core.kestrelserverlimits) | The authoritative defaults table |
| [HTTP/3 in Kestrel](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/http3) | Concept 16, including MsQuic requirements and `alt-svc` |
| [HTTP.sys](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/httpsys) · [IIS in-process hosting](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/in-process-hosting) | When they win |
| [Configure ASP.NET Core to work with proxies and load balancers](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer) | **Concept 20's forwarded-headers traps** |
| [YARP documentation](https://microsoft.github.io/reverse-proxy/) | A .NET reverse proxy; also the best worked example of connection/endpoint extensibility |
| [Health checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) | Liveness vs readiness (Concept 70) |

### Microsoft Learn — pipeline, routing, endpoints

| Resource | What it covers |
|---|---|
| [Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/) | **The canonical order diagram** (Concept 25) |
| [Write custom middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write) · [Factory-based middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/extensibility) | Concept 24, both activation models |
| [Routing](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing) | Templates, constraints, precedence, short-circuiting, link generation (Part D) |
| [Handle errors](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling) · [Handle errors in web APIs](https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors) | `IExceptionHandler`, ProblemDetails (Concept 29) |
| [Request timeouts middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/timeouts) | Concept 19's execution budget |

### Microsoft Learn — endpoint models

| Resource | What it covers |
|---|---|
| [Minimal APIs quick reference](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis) | The whole surface in one page |
| [Parameter binding](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/parameter-binding) · [Responses](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/responses) · [Filters](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters) · [Route handlers](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/route-handlers) | Concepts 54–56 |
| [Choose between controller-based and minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/apis) | The official version of Concept 60 |
| [Validation in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/validation) | `AddValidation()`, `[ValidatableType]`, async validation, localization (Concept 57) |
| [Filters in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/filters) · [Model binding](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/model-binding) | MVC's pipeline (Concept 58) |
| [Controller-based web APIs](https://learn.microsoft.com/en-us/aspnet/core/web-api/) | `[ApiController]` conventions (Concept 59) |
| [OpenAPI support](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/aspnetcore-openapi) | Document generation, transformers, build-time generation (Concept 62) |

### Microsoft Learn — cross-cutting middleware

| Resource | What it covers |
|---|---|
| [Authentication overview](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/) · [Authorization introduction](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/introduction) · [Policy-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies) | Concepts 63–64 |
| [CORS](https://learn.microsoft.com/en-us/aspnet/core/security/cors) · [Antiforgery](https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery) · [Enforce HTTPS](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl) | Concept 65 |
| [Output caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output) · [Response caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response) · [HybridCache](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) | Concept 66, and the continuation of Module 10 |
| [Rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) | Concept 67 |
| [Response compression](https://learn.microsoft.com/en-us/aspnet/core/performance/response-compression) · [Static files](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/static-files) | Concept 68, including the HTTPS compression caveat |
| [WebSockets](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/websockets) · [`IHttpClientFactory`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests) | Concepts 50, 69 |
| [Memory management and patterns in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/memory) | The server-side version of Module 14, with a demo app per pathology |

### Microsoft Learn — diagnostics, testing, AOT

| Resource | What it covers |
|---|---|
| [ASP.NET Core metrics](https://learn.microsoft.com/en-us/aspnet/core/log-mon/metrics/metrics) · [Built-in metrics reference](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-aspnetcore) | Concept 70's tables, authoritative |
| [HTTP logging](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-logging/) | Narrow, per-endpoint logging instead of custom middleware |
| [Integration tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) | `WebApplicationFactory`, `TestServer` (Concept 73) |
| [ASP.NET Core support for Native AOT](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot) | **The compatibility table and `CreateSlimBuilder` differences** (Concepts 6, 75) |
| [Request Delegate Generator](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/aot/request-delegate-generator/rdg) | Concept 53, including the `RDG###` diagnostics |
| [.NET Aspire](https://learn.microsoft.com/en-us/dotnet/aspire/) | Service defaults, health checks, OTel wiring as a reusable platform layer (Concept 72) |

### Release notes and current state

| Resource | What it covers |
|---|---|
| [What's new in ASP.NET Core in .NET 11](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-11) | `[ShortCircuit]`, async/localized validation, OpenAPI 3.2, native OTel attributes, Kestrel parser change |
| [What's new in ASP.NET Core in .NET 10](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0) | Minimal API validation, SSE, OpenAPI 3.1, `Microsoft.Extensions.Validation` |
| [Migrate from ASP.NET Core in .NET 10 to .NET 11](https://learn.microsoft.com/en-us/aspnet/core/migration/100-to-110) | Breaking changes, including the antiforgery middleware change |
| [What's new in .NET 11](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/overview) | The platform context |
| [.NET Blog](https://devblogs.microsoft.com/dotnet/) | Release announcements and the annual performance posts |

### Blogs, talks, and community

| Resource | What it covers |
|---|---|
| [Andrew Lock — .NET Escapades](https://andrewlock.net/) | **The best free long-form writing on ASP.NET Core internals.** Series on minimal APIs, hosting, DI, source generators |
| [Behind the scenes of minimal APIs (series)](https://andrewlock.net/series/behind-the-scenes-of-minimal-apis/) | `RequestDelegateFactory`, binding, filters — Concepts 53–56 |
| [`CreateBuilder` vs `CreateSlimBuilder`](https://andrewlock.net/exploring-the-dotnet-8-preview-comparing-createbuilder-to-the-new-createslimbuilder-method) | Exactly what slim leaves out (Concept 6) |
| [Steve Gordon](https://www.stevejgordon.co.uk/) | "ASP.NET Core Anatomy" deep dives — hosting, DI internals, `IHttpClientFactory` internals |
| [Safia Abdalla](https://blog.safia.rocks/) | ASP.NET Core team member on minimal APIs, OpenAPI, and validation design |
| [.NET on YouTube](https://www.youtube.com/@dotnet) | *Deep .NET*, .NET Conf sessions on Kestrel, minimal APIs, AOT |
| [MinimalApiPlayground](https://github.com/DamianEdwards/MinimalApiPlayground) — Damian Edwards | Working examples of nearly every minimal API capability |
| [Scrutor](https://github.com/khellang/Scrutor) | Decoration and scanning for `Microsoft.Extensions.DependencyInjection` (Concept 47) |
| [FastEndpoints](https://fast-endpoints.com/) · [Carter](https://github.com/CarterCommunity/Carter) | Two opinionated takes on organizing minimal APIs (Concept 61) |
| [OpenTelemetry .NET](https://github.com/open-telemetry/opentelemetry-dotnet) | Exporting the meters and activity sources in Concept 70 |
| [Testcontainers for .NET](https://github.com/testcontainers/testcontainers-dotnet) | Real dependencies in integration tests (Concept 73) |
| [NetArchTest](https://github.com/BenMorris/NetArchTest) | Architecture rules as unit tests (Concept 72) |

### Books (not free, listed for completeness)

- **ASP.NET Core in Action, 3rd edition** — Andrew Lock. The best single book on this module's material; the DI, configuration and middleware chapters are the reference.
- **Pro ASP.NET Core 8/9** — Adam Freeman. Encyclopedic and example-driven; good for MVC's machinery.
- **Dependency Injection Principles, Practices, and Patterns** — Seemann & van Deursen. The theory behind Part E, container-agnostic; the anti-pattern catalogue is where "service locator" got its bad name.
- **Designing Data-Intensive Applications** — Kleppmann. Still the distributed-systems companion to everything in Phase 3 that this module touches.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| The request path | Transport → connection middleware → protocol → pooled `HttpContext` + DI scope → pipeline → routing → endpoint → write → reset |
| The four layers | Host (process), server (connections), pipeline (built once), endpoint (per request) |
| `CreateBuilder` defaults | JSON + secrets + env + args config; console/debug/EventSource logging; Kestrel; scope validation in Development |
| `CreateSlimBuilder` | No startup assemblies, no IIS, no HTTPS by default, fewer log providers — for AOT and startup |
| Configuration | Ordered providers, last wins; `__` for `:` in env vars; reload feeds `IOptionsMonitor` only |
| Options | `IOptions` singleton, `IOptionsSnapshot` scoped, `IOptionsMonitor` live; `ValidateOnStart` |
| `Build()` | Freezes the collection; never `BuildServiceProvider()` in `ConfigureServices` |
| Startup phases | Image pull → runtime → host build → bind → first request; first request pays JIT, model build, serializer metadata |
| Hosted services | `StartAsync` delays readiness; unhandled `ExecuteAsync` exceptions stop the host; create your own scopes |
| Graceful shutdown | SIGTERM → stop accepting → drain → `ShutdownTimeout` (30 s default); `preStop` delay + failing readiness |
| Auto middleware | `WebApplication` adds dev page, `UseRouting`, auth/authz (if services registered), `UseEndpoints` |
| Feature collection | `HttpContext` is a facade; features are replaceable; that's how buffering and compression work |
| Kestrel layers | Transport → connection middleware (TLS!) → protocol → application |
| Pipelines | Pooled buffers, `ReadOnlySequence`, `AdvanceTo`, end-to-end back-pressure — your `await` is the back-pressure |
| HTTP/1.1 | One request per connection at a time; keep-alive 130 s; .NET 11 non-throwing parse path |
| HTTP/2 | Streams, flow-control windows, HPACK; `MaxStreamsPerConnection` 100; TCP HOL remains |
| HTTP/3 | QUIC/UDP, no TCP HOL, needs MsQuic + HTTPS, discovered via `alt-svc`, enable alongside 1.1/2 |
| Kestrel limits | Body 30 MB, headers 32 KB/100, request line 8 KB, connections unlimited by default |
| Kestrel timeouts | Keep-alive 130 s, headers 30 s, min data rates 240 B/s with 5 s grace; not an execution timeout |
| Request timeout | `AddRequestTimeouts` + `WithRequestTimeout`; cancels `RequestAborted` |
| Forwarded headers | First in the pipeline; configure `KnownProxies`/`KnownNetworks`; headers are spoofable |
| `RequestAborted` | Client went away; pass it everywhere; don't log as an error; not for must-complete work |
| Pipeline model | `RequestDelegate` composed in reverse; before `next` = inbound, after = outbound |
| `UseWhen` vs `MapWhen` | `UseWhen` rejoins the main pipeline; `Map`/`MapWhen` do not |
| Middleware activation | Convention-based = singleton (scoped deps in `InvokeAsync`); `IMiddleware` = per request |
| Canonical order | Forwarded → exception → HSTS → HTTPS → static → routing → CORS → authn → authz → limiter/timeouts/cache/session → endpoints |
| Short-circuiting | Don't call `next`; `MapShortCircuit`; `.ShortCircuit()` / `[ShortCircuit]` (.NET 11) |
| Response started | Status and headers frozen after first flush; use `OnStarting`/`OnCompleted` |
| Body wrapping | Replace `IHttpResponseBodyFeature`, bound the buffer, restore in `finally` — or don't do it |
| Exception handling | `UseExceptionHandler` outermost + `AddProblemDetails` + `IExceptionHandler` chain + `traceId`, log once |
| `HttpContext` | Pooled and reset; never store it, never use it after the response |
| Endpoint routing | `UseRouting` matches, `UseEndpoints` executes; policy runs in between using metadata |
| Route precedence | Literal > constrained param > param > constrained catch-all > catch-all; `Order` breaks ties |
| The matcher | A DFA; cost is URL length, not route count; wrong verb = 405 |
| Metadata | The extension model: authz, CORS, rate limits, caching, OpenAPI, short-circuit all read it |
| `MapGroup` | Prefix + filters + conventions for a family; nests |
| Link generation | `LinkGenerator` works outside a request; name your endpoints; don't trust the Host header |
| DI engines | Call-site graph compiled to IL/expressions; runtime interpreter under AOT |
| Lifetimes | Singleton = root, scoped = per scope, transient = per resolve (but tracked if disposable) |
| Request scope | Created by hosting, disposed after the response completes — including the body flush |
| Captive dependency | Longer-lived captures shorter-lived; `ValidateScopes` + `ValidateOnBuild` |
| Disposal | Container disposes what it creates; `AddSingleton(new X())` is yours to dispose |
| Transient disposable | Tracked by the resolving scope; from the root that means "forever" |
| Scopes in workers | `IServiceScopeFactory.CreateAsyncScope()`, one per unit of work |
| `IHttpContextAccessor` | `AsyncLocal`; null outside requests; resolve request facts once into a scoped record |
| Registration semantics | Last wins for one; all for `IEnumerable<T>`; `TryAdd*` for defaults; `TryAddEnumerable` for sets |
| Keyed services | `.NET 8+`, `[FromKeyedServices]`; not a service locator |
| `ActivatorUtilities` | Construct non-services with injected deps; cache with `CreateFactory` |
| `IHttpClientFactory` | Handler pooling + rotation (2 min); handler scope ≠ request scope; or `PooledConnectionLifetime` |
| DI performance | Resolution is nanoseconds; graph size and assembly scanning are the costs |
| Minimal API mechanics | `RequestDelegateFactory` at startup, or RDG interceptors at compile time |
| Binding order | Attributes → special types → services → `BindAsync` → `TryParse` → route/query → body |
| `TypedResults` | Concrete result types feed OpenAPI and tests; `Results<T1,T2>` is the union |
| Endpoint filters | Nested, inside the endpoint, see bound arguments; .NET 11 also see binding failures |
| Validation | `AddValidation()` source-generated; async + localized in .NET 11; invariants still belong in the domain |
| MVC filter order | Authorization → resource → binding → action → result filters → result execution; global→controller→action |
| `[ApiController]` | Auto-400, binding inference, ProblemDetails, attribute routing required |
| Minimal vs MVC | AOT (MVC: no), startup, views/negotiation, team — not microseconds |
| Organizing endpoints | Endpoint modules + groups + static handlers + explicit registration |
| OpenAPI | 3.1 default in .NET 10, 3.2 in .NET 11; generate at build, diff in CI, don't ship UI publicly |
| Authentication | Schemes + handlers; challenge (401) ≠ forbid (403); `UseAuthentication` sets `User` |
| Authorization | Policies = AND of requirements, OR of handlers; `FallbackPolicy` is the secure default |
| CORS | Browser-enforced, before auth, not an API security control |
| Caching | Output (server, tag-evictable), response (headers only), HybridCache (data, stampede-safe) |
| Rate limiting | Fixed/sliding/token-bucket/concurrency; per instance; `QueueLimit = 0` usually; send `Retry-After` |
| Static assets | `MapStaticAssets` for build-known assets; static files run before authorization |
| Bodies | `EnableBuffering` sparingly; stream large uploads with `MultipartReader`; no sync IO |
| Metrics | `http.server.request.duration` histogram with `http.route`; Kestrel connection meters |
| Tracing | `Microsoft.AspNetCore` activity source; .NET 11 populates OTel attributes natively |
| Health checks | Liveness = process, readiness = dependencies; short-circuit them |
| Per-request cost | Framework is microseconds; dependencies are milliseconds — profile before optimizing |
| Composition root | One extension method per module, one place for middleware order, validate at startup |
| Integration testing | `WebApplicationFactory` proves wiring, ordering, policies; not Kestrel, TLS or perf |
| Multi-tenancy | Resolve early, scoped `TenantContext`, tenant in every key, fail closed |
| Web AOT | Slim builder + RDG + JSON source gen + zero warnings; MVC and EF Core are the blockers |
| Anti-patterns | Service locator, captive dependency, God middleware, captured context, sync-over-async, silent 500s |

---

## Progress

Module 18 complete — **Phase 4 has one module left.** Modules 14–17 gave you the runtime, the concurrency model, the language and the measurement discipline; this module put them inside the framework where they are actually exercised, and turned "I configure ASP.NET Core" into "I can explain and defend every layer of it."

This module closes several loops from earlier phases:

- **Module 6's overload protection** now has its in-process implementation: Kestrel's connection and size limits, the rate-limiting middleware's four algorithms, and the queue-vs-reject decision restated as `QueueLimit`.
- **Module 10's caching taxonomy** gained its HTTP-layer members — output caching with tag eviction, response caching as headers only, and HybridCache as the data-layer default — all still governed by the same three questions.
- **Module 13's bulkheads, timeouts and graceful degradation** became concrete: concurrency limiters, request timeouts, `RequestAborted` propagation, readiness probes, and a shutdown sequence that makes a rolling deploy invisible.
- **Module 14's captive-dependency and retention problems** were diagnosed properly in Part E — root-provider transients, singleton-held scopes, and the gcdump procedure that finds them.
- **Module 15's ThreadPool starvation and `async` rules** reappear as `AllowSynchronousIO = false`, sync-over-async in middleware, fire-and-forget against a pooled `HttpContext`, and back-pressure through pipelines.
- **Module 16's language features** show up as the idiom of this framework: `static` handler methods, `record` request/response types, `[AsParameters]`, and in .NET 11 C# 15 union types flowing through `System.Text.Json` into OpenAPI.
- **Module 17's startup budget, AOT practice and measurement loop** became the web-specific checklist in Concepts 7, 31 and 75.

Threads left open on purpose:

- **EF Core's internals** — change tracking as a retention mechanism, query translation, the N+1 problem, compiled and precompiled queries, `DbContext` pooling, and migration strategy at scale — are **Module 19**. Everything this module said about scoped lifetimes and unit-of-work boundaries is its groundwork.
- **Clean Architecture and layering**, and where the composition root of Concept 72 belongs in it, are **Module 20**.
- **CQRS, MediatR and pipeline behaviours** as an alternative to endpoint filters — and the complexity tax that decision carries — are **Module 23**.
- **Polly's retry, circuit breaker, timeout, bulkhead and hedging strategies**, including `AddStandardResilienceHandler` on the typed clients of Concept 50, are **Module 25**.
- **Compute platform choice and cold start** — App Service vs AKS vs Container Apps vs Functions, minimum replicas, and where AOT fits each — is **Module 26**.
- **OAuth2/OIDC, Entra ID, Key Vault and threat modelling**, which Concepts 63–65 only touched at the pipeline level, are **Module 29**.
- **SLOs, exemplars, and building an observability platform** out of the free telemetry in Concept 70 are **Module 28**.

Next in the curriculum: **Module 19 — EF Core deep dive** (change tracking, query translation, the N+1 problem, compiled queries, and migration strategy at scale), which takes the scoped-lifetime and unit-of-work discipline from Part E and turns it into the data-access half of .NET mastery — and finally settles the threads Module 12 left open about ORMs at architectural scale.
