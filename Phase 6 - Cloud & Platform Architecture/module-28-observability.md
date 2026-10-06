# Module 28 — Observability: OpenTelemetry, Distributed Tracing, SLOs and Structured Logging
*Phase 6: Cloud & Platform Architecture · Senior/Architect Interview Prep for .NET & C#*

> **Platform state verified on October 6, 2026.** Observability tooling has moved fast in the last eighteen months, and several of the facts below are recent enough that a stale answer is a visible signal:
>
> - **OpenTelemetry .NET:** the core SDK (`OpenTelemetry`, `OpenTelemetry.Extensions.Hosting`, the OTLP exporter) is at **1.19.1** (September 21, 2026) on a roughly monthly cadence. Traces, metrics and logs are all stable in .NET; some instrumentation packages (notably SQL client) are still pre-release — check each package's README before relying on it.
> - **Semantic conventions:** the registry is at **v1.43** (July 2026). **HTTP** conventions have been **stable since v1.23 (November 2023)**, **database** conventions became **stable in v1.33 (2025)**; **GenAI** conventions are still **Development** status, and messaging conventions were not yet stable at last check — verify before you build dashboards that depend on their names.
> - **Profiles — the fourth signal:** OpenTelemetry **Profiles entered public Alpha in March 2026** (Collector support from v0.148.0, eBPF profiler for Linux). Not for critical production use yet.
> - **Azure Monitor on .NET — three instrumentation paths:**
>   - the **Azure Monitor OpenTelemetry Distro** (`Azure.Monitor.OpenTelemetry.AspNetCore` **1.6.0**, July 27, 2026; `AddOpenTelemetry().UseAzureMonitor()`), which samples with a **rate limit of 5 traces per second by default**;
>   - the new **Microsoft OpenTelemetry Distro** (`Microsoft.OpenTelemetry`, **1.0 since May 1, 2026**; `builder.UseMicrosoftOpenTelemetry(...)`), which bundles Azure Monitor, OTLP and AI-agent observability with an almost identical API and is what Microsoft now points existing Application Insights customers toward;
>   - **Application Insights .NET SDK 3.x** (`Microsoft.ApplicationInsights.AspNetCore` **3.0 GA February 2026**, now **3.1.2**), a re-implementation of the classic `TelemetryClient` API **on top of OpenTelemetry** — the upgrade path for 2.x code. Adaptive sampling, telemetry initializers/processors/modules, `TrackPageView` and a long list of 2.x packages are gone. **Don't mix 3.x with a distro in one app.**
> - **Native OTLP ingestion into Azure Monitor:** OTLP traces and logs land in **Log Analytics** using OpenTelemetry semantic conventions; OTLP metrics land in an **Azure Monitor workspace** (managed Prometheus, queried with PromQL). The **AKS** and **Azure Monitor Agent** OTLP paths are in **preview**; Microsoft's pages disagree on whether the **Collector** path is GA or preview, so verify before committing. The Collector path requires **Collector v0.132.0+** with the Azure authentication extension and **Entra ID** authentication.
> - **Azure Container Apps managed OpenTelemetry agent:** destinations are Application Insights, Datadog or any OTLP endpoint; **gRPC only**; the Application Insights destination takes **traces and logs, not metrics**, and the Application Insights resource must still allow local authentication.
> - **Dashboards with Grafana in the Azure portal** — **GA since November 2025**, at no extra cost; Azure Managed Grafana remains the full managed Grafana service.
> - **Log Analytics table plans** (approximate US list prices): **Analytics ~$2.30/GB**, **Basic ~$0.50/GB**, **Auxiliary ~$0.05/GB** ingested (first 5 GB per billing account per month free), with query charges on Basic and Auxiliary.
> - **.NET:** log **buffering** (global and per-request) and log **sampling** arrived in **.NET 9** via `Microsoft.Extensions.Telemetry` and `Microsoft.AspNetCore.Diagnostics.Middleware`. **Aspire 13.2** (March 2026) added dashboard telemetry export; **Aspire 13.3** added `aspire dashboard run` for the standalone dashboard. In **.NET 11 Preview 4**, even the `dotnet` CLI's own telemetry moved from `Microsoft.ApplicationInsights` to OpenTelemetry — a fair summary of where Microsoft's tooling has gone.
> - **SLOs on Azure:** at the time of writing there is **no first-class SLO object in Azure Monitor**. You build SLIs and burn-rate alerts from Prometheus recording rules, KQL log search alerts or workbooks, or use a dedicated SLO tool (Grafana SLO, Nobl9, Sloth/Pyrra on Prometheus). Check before you assert otherwise.
>
> Prices are approximate US list prices used for **arithmetic**, not procurement.

## Orientation

Here is the sentence to carry through the whole module: **observability is the ability to answer a question about your production system that you did not anticipate when you wrote the code — and it comes from emitting well-structured, correlated, affordable telemetry, then deciding in advance, with SLOs, which questions are worth waking someone up for.**

Every earlier module has quietly depended on this one. Module 13 told you to detect failures and degrade gracefully — detection is telemetry. Module 25 composed retries and circuit breakers — the only way to know whether they're helping is to measure them. Module 26's KEDA scaling runs on metrics. Module 27 ended Part G with "the few metrics per service that predict incidents" and promised the depth here. And every "how would you debug this?" question in a design interview is an observability question in disguise.

The module also sits on theory you already have:

- **Module 5** gave you latency numbers and percentiles; here you'll learn why percentiles can't be averaged and how histograms preserve them.
- **Module 6** introduced Little's Law and the Universal Scalability Law; saturation signals in this module are those laws, measured.
- **Module 11** introduced OpenTelemetry for messaging and trace-context propagation through messages.
- **Module 13** introduced availability arithmetic and composite SLAs; error budgets are that arithmetic turned into a management tool.
- **Modules 14, 15 and 17** covered the CLR, the thread pool and performance engineering; their counters (GC, thread-pool queue length, allocation rate) are the saturation metrics you'll alert on.
- **Module 18** covered the ASP.NET Core pipeline; that's where request telemetry is born.

What none of them did is treat telemetry as a **system you design**: the data model of each signal, the pipeline that carries it, the standards that make it portable, the economics that make it affordable, the objectives that turn it into decisions, and the Azure and .NET surfaces that implement all of it. That's this module.

Why it matters in an interview: almost every senior and architect loop contains *"How would you know this design is working in production?"* Weak answers say "we'd add logging and dashboards." Strong answers name **SLIs and SLOs per user journey**, describe **traces that cross service and message boundaries**, explain **metrics with bounded cardinality**, **structured logs correlated by trace ID**, a **sampling strategy**, **burn-rate alerts that page on symptoms**, and a **telemetry budget**. Current loops ask: *"Why can't I put the user ID on this metric?"*, *"Our p99 went up after a deploy — walk me through finding out why"*, *"How do you trace a request through a Service Bus queue?"*, *"Our Application Insights bill is $40,000 a month — what do you do?"*, *"Define an SLO for this API and the alert you'd page on"*, *"Head or tail sampling?"*, and *"We're on Application Insights SDK 2.x — what's the migration story?"*

This module has nine jobs:

1. **Build a first-principles model** — monitoring vs observability, the jobs telemetry does, the signals as data structures, cardinality, correlation, the telemetry pipeline as a distributed system, and the economics.
2. **Teach OpenTelemetry as an architecture** — spec, API/SDK split, resources, semantic conventions, OTLP, context propagation, the instrumentation modes, the Collector, sampling and distros.
3. **Teach distributed tracing precisely** — the span model, kinds and names, status and exceptions, asynchronous and messaging traces, reading traces, attribute discipline and broken traces.
4. **Teach metrics precisely** — instruments, temporality, histograms and percentiles, cardinality budgets, RED/USE/golden signals, exemplars, Prometheus and .NET's built-in metrics.
5. **Teach structured logging precisely** — message templates, the `ILogger` pipeline, high-performance logging, correlation, levels, redaction, volume control and audit.
6. **Teach SLOs and error budgets** — SLIs, SLOs, SLAs, choosing and writing them, budget arithmetic, budget policies, burn-rate alerting, alert design, asynchronous SLOs and SLOs in organizations.
7. **Map it onto Azure** — Azure Monitor's data platform, the three .NET instrumentation paths in 2026, OTLP ingestion, compute-specific collection, KQL, alerts, dashboards and cost.
8. **Make .NET do it right** — `System.Diagnostics` primitives, wiring OpenTelemetry, custom traces and metrics, enrichment, propagation through background work, the diagnostics toolbox and testing telemetry.
9. **Make it operable and defensible** — health checks and synthetics, incident response, choosing an observability architecture, hidden costs, anti-patterns and a design review.

Seven framings to carry through:

1. **Telemetry is data with a schema.** Spans, metrics and log records each have a data model, and most observability failures are modeling failures — wrong names, wrong attributes, wrong cardinality — not tooling failures.
2. **Correlation is the multiplier.** Three disconnected signals are three tools; three signals joined by trace ID, resource identity and exemplars are one investigation.
3. **Cardinality is the price.** Every attribute value multiplies storage and cost in metrics; traces and logs tolerate high cardinality but pay per event. Know which signal can afford which detail.
4. **Sampling and aggregation are the economic levers.** You can't keep everything; you choose what to aggregate, what to sample and what to keep at full fidelity, and you choose deliberately.
5. **Standards beat vendors.** Instrument with OpenTelemetry APIs and semantic conventions; treat the backend as a replaceable exporter. Lock-in belongs in the query layer, not the code.
6. **SLOs decide what matters.** Telemetry without objectives produces dashboards nobody reads and alerts everybody ignores. The SLO is the contract that tells you which signals to page on.
7. **Telemetry must never take the system down.** It is a best-effort, fail-open, bounded-overhead subsystem with its own failure modes, budget and owners.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Monitoring vs observability | Monitoring answers known questions; observability lets you ask new ones without shipping code |
| 2 | The jobs of telemetry | Detect, localize, diagnose, verify — each needs different signals |
| 3 | The signals as data structures | Metrics aggregate, traces record causality, logs record events, profiles record code cost |
| 4 | Cardinality and dimensionality | Series count is the product of attribute values — the cost model of metrics |
| 5 | Correlation | Trace IDs, shared resource identity and exemplars turn signals into one investigation |
| 6 | Telemetry is a distributed system | Emit, buffer, export, collect, store, query — each stage loses data and costs money |
| 7 | The economics | Aggregate, sample, retain — three levers on a bill that grows with traffic |
| 8 | What OpenTelemetry is | A spec, APIs, SDKs, conventions, a protocol and a Collector — not a backend |
| 9 | API vs SDK | Libraries depend on the API; applications choose the SDK and exporters |
| 10 | Resource and service identity | `service.name`, version, environment and instance identify every signal |
| 11 | Semantic conventions and stability | Shared attribute names make telemetry portable; know which are stable |
| 12 | OTLP | One protocol for all signals, over gRPC or HTTP, configured by environment |
| 13 | Context propagation | `traceparent`, `tracestate` and baggage carry the trace across processes |
| 14 | Four ways to instrument | Native, library, zero-code and manual — use all four deliberately |
| 15 | The Collector | Receivers, processors, exporters; agent and gateway; where policy lives |
| 16 | Sampling, properly | Head vs tail, parent-based consistency, rate limits, and metrics before sampling |
| 17 | Distros, vendors and lock-in | Distros are opinionated SDK bundles; lock-in lives in queries and dashboards |
| 18 | The span model | IDs, parent, name, kind, timing, attributes, events, links, status |
| 19 | Span kinds and names | Server, client, producer, consumer, internal; low-cardinality names |
| 20 | Status, errors and exceptions | When a span is an error, and how exceptions are recorded |
| 21 | Tracing asynchronous and messaging flows | Parents vs links, batches, fan-out and trace boundaries |
| 22 | Reading traces | Critical path, waterfalls, the shapes of N+1, retries and contention |
| 23 | What goes on a span | Business identifiers yes, secrets and payloads no, and limits |
| 24 | Broken traces and trust boundaries | Lost context, skew, and untrusted `traceparent` from the internet |
| 25 | The metrics data model | Instruments, measurements, attributes, aggregations, streams |
| 26 | Temporality | Cumulative vs delta, resets, and what your backend wants |
| 27 | Histograms and percentiles | Buckets preserve distributions; averages of percentiles are fiction |
| 28 | Cardinality budgets and views | Bound attributes, drop or rename with views, honor SDK limits |
| 29 | RED, USE and the golden signals | Three checklists for what to measure |
| 30 | Exemplars | A metric bucket that remembers a trace |
| 31 | Prometheus and the Azure Monitor workspace | Pull, PromQL, recording rules — managed on Azure |
| 32 | .NET's built-in metrics | ASP.NET Core, Kestrel, HttpClient, runtime — already emitted |
| 33 | Structured logging from first principles | Template plus properties, not strings |
| 34 | The `ILogger` pipeline | Categories, providers, filters, scopes |
| 35 | High-performance logging | Source-generated `LoggerMessage`, `IsEnabled`, no allocations when off |
| 36 | Correlating logs with traces | Trace and span IDs on every record, automatically |
| 37 | What to log, and at which level | Boundaries, decisions and failures — once |
| 38 | Sensitive data and redaction | Classification, redactors, and keeping PII out of telemetry |
| 39 | Volume control | Filtering, sampling, buffering and table plans |
| 40 | Logs, events and audit trails | Different purposes, different stores, different guarantees |
| 41 | Why SLOs exist | Reliability is a feature with a cost; 100% is the wrong target |
| 42 | SLI, SLO, SLA, error budget | Good events over valid events, a target, a contract, a budget |
| 43 | Choosing SLIs | Availability, latency, freshness, correctness — measured where users feel them |
| 44 | Writing an SLO | Target, window, threshold, exclusions — precisely |
| 45 | Error budget arithmetic | Nines to minutes and requests; composition across dependencies |
| 46 | The error budget policy | What the organization agrees to do when the budget is gone |
| 47 | Alerting on burn rate | Multi-window, multi-burn-rate alerts with a derivation |
| 48 | Alert design beyond SLOs | Page on symptoms, ticket on causes, every page actionable |
| 49 | SLOs for asynchronous systems | Freshness, lag, age and end-to-end latency from event time |
| 50 | SLOs in an organization | Owners, documents, reviews, dependencies and SLAs with margin |
| 51 | The Azure Monitor map | Log Analytics, Azure Monitor workspace, platform metrics, Application Insights |
| 52 | Instrumenting .NET for Azure Monitor in 2026 | Azure Monitor Distro, Microsoft OpenTelemetry Distro, or SDK 3.x |
| 53 | The OTLP-native path | Collector, AMA and AKS ingestion into Azure Monitor |
| 54 | Compute-specific collection | App Service, Functions, Container Apps, AKS |
| 55 | KQL for engineers | The queries you'll actually write |
| 56 | Alerts and action groups | Metric, log search and Prometheus alerts; routing and suppression |
| 57 | Dashboards and investigation tools | Application Map, transaction search, Live Metrics, workbooks, Grafana, Aspire |
| 58 | Cost governance | Table plans, sampling, transformations, retention and caps |
| 59 | `System.Diagnostics`: the native primitives | `ActivitySource`, `Meter`, `ILogger`, `EventSource` |
| 60 | Wiring OpenTelemetry in ASP.NET Core | Resource, providers, instrumentation, exporters, configuration |
| 61 | Custom traces | One `ActivitySource` per component, null-safe activities, links |
| 62 | Custom metrics | `IMeterFactory`, instruments, units, bounded tags |
| 63 | Enrichment, filtering and processors | Add context, drop noise, at the right layer |
| 64 | Propagation in .NET, including background work | HttpClient, Azure SDKs, channels, hosted services |
| 65 | The diagnostics toolbox | `dotnet-counters`, `-trace`, `-dump`, `dotnet-monitor`, profilers |
| 66 | Testing telemetry | `FakeLogger`, `MetricCollector<T>`, in-memory exporters |
| 67 | Health checks, synthetics and probes | Liveness, readiness, availability tests — and their limits |
| 68 | Incident response with telemetry | Detect, triage, mitigate, diagnose, learn |
| 69 | Choosing an observability architecture | Azure Monitor, a Grafana stack, a vendor — or a Collector in front of all three |
| 70 | Hidden costs | The line items nobody budgets for |
| 71 | Anti-patterns | The mistakes that make telemetry useless or ruinous |
| 72 | The observability design review, and when not to | A checklist to narrate, and the restraint that signals seniority |

---
# Part A — First principles: telemetry as data, as a pipeline and as a cost

## Concept 1 — Monitoring vs observability

The word *observability* comes from control theory: a system is **observable** if you can infer its internal state from its external outputs. Applied to software, the outputs are telemetry, and the question becomes: *can I explain what my system is doing, and why, from what it emits — without changing it?*

That gives a clean distinction:

| | **Monitoring** | **Observability** |
|---|---|---|
| The questions | Known in advance: "is the error rate above 1%?", "is the disk full?" | Not known in advance: "why are only Android users in Germany on the new plan slow since Tuesday?" |
| The data | Pre-aggregated metrics and checks designed for those questions | Rich, high-dimensional, correlated events (traces, structured logs, wide events) you can slice later |
| The workflow | Dashboards and threshold alerts | Exploration: filter, group by, compare, follow a trace |
| What it catches | **Known unknowns** — failure modes you predicted | **Unknown unknowns** — failure modes you didn't |
| Its limit | Every new question needs new instrumentation and a deploy | Expensive if you keep everything; useless if the data isn't structured |

They're not rivals. **Monitoring is a subset of observability**: you still need cheap, always-on metrics and alerts for the questions you *can* predict, and you need the richer data to answer the ones you can't. A system that is only monitored tells you *that* something is wrong; a system that is observable lets you find out *what* and *why* without shipping a new build at 3 a.m.

Why the distinction became important: in a monolith on three servers, a stack trace and a log file answered most questions. In a system of forty services, queues, caches, managed databases and third-party APIs, failures are **emergent** — no single component is broken, but a retry storm, a hot partition and a slow dependency combine into an outage. Those failures are almost by definition the ones nobody predicted, so a dashboard built for predicted failures shows green while customers suffer. That's the gap observability fills.

**The interview-grade sentence:** *"Monitoring answers the questions I knew to ask — thresholds and dashboards for predicted failure modes; observability is being able to answer new questions about production from the telemetry I already emit, without shipping code. In distributed systems most serious failures are emergent and unpredicted, so I want both: cheap metrics and SLO alerts for the known, and structured, correlated traces and logs rich enough to explore the unknown."*

---

## Concept 2 — The jobs of telemetry: detect, localize, diagnose, verify

Telemetry is not one job. In an incident it does four, in order, and each favors different signals:

| Job | The question | Best signals | Why |
|---|---|---|---|
| **1. Detect** | "Is something wrong for users right now?" | **Metrics** (SLI rates, latency histograms), synthetic checks | Cheap to compute continuously, cheap to alert on, complete (not sampled) |
| **2. Localize** | "Where — which service, endpoint, region, dependency, tenant?" | **Metrics with a few dimensions**, **traces** (service map, span breakdown) | Narrow from "the system" to "this component" quickly |
| **3. Diagnose** | "Why — what exactly is happening in that component?" | **Traces** (the slow span, its attributes), **logs** (the error and its context), **profiles** (where the CPU goes) | Detail and causality, at per-request granularity |
| **4. Verify** | "Did my fix (or rollback) work?" | **Metrics** again, compared before/after | Same signal that detected it, now recovering |

This sequence maps onto the incident clock. **MTTR** (mean time to restore) decomposes into **time to detect** + **time to localize** + **time to mitigate** + time to verify. Each component has a telemetry answer: SLO burn-rate alerts shrink detection; service maps and per-dependency metrics shrink localization; good traces and logs shrink diagnosis; and fast, comparable metrics shrink verification. When you design telemetry, design for the incident clock, not for a dashboard screenshot.

Two consequences shape the rest of the module:

1. **Detection must not depend on sampled data.** If you sample traces at 5%, you can't compute an accurate error rate from traces alone — the alert needs metrics computed from 100% of requests (or from traces *before* sampling — Concept 16).
2. **Diagnosis needs the detail that metrics can't hold.** The user ID, tenant ID, order ID, SQL text and exception message that explain a failure can't go on metrics (Concept 4) — they belong on spans and log records, which can be sampled, filtered and retained differently.

**The interview-grade sentence:** *"I design telemetry around the incident clock: metrics computed from every request detect user impact and verify fixes; metrics with a few bounded dimensions and the service map localize it; traces, structured logs and profiles diagnose it. So detection never relies on sampled data, and the high-cardinality detail lives on spans and logs, not on metrics."*

---

## Concept 3 — The signals as data structures

"Logs, metrics and traces" are often taught as three tools. It's more useful to see them as **three data structures**, each with a cost model and a query shape. Add **profiles**, now the fourth OpenTelemetry signal.

**Metrics — aggregates over time.** A metric is a *time series*: a name, a set of attribute values (dimensions), and numeric values aggregated per time interval — a count, a sum, a gauge reading, or a histogram of bucket counts.

- **Cost** is proportional to the **number of series** × **resolution**, *not* to traffic. A counter incremented a billion times costs the same as one incremented ten times.
- **Query shape:** rates, ratios, percentiles, comparisons over time — fast, because the data is pre-aggregated.
- **What it can't do:** answer anything about an individual request, or about a dimension you didn't record.

**Traces — causal graphs of work.** A trace is a tree (more generally a DAG) of **spans**; each span is a timed operation with attributes, events and a parent.

- **Cost** is proportional to **requests × spans per request × bytes per span** — it grows with traffic, which is why traces are sampled.
- **Query shape:** "show me this request," "show me slow requests where `tenant = X`," "what's the latency breakdown by span," service dependency maps.
- **What it's uniquely good at:** **causality and timing across processes** — what called what, in what order, how long each part took.

**Logs — discrete events.** A log record is a timestamp, a severity, a message (ideally a template), and properties.

- **Cost** is proportional to **events × bytes** — usually the largest bill of the three, because developers log liberally and per-event.
- **Query shape:** search and filter, group by property, count occurrences.
- **What it's good at:** recording **decisions and failures** with arbitrary context, especially for things that aren't requests (startup, background jobs, configuration changes).

**Profiles — where code spends resources.** A profile is a set of sampled stack traces with resource weights (CPU time, allocations, lock contention) over a period.

- **Cost** depends on sampling frequency and stack depth; continuous profilers sample at low frequency to keep overhead around a percent.
- **What it's uniquely good at:** "*which function* is burning the CPU?" — a question traces answer only down to the span boundary.
- **Status:** the OpenTelemetry Profiles signal entered **public Alpha in March 2026**; vendor profilers (Application Insights Profiler, Pyroscope, Datadog, Parca) remain the production options for now.

**Wide events — the unifying idea.** A school of thought (associated with Honeycomb and sometimes called "observability 2.0") argues that a single **wide, structured event per unit of work** — one record per request with dozens or hundreds of fields — can replace much of the three-signal split: metrics are just aggregations you compute at query time, logs are just events, and traces are events with parent IDs. In OpenTelemetry terms, a **span with rich attributes** *is* a wide event. The practical lesson, even if you keep separate backends: **put context on spans generously**, because a span is the cheapest place to keep high-cardinality detail that's already correlated.

| | Metrics | Traces | Logs | Profiles |
|---|---|---|---|---|
| Unit | Time series point | Span | Log record | Stack sample |
| Cost driver | Series count | Requests × spans | Events × bytes | Sample rate × depth |
| High cardinality | ❌ Expensive | ✅ Fine per span | ✅ Fine per record | n/a |
| Complete (unsampled) | ✅ Usually | ❌ Usually sampled | Often filtered | Sampled by design |
| Best job | Detect, verify | Localize, diagnose across services | Diagnose within a component; non-request events | Diagnose inside code |

**The interview-grade sentence:** *"I treat the signals as data structures with cost models: metrics are aggregated time series whose cost grows with series count, not traffic, so they're complete but low-cardinality; traces are causal graphs whose cost grows with traffic, so they're sampled but carry high-cardinality detail; logs are discrete events and usually the biggest bill; profiles show where code spends CPU and memory. A richly attributed span is effectively a wide event, so that's where I put per-request context."*

---

## Concept 4 — Cardinality and dimensionality: the cost model of metrics

**Cardinality** is the number of distinct values an attribute can take. **Dimensionality** is the number of attributes. For metrics, the two combine multiplicatively:

```
time series for one metric  =  Π (distinct values of each attribute)  (in the worst case)
```

A worked example. `http.server.request.duration`, a histogram, with these attributes:

| Attribute | Distinct values |
|---|---|
| `http.request.method` | 5 |
| `http.route` | 40 |
| `http.response.status_code` | 10 |
| `service.instance.id` | 20 pods |

Worst case: 5 × 40 × 10 × 20 = **40,000 series**. And a histogram isn't one series — with ~15 buckets plus sum and count, each combination is about **17 underlying series**, so roughly **680,000**. Most combinations never occur, so the real number is lower, but the shape is clear.

Now add `user.id` with 2 million users: the worst case becomes 1.36 **trillion**. That's the classic **cardinality explosion**: the metric backend's memory, storage and query time collapse, and on a managed service your bill follows. Prometheus-style systems typically degrade in the low millions of active series per instance; managed services charge per sample or per series.

What this means in practice:

1. **Bounded attributes only on metrics.** Method, route *template* (`/orders/{id}`, never `/orders/42`), status code *class* or code, region, deployment version, tenant *tier* — fine. User ID, order ID, raw URL, SQL text, exception message, email — **never on metrics**.
2. **Unbounded detail goes on spans and logs**, where each record is paid for once and cardinality costs nothing extra.
3. **"Per tenant" is a judgment call.** Fifty enterprise tenants is bounded; 50,000 self-service tenants isn't. Use a tier or a top-N bucket ("tenant or `other`") on metrics, and the real tenant ID on spans.
4. **Cardinality limits exist in the SDK.** The OpenTelemetry .NET SDK caps the number of attribute combinations per metric (a default of 2,000 metric points per metric stream, configurable); beyond it, measurements are folded into an overflow series (`otel.metric.overflow = true`) rather than growing without bound. Hitting the cap is a design bug, not a tuning problem (Concept 28).

**Traces and logs tolerate high cardinality** because they're paid for per record, and their backends index differently (columnar stores, trace-ID lookups). That's why the same `tenant.id` that would ruin a metric is exactly what you want on a span.

**The interview-grade sentence:** *"Metric cost is the product of the distinct values of every attribute, so one unbounded attribute — a user ID, a raw URL — turns forty thousand series into trillions and takes the metrics backend down with it. I keep metrics to bounded dimensions like method, route template, status and region, bucket tenants into tiers or a top-N, and put the high-cardinality identifiers on spans and log records, where each record is paid for once."*

---

## Concept 5 — Correlation: the multiplier

Three signals that can't be joined are three separate tools with three separate investigations. The value of observability comes from **moving between signals without losing your place**. Three mechanisms make that possible:

**1. Trace context on everything.** Every span carries a **trace ID** and **span ID**. If every **log record** written during that span also carries them — automatically, not by developer discipline — then "show me the logs for this slow request" is a filter, not a hunt. In .NET with OpenTelemetry, `ILogger` records written inside an active `Activity` get `TraceId` and `SpanId` attached by the SDK (Concept 36).

**2. Shared resource identity.** Every signal from a process carries the same **resource attributes**: `service.name`, `service.version`, `deployment.environment.name`, `service.instance.id`, plus cloud and Kubernetes attributes. That's how a metric spike in `orders-api v2.3.1 in prod` connects to the traces and logs of the same service and version. Inconsistent service names across signals (`OrdersApi` in metrics, `orders-api` in traces) are a surprisingly common reason correlation fails (Concept 10).

**3. Exemplars.** A metric bucket can carry an **exemplar**: a sampled measurement together with the trace ID of the request that produced it. Click the p99 spike on a latency chart and jump straight to a trace that was *in* that bucket (Concept 30).

Together they enable the canonical investigation path:

```
SLO burn-rate alert (metric)
   → latency histogram by route and region (metric, bounded dimensions)
      → exemplar on the slow bucket (trace ID)
         → the trace: slow span is a Cosmos DB query, 429-retried 4 times
            → logs for that span: "partition key range 3 throttled", tenant = contoso
               → profile or partition metrics to confirm the hot key
```

The "three pillars" framing gets criticized precisely because it suggests three independent stores and UIs. The critique is right about the goal — **one connected investigation** — even if, in practice, the data still lives in different stores. Your job as the architect is to guarantee the joins: the same trace context everywhere, the same resource identity everywhere, exemplars on the latency metrics.

**The interview-grade sentence:** *"Correlation is what makes three signals one investigation: trace and span IDs stamped automatically on every log record, the same resource identity — service name, version, environment, instance — on every signal, and exemplars that link a metric bucket to a trace. That gives the path from a burn-rate alert to a slow route, to an exemplar trace, to the slow span, to its logs, without guessing."*

---

## Concept 6 — Telemetry is a distributed system

Telemetry is itself a distributed data pipeline, with every property you'd examine in any other design: throughput, latency, loss, backpressure, cost and failure modes.

```
 in-process                           network                          backend
┌──────────────────────────────┐   ┌───────────────────────┐   ┌─────────────────────────────┐
│ instrument → SDK processors  │ → │ exporter → (agent /   │ → │ ingest → transform → store  │ → query, alert, dashboard
│ (sample, enrich, batch,      │   │  gateway Collector)   │   │ (index, aggregate, retain)  │
│  bounded in-memory queue)    │   │ retry, batch, filter  │   │                             │
└──────────────────────────────┘   └───────────────────────┘   └─────────────────────────────┘
```

Properties to design for:

1. **Overhead must be bounded.** Instrumentation runs on the hot path. Creating a span costs on the order of a microsecond and some allocations; logging a formatted string at a disabled level can cost more than you think if it allocates (Concept 35). Batching exporters move serialization and I/O off the request thread. Target low single-digit percent CPU overhead and measure it (Module 17's BenchmarkDotNet discipline applies).
2. **Telemetry must fail open.** If the backend is down or slow, the application must keep serving. The OpenTelemetry SDK's batch processors use **bounded queues** that **drop** when full (the SDK can now report this through its experimental self-observability metrics — `otel.sdk.processor.span.processed` with `error.type = queue_full`). Never make a request wait on telemetry export, and never let a telemetry exception escape into business code.
3. **Loss is normal — know where.** Data is lost when queues overflow, when a process crashes before flushing (flush on graceful shutdown — Module 26's termination grace period), when sampling drops it by design, when a Collector is overloaded, and when a backend throttles ingestion. Distinguish **intentional** loss (sampling, filtering) from **accidental** loss, and monitor the latter.
4. **Latency to insight.** Metrics are usually queryable within a minute; logs and traces in Log Analytics typically within a few minutes (ingestion latency). Live Metrics in Application Insights streams within a second or two but isn't stored. Alerts that depend on log queries inherit ingestion latency plus evaluation frequency — relevant to detection time (Concept 47).
5. **The pipeline needs its own monitoring.** Exporter errors, dropped spans, Collector queue length and memory, ingestion volume per table, and the daily ingestion trend are the telemetry about your telemetry. A dashboard for "is our observability working?" is not optional at scale.
6. **The observer effect is real.** Synchronous log sinks writing to disk or network, a profiler at high frequency, or `RecordException` capturing huge stack traces in a tight loop can themselves become the incident.

**The interview-grade sentence:** *"I treat telemetry as a distributed pipeline — instrument, process, batch, export, collect, ingest, store, query — with bounded overhead on the hot path, batching exporters with bounded queues that drop rather than block, flush on shutdown, a clear distinction between intentional loss like sampling and accidental loss, known ingestion latency for alerting, and its own health metrics. It must always fail open: a slow backend should cost me telemetry, never availability."*

---

## Concept 7 — The economics: aggregate, sample, retain

Telemetry spend grows with traffic, with the number of services, and with developer enthusiasm. It's not unusual for observability to become **10–30% of a cloud bill**, and for Log Analytics ingestion to be the single largest Azure Monitor line. An architect needs a cost model and three levers.

**The cost model**, roughly:

```
cost ≈ Σ over signals ( volume ingested × price per unit  +  volume retained × retention price  +  query/scan charges )
       + backend fixed costs (Collector compute, dashboards, alert rules)
```

with volume per signal:

- metrics ≈ active series × samples per minute (or per-series pricing);
- traces ≈ requests/s × spans per request × bytes per span × **sampling rate**;
- logs ≈ events/s × bytes per event × **(1 − filtered fraction)**.

**A quick estimate** (Module 5 style): an API at 2,000 requests/s, 8 spans per request at ~1 KB each, plus 5 log records per request at ~0.5 KB:

```
spans:   2,000 × 8 × 1 KB = 16 MB/s ≈ 1.38 TB/day
logs:    2,000 × 5 × 0.5 KB = 5 MB/s ≈ 432 GB/day
total:   ≈ 1.8 TB/day  → at ~$2.30/GB Analytics ingestion ≈ $4,100/day ≈ $125,000/month
with 5% trace sampling and Information-level logs cut to 1 per request:
spans:   ~69 GB/day;  logs: ~86 GB/day  → ~155 GB/day ≈ $360/day ≈ $11,000/month
```

The exact numbers matter less than the habit: **compute the telemetry bill like any other capacity estimate**, before production does it for you.

**The three levers:**

1. **Aggregate.** Convert per-event data into metrics *before* storage. A request-count metric computed in the SDK costs the same at 10 or 10,000 requests per second. Span-to-metrics (the Collector's `spanmetrics` connector, or Application Insights standard metrics computed pre-sampling) lets you sample traces aggressively while keeping accurate RED metrics.
2. **Sample.** Keep a representative or *interesting* subset of traces (and logs tied to them): head sampling for cheap baseline reduction, tail sampling to keep errors and slow requests (Concept 16). Sampling is safe *only* if detection doesn't depend on the sampled data (Concept 2).
3. **Retain and tier.** Not everything needs 90 days of interactive query. Hot data for incident response (days to weeks), cheaper tiers for compliance and occasional investigation (Basic/Auxiliary table plans, archive, data export to a lake). Retention is a per-table decision, not a workspace-wide default.

Plus the lever that precedes all three: **don't emit it**. Debug logs in production, health-check request spans, duplicate instrumentation (two HTTP instrumentations both producing spans), verbose framework categories at `Information` — the cheapest gigabyte is the one never sent.

**Telemetry budgets** make this sustainable: each service gets an ingestion budget (GB/day) with an alert at, say, 80%, and growth beyond it is a design conversation — not a surprise on the invoice. Daily caps exist as a last resort but cut off data indiscriminately, including the data you need during the incident that caused the spike (Concept 58).

**The interview-grade sentence:** *"I estimate telemetry like any capacity: requests times spans times bytes, events times bytes, active series — and at Analytics-tier prices a 2,000-request-per-second API can easily produce a six-figure monthly bill unsampled. The levers are to aggregate into metrics before storage so detection stays accurate, sample traces — head for baseline, tail to keep errors and slow requests — tier retention per table, and above all not emit noise like debug logs and health-check spans, with a per-service ingestion budget so growth is a design decision."*

---
# Part B — OpenTelemetry as an architecture

## Concept 8 — What OpenTelemetry is (and isn't)

**OpenTelemetry (OTel)** is a CNCF project, formed in 2019 by merging **OpenTracing** (a tracing API standard) and **OpenCensus** (Google's metrics-and-tracing libraries). It's now one of the CNCF's most active projects and the de facto industry standard for producing telemetry. The key word is *producing*: **OpenTelemetry is not a backend.** It doesn't store or visualize anything. It standardizes everything between your code and the backend of your choice.

Its parts:

| Part | What it is | Why it matters |
|---|---|---|
| **Specification** | The language-neutral definition of the API, SDK, data model and protocol | Every language implementation behaves the same way |
| **API** | Interfaces for creating spans, recording metrics and emitting logs | What libraries and applications code against |
| **SDK** | The implementation: sampling, processing, aggregation, batching, exporting | What the application configures at startup |
| **Semantic conventions** | Standard names and meanings for attributes (`http.request.method`, `db.system.name`, `service.name`) | Telemetry from different libraries and languages means the same thing |
| **OTLP** | The OpenTelemetry Protocol, one wire format for all signals | Any SDK can talk to any Collector or backend |
| **Collector** | A standalone, vendor-neutral agent/gateway that receives, processes and exports telemetry | Where routing, filtering, sampling and redaction policy lives |
| **Instrumentation libraries** | Ready-made instrumentation for frameworks (ASP.NET Core, HttpClient, SQL, gRPC…) | You don't hand-instrument infrastructure |
| **Zero-code (auto) instrumentation** | Agents that inject instrumentation without code changes | Coverage for code you can't or won't change |

**Signal status** in 2026: **traces, metrics and logs are stable** in the specification and in the .NET implementation; **profiles are Alpha** (March 2026). Baggage and context propagation are stable.

What changed in the industry is that every major backend now **accepts OpenTelemetry natively** — Azure Monitor (via distros and, increasingly, raw OTLP), AWS CloudWatch, Google Cloud, Datadog, Dynatrace, New Relic, Elastic, Grafana, Honeycomb, Splunk. The vendor-specific SDKs that defined the 2010s (Application Insights SDK 2.x, vendor agents) are being re-platformed onto OTel. Microsoft's own trajectory is explicit: the Application Insights .NET SDK 3.x is a compatibility layer over OpenTelemetry, and even the `dotnet` CLI's telemetry moved to OpenTelemetry in .NET 11 Preview 4.

**The interview-grade sentence:** *"OpenTelemetry is the CNCF standard for producing telemetry — a specification, APIs, SDKs, semantic conventions, the OTLP protocol, a vendor-neutral Collector and instrumentation libraries — not a backend. Traces, metrics and logs are stable, profiles are in alpha since March 2026, and every major backend including Azure Monitor now ingests it, so I instrument once against OTel and treat the backend as an exporter choice."*

---

## Concept 9 — API vs SDK: who depends on what

OpenTelemetry deliberately separates the **API** (what you call) from the **SDK** (what does the work).

- **Libraries depend only on the API.** A library — your company's shared data-access package, an open-source client — creates spans and records metrics through the API. If the hosting application never installs an SDK, every API call is a **no-op** with near-zero cost.
- **Applications own the SDK.** Only the final executable configures the SDK: which sources to listen to, sampling, processors, exporters, resource attributes.

This matters for three reasons: libraries can be instrumented without forcing a telemetry dependency on every consumer; there's exactly one place (startup) where telemetry policy is decided; and instrumentation stays vendor-neutral because exporters are an application concern.

**.NET is the special case — and the best-integrated one.** In .NET, the OpenTelemetry API for traces and metrics is **built into the runtime** in `System.Diagnostics`:

| OTel concept | .NET type | Where it lives |
|---|---|---|
| Tracer | `ActivitySource` | `System.Diagnostics.DiagnosticSource` (in the shared framework) |
| Span | `Activity` | same |
| SpanContext | `ActivityContext` | same |
| Meter / instruments | `Meter`, `Counter<T>`, `Histogram<T>`, `UpDownCounter<T>`, `Gauge<T>`, `Observable*` | `System.Diagnostics.Metrics` |
| Logger | `ILogger` (`Microsoft.Extensions.Logging`) | OTel's .NET log signal is a bridge from `ILogger` |
| Baggage | `Activity.Baggage` / OTel `Baggage` API | both exist; prefer the OTel `Baggage` API for propagation semantics |

So a .NET library author doesn't even reference an OpenTelemetry package: they create an `ActivitySource` and a `Meter`, and any listener — the OpenTelemetry SDK, Application Insights 3.x, `dotnet-counters`, `dotnet-trace` — can subscribe. ASP.NET Core, `HttpClient`, the Azure SDKs, Npgsql, EF Core, gRPC, MassTransit, NServiceBus, Orleans and many others emit this way natively. The `OpenTelemetry` NuGet packages are the **SDK** that subscribes to these sources and exports them.

One consequence trips people up: **nothing is collected from a source unless the SDK subscribes to it.** `AddSource("Contoso.Orders")` for traces and `AddMeter("Contoso.Orders")` for metrics are required (wildcards like `Contoso.*` are supported). Forgetting them is the most common "my custom spans don't show up" bug.

**The interview-grade sentence:** *"OpenTelemetry separates the API, which libraries code against and which is a no-op without an SDK, from the SDK, which only the application configures. In .NET the tracing and metrics APIs are built into System.Diagnostics — ActivitySource, Activity and Meter — and logging bridges from ILogger, so libraries emit natively with no OTel dependency and the SDK subscribes by source and meter name; forgetting AddSource or AddMeter is the classic reason custom telemetry is missing."*

---

## Concept 10 — Resource and service identity

Every signal an OpenTelemetry SDK emits carries a **resource**: a set of attributes describing *what produced it*. It's attached once per process (not per span), and it's the backbone of correlation (Concept 5).

The attributes that matter most:

| Attribute | Example | Notes |
|---|---|---|
| `service.name` | `orders-api` | **The** identity. Must be identical across all signals and stable across versions. Without it, SDKs fall back to `unknown_service:<process>` |
| `service.namespace` | `shop` | Groups services of one system |
| `service.version` | `2.3.1` or a git SHA | Enables "did the deploy cause it?" comparisons |
| `service.instance.id` | pod name or a GUID | Distinguishes replicas; mind cardinality on metrics |
| `deployment.environment.name` | `prod`, `staging` | (Renamed from `deployment.environment`; check your backend's expectations) |
| `cloud.provider`, `cloud.region`, `cloud.resource_id` | `azure`, `westeurope` | Usually added by **resource detectors** |
| `k8s.namespace.name`, `k8s.pod.name`, `k8s.deployment.name` | | Added by detectors or the Collector's `k8sattributes` processor |

Set them by code, or by the standard environment variables, which every SDK honors:

```bash
OTEL_SERVICE_NAME=orders-api
OTEL_RESOURCE_ATTRIBUTES=service.namespace=shop,service.version=2.3.1,deployment.environment.name=prod
```

**How Application Insights maps them:** Azure Monitor derives **Cloud Role Name** from `service.namespace` + `service.name` (as `[namespace].[name]`) and **Cloud Role Instance** from `service.instance.id` (or a host name). Application Map, the Performance blade and per-role filtering all depend on this. Microsoft's guidance is explicit: if two or more services send to the same Application Insights resource, you *must* set role names, or the map shows one blob.

**Design rules:**

1. **Name services by what they are, not where they run** — `orders-api`, not `aks-weu-pod-orders-7f9`.
2. **Version every deploy** — `service.version` is what makes canary comparison and "deploy markers" possible.
3. **Keep instance IDs off low-level metrics if replica counts are large and churny** — autoscaled services can mint thousands of instance IDs a day; that's a cardinality source (Concept 28).
4. **Set the resource in one shared place** — a ServiceDefaults project (Aspire's pattern) or a shared extension method — so every service gets it right.

**The interview-grade sentence:** *"Every OpenTelemetry signal carries a resource — service name, namespace, version, instance and environment, plus cloud and Kubernetes attributes from detectors — set once per process, ideally through OTEL_SERVICE_NAME and OTEL_RESOURCE_ATTRIBUTES. It's what correlates signals and what Application Insights turns into Cloud Role Name and Instance, so service names must be identical across signals, every deploy carries a version, and I watch instance IDs as a cardinality source on metrics."*

---

## Concept 11 — Semantic conventions and their stability

**Semantic conventions** are OpenTelemetry's shared vocabulary: standardized names, types, units and meanings for attributes, span names, span kinds and metrics, maintained as a versioned registry (v1.43 as of July 2026). They're why an HTTP span from ASP.NET Core, from Go's `net/http` and from a Java servlet all say `http.request.method`, `http.route`, `http.response.status_code` and `server.address` — and why a backend can draw the same latency chart for all three.

The domains you meet most in .NET systems:

| Domain | Key attributes / metrics | Stability (verify) |
|---|---|---|
| **HTTP** | `http.request.method`, `http.route`, `http.response.status_code`, `url.path`, `url.scheme`, `server.address`, `error.type`; metric `http.server.request.duration` (seconds) | **Stable since v1.23 (Nov 2023)** |
| **Database** | `db.system.name`, `db.namespace`, `db.collection.name`, `db.operation.name`, `db.query.text` (sanitized), `db.response.status_code`; metric `db.client.operation.duration` | **Stable since v1.33 (2025)** |
| **Messaging** | `messaging.system`, `messaging.destination.name`, `messaging.operation.type` (`send`, `receive`, `process`, `settle`), `messaging.message.id`, `messaging.batch.message_count` | Not yet stable at last check |
| **RPC/gRPC** | `rpc.system`, `rpc.service`, `rpc.method`, `rpc.grpc.status_code` | Check |
| **Exceptions** | `exception.type`, `exception.message`, `exception.stacktrace` (as a span event or log) | Stable |
| **Runtime (.NET)** | `dotnet.gc.collections`, `dotnet.gc.heap.total_allocated`, `dotnet.thread_pool.queue.length`, `dotnet.process.cpu.time` | Defined; emitted natively since .NET 9 |
| **GenAI** | `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens` | **Development** in 2026 |
| **Resource** | `service.*`, `deployment.*`, `cloud.*`, `k8s.*`, `host.*` | Mostly stable |

**Why stability is an architecture concern.** Dashboards, alerts, KQL queries, recording rules and SLO definitions **depend on attribute names**. When a convention changes before stabilizing — HTTP renamed `http.method` → `http.request.method`, `http.status_code` → `http.response.status_code`, `http.url` → `url.full`; database renamed `db.system` → `db.system.name`, `db.statement` → `db.query.text` — every query built on the old names silently returns nothing. Practical rules:

1. **Build durable assets on stable conventions only**, and pin instrumentation versions so names don't change under you.
2. **Use the migration switch** where instrumentation supports it: the `OTEL_SEMCONV_STABILITY_OPT_IN` environment variable (`http`, `http/dup`, `database`, `database/dup`) lets some instrumentations emit old, new or both names during a transition. `/dup` emits both, so you can migrate dashboards before dropping the old names.
3. **Follow conventions for your own telemetry.** Namespaced, lowercase, dot-separated (`contoso.order.id`, `contoso.payment.provider`); units in UCUM (`s`, `ms`, `By`, `{request}`); durations as histograms in **seconds** for anything that should line up with conventions.
4. **Know your backend's mapping.** Application Insights maps semantic-convention attributes onto its classic schema (requests, dependencies, traces, exceptions); OTLP-native ingestion stores data *with* semantic conventions. Queries differ between the two (Concept 52, 53).

**The interview-grade sentence:** *"Semantic conventions are OpenTelemetry's shared attribute vocabulary — HTTP has been stable since late 2023 and database since 2025, while messaging and GenAI are still evolving — and they matter architecturally because dashboards, alerts and SLOs are built on attribute names. So I build durable queries on stable conventions only, pin instrumentation versions, use OTEL_SEMCONV_STABILITY_OPT_IN with the dup mode during renames, and follow the same naming and unit rules for our own custom attributes."*

---

## Concept 12 — OTLP: one protocol for all signals

**OTLP (OpenTelemetry Protocol)** is the wire format between SDKs, Collectors and backends. It's defined in protobuf and carried in two ways:

| Transport | Default port | Notes |
|---|---|---|
| **OTLP/gRPC** | **4317** | HTTP/2, efficient, streaming; default in many SDKs |
| **OTLP/HTTP** | **4318** | HTTP/1.1 or 2 with protobuf (or JSON) bodies; paths `/v1/traces`, `/v1/metrics`, `/v1/logs`; friendlier to proxies, load balancers and firewalls |

One protocol for traces, metrics, logs (and now profiles) means one exporter configuration, one Collector receiver, and one firewall rule. The protocol defines **retryable** responses (e.g. gRPC `UNAVAILABLE`, HTTP 429/503 with `Retry-After`) and **partial success** (the backend accepted some items and tells you how many it rejected), so exporters can back off correctly instead of hammering an overloaded backend.

**Configuration by environment** — the same variables work across languages:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317     # or :4318 with http/protobuf
OTEL_EXPORTER_OTLP_PROTOCOL=grpc                           # grpc | http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=api-key=...                     # if the backend needs auth headers
OTEL_EXPORTER_OTLP_TIMEOUT=10000
# per-signal overrides exist: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT, ..._METRICS_..., ..._LOGS_...
OTEL_METRIC_EXPORT_INTERVAL=60000                          # ms between metric exports
```

In .NET, `UseOtlpExporter()` on the `OpenTelemetryBuilder` (from `OpenTelemetry.Exporter.OpenTelemetryProtocol`) registers OTLP for all three signals at once and reads these variables — the cleanest default for code that shouldn't know its backend. Aspire injects exactly these variables into every service it orchestrates, pointing at the Aspire dashboard (which is an OTLP receiver).

**Design notes:**

- **Prefer exporting to a local or nearby Collector** rather than straight to a remote backend from every process: the Collector batches, retries, buffers during backend outages and holds credentials (Concept 15).
- **Encrypt and authenticate** OTLP that crosses a network boundary (TLS, mTLS or bearer tokens via headers/extensions). Telemetry contains URLs, user identifiers and sometimes queries — treat it as sensitive data in transit.
- **Backends speak OTLP differently.** Azure Monitor's native OTLP ingestion requires Entra ID authentication through the Collector's Azure auth extension; the Azure Monitor *exporter* inside the distros speaks Azure Monitor's own ingestion protocol, not OTLP (Concepts 52–53).

**The interview-grade sentence:** *"OTLP is OpenTelemetry's single protobuf protocol for all signals, over gRPC on 4317 or HTTP on 4318, with defined retry and partial-success semantics so exporters back off properly. I configure it through the standard OTEL_EXPORTER_OTLP environment variables — UseOtlpExporter in .NET wires all three signals — export to a nearby Collector rather than directly to the backend, and secure it like any other channel carrying sensitive data."*

---

## Concept 13 — Context propagation: carrying the trace across processes

A distributed trace exists only because each process passes **trace context** to the next. Inside a process it lives in ambient state; between processes it travels in headers or message properties. Propagation is the most important — and most often broken — part of tracing.

**W3C Trace Context** (a W3C Recommendation) defines two HTTP headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                └─ trace-flags: 01 = sampled
             │  │                                └─ parent-id: 8-byte span ID of the caller's span
             │  └─ trace-id: 16-byte ID shared by every span in the trace
             └─ version
tracestate: contoso=t61rcWkgMzE,azure=...
             vendor-specific key/value list, propagated unchanged by others
```

- The **trace ID** is shared by the whole trace; each hop creates a new **span ID** whose parent is the incoming `parent-id`.
- The **sampled flag** carries the upstream sampling decision, so downstream services keep or drop *the same* traces (parent-based sampling — Concept 16). Without it, every service samples independently and you get fragments.
- **`tracestate`** lets vendors carry extra state (consistent-probability sampling thresholds use it, as `ot=th:...`).

**W3C Baggage** is a separate header for **application-defined key/value pairs** that should travel with the request:

```
baggage: tenant.id=contoso,user.tier=gold,canary=true
```

Baggage is propagated, *not* automatically attached to spans — you copy chosen keys onto spans or log records with a processor (Concept 63). It's powerful (tenant ID everywhere without changing every method signature) and dangerous: it's **sent to every downstream service, including third parties**, adds bytes to every call, and can be forged by clients. Never put secrets or PII in baggage; strip or validate it at trust boundaries (Concept 24).

**Propagators** implement the format: W3C Trace Context + W3C Baggage is the default; B3 (Zipkin) and Jaeger formats exist for legacy interop; Application Insights 2.x used `Request-Id` / `Correlation-Context`, which modern .NET still understands for compatibility.

**Inside a .NET process**, context lives in `Activity.Current`, which flows through `async`/`await` via `AsyncLocal<T>` (Module 15's `ExecutionContext`). That's why it "just works" across awaits — and why it can break when work escapes the execution context: `ThreadPool.UnsafeQueueUserWorkItem`, work captured by a long-lived singleton and executed later, or hand-offs through a `Channel<T>` or in-memory queue to a `BackgroundService` (Concept 64).

**Across messages**, context goes in message properties: Service Bus and Event Hubs SDKs write `Diagnostic-Id` (and `traceparent`) application properties automatically when tracing is enabled; Kafka uses message headers; CloudEvents has a distributed-tracing extension. Module 11 introduced this; Concept 21 covers how the consumer side should *use* that context.

**The interview-grade sentence:** *"Distributed tracing depends on propagation: W3C traceparent carries the trace ID, the caller's span ID and the sampled flag so every hop joins the same trace and honors the same sampling decision, tracestate carries vendor state, and W3C baggage carries application key/values like tenant ID — which go to every downstream, including third parties, so no secrets or PII. In .NET, Activity.Current flows through async/await via AsyncLocal, HTTP and the Azure messaging SDKs propagate automatically, and the gaps are hand-offs to background work, which I propagate explicitly."*

---

## Concept 14 — Four ways to instrument

| Mode | What it means | .NET examples | Strengths | Weaknesses |
|---|---|---|---|---|
| **Native instrumentation** | The library emits OTel-compatible telemetry itself | ASP.NET Core (`Microsoft.AspNetCore` activities, `Microsoft.AspNetCore.Hosting` metrics), `System.Net.Http` metrics, Azure SDKs (`Azure.*` sources), Npgsql, MassTransit, Orleans | Maintained by the library owners; accurate; no extra packages | Coverage varies; you still need to subscribe |
| **Instrumentation libraries** | A separate package that hooks a library's diagnostics and produces spans/metrics | `OpenTelemetry.Instrumentation.AspNetCore`, `.Http`, `.SqlClient`, `.GrpcNetClient`, `.EntityFrameworkCore`, `.StackExchangeRedis` | Semantic-convention-compliant output; enrichment hooks | Version coupling; some still pre-release; duplicates if combined with native |
| **Zero-code (auto) instrumentation** | An agent injects instrumentation at startup via the CLR profiler API and startup hooks | `opentelemetry-dotnet-instrumentation`; Application Insights auto-instrumentation on App Service/AKS | No code changes; good for legacy and third-party apps | Less control; startup overhead; harder to customize; another moving part |
| **Manual instrumentation** | Your code creates spans, metrics and logs | `ActivitySource.StartActivity`, `Meter.CreateCounter`, `ILogger` | Business meaning: "checkout", "price calculation", "orders placed" | Effort; consistency requires conventions |

The right answer is **all four, deliberately**:

- **Infrastructure telemetry** (HTTP in/out, database calls, cache, messaging) comes from native and library instrumentation — never hand-written.
- **Business telemetry** (domain operations, business events, business metrics like orders per minute or payment success rate) is manual — no library knows your domain.
- **Zero-code** fills gaps where you can't change code (vendor apps, legacy services) or wants a baseline across a fleet quickly.

Two rules prevent common messes: **avoid double instrumentation** (for example, enabling both native ASP.NET Core metrics and the instrumentation library's metrics, or two different HTTP client instrumentations, producing duplicate spans) — the Azure Monitor distro's docs even document "missing request telemetry" caused by an extra `OpenTelemetry.Instrumentation.AspNetCore` reference; and **check pre-release status** — the SQL client instrumentation is still pre-release (the Azure Monitor distro vendors it for that reason).

**The interview-grade sentence:** *"I combine four instrumentation modes on purpose: native instrumentation from ASP.NET Core, HttpClient and the Azure SDKs, instrumentation libraries for things like SQL, gRPC and Redis, zero-code agents only where I can't change the code, and manual instrumentation for the domain — business operations and business metrics no library can know. And I watch for double instrumentation and pre-release packages, which are the two classic sources of duplicate or missing telemetry."*

---

## Concept 15 — The OpenTelemetry Collector

The **Collector** is a standalone process (written in Go) that receives telemetry, processes it, and exports it anywhere. It's where **telemetry policy** lives — the place an architect can enforce redaction, sampling, routing and cost control without redeploying forty services.

**Components and pipelines:**

| Component | Role | Common examples |
|---|---|---|
| **Receivers** | Ingest data | `otlp` (gRPC/HTTP), `prometheus` (scrape), `filelog`, `hostmetrics`, `kubeletstats`, `k8s_events` |
| **Processors** | Transform in flight | `memory_limiter` (first!), `batch`, `resource`, `attributes`, `filter`, `transform` (OTTL), `k8sattributes`, `resourcedetection`, `tail_sampling`, `probabilistic_sampler`, `redaction` |
| **Exporters** | Send out | `otlp`, `otlphttp`, `prometheusremotewrite`, `azuremonitor`, `azuredataexplorer`, `debug`, vendor exporters |
| **Connectors** | Join pipelines — an exporter of one, a receiver of another | `spanmetrics` (RED metrics from spans), `servicegraph`, `routing`, `count` |
| **Extensions** | Non-pipeline capabilities | `health_check`, `pprof`, `zpages`, authentication (`azureauth`, `oauth2client`, `bearertokenauth`) |

```yaml
receivers:
  otlp:
    protocols: { grpc: { endpoint: 0.0.0.0:4317 }, http: { endpoint: 0.0.0.0:4318 } }

processors:
  memory_limiter: { check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 20 }
  k8sattributes: {}
  transform/redact:
    trace_statements:
      - context: span
        statements:
          - replace_pattern(attributes["url.full"], "token=[^&]+", "token=***")
          - delete_key(attributes, "http.request.header.authorization")
  tail_sampling:
    decision_wait: 10s
    policies:
      - { name: errors,  type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,    type: latency,     latency: { threshold_ms: 1000 } }
      - { name: baseline, type: probabilistic, probabilistic: { sampling_percentage: 5 } }
  batch: { send_batch_size: 8192, timeout: 5s }

connectors:
  spanmetrics: {}          # RED metrics computed from 100% of spans, before tail sampling
  forward: {}              # hands spans from the intake pipeline to the sampling pipeline

exporters:
  otlp/backend: { endpoint: tempo.observability:4317 }                                  # TLS on by default
  prometheusremotewrite: { endpoint: https://<azure-monitor-workspace>/api/v1/write }   # plus an auth extension

service:
  pipelines:
    traces/intake:  { receivers: [otlp],    processors: [memory_limiter, k8sattributes, transform/redact], exporters: [spanmetrics, forward] }
    traces/sampled: { receivers: [forward], processors: [tail_sampling, batch],                            exporters: [otlp/backend] }
    metrics:        { receivers: [otlp, spanmetrics], processors: [memory_limiter, batch],                 exporters: [prometheusremotewrite] }
```

**Deployment patterns:**

| Pattern | Where it runs | Good for |
|---|---|---|
| **Agent** | Per node (DaemonSet) or sidecar per pod/app | Local, low-latency OTLP endpoint; host and Kubernetes metadata; offloading batching and retry from apps |
| **Gateway** | Central, horizontally scaled deployment | Policy enforcement, tail sampling, routing to multiple backends, credential isolation |
| **Agent → gateway** | Both tiers | The standard at scale: agents enrich and forward, gateways decide |
| **No Collector** | SDK exports straight to the backend | Small systems, PaaS where the platform provides an agent (Container Apps managed agent, App Service auto-instrumentation) |

**Tail sampling has a topology requirement:** every span of a trace must reach the **same** gateway instance, so a load-balancing exporter tier routes by trace ID in front of the sampling tier.

**Operating the Collector:** it's production infrastructure. `memory_limiter` first in every pipeline, sized resources, horizontal scaling, its own metrics (queue size, refused spans, exporter failures) scraped and alerted, persistent queues (file storage extension) if you can't lose data during backend outages, and versions pinned — the Collector releases often (Azure Monitor's OTLP path requires v0.132.0 or later; profiles support appeared in v0.148.0), and component stability varies by component.

**Distributions** bundle components: the **core** distribution (minimal), **contrib** (everything, large attack surface), **k8s**, and vendor distributions. Production best practice is a **custom build** with the OpenTelemetry Collector Builder (`ocb`) containing only the components you use.

**The interview-grade sentence:** *"The Collector receives, processes and exports telemetry through pipelines of receivers, processors, connectors and exporters, so it's where I enforce policy without redeploying services — redaction with the transform processor, Kubernetes enrichment, spanmetrics computed from all spans before tail sampling, routing to several backends, and credential isolation. At scale it's agents per node forwarding to a gateway tier, with trace-ID-aware load balancing in front of tail sampling, memory_limiter first in every pipeline, and a custom minimal build that's monitored and pinned like any other production service."*

---

## Concept 16 — Sampling, properly

You can rarely afford to keep 100% of traces (Concept 7). **Sampling** decides which traces to keep. The design questions are *where* the decision is made, *on what information*, and *how consistently* across services.

**Head sampling — decide at the start.** The root service decides when the trace begins (before anything is known about its outcome), and the decision propagates via the sampled flag.

- **`TraceIdRatioBased(p)`** — keep a fraction `p`, decided deterministically from the trace ID, so any service applying the same ratio to the same trace ID agrees.
- **`ParentBased(root: …)`** — the default in OTel SDKs: if there's a parent, follow its sampled flag; otherwise apply the root sampler. This is what keeps traces **complete** across services.
- **Rate-limited** — keep up to N traces per second. The Azure Monitor distro and Application Insights SDK 3.x default to a **rate-limited sampler of 5 traces per second**, switchable to a fixed percentage (`SamplingRatio`) or via `OTEL_TRACES_SAMPLER=microsoft.rate_limited` / `microsoft.fixed_percentage`.
- **Pros:** cheap, simple, no buffering, the decision is made once. **Cons:** blind — it drops errors and slow requests at the same rate as boring ones.

**Tail sampling — decide at the end.** A Collector gateway buffers all spans of a trace for a decision window (say 10–30 s), then applies policies: keep all traces with errors, all traces slower than 1 s, all traces for a VIP tenant, and 5% of the rest.

- **Pros:** keeps the *interesting* traces — exactly what you need for diagnosis.
- **Cons:** memory and CPU for buffering; every span of a trace must reach the same Collector instance (load-balance by trace ID); a decision window means late spans (long async work) can miss it; the full volume must still be *exported* from every service to the gateway, so it saves backend cost, not network or SDK cost.

**Hybrid** is common at scale: modest head sampling (or none) for cost control at the edge, tail sampling at the gateway for quality.

**The rules that keep sampling honest:**

1. **Compute SLI metrics from unsampled data.** Request rate, error rate and latency histograms come from metrics instruments (or `spanmetrics` before sampling, or Application Insights standard metrics, which are pre-aggregated before sampling). Never compute an availability SLO from sampled spans unless you correct for the rate.
2. **Sample consistently.** Parent-based sampling everywhere; the same ratio sampler on the same trace ID; no service that independently re-samples a trace its parent kept. Inconsistent sampling produces broken trace trees.
3. **Record the sampling rate.** Backends can re-weight counts from sampled data only if each span knows its probability (Application Insights stores an item count per record; OTel's consistent probability sampling records a threshold in `tracestate`).
4. **Tie logs to trace sampling where useful.** Application Insights 3.x and the distros apply the trace's sampling decision to logs inside it by default (`EnableTraceBasedLogsSampler`), keeping log volume proportional and logs joinable to kept traces. .NET 9's log sampling offers a trace-based sampler too (Concept 39). Decide deliberately: error logs often deserve to be kept even when their trace wasn't.
5. **Make "always keep" explicit.** Errors, slow requests, specific tenants under investigation, and synthetic test traffic are usually worth 100% — tail policies or a custom head sampler can force them.

```csharp
// Head sampling in the OTel .NET SDK: parent-based with a 10% root ratio.
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.10))));
// Or by environment:  OTEL_TRACES_SAMPLER=parentbased_traceidratio   OTEL_TRACES_SAMPLER_ARG=0.1
```

**The interview-grade sentence:** *"Head sampling decides at the root — a trace-ID ratio or a rate limit, five traces per second by default in the Azure Monitor distro — and propagates through the sampled flag with parent-based sampling so traces stay complete, but it's blind to outcomes. Tail sampling in a Collector gateway buffers whole traces and keeps errors, slow requests and a baseline percentage, at the cost of buffering and trace-ID-affine routing. Either way I compute SLI metrics from unsampled data, sample consistently, record sampling rates so counts can be re-weighted, and decide deliberately whether logs follow their trace's decision."*

---

## Concept 17 — Distros, vendors and lock-in

A **distribution (distro)** is an opinionated bundle of the OpenTelemetry SDK, chosen instrumentations, resource detectors, defaults and an exporter for a particular backend. In .NET in 2026:

| Distro | Package / entry point | What it adds |
|---|---|---|
| **Azure Monitor OpenTelemetry Distro** | `Azure.Monitor.OpenTelemetry.AspNetCore` (1.6.0, July 2026) → `services.AddOpenTelemetry().UseAzureMonitor()` | ASP.NET Core, HttpClient and (vendored) SQL client instrumentation; Application Insights standard metrics; Azure resource detectors (App Service, VM, Container Apps); Live Metrics; rate-limited sampling; Entra auth for ingestion; offline storage and retry |
| **Microsoft OpenTelemetry Distro** | `Microsoft.OpenTelemetry` (1.0 since May 2026) → `builder.UseMicrosoftOpenTelemetry(o => o.Exporters = ExportTarget.AzureMonitor \| ExportTarget.Otlp)` | All of the above plus Azure SDK, OpenAI/Azure OpenAI, Semantic Kernel and Microsoft Agent Framework sources, Agent365 export, OTLP export alongside Azure Monitor; Microsoft positions it as the supported path for existing Application Insights customers, with an API nearly identical to the Azure Monitor distro |
| **Grafana OpenTelemetry distribution for .NET** | `Grafana.OpenTelemetry` | Grafana Cloud defaults, broad instrumentation set |
| **Vendor-neutral "distro"** | Your own shared `AddObservability()` extension over the upstream SDK + `UseOtlpExporter()` | Exactly what you choose; exports to a Collector |

**Where lock-in actually lives.** With OpenTelemetry, **instrumentation** — the expensive part, spread through every codebase — is portable. What isn't portable:

- **Queries, dashboards and alerts** (KQL vs PromQL vs TraceQL vs vendor query languages);
- **Backend-specific schema mappings** (Application Insights' `requests`/`dependencies` tables vs semantic-convention-native tables);
- **Backend-only features** (Application Insights Live Metrics, Profiler, Snapshot Debugger, smart detection; vendor APM features);
- **Distro-specific APIs**, if you call them from business code (keep them in startup).

So the architecture pattern is: **instrument with the OTel API and semantic conventions; isolate exporter and distro choices to a single startup module; optionally put a Collector in front so you can dual-export during a migration or evaluation; and accept that queries and dashboards are the migration cost.** That's a far better position than the 2010s, when changing APM vendors meant re-instrumenting everything.

**When to choose a distro vs vanilla OTel:** a distro when you're committed to that backend and want its features with minimum effort (most Azure-first .NET shops: the Azure Monitor or Microsoft distro); vanilla SDK + OTLP + Collector when you need multiple backends, multicloud, strict control over components, or a Grafana/vendor-neutral stack.

**The interview-grade sentence:** *"A distro bundles the OTel SDK with instrumentations, detectors, defaults and an exporter for one backend — on .NET that's the Azure Monitor distro with UseAzureMonitor, or the new Microsoft OpenTelemetry Distro, 1.0 since May 2026, which adds OTLP, Azure SDK and AI-agent sources with nearly the same API. With OpenTelemetry the instrumentation is portable; lock-in moves to queries, dashboards, alerts and backend-only features, so I keep distro choices in one startup module, optionally put a Collector in front to dual-export, and budget the query migration rather than a re-instrumentation."*

---
# Part C — Distributed tracing, precisely

## Concept 18 — The span model

The idea goes back to Google's **Dapper** paper (2010): give every request a trace ID, record each unit of work as a span with a parent, and reassemble the tree afterwards. Zipkin, Jaeger, X-Ray, Application Insights' operation model and OpenTelemetry all descend from it.

An OpenTelemetry **span** has:

| Field | Meaning | .NET (`Activity`) |
|---|---|---|
| **Trace ID** | 16 bytes, shared by every span in the trace | `Activity.TraceId` |
| **Span ID** | 8 bytes, unique to this span | `Activity.SpanId` |
| **Parent span ID** | The span that caused this one (absent for the root) | `Activity.ParentSpanId` |
| **Name** | Low-cardinality description of the operation | `Activity.DisplayName` / `OperationName` |
| **Kind** | Server, client, producer, consumer, internal | `Activity.Kind` (`ActivityKind`) |
| **Start / end time** | Wall-clock start and duration | `StartTimeUtc`, `Duration` |
| **Attributes** | Key/value metadata | `Activity.SetTag(...)` / `TagObjects` |
| **Events** | Timestamped annotations inside the span (e.g. an exception, a retry) | `Activity.AddEvent(...)` |
| **Links** | References to other spans (in this or other traces) that are related but not parents | `ActivityLink` (set at creation) |
| **Status** | Unset, Ok or Error, with an optional description | `Activity.SetStatus(...)` |
| **Trace flags / state** | Sampled flag; vendor state | `ActivityTraceFlags`, `TraceStateString` |

A **trace** is the set of spans sharing a trace ID; parent IDs make it a tree. A request that enters `api-gateway`, calls `orders-api`, which queries Cosmos DB twice and publishes to Service Bus, looks like:

```
trace 4bf9…4736
└─ SERVER  GET /checkout                     api-gateway      320 ms
   └─ CLIENT  POST orders-api/orders                          290 ms
      └─ SERVER  POST /orders                orders-api       280 ms
         ├─ INTERNAL  PriceCalculation                         40 ms
         ├─ CLIENT    Cosmos ReadItem orders                    6 ms
         ├─ CLIENT    Cosmos Upsert orders                     12 ms
         └─ PRODUCER  send orders-events                        9 ms
```

Note the **client/server pairing**: the caller's CLIENT span and the callee's SERVER span describe the same call from both sides. The gap between them is network time, queueing and connection setup — a useful diagnostic in its own right.

**The interview-grade sentence:** *"A span is a timed operation with a trace ID shared by the whole request, its own span ID and a parent ID, a low-cardinality name, a kind, attributes, timestamped events, links to related spans and a status; a trace is the tree those parent IDs form. In .NET it's System.Diagnostics.Activity, and the paired client and server spans for each hop let me see network and queueing time as the gap between them."*

---

## Concept 19 — Span kinds and names

**Kinds** tell backends how to draw and aggregate spans:

| Kind | Meaning | Example |
|---|---|---|
| **SERVER** | Handles a synchronous remote request | ASP.NET Core request, gRPC service method |
| **CLIENT** | Makes a synchronous remote call | `HttpClient`, SQL, Cosmos, Redis |
| **PRODUCER** | Creates a message for asynchronous processing | Service Bus send, Event Hubs send, Kafka produce |
| **CONSUMER** | Processes (or receives) a message | Service Bus processor handler, Event Hubs processor |
| **INTERNAL** | In-process work with no remote boundary | Domain operation, CPU-heavy calculation |

Application Insights maps SERVER and CONSUMER spans to **requests** and CLIENT, PRODUCER and INTERNAL spans to **dependencies** — which is why getting kinds right decides whether a message handler shows as a request with its own success rate and duration.

**Names must be low-cardinality.** Backends group spans by name to compute per-operation latency and error rates; a unique name per request makes that grouping impossible and is a cardinality bomb for span-derived metrics.

| ❌ Bad name | ✅ Good name | Put the detail in |
|---|---|---|
| `GET /orders/8f3a-42` | `GET /orders/{id}` | `url.path`, `contoso.order.id` |
| `SELECT * FROM Orders WHERE Id = 42` | `SELECT orders` (or `db.operation.name db.collection.name`) | `db.query.text` (sanitized) |
| `Process order 42 for contoso` | `ProcessOrder` | attributes |
| `send to orders-events-tenant-contoso` | `send orders-events` | `messaging.destination.name` (if templated, the template) |

The conventions give the patterns: HTTP server spans are named `{method} {http.route}`; HTTP client spans `{method}` (plus a template if known); database spans `{db.operation.name} {db.collection.name}`; messaging spans `{messaging.operation.name} {destination}`.

**How many spans?** Enough to explain latency, not one per method. Good manual spans mark **meaningful units of work** — a domain operation, a call to an external system without instrumentation, a CPU-heavy step, a retry loop as a whole. A span per loop iteration over 10,000 items is a cost and a UI problem; use one span with an attribute (`item.count`) or span events for notable iterations.

**The interview-grade sentence:** *"Span kinds — server, client, producer, consumer and internal — tell the backend how to aggregate and draw spans, and in Application Insights server and consumer become requests while the rest become dependencies, so kinds decide whether a message handler gets its own success rate. Names must be low-cardinality — GET /orders/{id}, not the raw path — with the identifiers in attributes, and I add manual spans for meaningful units of work, not per method or per loop iteration."*

---

## Concept 20 — Status, errors and exceptions

**Span status** has three values: **Unset** (the default — the operation completed and the instrumentation has no opinion), **Error**, and **Ok** (an explicit override meaning "this is definitely a success", rarely needed). The rules from the HTTP conventions are worth knowing exactly, because they decide error rates:

| Situation | Server span | Client span |
|---|---|---|
| 2xx / 3xx | Unset | Unset |
| **4xx** | **Unset** — the *client* made a mistake; the server worked correctly | **Error** — from the caller's perspective, the call failed |
| 5xx | Error | Error |
| Exception, timeout, connection failure | Error, with `error.type` | Error, with `error.type` |

That asymmetry matters for SLOs: a burst of 404s from a scanner shouldn't burn the API's availability budget, but a client that gets 404 from a dependency it expected to find has a real error. Some 4xx *are* server problems in disguise — a 429 from your own rate limiter during an overload, or a 409 from a concurrency bug — so SLI definitions should say explicitly which status codes count as bad (Concept 44).

**`error.type`** is the stable, low-cardinality attribute for *what kind* of failure occurred — an exception type name (`System.TimeoutException`), an HTTP status code as a string, or a domain value (`payment_declined`). It's the right dimension for failure breakdowns on spans and metrics (unlike `exception.message`, which is unbounded).

**Exceptions** are recorded as a span **event** named `exception` with `exception.type`, `exception.message` and `exception.stacktrace`. In .NET:

```csharp
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);   // description: keep short; no PII
    activity?.AddException(ex);                                   // .NET 9+: records the OTel "exception" event
    activity?.SetTag("error.type", ex.GetType().FullName);
    throw;
}
```

(On older targets, the OTel SDK's `activity.RecordException(ex)` extension does the same.) Instrumentation libraries can record exceptions for you (`RecordException = true` on the ASP.NET Core and HttpClient trace options). Note that Application Insights shows exceptions from span events *and* from `ILogger.LogError(ex, ...)` — record an exception **once**, at the boundary that handles it, or you'll count it twice (Concept 37).

**Handled vs unhandled.** An exception that's caught and recovered from (a retry that eventually succeeded, a fallback) is not a failed span — at most a span event (`retry`, `fallback_used`) on an Unset span. Marking every caught exception as Error inflates error rates and destroys their meaning.

**The interview-grade sentence:** *"Span status is Unset, Error or rarely Ok, and the HTTP rules are asymmetric: a 4xx is Unset on the server span because the server behaved correctly but Error on the client span, while 5xx and exceptions are errors on both — so I define in the SLI which status codes count as bad, including 429s from my own overload protection. I record exceptions once, as the OTel exception event at the boundary that handles them, use error.type as the low-cardinality failure dimension, and don't mark recovered failures like successful retries as errors."*

---

## Concept 21 — Tracing asynchronous and messaging flows

Synchronous calls form a clean tree. Asynchronous messaging (Module 11) forces three decisions.

**1. Parent or link?** When a consumer processes a message, should its span be a **child** of the producer's span (same trace) or start a **new trace** with a **link** to the producer?

| | **Child (same trace)** | **New trace + link** |
|---|---|---|
| Visualization | One end-to-end trace from HTTP request through queue to consumer | Separate traces, navigable via the link |
| Good when | One message → one consumer, short delay (seconds), end-to-end latency matters | Fan-out to many consumers, batches, long delays (minutes to days), replays |
| Problems avoided | — | Gigantic traces that never "finish"; traces kept open for hours; one request's trace polluted by hundreds of consumers |
| Sampling | Inherits the producer's decision | Consumer can sample independently |

OpenTelemetry's messaging conventions model a **PRODUCER** span (`send` or `create`), and on the consumer side a **`process`** span (CONSUMER kind) that typically **links** to the creation context and may be parented to it for single-message delivery; **`receive`** spans represent pulling from the broker. The .NET Azure Service Bus and Event Hubs SDKs follow this: producer spans inject `Diagnostic-Id`/`traceparent` into message properties; processor spans use the context, and batch receive operations carry **links to every message** in the batch.

**2. Batches.** A consumer that processes 100 messages in one batch can't have 100 parents. The correct shape is one span for the batch operation with **100 links** (one per message's creation context) and `messaging.batch.message_count = 100`, plus optional per-message child spans if each message is processed individually.

**3. Trace boundaries for long-running flows.** A saga that spans days (Module 12, Module 24) shouldn't be one trace. Use a trace per step, linked, and a **business correlation ID** (order ID, saga ID) as an attribute on every span and log record, so you can query "everything about order 42" across traces. Trace IDs are for causal chains of seconds to minutes; business IDs are for lifecycles.

**What end-to-end latency means in async systems.** The trace gives you *processing* latency per hop; it doesn't directly give you **time in queue**. Measure it explicitly: the consumer computes `now − enqueued time` (Service Bus `EnqueuedTime`, Event Hubs `EnqueuedTime`) and records it as a histogram (`messaging.process.duration` covers processing; a custom `contoso.queue.wait_time` or lag metric covers waiting). That's the SLI for asynchronous systems (Concept 49).

**The interview-grade sentence:** *"For messaging I use producer spans that inject trace context into message properties — the Azure Service Bus and Event Hubs SDKs do it as Diagnostic-Id and traceparent — and consumer process spans that are children only for one-to-one, short-delay delivery and otherwise start a new trace with a link, which is mandatory for batches: one span, one link per message. Long-running sagas get a trace per step plus a business correlation ID on everything, and I measure time-in-queue explicitly from the enqueued timestamp, because traces show processing time, not waiting time."*

---

## Concept 22 — Reading traces: critical path and the shapes of common problems

A trace waterfall is a diagnostic picture with recognizable shapes. Being able to read them aloud is a strong interview move.

**The critical path** is the chain of spans that determines the end-to-end duration — the work that, if made faster, would make the request faster. Parallel calls off the critical path don't matter for latency until they become the longest branch. Optimizing a span that isn't on the critical path wastes effort.

**Shapes worth recognizing:**

| Shape in the waterfall | Likely cause | Next step |
|---|---|---|
| **A ladder** — dozens of short, identical sequential DB spans | **N+1 queries** (Module 19) | Batch, include, or project; check the ORM query |
| **A staircase** of calls that could be parallel | Sequential `await`s on independent calls | `Task.WhenAll` (bounded) |
| **Repeated identical client spans with growing gaps** | **Retries with backoff** (Module 25) | Check the dependency's health and the retry policy; count attempts per call |
| **A long gap between client and server span start** | Connection pool exhaustion, DNS, TLS handshake, thread-pool starvation, network | Pool metrics, `dotnet.thread_pool.queue.length`, `http.client.open_connections` |
| **A long server span with no children** | CPU work, lock contention, sync-over-async, GC pause, or un-instrumented I/O | Profile; add a span; check GC metrics |
| **A gap inside a span before its first child** | Middleware, authentication, deserialization, waiting for a lock or semaphore | Middleware spans; check concurrency limiters |
| **One slow call among many fast ones of the same kind** | Hot partition, cold cache, slow replica, specific tenant | Compare attributes (partition, tenant, region) across slow vs fast spans |
| **Trace ends abruptly / orphan spans** | Broken propagation or sampling inconsistency | Concept 24 |

**From one trace to many.** A single trace is an anecdote. The senior move is to **aggregate**: compare the attribute distributions of slow traces vs normal traces ("94% of traces above 2 s have `cosmos.partition_key_range_id = 3`"). Application Insights' performance blade, Honeycomb's BubbleUp, Grafana Tempo's TraceQL metrics and similar features exist for exactly this comparison. Without them, a KQL query grouping slow dependency spans by attribute does the same job.

**The interview-grade sentence:** *"I read traces for the critical path — the chain that sets end-to-end latency — and for recognizable shapes: a ladder of identical DB spans is N+1, a staircase is sequential awaits that could be parallel, repeated client spans with growing gaps are retries, a gap between client and server spans is pool exhaustion or thread-pool starvation, and a long childless span is CPU, locks or GC. Then I move from one trace to many by comparing attribute distributions of slow and normal traces, because a single trace is an anecdote."*

---

## Concept 23 — What goes on a span

Because spans tolerate high cardinality (Concept 4), they're where per-request context belongs. The discipline is choosing context that **explains behavior** without leaking data or exploding size.

**Put on spans:**

- **Business identifiers** that you'll filter by during an incident: `contoso.tenant.id`, `contoso.order.id`, `contoso.customer.tier`, `contoso.feature_flag.checkout_v2 = true`.
- **Decision context**: which branch was taken, cache hit or miss, which provider was chosen, how many items were processed, the retry attempt number.
- **Dependency context** from conventions: `db.namespace`, `db.collection.name`, `server.address`, `messaging.destination.name`, partition or region where the SDK exposes it.
- **Deployment context** on the resource, not on each span: version, environment, region.

**Keep off spans:**

- **Secrets and credentials** — tokens, API keys, connection strings, `Authorization` headers, SAS query strings in URLs (the Microsoft distro removed its URL query redaction override in favor of OTel defaults; check that query strings are redacted in your stack).
- **PII you don't need** — email, names, addresses, raw IPs (where regulated), free-text user input. If you need a user correlation, use an opaque internal ID.
- **Payloads** — request/response bodies, full documents, large SQL with literal values (use parameterized/sanitized `db.query.text`).
- **Unbounded blobs** — giant exception messages, serialized objects.

**Limits:** the OTel SDKs cap attributes per span (default **128**), events per span (128), links (128), and optionally attribute value length (`OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT`). Backends have their own limits (Application Insights truncates property values and caps custom properties). Exceeding them silently drops data — another reason to be deliberate.

**Naming your attributes:** namespaced (`contoso.*` or your domain's prefix), lowercase, dot-separated, consistent types (don't let `order.id` be a string in one service and a number in another), and **reuse semantic conventions** where they exist rather than inventing `http_status`.

**The interview-grade sentence:** *"On spans I put what explains behavior and what I'll filter by in an incident — tenant, order and feature-flag identifiers, branch decisions, cache hits, retry attempts and dependency details from the conventions — under a namespaced, consistently typed scheme; I keep secrets, tokens, payloads, unneeded PII and unbounded blobs off them, and I remember the SDK's 128-attribute default and backend truncation limits silently drop anything beyond them."*

---

## Concept 24 — Broken traces and trust boundaries

A trace is only as good as its weakest propagation hop. **Broken traces** — orphan spans, traces that stop at a service, two halves of one request in two traces — are the most common tracing complaint. The usual causes:

1. **A hop that doesn't propagate.** A reverse proxy or gateway that strips unknown headers, a custom HTTP client (raw sockets, an old SDK), a message transport whose producer doesn't write context, a legacy service on an old SDK using only the `Request-Id` format.
2. **Context lost in process.** Work handed to a `BackgroundService` via a `Channel<T>`, a timer callback, `Task.Run` from a singleton constructed outside the request, `ThreadPool.UnsafeQueueUserWorkItem`, or `ExecutionContext.SuppressFlow` (Concept 64).
3. **Inconsistent sampling.** A downstream service that ignores the parent's sampled flag and drops spans the parent kept — or keeps spans whose parent was dropped (orphans).
4. **Unsubscribed sources.** A service emits spans from an `ActivitySource` the SDK isn't listening to; the trace jumps over it.
5. **Clock skew.** Spans from different hosts with skewed clocks appear to start before their parents. Backends usually correct small skew using client/server pairs; large skew means NTP problems.
6. **Different backends per service.** Half the services export to one backend, half to another — each sees half the trace. A Collector that routes everything to the same place fixes it.

**Trust boundaries — the security side of propagation.** Your public API receives `traceparent`, `tracestate` and `baggage` from the internet. Accepting them blindly lets a client:

- choose trace IDs (collisions, or forcing correlation with other users' traces in a shared backend);
- force sampling on (`sampled = 1` on every request) to inflate your telemetry bill — a cost attack;
- inject baggage that downstream services trust (`tenant.id=other-tenant`, `role=admin` — never let baggage drive authorization);
- send oversized headers.

**Rules at the edge:** at the first service you own (or the API gateway / Front Door / APIM), **start a new trace** for untrusted callers — keep the incoming trace ID as an attribute or link if you want to correlate with a partner's telemetry — **don't honor an external sampled flag**, **drop or allow-list baggage**, and cap header sizes. Between your own services, inside the trust boundary, propagate everything normally. ASP.NET Core and OTel let you customize this (a custom propagator, or middleware that clears `Activity.Current` parent for external requests); APIM and Front Door policies can strip or rewrite headers.

**The interview-grade sentence:** *"Traces break at hops that don't propagate — proxies stripping headers, old SDKs, background hand-offs — at inconsistent sampling decisions, at unsubscribed sources, under clock skew and when services export to different backends. And propagation is a trust boundary: at the edge I start a new trace for external callers, keep their trace ID only as a link or attribute, ignore their sampled flag so nobody can force my bill up, and drop or allow-list baggage so it never carries authority; inside my own boundary I propagate everything."*

---
# Part D — Metrics, precisely

## Concept 25 — The metrics data model

OpenTelemetry metrics have three layers: **instruments** you record into, **measurements** with attributes, and **aggregations** that turn measurements into exported **metric streams** (time series).

**Instruments** — choose by what the number *means*:

| Instrument | Semantics | Typical aggregation | .NET type | Examples |
|---|---|---|---|---|
| **Counter** | Monotonically increasing count or sum | Sum | `Counter<T>` | requests, orders placed, bytes sent |
| **UpDownCounter** | Sum that can go up and down | Sum (non-monotonic) | `UpDownCounter<T>` | active requests, items in an in-memory queue |
| **Histogram** | Distribution of values | Histogram (buckets, sum, count, min, max) | `Histogram<T>` | request duration, payload size, batch size |
| **Gauge** | A current value you set, not summable | Last value | `Gauge<T>` (.NET 9+) | temperature, configured limit, cache size reported on change |
| **Observable Counter / UpDownCounter / Gauge** | Asynchronous: a callback reports the value at collection time | Sum / last value | `ObservableCounter<T>`, `ObservableUpDownCounter<T>`, `ObservableGauge<T>` | CPU time, memory, connection-pool size, queue depth read from the broker |

Choosing wrong produces wrong math. Recording request duration in a Counter gives you a total that's meaningless; recording "current queue depth" as a Counter double-counts; recording request counts as a Gauge loses everything between collections. **If you'll ever want a percentile, it's a Histogram. If summing across instances makes sense, it's a counter. If it doesn't (a temperature, a configured limit), it's a gauge.**

**Synchronous vs asynchronous.** Synchronous instruments are called in your code at the moment of the event (`counter.Add(1)`), cost a few nanoseconds to tens of nanoseconds, and are thread-safe. Observable instruments are called by the SDK at export time (every `OTEL_METRIC_EXPORT_INTERVAL`, 60 s by default) — ideal for values you can read cheaply but don't want to push on every change.

**Measurements and attributes.** Every recording carries attributes (`TagList` in .NET): `requests.Add(1, new("http.route", route), new("http.response.status_code", 200))`. The SDK aggregates measurements with identical attribute sets into one **metric point**; each distinct set is a time series (Concept 4).

**Views** let the application change aggregation without touching instrumentation: rename a metric, drop attributes, change histogram buckets, or drop an instrument entirely (Concept 28).

**The interview-grade sentence:** *"I pick the instrument by meaning: counters for things that only accumulate, up-down counters for sums that go both ways like active requests, histograms for anything I'll want a percentile of, gauges for values that can't be summed, and observable instruments when the SDK should read a value at collection time. Each distinct attribute set is a separate time series, and views let the application rename, re-bucket or drop dimensions without changing instrumentation."*

---

## Concept 26 — Temporality: cumulative vs delta

**Aggregation temporality** is how a sum or histogram is reported over time:

| | **Cumulative** | **Delta** |
|---|---|---|
| What each export contains | Total since the process (or series) started | Change since the last export |
| Example (requests) | 1,000 → 1,450 → 1,900 | 1,000 → 450 → 450 |
| Survives a lost export | ✅ the next value includes it | ❌ that interval is lost |
| Handles restarts | Needs **reset detection** (value drops → new start time) | Natural |
| Memory in the SDK | Must keep every series ever seen | Can forget series after export |
| Who prefers it | **Prometheus**, most OSS backends | **Azure Monitor**, Datadog, some SaaS backends |

Why it matters to an architect:

1. **Exporter defaults differ.** The OTLP exporter can be configured with a temporality preference (`OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative|delta|lowmemory`). The Azure Monitor exporter uses delta. A Collector can convert delta to cumulative (`deltatocumulative` processor) for Prometheus-style backends.
2. **Rates come from differences.** With cumulative counters, backends compute `rate()` from differences between samples and must handle resets when pods restart — PromQL's `rate()` does this, which is why you never graph a raw cumulative counter.
3. **Memory and cardinality interact.** With cumulative temporality the SDK keeps state for every attribute combination it has ever seen (until restart), so a short-lived high-cardinality attribute leaks memory; delta lets the SDK drop idle series.
4. **Gauges have no temporality** — they're last values, reported as they are.

**The interview-grade sentence:** *"Temporality is whether sums are reported as running totals or per-interval changes: Prometheus wants cumulative, which survives lost exports but needs reset handling and keeps state for every series ever seen; Azure Monitor and several SaaS backends want delta, which loses an interval if an export is lost but lets the SDK forget idle series. I set the exporter's temporality preference to match the backend, convert in the Collector when needed, and always derive rates rather than graphing raw counters."*

---

## Concept 27 — Histograms and percentiles

Latency is a **distribution**, and the mean hides the tail where users suffer (Module 5). Percentiles — p50, p95, p99, p99.9 — describe the tail, but they come with mathematical traps that interviewers love.

**Trap 1 — you can't average percentiles.** If pod A's p99 is 100 ms and pod B's p99 is 900 ms, the fleet p99 is *not* 500 ms; it depends on both distributions and their traffic. The same holds across time: the average of 60 per-minute p99s is not the hourly p99. Any dashboard that averages percentiles is showing a number with no meaning.

**Trap 2 — client-side summaries can't be re-aggregated.** A *summary* (pre-computed quantiles per instance, Prometheus' `Summary` type) has the same problem. **Histograms can be aggregated**: buckets from many instances and many intervals simply add, and a percentile is estimated from the merged buckets. That's why OpenTelemetry and Prometheus best practice is **record histograms, compute percentiles at query time.**

**Explicit-bucket histograms** count measurements into fixed boundaries. The OTel default boundaries are `[0, 5, 10, 25, 50, 75, 100, 250, 500, 750, 1000, 2500, 5000, 7500, 10000]` — designed for **milliseconds**. The .NET HTTP metrics record **seconds** and the runtime supplies seconds-appropriate boundaries (from 5 ms up to 10 s) via `InstrumentAdvice` — if you record your own durations in seconds with default buckets, everything lands in the first bucket. Rules:

- **Choose bucket boundaries around your SLO thresholds.** If the SLO is "99% of requests under 300 ms", you need a boundary *at* 300 ms (or very near it), because a percentile estimate is interpolated within a bucket and the "fraction under 300 ms" is exact only at a boundary.
- **Bucket resolution sets percentile accuracy.** With buckets at 250 and 500 ms, any p99 between them is an interpolation.

**Exponential (base-2) histograms** pick buckets automatically on a logarithmic scale with a configurable maximum bucket count (160 by default), giving bounded relative error across many orders of magnitude without hand-tuning. They're supported in the OTel .NET SDK (via a view: `new Base2ExponentialBucketHistogramConfiguration()`) and by backends such as Prometheus native histograms; check backend support before enabling — Azure Monitor's exporter path handles explicit buckets.

**Trap 3 — coordinated omission.** A load generator that waits for each response before sending the next under-reports latency during stalls, because it stops *issuing* requests exactly when the system is slow (Gil Tene's classic talk). In production, measure at the server *and* the client, and remember that a stalled server records nothing while it's stalled.

**Trap 4 — percentiles per low-traffic series are noisy.** A p99 over 50 requests is the second-slowest request. Report percentiles at an aggregation level with enough traffic, or use "fraction of requests slower than X" — which is also exactly how latency SLIs are defined (Concept 44).

**The interview-grade sentence:** *"Latency is a distribution, so I record histograms and compute percentiles at query time: percentiles can't be averaged across instances or across time, and pre-computed summaries can't be merged, but histogram buckets simply add. Bucket boundaries must bracket the SLO threshold — and the OTel defaults are millisecond-shaped, so durations in seconds need seconds-shaped buckets or exponential histograms — and I watch for coordinated omission and for percentiles over too little traffic, where 'fraction slower than 300 ms' is the more honest number."*

---

## Concept 28 — Cardinality budgets and views

Concept 4 gave the arithmetic; operating it needs **budgets** and **tools**.

**A cardinality budget per metric** is a design artifact: list the attributes, their maximum distinct values, and the product. `http.server.request.duration` in a service with 30 routes × 5 methods × 12 status codes × 3 regions ≈ 5,400 combinations, each a histogram — acceptable. Adding `tenant.id` with 8,000 tenants turns it into 43 million — not acceptable. Write the budget down in the same place you document the metric.

**Where cardinality sneaks in:**

| Source | Example | Fix |
|---|---|---|
| Raw paths instead of route templates | `/orders/8f3a…` | Use `http.route` (ASP.NET Core does) |
| Status codes with free text | `error.message` as an attribute | Use `error.type` |
| Instance IDs on autoscaled pods | thousands of `service.instance.id` values per day | Keep on resource; aggregate it away in dashboards; drop in views for very high-churn fleets |
| Tenant or user IDs | `tenant.id` with 50,000 values | Tier, top-N + `other`, or move to spans |
| Dynamic queue or topic names | per-tenant queues | Template or bucket |
| Exceptions with message text | `exception.message` | `exception.type` only |

**Views** in the OTel .NET SDK reshape metrics at the application:

```csharp
builder.Services.AddOpenTelemetry().WithMetrics(m => m
    .AddMeter("Contoso.Orders")
    // Keep only bounded attributes on a noisy instrument.
    .AddView("contoso.orders.processed",
        new MetricStreamConfiguration { TagKeys = ["contoso.order.channel", "contoso.order.outcome"] })
    // SLO-aligned buckets (seconds) for our own latency histogram.
    .AddView("contoso.checkout.duration",
        new ExplicitBucketHistogramConfiguration { Boundaries = [0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5] })
    // Drop an instrument we never query.
    .AddView("http.server.active_requests", MetricStreamConfiguration.Drop)
    // Raise (or lower) the per-stream cardinality cap deliberately; default is 2000.
    .AddView("contoso.cache.lookups", new MetricStreamConfiguration { CardinalityLimit = 500 }));
```

**When the cap is hit**, the .NET SDK folds new combinations into an overflow point tagged `otel.metric.overflow = true`. Alert on its presence; it means a design assumption broke.

**Downstream controls**: the Collector can drop or aggregate attributes (`transform`, `metricstransform`, `filter`), and backends can enforce series limits. But the cheapest cardinality is the one the SDK never creates.

**The interview-grade sentence:** *"Every metric gets a written cardinality budget — the product of its attributes' distinct values — and I watch the usual leaks: raw paths instead of route templates, error messages instead of error.type, tenant or user IDs, dynamic queue names and pod churn. Views in the SDK keep only bounded tag keys, set SLO-aligned buckets, drop unused instruments and set per-stream cardinality limits, and I alert on the otel.metric.overflow series, because hitting the cap means a design assumption broke."*

---

## Concept 29 — RED, USE and the golden signals

Three checklists answer "what should I measure?" They overlap on purpose.

**RED — for request-driven services** (Tom Wilkie):

- **Rate** — requests per second;
- **Errors** — failed requests per second (or the error ratio);
- **Duration** — the latency distribution (histogram).

Every service endpoint, every consumer and every outbound dependency gets RED. With OpenTelemetry's HTTP conventions, one histogram (`http.server.request.duration`) supplies all three: count = rate, count filtered by status = errors, buckets = duration.

**USE — for resources** (Brendan Gregg):

- **Utilization** — how busy the resource is (CPU %, pool in use / pool size);
- **Saturation** — how much work is waiting (run-queue length, thread-pool queue length, connection-pool waiters, queue depth);
- **Errors** — resource errors (timeouts acquiring a connection, OOM, disk errors).

USE applies to CPU, memory, disks and network, but also to **logical resources**: thread pool, connection pools, semaphores and rate limiters, Cosmos RU per partition, Service Bus MU CPU, consumer lag.

**The four golden signals** (Google SRE book): **latency**, **traffic**, **errors**, **saturation** — essentially RED plus the most important saturation signal. Note the SRE book's subtlety: **separate the latency of successful and failed requests** — a fast error is not good performance, and a slow error is worse than a fast one.

**Saturation is the leading indicator.** Errors and latency tell you users are already hurting; saturation tells you they're about to. For a .NET service the saturation signals worth alerting on (as tickets, or as pages when tied to SLO burn):

| Resource | Signal | Source |
|---|---|---|
| Thread pool | `dotnet.thread_pool.queue.length` rising, thread count climbing | `System.Runtime` meter (Module 15) |
| GC | `dotnet.gc.pause.time` share of wall time, gen 2 collections, heap growth | `System.Runtime` (Module 14) |
| Kestrel | queued connections, rejected connections, TLS handshake time | `Microsoft.AspNetCore.Server.Kestrel` |
| Outbound HTTP | `http.client.open_connections` (idle vs active), `http.client.connection.duration`, request queue time | `System.Net.Http` |
| Rate limiters / bulkheads | queued and rejected leases | `Microsoft.AspNetCore.RateLimiting`, Polly telemetry (Module 25) |
| Database | pool wait time, Cosmos normalized RU per partition, 429 rate | provider metrics, Azure platform metrics (Module 27) |
| Messaging | queue depth, age of oldest message, consumer lag | broker metrics (Module 27) |

**The interview-grade sentence:** *"I use RED for every endpoint, consumer and dependency — rate, errors and a duration histogram, which the OTel HTTP metric supplies on its own — USE for resources, including logical ones like the thread pool, connection pools, rate limiters and Cosmos partitions, and the golden signals as the SRE summary, keeping success and failure latency separate. Saturation is my leading indicator: thread-pool queue length, GC pause share, connection-pool waits and queue age warn me before errors and latency do."*

---

## Concept 30 — Exemplars

An **exemplar** is a sample measurement attached to an aggregated metric point, carrying the **trace ID and span ID** of the request that produced it (plus its value and timestamp). It answers the most common question after "the p99 spiked": *show me one of those slow requests.*

How it works in OpenTelemetry:

1. When a histogram or counter records a measurement *inside a sampled span*, the SDK's **exemplar reservoir** may keep it (by default, one per histogram bucket per export interval, chosen with reservoir sampling).
2. The exemplar is exported with the metric point.
3. The backend shows exemplars as dots on the latency chart; clicking one opens the trace.

In the .NET SDK, exemplars are configured with an **exemplar filter**: `TraceBased` (keep only measurements recorded inside sampled traces — the sensible default when enabled), `AlwaysOn` or `AlwaysOff`:

```csharp
.WithMetrics(m => m.SetExemplarFilter(ExemplarFilterType.TraceBased))
```

Requirements and caveats: the measurement must happen **inside an active, sampled `Activity`** (built-in ASP.NET Core and HttpClient histograms are), the exporter and backend must support exemplars (OTLP → Prometheus/Mimir/Grafana, and others; check your Azure path), and the trace must still exist in the trace backend — an exemplar pointing to a trace dropped by tail sampling is a dead link. Prefer tail-sampling policies that **keep slow and failing traces**, which are exactly the ones exemplars on high buckets point to.

**The interview-grade sentence:** *"Exemplars attach a sampled measurement with its trace and span IDs to a metric point — one per histogram bucket by default — so a click on the p99 spike opens a trace that was actually in that bucket. They need the measurement recorded inside a sampled activity, an exporter and backend that carry them, and a sampling policy that keeps slow and failing traces, or the exemplar links to a trace that no longer exists."*

---

## Concept 31 — Prometheus and the Azure Monitor workspace

**Prometheus** is the reference open-source metrics system, and its model has become the lingua franca for metrics queries:

- **Pull-based**: Prometheus **scrapes** an HTTP `/metrics` endpoint on each target at an interval (typically 15–60 s). Targets are discovered (Kubernetes service discovery). OTel's Prometheus exporter for ASP.NET Core can expose such an endpoint; alternatively, OTLP push into a Prometheus-compatible store via remote write or native OTLP ingestion.
- **Data model**: metric name + labels (attributes) + float samples; counters, gauges, histograms (`_bucket`, `_sum`, `_count` series; now also native histograms) and summaries.
- **Naming**: OTel names are translated — `http.server.request.duration` (unit `s`) becomes `http_server_request_duration_seconds_bucket`, `_sum`, `_count`.
- **PromQL** — the query language you should be able to write a few lines of:

```promql
# Request rate per route (5-minute window)
sum by (http_route) (rate(http_server_request_duration_seconds_count[5m]))

# Error ratio (5xx) — the availability SLI's complement
sum(rate(http_server_request_duration_seconds_count{http_response_status_code=~"5.."}[5m]))
  / sum(rate(http_server_request_duration_seconds_count[5m]))

# p99 latency from histogram buckets, merged across instances
histogram_quantile(0.99, sum by (le) (rate(http_server_request_duration_seconds_bucket[5m])))

# Fraction of requests faster than 300 ms — the latency SLI (needs a 0.3 bucket boundary)
sum(rate(http_server_request_duration_seconds_bucket{le="0.3"}[5m]))
  / sum(rate(http_server_request_duration_seconds_count[5m]))
```

- **Recording rules** precompute expensive expressions (SLI ratios over 5m, 30m, 1h, 6h windows) into new series; **alerting rules** evaluate conditions and fire alerts. SLO tooling such as **Sloth** and **Pyrra** generates both from an SLO definition (Concept 47).

**On Azure**, the managed equivalent is **Azure Monitor managed service for Prometheus**, which stores data in an **Azure Monitor workspace** (a distinct resource from a Log Analytics workspace):

- Collection from AKS via the managed Prometheus add-on (the Azure Monitor metrics add-on), from other sources by remote write, and — via OTLP ingestion — OpenTelemetry metrics land here too.
- Query with PromQL; visualize in **Dashboards with Grafana** in the portal or **Azure Managed Grafana**.
- **Prometheus rule groups** are Azure resources holding recording and alerting rules, routed to action groups.
- Pricing is per samples ingested and queried (check current rates); there's no per-series license, but series × scrape frequency drives samples.

**The interview-grade sentence:** *"Prometheus is the reference metrics model — pull-based scraping or remote write, labels, histograms as bucket, sum and count series, and PromQL — and I can write rate, error-ratio, histogram_quantile and fraction-under-threshold queries, with recording rules to precompute SLI windows and alerting rules on top. On Azure that's managed Prometheus in an Azure Monitor workspace — separate from Log Analytics — fed by the AKS add-on, remote write or OTLP, visualized in portal Grafana dashboards or Azure Managed Grafana, with Prometheus rule groups for recording and alerting."*

---

## Concept 32 — .NET's built-in metrics

Since .NET 8, the runtime and ASP.NET Core emit **semantic-convention metrics natively** through `System.Diagnostics.Metrics`. You subscribe by meter name; you don't need instrumentation packages for them.

| Meter | Key instruments | What they answer |
|---|---|---|
| `Microsoft.AspNetCore.Hosting` | `http.server.request.duration` (histogram, seconds), `http.server.active_requests` | RED for every endpoint, tagged with `http.route`, method, status, `error.type` |
| `Microsoft.AspNetCore.Server.Kestrel` | `kestrel.active_connections`, `kestrel.queued_connections`, `kestrel.connection.duration`, `kestrel.rejected_connections`, `kestrel.tls_handshake.duration` | Connection-level saturation |
| `Microsoft.AspNetCore.Routing`, `.Diagnostics`, `.RateLimiting`, `.Http.Connections` (SignalR) | match attempts, handled exceptions, rate-limiter leases and queues, SignalR connections | Middleware behavior |
| `System.Net.Http` | `http.client.request.duration`, `http.client.open_connections`, `http.client.connection.duration`, `http.client.request.time_in_queue` | Outbound RED and pool saturation |
| `System.Net.NameResolution` | `dns.lookup.duration` | DNS problems |
| `System.Runtime` (.NET 9+) | `dotnet.gc.collections`, `dotnet.gc.heap.total_allocated`, `dotnet.gc.pause.time`, `dotnet.gc.last_collection.heap.size`, `dotnet.thread_pool.thread.count`, `dotnet.thread_pool.queue.length`, `dotnet.thread_pool.work_item.count`, `dotnet.monitor.lock_contentions`, `dotnet.exceptions`, `dotnet.process.cpu.time`, `dotnet.process.memory.working_set` | GC, thread-pool, lock and exception health (Modules 14–15) |
| `Microsoft.EntityFrameworkCore` | (as counters/metrics, depending on version) queries, SaveChanges, compiled-query cache hits | ORM behavior (Module 19) |
| Azure SDKs, Npgsql, `Microsoft.Extensions.Diagnostics.ResourceMonitoring`, Polly | client metrics, container CPU/memory utilization, resilience events | Dependencies and resilience |

On .NET 8, the `System.Runtime` meter isn't present; the `OpenTelemetry.Instrumentation.Runtime` package supplies equivalent metrics (with older names). On .NET 9+, prefer the built-in meter.

```csharp
builder.Services.AddOpenTelemetry().WithMetrics(m => m
    .AddMeter("Microsoft.AspNetCore.Hosting", "Microsoft.AspNetCore.Server.Kestrel",
              "System.Net.Http", "System.Net.NameResolution", "System.Runtime")
    .AddMeter("Contoso.*"));
// Or use the instrumentation helpers: .AddAspNetCoreInstrumentation().AddHttpClientInstrumentation().AddRuntimeInstrumentation()
```

You can inspect any of these live without deploying anything: `dotnet-counters monitor -n Orders.Api --counters Microsoft.AspNetCore.Hosting,System.Runtime` (Concept 65).

**Enriching built-in metrics.** ASP.NET Core lets you add tags to `http.server.request.duration` per request through `IHttpMetricsTagsFeature` (for example a bounded `contoso.tenant.tier`) — use it sparingly, cardinality rules apply.

**The interview-grade sentence:** *"Since .NET 8, ASP.NET Core, Kestrel and HttpClient emit semantic-convention metrics natively — http.server.request.duration in seconds gives me RED per route, Kestrel and the HttpClient connection metrics give me connection saturation — and since .NET 9 the System.Runtime meter adds GC, thread-pool, lock-contention and exception metrics, so I subscribe to those meters instead of adding instrumentation packages, inspect them live with dotnet-counters, and add bounded tags through IHttpMetricsTagsFeature when I really need them."*

---
# Part E — Structured logging, precisely

## Concept 33 — Structured logging from first principles

A plain-text log line is written for a human reading one line. A **structured** log record is written for a machine querying millions: it separates the **message template** from the **properties** that fill it.

```csharp
// ❌ Interpolated: one unique string per call; properties lost; allocates even when the level is off.
logger.LogInformation($"Order {orderId} placed by {customerId} for {total:C}");

// ✅ Template: one template for all calls; OrderId, CustomerId and Total stored as typed properties.
logger.LogInformation("Order {OrderId} placed by {CustomerId} for {Total}", orderId, customerId, total);
```

What the template gives you:

1. **Grouping by event type.** All occurrences share one template (`Order {OrderId} placed by …`), so "how often did this happen?" is a `count by template`. Interpolated strings are all unique.
2. **Queryable properties.** `OrderId = 42` is a field you can filter on, not a substring you regex for. In Application Insights / Log Analytics, properties land in `customDimensions` (classic schema) or attribute columns (OTLP schema).
3. **Type preservation.** `Total` stays a number; `Elapsed` stays a duration — range queries work.
4. **Cheap when disabled.** With templates, the formatting work can be skipped if the level is off (fully, with the source generator — Concept 35).

Conventions that keep structured logs useful:

- **PascalCase property names** in templates (the .NET convention), consistent across services for the same concept (`OrderId`, never `orderId` in one service and `Order_ID` in another). Or adopt semantic-convention names for shared concepts.
- **Templates are constants.** Never build the template string dynamically.
- **One event, one template.** Don't reuse "Processing failed" for ten different failures; give each its own event (and an **EventId** — Concept 35).
- **Messages read well as text, too.** Someone will read them in a console during an incident.

`ILogger`'s message templates follow the **Message Templates** specification also used by Serilog (with Serilog's `@` destructuring and `$` stringification operators as Serilog-specific extensions).

**The interview-grade sentence:** *"A structured log record separates a constant message template from typed properties, so every occurrence of an event shares one template I can count, properties are fields I can filter and range-query rather than substrings, and formatting can be skipped when the level is off. String interpolation in log calls destroys all of that, so I enforce constant templates, one template per event, and consistent property names across services."*

---

## Concept 34 — The `ILogger` pipeline

`Microsoft.Extensions.Logging` is the standard .NET logging abstraction, and OpenTelemetry's .NET log signal is a **bridge from it** — you keep writing to `ILogger`; OTel is one more provider.

The moving parts:

| Part | Role |
|---|---|
| **`ILogger<T>` / category** | Every logger has a category, usually the full type name (`Contoso.Orders.Api.OrdersController`); filtering is by category prefix |
| **Log level** | `Trace` < `Debug` < `Information` < `Warning` < `Error` < `Critical` (< `None`) |
| **Providers** | Sinks: Console, Debug, EventSource, EventLog, **OpenTelemetry**, third-party (Serilog, NLog) |
| **Filters** | Minimum level per category per provider, from configuration (`Logging:LogLevel`) or code |
| **Scopes** | Ambient key/values attached to every record within a `using (logger.BeginScope(...))` block |
| **`ILoggerFactory`** | Builds loggers; one per application |

Configuration is layered and hierarchical:

```jsonc
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",              // framework noise down
      "Microsoft.EntityFrameworkCore.Database.Command": "Warning",
      "System.Net.Http.HttpClient": "Warning",
      "Contoso": "Information"
    },
    "OpenTelemetry": {                                // per-provider override
      "LogLevel": { "Default": "Information", "Contoso.Orders.Pricing": "Debug" }
    }
  }
}
```

Points that matter in production:

- **Filtering happens before the provider does any work**, so a disabled category costs very little — *if* the call site doesn't allocate (Concept 35).
- **Levels can be changed at runtime** by reloading configuration (App Configuration with refresh, or Kubernetes config reloads) — the operational switch for "turn Debug on for `Contoso.Orders.Pricing` for ten minutes."
- **Scopes are off by default in the OTel provider**; enable `IncludeScopes` deliberately, and use `List<KeyValuePair<string, object?>>` as scope state for performance. In Azure Monitor, scope values become custom properties.
- **Provider order and duplication**: the default host adds Console, Debug, EventSource and (on Windows) EventLog. In containers, Console logs are often *also* collected by the platform (Container Apps, AKS Container Insights) — the same record ingested twice into two tables is a common, silent cost (Concept 70). Decide which path is the system of record.
- **Serilog or NLog?** Both plug in as providers (or replace the factory). They bring rich sinks and enrichers, but with OpenTelemetry handling export, many teams now keep plain `ILogger` + OTel and drop the third-party framework. If you keep Serilog, use its OpenTelemetry sink or route through `ILogger` so trace correlation and export stay consistent.

**The interview-grade sentence:** *"ILogger is a pipeline of categories, levels, per-provider filters, scopes and providers, and OpenTelemetry is just another provider bridging those records into OTel logs. I keep framework categories at Warning, tune levels per category and per provider through configuration that can reload at runtime, enable scopes in the OTel provider deliberately, and make sure the same records aren't ingested twice — once through OTel and once through the platform's console log collection."*

---

## Concept 35 — High-performance logging

Logging sits on hot paths; done carelessly, it's a measurable share of CPU and allocations (Module 17). The costs of a naive `logger.LogInformation("Order {OrderId} …", orderId)`:

1. A `params object[]` array allocation for the arguments;
2. **boxing** of value-type arguments (`int`, `Guid`, `decimal`);
3. template parsing (cached, but still a lookup);
4. all of the above **even when the level is disabled**, because the arguments are evaluated before the call.

**The fix is the compile-time logging source generator** (`[LoggerMessage]`, .NET 6+):

```csharp
public static partial class OrderLog
{
    [LoggerMessage(EventId = 1201, Level = LogLevel.Information,
        Message = "Order {OrderId} placed by {CustomerId} for {Total}")]
    public static partial void OrderPlaced(this ILogger logger, Guid orderId, string customerId, decimal total);

    [LoggerMessage(EventId = 1202, Level = LogLevel.Warning,
        Message = "Payment declined for order {OrderId}: {DeclineCode}")]
    public static partial void PaymentDeclined(this ILogger logger, Guid orderId, string declineCode);

    [LoggerMessage(EventId = 1299, Level = LogLevel.Error, Message = "Order {OrderId} processing failed")]
    public static partial void OrderFailed(this ILogger logger, Exception exception, Guid orderId);
}

// Call site: strongly typed, no params array, no boxing, IsEnabled checked first.
logger.OrderPlaced(order.Id, order.CustomerId, order.Total);
```

What it gives you: generated code that checks `IsEnabled` first, uses strongly typed generic state (no boxing, no array), caches the parsed template, and gives each event a stable **EventId** and **EventName** — which become queryable fields and a contract for alerts ("alert when EventId 1299 count > 0").

**Related tools:**

- **`[LogProperties]`** (from `Microsoft.Extensions.Telemetry.Abstractions`) logs selected properties of an object as individual fields, with **`[TagProvider]`** for custom extraction and data-classification attributes for redaction (Concept 38).
- **`logger.IsEnabled(level)`** guards around *expensive argument computation* (serializing an object, computing a summary) even with the generator.
- **Avoid string interpolation and `ToString()` in arguments**; pass the values and let the formatter run only when needed.
- **Avoid logging large objects** — serializing a 50 KB DTO per request is a performance *and* cost problem.

A useful benchmark result to quote (orders of magnitude, measure your own): a disabled-level source-generated log call costs a few nanoseconds and zero allocations; a disabled-level classic call with value-type arguments allocates the params array and boxes.

**The interview-grade sentence:** *"Classic ILogger calls allocate a params array and box value types even when the level is off, so on hot paths I use the LoggerMessage source generator: strongly typed methods that check IsEnabled first, don't allocate or box, cache the template and give every event a stable EventId I can alert on. LogProperties handles structured objects with classification for redaction, and I guard genuinely expensive argument computation with IsEnabled."*

---

## Concept 36 — Correlating logs with traces

A log record is far more valuable when it's attached to the request that produced it. With OpenTelemetry in .NET, **this is automatic**: the OTel logging provider stamps each record with the `TraceId`, `SpanId` and trace flags of `Activity.Current` at the moment of logging. In Application Insights, the record's `operation_Id` (trace ID) and `operation_ParentId` (span ID) let the transaction view show logs inline with the spans.

```csharp
builder.Logging.AddOpenTelemetry(o =>
{
    o.IncludeFormattedMessage = true;     // keep the rendered text as well as the template
    o.IncludeScopes = true;               // scope key/values as attributes
    o.ParseStateValues = true;            // template properties as attributes
});
```

Even without OpenTelemetry, `LoggerFactoryOptions.ActivityTrackingOptions` (`TraceId | SpanId | ParentId | Baggage | Tags`) adds these as scope values for other providers (console JSON formatter, Serilog).

**What breaks correlation:**

1. **No active `Activity`** — background jobs, startup code, timers and message handlers that don't start a span. Fix: start an `Activity` for each unit of background work (one per message, per job run) so its logs have a trace (Concept 64).
2. **A different provider path** — logs shipped through a file or console scraper without trace fields. Use the JSON console formatter with scopes, or send logs through OTel.
3. **Sampling mismatch** — with trace-based log sampling, logs follow their trace; without it, you can have a log whose trace was dropped (the trace ID then points nowhere). That's acceptable for error logs; decide on purpose (Concept 39).

**Business correlation, too.** Trace IDs connect a request; **business IDs** connect a lifecycle. Put `OrderId`, `TenantId` and `CorrelationId` (for sagas and long flows) in scopes or as properties consistently, so "everything about order 42 this week" is one query across traces.

**The interview-grade sentence:** *"With the OpenTelemetry logging provider, every ILogger record written inside an Activity carries its trace and span IDs automatically — which is what lets Application Insights show logs inline in a transaction — and ActivityTrackingOptions does the same for other providers. Correlation breaks where there's no active Activity, so I start one per message, job or background unit of work, and I add business IDs like order and tenant as scope properties so lifecycles longer than one trace are still one query."*

---

## Concept 37 — What to log, and at which level

The hardest logging question isn't how but **what**. A useful policy:

| Level | Meaning | Production default | Examples |
|---|---|---|---|
| **Critical** | The process or a whole capability can't continue | On, alert | Can't connect to the database at startup; configuration invalid; data corruption detected |
| **Error** | An operation failed and was not recovered; someone may need to act | On, counted, often alerted via metrics | Unhandled exception at a boundary; message dead-lettered; payment call failed after retries |
| **Warning** | Something unexpected happened but was handled; worth knowing in aggregate | On | Retry succeeded after failures; fallback used; deprecated API called; request validation rejected odd input |
| **Information** | Significant business or lifecycle events | On, **selectively** | Order placed; job started/completed with counts; configuration loaded; circuit opened/closed |
| **Debug** | Developer diagnostics | Off; enable per category on demand | Branch decisions, intermediate values |
| **Trace** | Very verbose, possibly sensitive | Off | Payload dumps in dev only |

Principles that separate seniors from everyone else:

1. **Log at boundaries and decisions, not everywhere.** Entry/exit of every method is what traces are for. Log what traces can't express: business events, decisions, unusual conditions, and failures with context.
2. **Log an exception once, where it's handled.** "Log and rethrow" at five layers produces five copies of one failure (and five Application Insights exceptions). Either handle (log + recover) or propagate (don't log). The outermost boundary — the ASP.NET Core exception handler, the message processor's error handler — logs unhandled failures.
3. **Errors mean action.** If an `Error` is routinely ignored, it's mislabeled (should be `Warning`) or there's a bug to fix. Error-level noise trains everyone to ignore errors.
4. **Don't log what metrics count better.** "Request completed in 23 ms" per request at `Information` duplicates the request span and the duration histogram at a far higher price.
5. **Include the context needed to act** — identifiers, the operation, the dependency, the outcome — but not payloads or secrets (Concept 38).
6. **Startup and configuration** deserve `Information`: the effective configuration (sanitized), versions, feature flags — the first thing you need when "it works in staging but not prod."

**The interview-grade sentence:** *"My logging policy is to log at boundaries and decisions — business events, handled anomalies, unrecovered failures and sanitized startup configuration — not method entry and exit, which traces cover, and not per-request timings, which metrics count far more cheaply. An exception is logged once, where it's handled, never logged-and-rethrown at every layer; Error means someone should act, so routinely ignored errors are either mislabeled or bugs; and Debug stays off by default, switchable per category at runtime."*

---

## Concept 38 — Sensitive data and redaction

Telemetry is a **data store with broad access**: dozens of engineers, support staff and vendors can query it, and it's retained for weeks to years. Personal data, secrets and payment data that leak into logs and spans create GDPR, PCI DSS and HIPAA exposure, and leaked tokens are a direct security incident.

**Defense in depth, from the code outward:**

1. **Don't emit it.** Use opaque internal IDs instead of emails or names; never log request bodies, headers, connection strings or tokens; sanitize SQL (parameterized `db.query.text` without literals).
2. **Classify and redact in-process** with `Microsoft.Extensions.Compliance`:
   - define **data classifications** (`[PersonalData]`, `[SensitiveData]` — your own taxonomy);
   - annotate model properties or logging-method parameters with them;
   - register **redactors** (erasing, star, HMAC-based for stable pseudonyms that still allow correlation) per classification;
   - enable redaction in logging (`builder.Logging.EnableRedaction()`), so source-generated log methods and `[LogProperties]` redact automatically.

```csharp
public sealed class PersonalDataAttribute() : DataClassificationAttribute(ContosoTaxonomy.PersonalData);

public static partial class CustomerLog
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Customer {CustomerId} updated email to {Email}")]
    public static partial void EmailChanged(this ILogger logger, string customerId, [PersonalData] string email);
}

builder.Logging.EnableRedaction();
builder.Services.AddRedaction(r =>
{
    r.SetRedactor<ErasingRedactor>(new DataClassificationSet(ContosoTaxonomy.PersonalData));
    // or SetHmacRedactor(...) for a keyed, stable pseudonym that still lets you correlate
});
```

3. **Scrub in the pipeline** — Collector `transform`/`redaction` processors remove or mask attributes (`http.request.header.authorization`, query-string tokens), and Azure Monitor **data collection rule transformations** can drop or mask columns at ingestion. These catch what code missed; they aren't a substitute for not emitting.
4. **Control access** — table-level and resource-context RBAC in Log Analytics, separate workspaces for regulated data, private links for ingestion and query, and retention matched to policy.
5. **Make deletion possible** — GDPR erasure requests reach telemetry too; Log Analytics supports a purge API, but it's slow and operationally heavy. Not storing personal data is far cheaper than purging it.

**URL and query strings** deserve a specific mention: SAS tokens, OAuth codes and API keys in query strings end up in `url.full` and server logs. Ensure your instrumentation redacts query strings (OTel .NET HttpClient and ASP.NET Core instrumentation redact query values by default in recent versions; verify for your versions and any custom spans).

**The interview-grade sentence:** *"Telemetry is a broadly readable, long-retained data store, so personal data and secrets mustn't reach it: first don't emit them — opaque IDs, no bodies, headers or literal SQL — then classify and redact in-process with Microsoft.Extensions.Compliance data classifications and erasing or HMAC redactors wired into the logging generator, then scrub in the Collector or with Azure Monitor ingestion-time transformations as a backstop, and finally control access and retention, because purging personal data from a workspace later is slow and painful."*

---

## Concept 39 — Volume control: filtering, sampling, buffering and table plans

Logs are usually the largest telemetry bill (Concept 7). Five tools, in the order you should reach for them:

**1. Filtering by level and category** (Concept 34) — the bluntest and cheapest. Framework categories at `Warning`, your code at `Information`, `Debug` only on demand.

**2. Log sampling** (.NET 9+, `Microsoft.Extensions.Telemetry`): keep a fraction of records matching rules, e.g. 10% of `Information` from a chatty category, 100% of `Warning` and above. A **trace-based** sampler keeps logs whose trace was sampled, aligning log and trace retention. The Azure Monitor distro and Application Insights SDK 3.x apply trace-based log sampling by default (`EnableTraceBasedLogsSampler`).

**3. Log buffering** (.NET 9+): the most interesting new tool. Logs below a threshold are held in memory instead of emitted; if something interesting happens — an error, an exception, a slow request — you **flush** the buffer and get the full detailed context *for that operation only*; otherwise the buffer is discarded.

- **Global buffering** (`builder.Logging.AddGlobalBuffer(LogLevel.Information)`) with an explicit flush via `GlobalLogBuffer`.
- **Per-request buffering** in ASP.NET Core (`builder.Logging.AddPerIncomingRequestBuffer(LogLevel.Information)` from `Microsoft.AspNetCore.Diagnostics.Middleware`), flushed through `PerRequestLogBuffer` — e.g. in the exception handler — so a failed request ships its Debug/Information trail and a successful one ships nothing below Warning.
- Caveats: buffered records are lost if the process dies before flushing; buffers have size limits; and it's a newer API — check maturity for your version.

```csharp
builder.Logging.AddPerIncomingRequestBuffer(LogLevel.Information);   // hold Info and below per request

app.UseExceptionHandler(handler => handler.Run(async ctx =>
{
    ctx.RequestServices.GetRequiredService<PerRequestLogBuffer>().Flush();   // emit the full trail for failures
    await Results.Problem().ExecuteAsync(ctx);
}));
```

**4. Pipeline filtering** — Collector `filter` processors and Azure Monitor **DCR transformations** drop noisy records (health-check logs, known benign warnings) before they're billed. On Azure, filtering more than 50% of incoming data through ingestion-time transformations can incur a log processing charge — check the pricing page.

**5. Table plans and retention** (Concept 58): route verbose, rarely queried logs to **Basic** (~$0.50/GB) or **Auxiliary** (~$0.05/GB) tables instead of **Analytics** (~$2.30/GB), and set retention per table. Basic and Auxiliary have query charges and reduced query capabilities, so they suit "keep for occasional investigation or compliance," not dashboards and alerts.

**The interview-grade sentence:** *"I control log volume in layers: level and category filters first; .NET 9 log sampling — including trace-based sampling so logs follow kept traces — for chatty categories; .NET 9 log buffering, especially per-request buffering that holds Debug and Information and flushes the full trail only when a request fails; pipeline filtering in the Collector or Azure Monitor ingestion transformations; and finally table plans, with verbose logs in Basic or Auxiliary at a fraction of the Analytics price and per-table retention."*

---

## Concept 40 — Logs, events and audit trails

Three things that look alike and must be designed apart:

| | **Diagnostic logs** | **Business / product events** | **Audit trail** |
|---|---|---|---|
| Purpose | Help engineers understand and fix the system | Measure the product: funnels, usage, conversions | Prove who did what, when — for security, compliance, disputes |
| Consumers | Engineers, on-call | Product, analytics, data science | Security, compliance, auditors, legal |
| Completeness | Best effort; sampled and filtered | High; often analytically corrected | **Must be complete** — no sampling, no filtering, no loss |
| Integrity | Not required | Moderate | **Tamper-evident**, append-only, access-controlled |
| Retention | Days to weeks (months for some) | Months to years | Years, per regulation |
| Store | Observability backend | Data platform (Fabric, Event Hubs → lake), product analytics | Dedicated store: append-only table, immutable blob (WORM), Microsoft Purview / Sentinel for security events |
| Schema | Evolving | Versioned contract | Strict contract |

The mistakes this table prevents:

- **Using diagnostic logs as an audit trail.** Logs are sampled, filtered, dropped under backpressure, editable by admins and expire. An auditor asking "show me every change to this customer's bank details" needs a dedicated, complete, tamper-evident record — written **transactionally** with the change (an outbox or an append-only table in the same transaction — Module 11), not a fire-and-forget `LogInformation`.
- **Using logs for product analytics.** Product events need a stable schema, completeness and joinability with business data; parsing them out of logs is fragile. Emit them deliberately (an event stream, Module 27), not as log side effects.
- **Using custom events in Application Insights for everything.** Application Insights *custom events* (`microsoft.custom_event.name` on a log record with the OTel distro, or `TrackEvent` in SDK 3.x) are useful for lightweight product telemetry and feature usage, but they're subject to sampling and telemetry retention — fine for "how many people used the new filter," not for billing or compliance.

**The interview-grade sentence:** *"I keep three record types apart: diagnostic logs for engineers, which can be sampled, filtered and short-lived; product events with versioned schemas and high completeness, sent to the data platform; and audit trails that must be complete, tamper-evident and retained for years, written transactionally with the change into an append-only or immutable store — never reconstructed from diagnostic logs, which are sampled, droppable and expire."*

---
# Part F — SLIs, SLOs and error budgets

## Concept 41 — Why SLOs exist

Every reliability conversation without SLOs collapses into one of two failure modes: **"it should never go down"** (an infinitely expensive, unachievable goal that makes every incident a crisis and every release a fight), or **"it seems fine"** (no shared definition of fine, so reliability erodes until customers leave). Service level objectives replace both with a number everyone agreed to.

The reasoning, from first principles:

1. **Users can't tell the difference between very reliable and perfectly reliable.** Their device, network, ISP and browser fail far more often than 0.01% of the time; the difference between 99.99% and 99.999% of *your* service is invisible to them.
2. **Each extra nine costs roughly an order of magnitude more** — redundancy, multi-region, slower release processes, more on-call, more testing (Module 13). The marginal cost rises steeply while the marginal user benefit flattens.
3. **Reliability competes with velocity.** Every change is a risk; freezing changes maximizes short-term reliability and kills the product.
4. **So reliability is a feature with a target**, chosen where users are happy and the business can afford it — and the gap between the target and 100% is a **budget** to spend on change.

That last point is the heart of SRE as Google describes it: the **error budget** turns the reliability-vs-velocity argument into arithmetic. If the service is well inside its budget, ship faster and take risks. If the budget is exhausted, slow down and invest in reliability. Neither side "wins" by opinion.

SLOs also do something for observability specifically: they **decide what to page on**. Hundreds of metrics exist; a handful of SLIs express what users experience; alerts on SLO burn are the ones that justify waking a human (Concept 47).

**The interview-grade sentence:** *"SLOs replace 'never go down' and 'seems fine' with an agreed number: users can't perceive the difference between very and perfectly reliable, each extra nine costs about an order of magnitude more, and reliability competes with velocity — so we pick the target where users are happy and the business can pay, treat the gap to 100% as an error budget to spend on change, and page on burning that budget rather than on hundreds of internal metrics."*

---

## Concept 42 — SLI, SLO, SLA and error budget

Four terms with precise meanings:

**SLI — service level indicator.** A quantitative measure of service behavior *as experienced by users*. The most useful form is a **ratio of good events to valid events**:

```
SLI = good events / valid events        (expressed as a percentage, 0–100%)
```

- *availability SLI* = requests answered successfully / valid requests;
- *latency SLI* = requests answered in under 300 ms / valid requests;
- *freshness SLI* = reads that returned data less than 1 minute old / reads.

Good-over-valid has two advantages: it's always between 0 and 100% (so all SLIs read the same way), and **"valid" lets you exclude what shouldn't count** — health checks, synthetic probes (or count them separately), requests rejected for authentication, clients sending malformed input.

**SLO — service level objective.** A **target value for an SLI over a time window**: *"99.9% of valid requests to the checkout API succeed, measured over a rolling 28 days."* It's an internal engineering and business goal.

**SLA — service level agreement.** A **contract** with consequences (credits, refunds, penalties) if a level isn't met. SLAs are business and legal documents; they should be **looser than the SLO**, so the team gets warned and acts long before the contract is breached. Azure's own SLAs (99.95%, 99.99% and so on, per service and configuration) are the *inputs* to your design (Module 13), not your SLOs.

**Error budget** = 1 − SLO, over the SLO window:

```
SLO 99.9% over 28 days → budget 0.1% of valid events
10,000,000 valid requests in the window → 10,000 may fail
```

The budget is what you spend on deploys, experiments, planned maintenance, dependency failures and plain bad luck. **Burn rate** is how fast you're spending it relative to "exactly on budget" (Concept 47).

**The interview-grade sentence:** *"An SLI is good events over valid events as users experience them — always a percentage, with 'valid' excluding things like health checks and malformed requests; an SLO is a target for it over a window, like 99.9% of valid checkout requests succeed over 28 days; an SLA is a contract with penalties that must be looser than the SLO; and the error budget is one minus the SLO — at 99.9% and ten million requests, ten thousand may fail — which we spend on change and measure with burn rate."*

---

## Concept 43 — Choosing SLIs

Start from **user journeys**, not from components: "a customer searches the catalog," "a customer checks out," "a merchant receives payout reports," "a device's telemetry appears on the dashboard." For each journey, ask what users would consider *broken* or *bad*. The SLI menu (from Google's SRE workbook, adapted):

| Service type | SLI kinds | Example |
|---|---|---|
| **Request/response** (APIs, web) | **Availability**, **latency**, quality (degraded responses) | % of checkout requests succeeding; % served under 400 ms; % of search results not served from fallback |
| **Data processing / pipelines** | **Freshness**, **correctness**, **coverage**, throughput | % of records processed within 5 minutes of arrival; % of outputs passing validation; % of input partitions processed by the deadline |
| **Storage** | **Durability**, availability, latency | % of written objects readable later; % of reads succeeding |
| **Asynchronous / event-driven** | **End-to-end latency** (event time to effect), **age of oldest message**, DLQ rate | % of orders whose confirmation email is sent within 2 minutes (Concept 49) |

**Where to measure** — closer to the user is more accurate, farther is cheaper and easier:

| Measurement point | Pros | Cons |
|---|---|---|
| **Client / RUM** (browser, mobile SDK) | Closest to the real experience, includes network | Noisy, lossy, privacy constraints, hard to alert on |
| **Edge / load balancer** (Front Door, Application Gateway, APIM logs) | Sees requests your service never processed (crashes, timeouts at the edge) | Coarser; fewer business dimensions |
| **Server** (`http.server.request.duration`) | Rich, cheap, already instrumented | Misses requests that never arrived or died before the handler |
| **Synthetic probes** | Detects outages with zero traffic; tests full journeys | Not real users; small sample |

A common senior answer: **server-side SLIs from metrics as the primary**, **edge metrics to catch what the server can't see**, and **synthetics for low-traffic paths** and outside-in verification.

**Choosing well:**

- **Few SLIs per journey** — usually one availability and one latency. More SLIs dilute attention.
- **Prefer symptoms users feel** — success and latency — over internal causes (CPU, queue depth). Causes are for dashboards and tickets (Concept 48).
- **Per-dependency SLIs** for platform teams and critical third parties, to know whose budget an outage consumed.
- **Critical vs non-critical paths** — checkout and login deserve stricter SLOs than "recommended products."

**The interview-grade sentence:** *"I choose SLIs from user journeys: availability and latency for request paths, freshness, correctness and coverage for pipelines, durability for storage, and event-to-effect latency and oldest-message age for asynchronous flows — usually one availability and one latency SLI per journey. I measure server-side from metrics as the primary, use edge metrics from Front Door or APIM to catch requests that never reached the service, add synthetics for low-traffic paths, and keep internal causes like CPU out of SLIs."*

---

## Concept 44 — Writing an SLO precisely

An SLO that two people read differently isn't an SLO. Specify each part:

**1. The SLI implementation** — the exact query or metric:

```
Availability (checkout-api):
  valid = POST /checkout requests from external clients, excluding status 400, 401, 403, 404, 422
          and requests with header X-Synthetic: true
  good  = valid requests with status < 500 AND not 429 from our own limiter
          AND completed (no client-observed timeout at the edge)
Latency (checkout-api):
  good  = valid requests whose server duration ≤ 0.4 s     (histogram bucket boundary exactly at 0.4)
```

Status-code choices are product decisions: is a 429 *your* failure (your capacity) or the client's (abusive caller)? Is a 422 a client error or a symptom of your broken validation? Write it down.

**2. The target** — e.g. 99.9% availability, 99% latency-under-400-ms. Set it from **historical performance and user expectations**, not aspiration: if the service has achieved 99.7% for six months, a 99.99% SLO just produces permanent "violated" status that everyone ignores. Start achievable, tighten as you improve. Multiple latency thresholds are common: *"90% under 200 ms and 99% under 1 s."*

**3. The window**:

| Window | Pros | Cons |
|---|---|---|
| **Rolling (e.g. 28 or 30 days)** | Continuous; matches user memory; no reset cliff | Old incidents linger until they roll off |
| **Calendar (month, quarter)** | Aligns with business reporting and SLAs | Budget resets abruptly; "it's the 29th, spend it" incentives |

28 days is popular because it always contains four of each weekday, so weekly traffic patterns don't skew it.

**4. Request-based or time-based ("windows-based")**:

- **Request-based:** good requests / valid requests over the window. Weights by traffic: an outage at peak costs more budget than one at 3 a.m. — usually what users experience.
- **Time-based:** fraction of minutes (or 5-minute windows) that were "good" (e.g. error ratio < 1% in that minute). Treats every minute equally; useful for low-traffic services where one failed request shouldn't be a bad hour, and closer to how many SLAs are written.

**5. Exclusions and ownership** — planned maintenance (if users were informed), dependency failures (usually *not* excluded — users don't care whose fault it is), the owning team, and the review cadence.

**The interview-grade sentence:** *"A precise SLO specifies the SLI implementation — which requests are valid, which status codes count as bad, including whether my own 429s count — a target grounded in historical performance rather than aspiration, latency thresholds aligned exactly with histogram buckets, the window — usually a rolling 28 days, which always contains four of each weekday — whether it's request-based, weighting outages by traffic, or time-based for low-traffic services, and explicit exclusions, an owner and a review cadence."*

---

## Concept 45 — Error budget arithmetic

**Nines to time** (30-day window = 43,200 minutes):

| SLO | Error budget | Downtime-equivalent per 30 days | Per day |
|---|---|---|---|
| 99% | 1% | 7 h 12 min | 14.4 min |
| 99.5% | 0.5% | 3 h 36 min | 7.2 min |
| 99.9% | 0.1% | 43.2 min | 1.44 min |
| 99.95% | 0.05% | 21.6 min | 43 s |
| 99.99% | 0.01% | 4.3 min | 8.6 s |
| 99.999% | 0.001% | 26 s | 0.86 s |

Two sanity checks to say aloud: **at 99.99%, a single slow manual rollback consumes the month**, so four nines requires automated detection *and* automated mitigation (Module 13's canaries and rollbacks); and **at 99.999%, human response is impossible** — the budget is shorter than paging someone.

**Requests, not just minutes.** With a request-based SLO, budget is a count: 99.9% of 10 M requests = 10,000 failures. A 10-minute total outage at 400 req/s burns 240,000 requests — **24× the monthly budget** — while a 10-minute outage at 3 a.m. at 20 req/s burns 12,000 (1.2×). This is why request-based budgets are traffic-weighted.

**Partial degradation counts proportionally.** An incident where 5% of requests fail for 2 hours at 300 req/s burns 0.05 × 300 × 7,200 = 108,000 requests — over ten times a 10 M-request budget at 99.9%, though no dashboard showed "down."

**Composition across dependencies** (Module 13):

- **Serial dependencies multiply**: a request that needs API (99.95%) → Cosmos (99.99%) → payment provider (99.9%) has at best 0.9995 × 0.9999 × 0.999 ≈ **99.84%** — so a 99.9% SLO on this journey is already impossible unless you remove the payment provider from the critical path (async capture, retries with a queue, a fallback).
- **Redundancy in parallel**: two independent regions at 99.9% each, with perfect failover, give 1 − (0.001)² = 99.9999% for that layer — in theory. Correlated failures and failover time make reality much worse; budget the failover time explicitly.
- **Your SLO can't exceed the product of your hard dependencies' availability** — the first sanity check of any proposed target.

**The interview-grade sentence:** *"At thirty days, 99.9% is 43 minutes and 99.99% is about four minutes, which means four nines needs automated detection and rollback and five nines rules out human response. Request-based budgets are traffic-weighted — ten minutes down at peak can be twenty times the monthly budget, and five percent errors for two hours burns more than a short full outage — and serial dependencies multiply, so a journey through 99.95, 99.99 and 99.9 percent services is capped near 99.84 percent unless I take the weakest one off the critical path."*

---

## Concept 46 — The error budget policy

An SLO without consequences is decoration. The **error budget policy** is a short, pre-agreed document stating what happens at each budget level — signed off by engineering *and* product leadership *before* the first incident, so nobody negotiates under pressure.

A typical policy:

| Budget state (rolling window) | Consequence |
|---|---|
| **> 50% remaining** | Normal velocity; experiments and risky changes allowed |
| **25–50% remaining** | Extra care: canary all releases, no risky migrations without review |
| **< 25% remaining** | Reliability work prioritized; feature releases need SRE/lead approval |
| **Exhausted** | **Feature freeze** for the service except fixes, security patches and reliability work, until the budget recovers to a threshold (or for a fixed period); postmortem required for any single incident that consumed > 20% of budget |
| **Repeatedly exhausted** | Escalate: architecture review, staffing, or a deliberate decision to **lower the SLO** (and tell users) |

Rules that make policies work:

1. **Agree it in advance, with product.** The policy is a contract between the people who want features and the people who carry pagers.
2. **Apply it to causes honestly.** If a dependency burned the budget, the response may be resilience work against that dependency (async, caching, fallback) rather than a freeze on your releases — the policy should say so.
3. **Budget is also permission.** If you consistently end the window with 80% of budget left, you're probably over-investing in reliability or under-taking risk: ship faster, run game days, or tighten the SLO.
4. **Review SLOs quarterly**: still the right journeys, thresholds and targets? SLOs that nobody looks at are worse than none.

**The interview-grade sentence:** *"The error budget policy is a short document agreed with product before any incident: normal velocity with plenty of budget, extra care as it shrinks, a freeze on features except fixes and reliability work once it's exhausted, a postmortem for any incident that burns more than about a fifth of it, and escalation — architecture, staffing or a lower SLO — if it's exhausted repeatedly. And unspent budget is permission to ship faster or take more risk, not a trophy."*

---

## Concept 47 — Alerting on burn rate

The naive SLO alert — "page if the error rate over 5 minutes exceeds 0.1%" — is both too noisy (a 5-minute blip fires it though it barely touches the budget) and, set higher, too slow. **Burn-rate alerting** fixes both.

**Burn rate** = observed error ratio ÷ budgeted error ratio (1 − SLO).

- Burn rate **1** spends exactly the budget over the SLO window.
- Burn rate **14.4** over a 30-day window spends it in 30 × 24 / 14.4 = **50 hours**.
- Burn rate **720** (100% errors at 99.9%) spends it in **1 hour**.

The fraction of the budget consumed during an alert window `w` at burn rate `b` is `b × w / T` (T = SLO window). Choose "how much budget lost justifies a page" and solve for `b`:

| Severity | Budget consumed | Long window | Burn rate threshold | Short window (≈ 1/12) | Action |
|---|---|---|---|---|---|
| **Page** | 2% | 1 h | 0.02 × 720 / 1 = **14.4** | 5 min | Page now |
| **Page** | 5% | 6 h | 0.05 × 720 / 6 = **6** | 30 min | Page |
| **Ticket** | 10% | 3 days | 0.10 × 720 / 72 = **1** | 6 h | Ticket, next business day |

(These are the Google SRE workbook's recommended starting values for a 30-day window.)

**Why two windows per alert.** The **long window** ensures the problem is significant (enough budget actually lost); the **short window** ensures it's *still happening*, so the alert resets quickly after recovery instead of staying red for an hour. Both must exceed the threshold:

```promql
# Fast-burn page for a 99.9% availability SLO (recording rules precompute these ratios)
(
  slo:checkout_errors:ratio_rate1h  > (14.4 * 0.001)
and
  slo:checkout_errors:ratio_rate5m  > (14.4 * 0.001)
)
or
(
  slo:checkout_errors:ratio_rate6h  > (6 * 0.001)
and
  slo:checkout_errors:ratio_rate30m > (6 * 0.001)
)
```

The KQL equivalent on Application Insights data runs as a **log search alert** every 5 minutes (Concept 56):

```kusto
// Fast-burn check for a 99.9% availability SLO on checkout, over Application Insights requests.
let slo  = 0.999;
let burn = 14.4;
let checkout = requests | where name == "POST /checkout";
// itemCount re-weights records that survived sampling.
let r1h = toscalar(checkout | where timestamp > ago(1h)
                   | summarize todouble(sumif(itemCount, success == false)) / sum(itemCount));
let r5m = toscalar(checkout | where timestamp > ago(5m)
                   | summarize todouble(sumif(itemCount, success == false)) / sum(itemCount));
print r1h, r5m, threshold = burn * (1 - slo)
| where r1h > threshold and r5m > threshold
```

(Remember that Application Insights' `success` reflects its own mapping of status codes; if your SLI excludes some 4xx or counts your own 429s as bad, filter on `resultCode` explicitly. In production, prefer metrics — standard metrics or Prometheus recording rules — over log queries for SLI alerts: they're complete, cheaper to evaluate and don't inherit log ingestion latency.)

**Properties to discuss — the four alerting qualities** from the SRE workbook: **precision** (alerts are real), **recall** (real problems alert), **detection time**, and **reset time**. Multi-window, multi-burn-rate alerts score well on all four. Worked detection time: a full outage at 99.9% pushes the 1-hour error ratio past 1.44% within about **52 seconds** (0.0144 h), plus evaluation interval and ingestion latency — so detection in a few minutes.

**Low-traffic services** break ratio-based alerts: at 10 requests per hour, one failure is a 10% error ratio. Remedies: **synthetic traffic** to provide a baseline, **time-based SLIs**, **minimum request-count conditions** in the alert, or **aggregating** several low-traffic services into one SLO.

**Tooling.** Sloth and Pyrra generate Prometheus recording and alerting rules from an SLO spec (OpenSLO-compatible definitions exist); Grafana SLO and vendor tools do the same in their products. On Azure, you implement it with Prometheus rule groups on managed Prometheus or log search alerts in KQL — there's no built-in SLO object as of this writing.

**The interview-grade sentence:** *"I alert on burn rate — the observed error ratio over the budgeted one — with multi-window, multi-burn-rate rules: page when 2% of a 30-day budget burns in an hour, a burn rate of 14.4 confirmed over 5 minutes, or 5% in six hours at a burn rate of 6 confirmed over 30 minutes, and ticket at 10% over three days. The long window proves significance, the short one proves it's still happening so alerts reset quickly; a full outage at 99.9% crosses the fast threshold in under a minute, and low-traffic services need synthetics, minimum-count conditions or time-based SLIs."*

---

## Concept 48 — Alert design beyond SLOs

Burn-rate alerts cover user-facing symptoms. A complete alerting design adds a few other classes and enforces discipline across all of them — Rob Ewaschuk's "My Philosophy on Alerting" and the SRE book are the classic sources.

**Principles:**

1. **Page on symptoms, ticket on causes.** Users feel errors and latency (symptoms); CPU at 90%, a full disk at 80% or a growing queue are causes or predictions. Causes go to **tickets** or dashboards unless they *will* become user-facing faster than a human can respond (disk full in 2 hours → page; disk 80% → ticket).
2. **Every page must be actionable, urgent and novel.** If the right response is "wait and see," it's not a page. If it fires every day, it's not novel — fix the cause or delete the alert.
3. **Every alert has a runbook link and an owner** — what it means, how to verify, how to mitigate, who escalates.
4. **Few pages.** A common target is no more than a couple of pages per on-call shift; more means people stop reading them (alert fatigue), which is how real incidents get missed.
5. **Alert on the absence of data.** A service that stops emitting looks healthy to a threshold alert. Use "no data" conditions, heartbeat metrics, and synthetic checks.

**Alert classes worth having, beyond SLO burn:**

| Class | Example | Severity |
|---|---|---|
| Fast-burn SLO | Checkout availability burn ≥ 14.4 | Page |
| Imminent resource exhaustion | Disk / quota / certificate expiring within hours | Page |
| Data-loss risk | Dead-letter queue growing, backup failures, replication lag beyond RPO | Page or high ticket |
| Pipeline health | Oldest message age, consumer lag beyond the freshness SLO | Page (it's the SLO) or ticket |
| Saturation trends | Thread-pool queue rising, RU 429 rate rising, connection pool waits | Ticket / dashboard |
| Telemetry health | Ingestion volume dropped to zero, exporter errors, Collector refusing data | Ticket (page if blind) |
| Cost anomalies | Ingestion 3× the weekly average | Ticket |
| Security signals | Auth failure spikes, disabled local auth re-enabled | Security team routing |

**Dynamic thresholds and anomaly detection** (Azure Monitor's dynamic thresholds, smart detection) can help with seasonal metrics where a fixed threshold doesn't fit, but they're **hard to explain** at 3 a.m. and learn bad baselines during long incidents. Use them for tickets and exploration; prefer SLO burn for pages.

**The interview-grade sentence:** *"Beyond burn-rate pages I alert on imminent exhaustion like quotas, disks and certificates, data-loss risks like growing dead-letter queues or replication lag beyond the RPO, pipeline freshness, and the absence of data — with saturation trends, telemetry-pipeline health and cost anomalies as tickets. Every page must be actionable, urgent and novel with a runbook and an owner, pages stay to a couple per shift, and dynamic thresholds are for tickets and exploration rather than waking people, because they're hard to reason about at 3 a.m."*

---

## Concept 49 — SLOs for asynchronous systems

Request-based SLIs don't fit queues, streams and pipelines (Modules 11, 27): the producer gets a fast 202, and the user's experience is decided later, by a consumer. Asynchronous systems need SLIs about **time-to-effect** and **correctness of outcomes**.

| SLI | Definition | Measured from |
|---|---|---|
| **End-to-end latency (freshness)** | % of events whose effect is visible within X of the event time (order placed → confirmation email sent within 2 min; telemetry visible on dashboard within 30 s) | Event timestamp carried in the message vs completion time in the consumer — a histogram of `now − event_time` |
| **Age of oldest unprocessed message** | Max age in the queue/backlog | Broker metrics: Service Bus age of oldest message (if exposed in your tier/tooling), Event Hubs consumer lag converted to time, change-feed estimator |
| **Consumer lag in time** | How far behind real time the consumer is | Event Hubs enqueued time of last processed vs latest; change-feed estimator |
| **Processing success** | % of messages processed without dead-lettering | Completed vs dead-lettered counts |
| **Correctness / completeness** | % of expected outputs produced (all partitions of the nightly export present by 06:00) | Reconciliation jobs, row counts |

Design notes:

1. **Carry event time in the message** (CloudEvents `time`, or a domain timestamp) and measure end-to-end from it, not from enqueue time — the producer's own outbox delay is part of what users experience.
2. **Histogram of end-to-end latency, with the SLO threshold as a bucket boundary**, recorded by the final consumer (or by each stage, for attribution).
3. **Queue depth is not an SLI.** Ten thousand messages may be fine (fast consumers) or awful (stalled). **Age and lag in time** are what users feel. Depth is a capacity/scaling signal.
4. **Dead-letter growth is a correctness alert** — each DLQ message is usually a user whose order, email or payment didn't happen.
5. **Batch pipelines** use time-based SLOs: "on 99% of days, the report is complete by 06:00."
6. **Fan-out**: an event that triggers five consumers has five effects; define the SLO per consumer journey that matters, not one blended number.

**The interview-grade sentence:** *"Asynchronous systems need time-to-effect SLIs: the percentage of events whose effect is visible within a threshold of the event time — carried in the message, so outbox delay counts — recorded as a histogram by the consumer, plus the age of the oldest message or consumer lag in time, dead-letter rate as a correctness signal, and time-based SLOs for batch deadlines. Queue depth is a scaling signal, not an SLI, because ten thousand messages can be fine or a disaster depending on how fast they drain."*

---

## Concept 50 — SLOs in an organization

SLOs are as much a social system as a technical one.

**The SLO document** for each service (one page): the user journeys covered; SLI definitions and their implementation (queries); targets and windows; the rationale for each target; exclusions; the error budget policy; the alerting rules and runbooks; owners; review date. Keep it in the repository next to the service, versioned, and generate alert rules from it where tooling allows (OpenSLO, Sloth).

**Ownership.** Each SLO has an owning team with authority over its releases. A shared platform SLO (identity, API gateway, messaging) is owned by the platform team and consumed by product teams as a **dependency SLO** — which they should design around (Concept 45's multiplication).

**Hierarchy of SLOs:**

- **Journey SLOs** (what users feel: "checkout succeeds") — few, business-facing.
- **Service SLOs** (per service on the journey) — for owning teams' budgets.
- **Dependency/platform SLOs** — what platform teams promise internally.

A journey SLO tighter than the product of its services' SLOs is a design problem, not a monitoring one.

**SLAs from SLOs.** If you sell an SLA, derive it from measured SLO performance with a margin — e.g. internal SLO 99.95%, external SLA 99.9% — and make sure the SLA's measurement method (often time-based "minutes of downtime") is one you can actually compute and defend from your telemetry.

**Reviews.** A monthly or quarterly SLO review looks at budget consumption, top budget-burning incidents, alert precision (pages that weren't real), and whether targets still match user expectations. It's also where you decide to tighten an SLO that's always met or loosen one that's never been achievable.

**Common organizational failures:** SLOs defined by a central team without the service owners; SLOs that exist on a dashboard but drive no decisions; targets set by marketing ("five nines!"); and an SLO per metric (dozens per service), which is just monitoring with extra steps.

**The interview-grade sentence:** *"Organizationally, each service has a one-page SLO document in its repository — journeys, SLI queries, targets with rationale, exclusions, budget policy, alerts and runbooks, owner and review date — with alert rules generated from it where possible; SLOs are layered from a few journey SLOs down to service and platform dependency SLOs, external SLAs are derived from measured SLOs with a margin and a defensible measurement method, and quarterly reviews tighten, loosen or retire them — otherwise they're dashboards that drive no decisions."*

---
# Part G — Observability on Azure

## Concept 51 — The Azure Monitor map

"Azure Monitor" is an umbrella over several stores and services. Getting the map right is the first Azure signal in an interview.

| Component | What it stores / does | Queried with | Typical content |
|---|---|---|---|
| **Azure Monitor Metrics (platform metrics)** | Time series emitted automatically by Azure resources, 93 days | Metrics explorer, REST, Grafana | Service Bus active messages, Cosmos normalized RU, App Service CPU, Front Door request count |
| **Log Analytics workspace** | Logs and traces in tables, retention per table (interactive up to 2 years, long-term up to 12) | **KQL** | Application Insights data (`AppRequests`, `AppDependencies`, `AppTraces`, `AppExceptions`, `AppMetrics`…), resource logs (`AzureDiagnostics` or resource-specific tables), Container Insights, Activity Log, custom tables, OTLP logs/traces |
| **Azure Monitor workspace** | **Prometheus metrics** (managed Prometheus), including OTLP metrics | **PromQL** | AKS/container metrics, application OTel metrics via OTLP or remote write |
| **Application Insights** | An **application-centric experience** over data stored in a Log Analytics workspace (workspace-based; classic resources were retired) | Portal experiences + KQL | Application Map, transaction search, performance, failures, Live Metrics, availability tests |
| **Data collection rules (DCRs)** | Define what's collected, how it's transformed (KQL transformations at ingestion), and where it goes | — | Filtering, column masking, routing to tables/workspaces |
| **Azure Monitor Agent (AMA)** | Agent on VMs/Arc servers; with DCRs, collects logs, perf counters, and (preview) OTLP | — | Host telemetry |
| **Alerts and action groups** | Metric, log search, Prometheus, activity-log alerts; notification/automation targets | — | Pages, tickets, runbooks |
| **Visualization** | Workbooks, Azure dashboards, **Dashboards with Grafana** (portal, GA Nov 2025), **Azure Managed Grafana** | — | Ops dashboards, SLO reports |

Two schema facts that trip up engineers:

1. **Application Insights tables have two names.** In the Application Insights resource's own query context, you write `requests`, `dependencies`, `traces`, `exceptions`, `customMetrics` (classic names with `timestamp`, `operation_Id`, `customDimensions`). In the Log Analytics workspace, the same data lives in `AppRequests`, `AppDependencies`, `AppTraces`, `AppExceptions`, `AppMetrics` (with `TimeGenerated`, `OperationId`, `Properties`). Same data, different column names.
2. **OTLP-native data uses OpenTelemetry semantic conventions**, not the classic Application Insights schema (Concept 53). If a system mixes distro-ingested and OTLP-ingested telemetry, queries differ by path.

**Workspace design** is an architecture decision: one workspace per environment per region is a common default; separate workspaces for **regulated data** (access isolation), for **cost attribution** (or use resource-context RBAC and table-level access within one), and for **data residency**. Fewer workspaces make cross-service queries simpler; Application Insights resources can be one per service or one per system (one per system with distinct Cloud Role Names makes Application Map work across services).

**The interview-grade sentence:** *"Azure Monitor is several stores: platform metrics from every resource, Log Analytics workspaces holding logs and traces — including all Application Insights data, which is workspace-based — queried with KQL, and Azure Monitor workspaces holding managed Prometheus metrics queried with PromQL; data collection rules decide what's collected and transform it at ingestion, and Application Insights is the application experience on top. I remember Application Insights tables appear as requests and dependencies in the resource view but AppRequests and AppDependencies in the workspace, and that OTLP-ingested data follows semantic conventions instead."*

---

## Concept 52 — Instrumenting .NET for Azure Monitor in 2026

There are now three supported ways to get .NET telemetry into Application Insights, plus the vendor-neutral path in Concept 53. All are OpenTelemetry underneath.

| Path | Package / API | Choose when | Notes |
|---|---|---|---|
| **Azure Monitor OpenTelemetry Distro** | `Azure.Monitor.OpenTelemetry.AspNetCore` → `services.AddOpenTelemetry().UseAzureMonitor()`; `Azure.Monitor.OpenTelemetry.Exporter` for non-ASP.NET Core apps | Azure-first ASP.NET Core services that want the documented path | Live Metrics, standard metrics, resource detectors, rate-limited sampling (5 traces/s default), Entra auth, offline storage |
| **Microsoft OpenTelemetry Distro** | `Microsoft.OpenTelemetry` → `builder.UseMicrosoftOpenTelemetry(o => …)` (1.0 since May 2026) | New Azure Monitor work, AI-agent workloads, or Azure Monitor **plus** OTLP export | Nearly the same options as the Azure Monitor distro; adds Azure SDK, OpenAI, Semantic Kernel and Agent Framework sources, Agent365 export and `ExportTarget.Otlp` |
| **Application Insights SDK 3.x** | `Microsoft.ApplicationInsights.AspNetCore` / `.WorkerService` 3.x (3.0 GA Feb 2026) | Upgrading 2.x code that uses `TelemetryClient` heavily | OTel inside; classic `Track*` calls mapped to OTel; `TelemetryInitializer`/`Processor`/modules removed (use OTel processors); adaptive sampling replaced by `SamplingRatio` / `TracesPerSecond`; `TrackPageView` removed; many 2.x packages discontinued (`DependencyCollector`, `PerfCounterCollector`, `EventCounterCollector`, `Microsoft.Extensions.Logging.ApplicationInsights`, `Log4NetAppender`, …) |
| **Plain OTel SDK + Azure Monitor exporter** | `OpenTelemetry.Extensions.Hosting` + `Azure.Monitor.OpenTelemetry.Exporter` | You want full control of instrumentation and processors | You assemble the pieces the distro would add |

**Rules that apply to all of them:**

1. **Don't mix** SDK 3.x and a distro in one application, and don't mix 2.x and 3.x packages.
2. **Connection string, not instrumentation key** — via `APPLICATIONINSIGHTS_CONNECTION_STRING` (Microsoft's recommendation for production), not hard-coded. Instrumentation-key-only ingestion is no longer supported.
3. **Entra ID authentication for ingestion** (`Credential` option with a managed identity, and **disable local authentication** on the Application Insights resource) prevents anyone with the connection string from injecting telemetry. Exception to watch: the Container Apps managed OTel agent's Application Insights destination currently needs local auth left enabled.
4. **Set Cloud Role Name** (`service.name`/`service.namespace`) whenever more than one service shares a resource.
5. **Custom sources and meters** must be added: `ConfigureOpenTelemetryTracerProvider(b => b.AddSource("Contoso.*"))`, `ConfigureOpenTelemetryMeterProvider(b => b.AddMeter("Contoso.*"))`.
6. **Sampling** defaults to **rate-limited 5 traces/s** in the distro and SDK 3.x — fine for small services, too low for diagnosis at scale and too high for very large fleets. Choose explicitly (`TracesPerSecond`, or `SamplingRatio` with `TracesPerSecond = null`), and remember standard metrics are computed before sampling so the Performance and Failures blades stay accurate.
7. **Azure SDK tracing**: the Azure SDKs emit `Activity` spans; in the Azure Monitor distro, enabling them has historically required the `Azure.Experimental.EnableActivitySource` switch (`AZURE_EXPERIMENTAL_ENABLE_ACTIVITY_SOURCE=true`); the Microsoft distro enables Azure SDK instrumentation by default — verify for your versions.
8. **Custom events**: with the distros, an `ILogger` record with a `microsoft.custom_event.name` attribute becomes an Application Insights custom event.

```csharp
// Azure Monitor distro, production-shaped.
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(serviceName: "orders-api", serviceNamespace: "shop",
                                         serviceVersion: typeof(Program).Assembly.GetName().Version?.ToString()))
    .UseAzureMonitor(o =>
    {
        o.Credential      = new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(clientId));
        o.TracesPerSecond = null;          // switch from rate-limited to ratio sampling
        o.SamplingRatio   = 0.2f;
    })
    .WithTracing(t => t.AddSource("Contoso.*"))
    .WithMetrics(m => m.AddMeter("Contoso.*", "System.Runtime"));
```

**Migration from SDK 2.x** (Worked example 5): inventory 2.x packages and third-party sinks (e.g. `Serilog.Sinks.ApplicationInsights` must support 3.x), decide SDK 3.x (minimal code change) vs a distro (cleaner long-term), replace initializers/processors with OTel processors, replace adaptive sampling settings, move `InstrumentationKey` to connection strings, replace channel mocks in tests with in-memory exporters, and **validate dashboards and alerts** — some property names and metric names change.

**The interview-grade sentence:** *"For .NET on Azure in 2026 I'd choose between the Azure Monitor distro with UseAzureMonitor, the Microsoft OpenTelemetry Distro — 1.0 since May with the same options plus OTLP, Azure SDK and AI-agent sources — and Application Insights SDK 3.x, GA since February, which re-implements TelemetryClient on OpenTelemetry for 2.x upgrades but drops initializers, adaptive sampling and many 2.x packages; never two of them in one app. In every case I use a connection string from the environment, Entra ingestion with local auth disabled, Cloud Role Names, explicit sampling instead of the five-traces-per-second default, and AddSource and AddMeter for our own telemetry."*

---

## Concept 53 — The OTLP-native path into Azure Monitor

The alternative to a Microsoft distro is **vendor-neutral OpenTelemetry end to end**: upstream SDKs export OTLP; Azure Monitor ingests OTLP.

**Three ingestion mechanisms:**

| Mechanism | For | Status (verify) |
|---|---|---|
| **OpenTelemetry Collector → Azure Monitor endpoints** | Any environment where a Collector runs, including other clouds and on-premises | Microsoft's pages disagree (GA in the overview, preview in the how-to); requires Collector **v0.132.0+** with the **Azure authentication extension** and **Entra ID** |
| **Azure Monitor Agent (AMA) OTLP receiver** | VMs, scale sets, Arc-enabled servers — AMA receives OTLP locally and forwards, configured by a DCR | Preview |
| **AKS application monitoring with OTLP** | AKS workloads — managed autoinstrumentation or autoconfiguration at namespace or deployment scope | Preview |

**Where data lands:**

- **Traces and logs** → **Log Analytics** tables that use **OpenTelemetry semantic conventions** (not the classic Application Insights schema), with an optional **Application Insights resource with OTLP support** layered on top to enable Application Insights troubleshooting experiences;
- **Metrics** → an **Azure Monitor workspace** (managed Prometheus), queried with PromQL and visualized with Grafana.

**Why choose it:** you already standardize on upstream OTel SDKs across languages; you need multicloud or hybrid; you want a Collector for policy (redaction, tail sampling, routing to several backends); or you want metrics in Prometheus form rather than Application Insights `customMetrics`. **Why not:** some Application Insights features depend on the distro (Live Metrics, Profiler integration, certain standard metrics), queries differ from the classic schema, and parts are still preview — so production SLAs and feature completeness need checking.

**A pragmatic hybrid** many teams adopt: services instrument with upstream OTel and export OTLP to a Collector; the Collector forwards to Azure Monitor (OTLP endpoints or the `azuremonitor` exporter) *and* to a second destination during evaluation or for a specific team. That keeps service code backend-neutral.

**The interview-grade sentence:** *"The OTLP-native path keeps services on upstream OpenTelemetry and ingests OTLP into Azure Monitor through a Collector — v0.132 or later with Entra authentication — through the Azure Monitor Agent on VMs, or through AKS-integrated monitoring, the latter two in preview; traces and logs land in Log Analytics using semantic conventions, optionally with an OTLP-enabled Application Insights resource for its experiences, and metrics land in an Azure Monitor workspace as Prometheus. I'd choose it for multicloud, multi-language or Collector-policy needs, accepting that some distro-only features and the classic query schema don't carry over."*

---

## Concept 54 — Compute-specific collection

Each compute platform from Module 26 has its own collection story. Know the defaults and the gaps.

| Platform | Built-in collection | Application telemetry | Gotchas |
|---|---|---|---|
| **App Service** | Platform metrics; diagnostic settings for HTTP logs, console logs, app logs; auto-instrumentation (Application Insights agent) for .NET | Distro or SDK 3.x in code (preferred for control), or auto-instrumentation via app settings | Don't enable auto-instrumentation *and* an in-code distro (double telemetry); HTTP logs to Log Analytics can be large |
| **Azure Functions** (isolated worker) | Functions host emits its own telemetry (invocations, scale) | OpenTelemetry in the worker: the Functions OTel integration lets host and worker export via OTLP or Azure Monitor; configure the host (`host.json` telemetry mode) and the worker consistently | Host and worker each emit; without coordination you get duplicate request telemetry or disconnected traces; sampling is configured in more than one place |
| **Container Apps** | System logs, console logs (Log Analytics or Azure Monitor), platform metrics | **Managed OpenTelemetry agent** in the environment (destinations: Application Insights, Datadog, OTLP endpoints; gRPC only) — apps send OTLP to it via injected `OTEL_*` variables; or an in-app distro; or a Collector app | The managed agent's Application Insights destination takes traces and logs, **not metrics**, and requires local auth on the resource; console logs collected by the platform *and* exported via OTel are double-billed |
| **AKS** | **Container Insights** (logs, inventory), **managed Prometheus** add-on (metrics to an Azure Monitor workspace), control plane diagnostics | Distro in code; OTel Operator for auto-instrumentation; Collector DaemonSet/gateway; AKS OTLP application monitoring (preview) | Container Insights log volume (stdout of every pod) is a classic cost driver; use cost-optimized profiles, filter namespaces, and route verbose logs to Basic tables |
| **VMs / Arc servers** | AMA with DCRs (perf counters, logs; OTLP preview) | Distro or OTel SDK + Collector | Legacy MMA agent retired; use AMA |
| **API Management / Front Door / Application Gateway** | Resource logs and metrics; APIM can log to Application Insights with sampling and propagate `traceparent` | — | Edge logs are your outside-in SLI source (Concept 43); sample APIM's Application Insights logging — it can be very chatty |

**The cross-platform rule:** decide **one path per signal per service**, and turn off the others. Most "why is our bill double?" investigations end at two collection paths for the same data — platform console log collection plus OTel logs, or auto-instrumentation plus a distro.

**The interview-grade sentence:** *"Each compute has its own collection story: App Service offers auto-instrumentation but I prefer the distro in code and never both; Functions isolated has host and worker telemetry that must be configured together; Container Apps has a managed OTel agent — gRPC only, and its Application Insights destination takes traces and logs but not metrics; AKS has Container Insights for logs and managed Prometheus for metrics, with stdout volume as the classic cost trap. The rule across all of them is one collection path per signal per service, because duplicated paths are the usual cause of a doubled bill."*

---

## Concept 55 — KQL for engineers

KQL (Kusto Query Language) is how you query Log Analytics and Application Insights. Senior engineers are expected to write simple queries live. The pipeline model: start from a table, pipe through operators.

**The operators you'll use 90% of the time:** `where`, `project`, `extend`, `summarize … by`, `bin()`, `percentile()` / `percentiles()`, `count()`, `dcount()`, `top`, `order by`, `join`, `parse` / `extract`, `mv-expand`, `render`, `let`, `ago()`, `todynamic()`.

```kusto
// 1. Request rate, failure rate and p95 per route, 5-minute bins (Application Insights resource context)
requests
| where timestamp > ago(6h)
| summarize requests = sum(itemCount),
            failed   = sumif(itemCount, success == false),
            p95_ms   = percentile(duration, 95)
          by name, bin(timestamp, 5m)
| extend failure_rate = todouble(failed) / requests
| render timechart

// 2. Slowest dependencies by target and type in the last hour
dependencies
| where timestamp > ago(1h)
| summarize calls = sum(itemCount), p99_ms = percentile(duration, 99), failures = sumif(itemCount, success == false)
          by type, target, name
| top 20 by p99_ms desc

// 3. Everything about one request: spans and logs by trace ID
union requests, dependencies, traces, exceptions
| where operation_Id == "4bf92f3577b34da6a3ce929d0e0e4736"
| project timestamp, itemType, name, duration, success, message, severityLevel, customDimensions
| order by timestamp asc

// 4. Which attribute explains the slow requests? Compare slow vs all.
requests
| where timestamp > ago(1h) and name == "POST /orders"
| extend tenant = tostring(customDimensions["contoso.tenant.id"]), slow = duration > 1000
| summarize total = sum(itemCount), slow_count = sumif(itemCount, slow) by tenant
| extend slow_share = todouble(slow_count) / total
| top 10 by slow_count desc

// 5. Exceptions after a deploy, grouped by type and version
exceptions
| where timestamp > ago(24h)
| summarize count = sum(itemCount) by type, application_Version, bin(timestamp, 1h)
| render columnchart

// 6. Ingestion cost by table — the first query in any cost review (workspace context)
Usage
| where TimeGenerated > ago(30d) and IsBillable
| summarize GB = sum(Quantity) / 1024 by DataType
| order by GB desc
```

Habits: **always use `itemCount`** (re-weights sampling) rather than `count()` for Application Insights counts; **filter on time first** (it's the partition key and the biggest cost lever for queries); **`project` early** to reduce columns; **cast `customDimensions` values** (`tostring`, `toint`) because they're dynamic; prefer **`summarize` + `bin`** over pulling raw rows to the client.

**The interview-grade sentence:** *"In KQL I start from the table, filter on time first, project early, and summarize by bin for time series — rates, failure ratios and percentiles per route, slowest dependencies by target, a union of requests, dependencies, traces and exceptions on one operation ID to see a whole request, and slow-versus-total comparisons by a custom dimension to find which tenant or partition explains the latency — using itemCount instead of count so sampled data is re-weighted, and the Usage table as the first query in any cost review."*

---

## Concept 56 — Alerts and action groups

Azure Monitor alert types and when to use each:

| Alert type | Evaluates | Latency | Cost | Best for |
|---|---|---|---|---|
| **Metric alert** | Platform or custom metrics (static or dynamic thresholds, multiple dimensions) | ~1 minute | Low, per time series monitored | Resource health, SLI ratios computed as metrics, saturation |
| **Log search alert** | A KQL query on a schedule (1–15 min, or longer) | Ingestion latency + frequency (several minutes) | Per rule × frequency | Complex conditions, SLI ratios over logs, burn rate in KQL, absence of logs |
| **Prometheus alert (rule group)** | PromQL over an Azure Monitor workspace | ~1 minute | Per rule evaluation | Container and OTel metrics; burn-rate rules from Sloth/Pyrra |
| **Activity log alert** | Control-plane events, service health, resource health | Minutes | Free | Deletions, role changes, Azure service incidents |
| **Smart detection / anomaly** | Application Insights failure anomalies | Varies | Included | Supplementary, not primary |

**Action groups** define *who/what is notified*: email, SMS, voice, push, webhooks, ITSM, Logic Apps, Functions, Automation runbooks, Event Hubs — and integrations to PagerDuty, Opsgenie, ServiceNow, Teams. **Alert processing rules** suppress (maintenance windows) or route alerts at scale without editing each rule.

Design notes:

1. **Alert rules are code.** Define them in Bicep/Terraform next to the service, generated from the SLO spec where possible, reviewed like any change.
2. **Severity mapping**: Sev0/1 → page; Sev2 → ticket; Sev3/4 → informational. Keep the mapping consistent across teams.
3. **Stateful alerts** (auto-resolve) for metrics; mind that log alerts may fire repeatedly per evaluation unless configured for stateful behavior.
4. **Don't alert on every resource's default metrics.** Azure offers "recommended alerts" per resource — useful as a baseline for platform health, but an estate with thousands of them reproduces alert fatigue at cloud scale.
5. **Test alerts** — synthetic failure injection (Chaos Studio, a feature flag that forces errors in staging) proves the alert fires and the runbook works.

**The interview-grade sentence:** *"On Azure I use metric alerts for resource health and metric-based SLIs because they're fast and cheap, Prometheus rule groups for burn-rate rules over managed Prometheus, log search alerts when the condition needs KQL — accepting ingestion latency plus evaluation frequency — and activity-log alerts for control-plane and service-health events. Action groups route to paging and ITSM, alert processing rules handle maintenance suppression, and all of it lives in Bicep or Terraform next to the service and gets tested with injected failures."*

---

## Concept 57 — Dashboards and investigation tools

| Tool | Use it for | Notes |
|---|---|---|
| **Application Map** | Service topology with call rates, failures and latency per edge | Needs correct Cloud Role Names; shows dependencies discovered from spans |
| **Transaction search / end-to-end transaction details** | One request's spans and logs as a waterfall | The trace view; joins by operation ID |
| **Performance and Failures blades** | Per-operation latency distributions and failure breakdowns, drill to samples | Built on standard metrics (pre-sampling) plus sampled details |
| **Live Metrics** | Real-time (≈1 s) request, failure and dependency rates and sample telemetry, unsampled and unstored | Deploy verification, incident watching; distro feature |
| **Availability tests** | Standard (single URL) tests from multiple regions; custom TrackAvailability tests | Outside-in synthetics (Concept 67) |
| **Profiler / Snapshot Debugger** | Code-level traces of slow requests; snapshots at exceptions | Application Insights features; check support for your hosting and SDK path |
| **Workbooks** | Parameterized, interactive reports mixing KQL, metrics and text | SLO reports, incident review templates |
| **Dashboards with Grafana (portal)** | Grafana dashboards over Azure Monitor metrics, logs, traces, Prometheus and ADX — **free, GA since Nov 2025** | Prebuilt dashboards for Application Insights, AKS, Container Apps; portable to any Grafana |
| **Azure Managed Grafana** | Full managed Grafana: plugins, alerting, multiple data sources, teams | When you need more than portal dashboards |
| **Aspire dashboard** | Local and dev/test OTLP receiver with traces, metrics, structured logs and GenAI views; standalone via `aspire dashboard run` (13.3) or the container image | Not a production backend; great for "does my telemetry look right?" |

**Dashboard design** that works under stress:

1. **A hierarchy**: a system **overview** (journey SLOs and budgets), a **service** dashboard per service (RED per endpoint, dependencies, saturation, deploy markers), and **component** dashboards (database, broker, cache). Incidents move top-down.
2. **SLIs first, causes below.** The top row answers "are users OK?"
3. **Deploy and feature-flag annotations** on time series — most incidents follow a change.
4. **Show distributions, not averages** — percentiles from histograms or heatmaps.
5. **Every panel has an owner and a reason**; dashboards nobody opened in 90 days get archived.

**The interview-grade sentence:** *"For investigation on Azure I use Application Map for topology, end-to-end transaction details for a single trace with its logs, the Performance and Failures blades — built on pre-sampling standard metrics — for distributions and samples, Live Metrics for deploy verification, and Profiler when the time is inside code; for dashboards, portal Grafana dashboards, free and GA since November 2025, or Azure Managed Grafana, organized as a hierarchy from journey SLOs to services to components with deploy annotations and percentiles; and the Aspire dashboard locally to check telemetry before it ever reaches Azure."*

---

## Concept 58 — Cost governance

Azure Monitor cost is dominated by **Log Analytics ingestion**, followed by retention, Basic/Auxiliary query scans, managed Prometheus samples, alert rules and Grafana. A governance playbook:

**1. Measure.** The `Usage` table (GB by table), `_BilledSize` per record (bytes by service, category or operation), and Application Insights' own volume views. Find the top tables and the top emitters within them.

```kusto
// Bytes by Application Insights role and record type over 7 days (workspace context)
union withsource = Table AppRequests, AppDependencies, AppTraces, AppExceptions, AppMetrics
| where TimeGenerated > ago(7d)
| summarize GB = sum(_BilledSize) / 1e9 by Table, AppRoleName
| order by GB desc
```

**2. Choose table plans** per table:

| Plan | Ingestion (approx.) | Query | Retention | Use for |
|---|---|---|---|---|
| **Analytics** | ~$2.30/GB (commitment tiers discount it) | Full KQL, alerts, included | Interactive up to 2 years | Data you query and alert on often |
| **Basic** | ~$0.50/GB | Per-GB scanned; reduced KQL; simple alerts in preview | 30 days interactive + long-term | Verbose logs for occasional troubleshooting |
| **Auxiliary** | ~$0.05/GB | Per-GB scanned; slower; limited | 30 days interactive + long-term | Audit, compliance, high-volume rarely queried data |

**3. Reduce at the source** — the highest-leverage step: lower log levels on noisy categories, remove duplicate collection paths (Concept 54), drop health-check request telemetry (filter in instrumentation), sample traces explicitly, keep large payloads out.

**4. Reduce at ingestion** — DCR **transformations** (KQL at ingestion) drop rows or columns, mask fields and split data between tables (e.g. errors to Analytics, verbose to Basic). Filtering more than half of incoming data this way can incur a processing charge.

**5. Commitment tiers** — at steady volumes above ~100 GB/day, commitment tiers cut the per-GB price materially; dedicated clusters at higher volumes.

**6. Retention per table** — 30–90 days interactive for operational tables; long-term retention (cheap, slow to search) for compliance; don't keep Debug-level data for two years.

**7. Daily cap — last resort.** A cap stops ingestion when reached, **including the telemetry you need for the incident that caused the spike**. Prefer budget alerts (Cost Management) and ingestion-anomaly alerts per table, with an owner who acts.

**8. Showback.** Attribute ingestion to teams (by role name, resource or table) and publish it; teams that see their telemetry bill optimize it.

**The interview-grade sentence:** *"Azure Monitor cost is mostly Log Analytics ingestion, so I measure first — the Usage table and _BilledSize by table and role — then reduce at the source by fixing log levels, duplicate collection paths, health-check telemetry and sampling, reduce at ingestion with DCR transformations, put verbose tables on Basic or Auxiliary plans at roughly a fifth to a fiftieth of the Analytics price, use commitment tiers at steady volume, set retention per table, show the bill back to teams, and use a daily cap only as a last resort because it cuts off exactly the data an incident needs."*

---
# Part H — Making .NET do it right

## Concept 59 — `System.Diagnostics`: the native primitives

.NET's observability primitives live in the runtime, and every higher-level tool — OpenTelemetry, Application Insights 3.x, `dotnet-counters`, `dotnet-trace`, `dotnet-monitor`, the Aspire dashboard — is a **listener** on them.

| Primitive | Namespace / package | Purpose | Listened to by |
|---|---|---|---|
| **`ActivitySource` / `Activity`** | `System.Diagnostics` | Distributed tracing spans (OTel tracing API) | `ActivityListener` → OTel SDK, App Insights 3.x |
| **`Meter` and instruments** | `System.Diagnostics.Metrics` | Metrics (OTel metrics API) | `MeterListener` → OTel SDK, `dotnet-counters`, `dotnet-monitor` |
| **`ILogger`** | `Microsoft.Extensions.Logging` | Structured logs | Logger providers → OTel provider, console, … |
| **`EventSource`** | `System.Diagnostics.Tracing` | High-performance runtime/library events (GC, thread pool, network internals) | **EventPipe** → `dotnet-trace`, `EventListener`; OTel SDK self-diagnostics |
| **`DiagnosticSource` / `DiagnosticListener`** | `System.Diagnostics` | In-process, object-rich diagnostic events (older instrumentation hook) | Instrumentation libraries (legacy paths) |
| **Event counters** (`EventCounter`, `PollingCounter`) | `System.Diagnostics.Tracing` | The pre-.NET-8 metrics mechanism | `dotnet-counters`; superseded by `Meter` |

Two properties make them suitable for libraries and hot paths:

1. **Near-zero cost when nobody listens.** `ActivitySource.StartActivity` returns **`null`** if no listener is interested (or the sampler drops it) — which is why every call site uses `?.` — and instruments skip recording when no `MeterListener` has enabled them.
2. **No dependency on a vendor.** A library emits; an application decides.

**EventPipe** is the cross-platform transport behind `dotnet-trace`, `dotnet-counters` and `dotnet-monitor`: it streams `EventSource` events (and metrics) out of a running process over a diagnostic IPC channel — no code changes, no restart. That's your live-debugging escape hatch when the telemetry you exported isn't enough (Concept 65).

**The interview-grade sentence:** *"In .NET the observability primitives live in the runtime: ActivitySource and Activity for spans, Meter and its instruments for metrics, ILogger for structured logs, and EventSource over EventPipe for low-level runtime events — and OpenTelemetry, Application Insights 3.x and the dotnet diagnostic tools are all listeners on them. They cost almost nothing when nobody listens — StartActivity returns null, hence the null-conditional at every call site — which is why libraries can instrument freely without taking a vendor dependency."*

---

## Concept 60 — Wiring OpenTelemetry in ASP.NET Core

A production-shaped, vendor-neutral setup, typically placed in a shared **ServiceDefaults** project (Aspire's pattern) so every service gets the same telemetry:

```csharp
public static class ObservabilityExtensions
{
    public static IHostApplicationBuilder AddObservability(this IHostApplicationBuilder builder)
    {
        // Logs: ILogger → OpenTelemetry, with trace correlation and template properties.
        builder.Logging.AddOpenTelemetry(o =>
        {
            o.IncludeFormattedMessage = true;
            o.IncludeScopes           = true;
        });

        builder.Services.AddOpenTelemetry()
            // Resource: name/version/env come from OTEL_SERVICE_NAME / OTEL_RESOURCE_ATTRIBUTES where set.
            .ConfigureResource(r => r
                .AddService(serviceName: builder.Environment.ApplicationName,
                            serviceVersion: typeof(ObservabilityExtensions).Assembly.GetName().Version?.ToString())
                .AddAttributes([new("deployment.environment.name", builder.Environment.EnvironmentName)]))
            .WithTracing(t => t
                .AddAspNetCoreInstrumentation(o =>
                {
                    o.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");   // no health-check spans
                    o.RecordException = true;
                })
                .AddHttpClientInstrumentation()
                .AddSource("Contoso.*")                 // our ActivitySources
                .AddSource("Azure.*"))                  // Azure SDK spans (Service Bus, Cosmos, Storage…)
            .WithMetrics(m => m
                .AddAspNetCoreInstrumentation()         // or AddMeter("Microsoft.AspNetCore.Hosting", "Microsoft.AspNetCore.Server.Kestrel")
                .AddHttpClientInstrumentation()         // or AddMeter("System.Net.Http")
                .AddMeter("System.Runtime")             // .NET 9+ runtime metrics
                .AddMeter("Contoso.*"))
            // One call configures OTLP for traces, metrics and logs from OTEL_EXPORTER_OTLP_* settings.
            .UseOtlpExporter();

        return builder;
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddObservability();
```

What to notice:

1. **One OpenTelemetry builder, three signals, one exporter call.** Swapping `UseOtlpExporter()` for `UseAzureMonitor()` (Azure Monitor distro) or `UseMicrosoftOpenTelemetry(...)` changes the backend without touching instrumentation.
2. **Configuration from the environment** (`OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_TRACES_SAMPLER`) keeps code identical across environments; Aspire, Container Apps' managed agent and the OTel Operator inject these.
3. **Filter health checks at the instrumentation**, not in the backend — they're often the majority of requests on low-traffic services and pure noise.
4. **Only one SDK registration per service collection.** Calling `AddOpenTelemetry()` repeatedly is additive and fine; calling `UseAzureMonitor()` twice throws; mixing a distro with a manual exporter for the same signal produces duplicates.
5. **Flush on shutdown.** The host disposes the providers on graceful shutdown, which flushes batches — another reason for the termination grace periods in Module 26.

For **worker services and console apps**, the same `AddOpenTelemetry()` works on `Host.CreateApplicationBuilder`; for non-hosted apps, `OpenTelemetrySdk.Create(b => …)` builds and owns the providers (dispose at exit).

**The interview-grade sentence:** *"I wire OpenTelemetry once, in a shared ServiceDefaults-style extension: ILogger bridged to OTel with scopes, one OpenTelemetry builder with the resource, tracing for ASP.NET Core, HttpClient, our Contoso sources and the Azure SDK sources, metrics for the hosting, HttpClient, runtime and Contoso meters, health checks filtered out at instrumentation, and UseOtlpExporter configured from OTEL_ environment variables — so switching to UseAzureMonitor or the Microsoft distro changes the exporter, not the instrumentation, and the host flushes everything on graceful shutdown."*

---

## Concept 61 — Custom traces

Rules for manual spans in .NET:

**1. One static `ActivitySource` per component**, named like a namespace and versioned:

```csharp
internal static class Telemetry
{
    public const string SourceName = "Contoso.Orders";
    public static readonly ActivitySource Source = new(SourceName, version: "1.0.0");
}
```

**2. Create spans around meaningful work, null-safely, disposed with `using`:**

```csharp
public async Task<Quote> PriceAsync(Order order, CancellationToken ct)
{
    using Activity? activity = Telemetry.Source.StartActivity("PriceOrder", ActivityKind.Internal);
    activity?.SetTag("contoso.order.id", order.Id);
    activity?.SetTag("contoso.order.line_count", order.Lines.Count);

    try
    {
        Quote quote = await _pricing.CalculateAsync(order, ct);
        activity?.SetTag("contoso.pricing.rule_set", quote.RuleSetVersion);
        if (quote.UsedFallbackRates) activity?.AddEvent(new ActivityEvent("pricing.fallback_rates_used"));
        return quote;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, "pricing failed");
        activity?.AddException(ex);
        throw;
    }
}
```

**3. Avoid expensive tag computation when not sampled.** `activity` is `null` when nobody listens; when it exists but isn't recorded (`activity.IsAllDataRequested == false`), skip expensive attributes:

```csharp
if (activity is { IsAllDataRequested: true })
    activity.SetTag("contoso.order.summary", BuildSummary(order));   // only pay when the span is recorded
```

**4. Set the kind correctly** — `Client` for calls to un-instrumented remote systems (a legacy SOAP service, an SDK without tracing), `Producer`/`Consumer` for messaging you implement yourself, `Internal` otherwise (Concept 19).

**5. Links at creation time** — for batches or fan-in, pass `ActivityLink`s to `StartActivity` (links can't be added after sampling decisions in older APIs; .NET 9 added `Activity.AddLink`, but samplers only see links passed at creation):

```csharp
var links = messages.Select(m => new ActivityLink(
    Propagators.DefaultTextMapPropagator.Extract(default, m.ApplicationProperties, Getter).ActivityContext));
using var batch = Telemetry.Source.StartActivity("process orders-batch", ActivityKind.Consumer,
                                                 parentContext: default, links: links);
batch?.SetTag("messaging.batch.message_count", messages.Count);
```

**6. Register the source** in the SDK (`AddSource("Contoso.*")`) — or nothing is collected (Concept 9).

**7. Don't wrap everything.** A span per repository method, mapper or validator produces noisy waterfalls and cost; rely on instrumentation for I/O, and add spans for domain operations and un-instrumented calls.

**The interview-grade sentence:** *"For custom tracing I keep one static, versioned ActivitySource per component, start activities null-safely with using around meaningful domain operations or un-instrumented calls, with the right kind, namespaced tags, events for notable moments and AddException plus an Error status on failure; I skip expensive tags unless IsAllDataRequested is true, pass links at creation for batches so samplers can see them, register the source with AddSource, and resist wrapping every method in a span."*

---

## Concept 62 — Custom metrics

Business and component metrics are where custom instrumentation pays most: *orders placed per minute*, *payment success ratio by provider*, *cart value distribution*, *cache hit ratio*, *outbox lag*.

**Use `IMeterFactory` with dependency injection** (.NET 8+), so meters are owned by the DI container, testable, and correctly isolated between test hosts:

```csharp
public sealed class OrderMetrics
{
    public const string MeterName = "Contoso.Orders";

    private readonly Counter<long>     _placed;
    private readonly Histogram<double> _checkoutDuration;
    private readonly Histogram<double> _orderValue;
    private readonly UpDownCounter<int> _inFlight;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        Meter meter = meterFactory.Create(MeterName, version: "1.0.0");

        _placed = meter.CreateCounter<long>("contoso.orders.placed", unit: "{order}",
            description: "Orders successfully placed");

        _checkoutDuration = meter.CreateHistogram<double>("contoso.checkout.duration", unit: "s",
            description: "End-to-end checkout processing time",
            advice: new InstrumentAdvice<double> { HistogramBucketBoundaries = [0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5] });

        _orderValue = meter.CreateHistogram<double>("contoso.order.value", unit: "{EUR}");
        _inFlight   = meter.CreateUpDownCounter<int>("contoso.checkout.in_flight", unit: "{request}");
    }

    public void OrderPlaced(string channel, string paymentProvider, double valueEur, TimeSpan elapsed)
    {
        var tags = new TagList { { "contoso.order.channel", channel },               // web | mobile | partner
                                 { "contoso.payment.provider", paymentProvider } };   // bounded set
        _placed.Add(1, tags);
        _orderValue.Record(valueEur, tags);
        _checkoutDuration.Record(elapsed.TotalSeconds, tags);
    }

    public IDisposable TrackInFlight() { _inFlight.Add(1); return new Decrement(_inFlight); }
    private sealed class Decrement(UpDownCounter<int> c) : IDisposable { public void Dispose() => c.Add(-1); }
}

// Registration
builder.Services.AddSingleton<OrderMetrics>();
// and in the OTel builder: .WithMetrics(m => m.AddMeter(OrderMetrics.MeterName))
```

Rules:

1. **Names**: lowercase, dot-separated, namespaced (`contoso.orders.placed`), no units in the name; **units** in UCUM (`s`, `By`, `{order}`); **durations in seconds** as histograms with **`InstrumentAdvice` bucket boundaries** around SLO thresholds (.NET 9+) or a view.
2. **Tags from bounded sets only** — channel, provider, outcome, region; never order IDs, customer IDs or free text (Concept 28). Use `TagList` (a struct) to avoid allocations for up to eight tags.
3. **Record once, at the right place** — business metrics where the business event is committed (after the transaction), not where it's attempted, or you count failures as successes.
4. **Observable instruments for state you can read** — outbox backlog, cache size, connection-pool usage — via callbacks, rather than incrementing and decrementing from many places.
5. **Business metrics are SLIs too.** "Orders placed per minute" dropping to zero while HTTP availability is 100% is the classic silent failure — a broken checkout button, a payment provider rejecting everything with 200 OK. Alert on business throughput anomalies relative to the same hour last week.

**The interview-grade sentence:** *"Custom metrics go in a DI-registered class built from IMeterFactory — counters for business events like orders placed, histograms in seconds with InstrumentAdvice bucket boundaries around SLO thresholds, up-down counters for in-flight work and observable instruments for readable state like outbox backlog — with namespaced names, UCUM units, TagList tags from bounded sets only, and recording at the point the business event is committed. And business throughput is itself an SLI: orders per minute falling to zero while HTTP availability is a hundred percent is the classic silent outage."*

---

## Concept 63 — Enrichment, filtering and processors

You'll often want to add context to, or remove noise from, telemetry produced by instrumentation you don't own. Choose the layer deliberately:

| Need | Best layer | .NET mechanism |
|---|---|---|
| Add request-specific tags to the HTTP server span | Instrumentation hook | `AspNetCoreTraceInstrumentationOptions.EnrichWithHttpRequest` / `EnrichWithHttpResponse`, or `Activity.Current?.SetTag(...)` in middleware |
| Add a tag to every span (e.g. tenant from baggage) | Span processor | `BaseProcessor<Activity>.OnStart` |
| Add tags to the built-in HTTP request metric | Metrics feature | `IHttpMetricsTagsFeature` |
| Drop spans (health checks, static files) | Instrumentation filter | `o.Filter = ctx => …` |
| Drop or redact attributes for all services | Collector | `transform` / `attributes` / `redaction` processors |
| Change metric dimensions or buckets | View | `AddView(...)` |
| Drop log categories | Logging filters | configuration `LogLevel` |
| Drop or mask at ingestion (Azure) | DCR transformation | KQL in the data collection rule |

**A span processor that copies tenant ID from baggage onto every span:**

```csharp
public sealed class TenantEnrichingProcessor : BaseProcessor<Activity>
{
    public override void OnStart(Activity activity)
    {
        string? tenant = Baggage.GetBaggage("contoso.tenant.id");     // set at the edge after authentication
        if (tenant is not null) activity.SetTag("contoso.tenant.id", tenant);
    }
}

// .WithTracing(t => t.AddProcessor<TenantEnrichingProcessor>())
```

```csharp
// At the edge, after authentication — derive the tenant from the validated token, never from client input.
app.Use(async (ctx, next) =>
{
    if (ctx.User.FindFirst("tid")?.Value is { } tenantId)
    {
        Baggage.SetBaggage("contoso.tenant.id", tenantId);
        Activity.Current?.SetTag("contoso.tenant.id", tenantId);
    }
    await next();
});
```

Principles: **enrich as close to the source as possible** (the code knows the domain context), **filter as early as possible** (every later stage costs money), and **enforce policy centrally** (the Collector or DCR catches what services forget). Remember that processors run on the hot path — keep them allocation-light and exception-safe.

**The interview-grade sentence:** *"I enrich as close to the source as possible — EnrichWithHttpRequest or middleware for request context, a span processor that copies a tenant ID from baggage set after authentication onto every span, IHttpMetricsTagsFeature for bounded tags on the request metric — filter as early as possible with instrumentation filters, views and logging levels, and enforce organization-wide policy like redaction in the Collector or Azure Monitor ingestion transformations, keeping processors cheap and exception-safe because they run on the hot path."*

---

## Concept 64 — Propagation in .NET, including background work

What propagates automatically:

| Boundary | Propagation | Notes |
|---|---|---|
| `async`/`await` within a request | ✅ `Activity.Current` flows via `ExecutionContext` | Including `Task.Run` started inside the request |
| Outgoing `HttpClient` | ✅ W3C `traceparent`/`tracestate` (+ baggage) injected by `SocketsHttpHandler`'s diagnostics | Customize with `DistributedContextPropagator` (e.g. suppress for external hosts) |
| Incoming ASP.NET Core request | ✅ Extracted by hosting | Edge policy in Concept 24 |
| gRPC (`Grpc.Net.Client`) | ✅ | |
| Azure Service Bus / Event Hubs SDKs | ✅ `Diagnostic-Id`/`traceparent` application properties on send; processor spans use them | Enable Azure SDK tracing (`AddSource("Azure.*")`) |
| MassTransit, NServiceBus, Wolverine, Orleans | ✅ Native `ActivitySource`s | Add their sources |
| **`Channel<T>` / in-memory queue → `BackgroundService`** | ❌ The consumer runs in its own context | Capture and restore explicitly |
| **Timers, `IHostedService` loops, scheduled jobs** | ❌ No ambient request | Start a root activity per unit of work |
| **Custom transports, raw sockets, legacy SDKs** | ❌ | Inject/extract with the propagator |

**Capturing context across an in-memory hand-off:**

```csharp
public sealed record WorkItem(Guid OrderId, ActivityContext Parent, IEnumerable<KeyValuePair<string, string>> Baggage);

// Producer (inside the request):
await channel.Writer.WriteAsync(new WorkItem(orderId,
    Activity.Current?.Context ?? default,
    Baggage.Current.GetBaggage()), ct);

// Consumer (BackgroundService):
await foreach (WorkItem item in channel.Reader.ReadAllAsync(stoppingToken))
{
    Baggage.Current = Baggage.Create(item.Baggage.ToDictionary(kv => kv.Key, kv => kv.Value));
    // Child of the request if the delay is short; otherwise start a new trace with a link (Concept 21).
    using Activity? activity = Telemetry.Source.StartActivity(
        "SendConfirmation", ActivityKind.Consumer, parentContext: item.Parent);
    await _mailer.SendConfirmationAsync(item.OrderId, stoppingToken);
}
```

**Injecting into and extracting from a custom carrier** (any `IDictionary<string, string>`-like headers):

```csharp
TextMapPropagator propagator = Propagators.DefaultTextMapPropagator;

// Inject (producer)
propagator.Inject(new PropagationContext(Activity.Current!.Context, Baggage.Current),
                  message.Headers, (headers, key, value) => headers[key] = value);

// Extract (consumer)
PropagationContext parent = propagator.Extract(default, message.Headers,
    (headers, key) => headers.TryGetValue(key, out var v) ? [v] : []);
Baggage.Current = parent.Baggage;
using var activity = Telemetry.Source.StartActivity("process custom-queue", ActivityKind.Consumer, parent.ActivityContext);
```

**Root activities for jobs:** a `BackgroundService` loop, a Quartz/Hangfire job or a timer should start one activity per iteration/run (`ActivityKind.Internal` or `Consumer`) so its spans and logs form a trace — otherwise its logs have no trace ID and its dependency calls appear as orphans.

**The interview-grade sentence:** *"In .NET, context flows automatically through async/await, HttpClient, ASP.NET Core, gRPC, the Azure messaging SDKs and the major messaging frameworks; it doesn't flow through in-memory hand-offs like Channels to BackgroundServices, timers or custom transports. There I capture Activity.Current's context and the baggage into the work item and restore them as the parent or a link, use the default TextMapPropagator to inject and extract on custom carriers, and start a root activity per job run so background work has traces and its logs have trace IDs."*

---

## Concept 65 — The diagnostics toolbox

When exported telemetry isn't enough — the span says "slow" but not why — .NET's diagnostic tools read the live process through EventPipe and diagnostic IPC (Modules 14, 15, 17):

| Tool | What it does | When |
|---|---|---|
| **`dotnet-counters`** | Live view of metrics/counters: `monitor` or `collect` (CSV/JSON), any `Meter` or `EventCounter` | First look at a sick process: GC, thread pool, exceptions, request rates |
| **`dotnet-trace`** | Collects an EventPipe trace (CPU sampling, events) into `.nettrace`; view in PerfView, Visual Studio, Speedscope | CPU hot paths, contention, GC behavior, startup |
| **`dotnet-dump`** | Captures and analyzes process dumps (`analyze` with SOS commands: `dumpheap`, `gcroot`, `threads`, `clrstack`) | Hangs, deadlocks, memory leaks, thread-pool starvation (Module 15) |
| **`dotnet-gcdump`** | Lightweight GC heap snapshot | Managed memory growth |
| **`dotnet-stack`** | Prints managed stacks of all threads | Quick hang diagnosis |
| **`dotnet-monitor`** | A sidecar/agent exposing an HTTP API to collect dumps, traces, logs and metrics on demand or by **triggers** (e.g. "collect a trace when CPU > 80% for 30 s") | Containers and Kubernetes, where you can't `exec` freely |
| **Application Insights Profiler / Code Optimizations / Snapshot Debugger** | Sampled code-level traces of slow requests; snapshots of locals at exceptions | Production on App Service/AKS where supported |
| **Continuous profilers** (Pyroscope, Datadog, the OTel eBPF profiler in alpha) | Always-on low-frequency CPU/alloc profiling, correlatable with traces | Fleet-wide "where does CPU go?" |

**In containers**, these tools need the diagnostic socket: run `dotnet-monitor` as a sidecar sharing `/tmp`, or `kubectl debug` with an ephemeral container that includes the tools, and set `DOTNET_DiagnosticPorts` when needed. Chiseled and distroless images don't ship the tools — plan for it before the incident.

**Operational rules:** dumps contain **secrets and personal data** (memory is everything) — store them in access-controlled storage, short-lived; `dotnet-trace` CPU sampling has low overhead but full event tracing doesn't; and capture *before* restarting a hung process, or the evidence is gone.

**The interview-grade sentence:** *"When exported telemetry says 'slow' but not why, I go to the .NET diagnostic tools over EventPipe: dotnet-counters for a live look at GC, thread-pool and exception rates, dotnet-trace for CPU sampling and contention, dotnet-dump or dotnet-stack for hangs, deadlocks and starvation, dotnet-gcdump for memory growth, and dotnet-monitor as a Kubernetes sidecar with triggers — plus Application Insights Profiler or a continuous profiler for fleet-wide CPU. I plan container access in advance, capture before restarting, and treat dumps as sensitive data."*

---

## Concept 66 — Testing telemetry

Telemetry is a **contract**: dashboards, alerts and SLOs depend on span names, attribute names, metric names and units, and log EventIds. Contracts deserve tests.

**Logs — `FakeLogger`** (`Microsoft.Extensions.Diagnostics.Testing`):

```csharp
var logger = new FakeLogger<CheckoutService>();
var sut = new CheckoutService(logger, /* … */);

await sut.CheckoutAsync(order);

FakeLogRecord record = logger.Collector.GetSnapshot().Single(r => r.Id.Id == 1201);
Assert.Equal(LogLevel.Information, record.Level);
Assert.Equal(order.Id.ToString(), record.GetStructuredStateValue("OrderId"));
```

**Metrics — `MetricCollector<T>`** with a test `IMeterFactory`:

```csharp
var services = new ServiceCollection().AddMetrics().AddSingleton<OrderMetrics>().BuildServiceProvider();
var meterFactory = services.GetRequiredService<IMeterFactory>();
using var placed = new MetricCollector<long>(meterFactory, OrderMetrics.MeterName, "contoso.orders.placed");

services.GetRequiredService<OrderMetrics>().OrderPlaced("web", "adyen", 42.0, TimeSpan.FromMilliseconds(180));

CollectedMeasurement<long> m = Assert.Single(placed.GetMeasurementSnapshot());
Assert.Equal(1, m.Value);
Assert.Equal("web", m.Tags["contoso.order.channel"]);
```

**Traces — an `ActivityListener`** or the OTel **in-memory exporter** (`OpenTelemetry.Exporter.InMemory`):

```csharp
var exported = new List<Activity>();
using TracerProvider tracer = Sdk.CreateTracerProviderBuilder()
    .AddSource(Telemetry.SourceName)
    .AddInMemoryExporter(exported)
    .Build();

await pricing.PriceAsync(order, CancellationToken.None);
tracer.ForceFlush();

Activity span = Assert.Single(exported, a => a.DisplayName == "PriceOrder");
Assert.Equal(order.Id, span.GetTagItem("contoso.order.id"));
```

**Integration level:** with `WebApplicationFactory<T>` (Module 18) or Aspire's testing host, assert that a request produces a server span with the expected `http.route`, that trace context propagates to a downstream fake, and that a message published to the Service Bus emulator carries `Diagnostic-Id`.

**What to test:** names and attributes your **alerts and SLO queries depend on**, EventIds referenced by alerts, that PII is redacted (a `[PersonalData]` parameter appears redacted in `FakeLogger` with redaction enabled), and that health checks are filtered. Don't test every span — test the contract surface.

**The interview-grade sentence:** *"I treat telemetry as a contract and test the surface that alerts and SLOs depend on: FakeLogger to assert EventIds, levels, structured properties and redaction; MetricCollector with a test IMeterFactory to assert metric names, values and bounded tags; the in-memory exporter or an ActivityListener to assert span names and attributes; and WebApplicationFactory or the Aspire test host to check routes, propagation to downstream fakes and trace context on messages to the emulator."*

---
# Part I — Operating and deciding

## Concept 67 — Health checks, synthetics and probes

Three mechanisms that look like observability but answer narrower questions:

**Health checks** (`Microsoft.Extensions.Diagnostics.HealthChecks`, `app.MapHealthChecks`) answer *"should the platform route traffic to / restart this instance?"* — they're **control signals for orchestrators**, not user-experience metrics.

- **Liveness** (`/health/live`): is the process able to make progress? Keep it trivial — no dependency checks — or a database blip restarts your whole fleet (a cascading failure, Module 13).
- **Readiness** (`/health/ready`): can this instance serve now? May check *local* prerequisites (warm-up done, configuration loaded, critical connections initialized) — but checking shared dependencies in readiness takes *all* instances out at once when the dependency hiccups.
- **Startup** probes protect slow starts from liveness kills.
- Exclude health endpoints from tracing and SLIs (Concept 60), and keep them cheap and unauthenticated only on internal ports.

**Synthetic monitoring** answers *"does the journey work from the outside, even with no users?"*

- **Application Insights standard availability tests**: HTTP checks from multiple Azure regions, with SSL expiry checks, results in the `availabilityResults` table and alertable.
- **Custom synthetic transactions**: a scheduled Function or Playwright test that logs in, searches and checks out against production (with a test tenant), reporting via `TrackAvailability` or as OTel spans tagged synthetic.
- Use them for **low-traffic services** (an SLI baseline — Concept 47), **critical journeys** (checkout, login), **external dependencies**, and **certificate/DNS expiry**.
- Tag synthetic traffic (`X-Synthetic: true` → an attribute) so you can include it in one SLI and exclude it from others.

**Probes from the edge** — Front Door and Application Gateway health probes — decide routing between regions and backends; their results are another outside-in signal.

**What none of them do:** tell you how *real* users experience the service. A green health check with a 30% checkout failure rate is common. Health checks and synthetics complement SLIs from real traffic; they don't replace them.

**The interview-grade sentence:** *"Health checks are control signals for orchestrators: liveness stays trivial with no dependency checks so a database blip doesn't restart the fleet, readiness checks only local prerequisites, and both are excluded from traces and SLIs. Synthetics — Application Insights availability tests or scripted Playwright journeys tagged as synthetic — give outside-in coverage for low-traffic services, critical journeys and certificate expiry. None of them measure real users, so they complement SLIs from real traffic rather than replace them."*

---

## Concept 68 — Incident response with telemetry

Telemetry exists for the incident; design it backwards from how incidents are run.

**1. Detect** — a burn-rate page (Concept 47) or a synthetic failure, routed to the owning on-call with a runbook link. The page text names the SLO, the burn rate, the window and a dashboard link.

**2. Triage (first five minutes)** — answer, from the top-level dashboard:
- *Who is affected?* All users or a segment (region, tenant tier, client version, endpoint)?
- *Since when?* Correlate with deploy markers, feature-flag changes, config changes, and Azure Service Health.
- *How bad?* Burn rate and budget remaining set the urgency.

**3. Mitigate before diagnosing.** If a deploy or flag change correlates, **roll back or turn it off** first (Module 13): restoring service beats understanding it. Other mitigations: fail over, shed load, scale out, disable a non-critical feature, route around a dependency. Telemetry's job here is **verification** — watch the SLI recover (Live Metrics, 1-minute metrics).

**4. Diagnose** — localize with service dashboards and the map, then go from exemplars and slow-trace comparisons to spans, logs and profiles (Concept 5's path). Keep an incident timeline as you go.

**5. Learn** — a **blameless postmortem**: timeline, impact (budget consumed), detection and response times, contributing factors, what went well, and action items — including **observability action items** ("we couldn't tell which tenant was affected for 20 minutes → add tenant tier to the request metric"). Every postmortem should ask: *what telemetry would have made this faster to detect, localize or diagnose?*

**Roles** for larger incidents: an **incident commander** (coordinates, decides), **operations lead** (hands on keyboard), **communications lead** (status page, stakeholders), and a scribe. Telemetry links in the incident channel make the shared picture.

**Metrics about incident response itself** — time to detect, time to mitigate, pages per shift, false-page rate, percentage of incidents first detected by customers — are how you know whether observability investment is working.

**The interview-grade sentence:** *"I design telemetry backwards from incidents: a burn-rate page with the SLO, burn rate and dashboard link; five minutes of triage on who's affected, since when — against deploy and flag markers — and how bad; mitigation by rollback, flag-off, failover or load shedding before diagnosis, verified on fast metrics; then localization and diagnosis from exemplars to traces, logs and profiles; and a blameless postmortem that always asks what telemetry would have shortened detection or diagnosis — tracked through time-to-detect, time-to-mitigate and how many incidents customers found first."*

---

## Concept 69 — Choosing an observability architecture

Three broad architectures, and a pattern that makes the choice reversible.

| | **Azure Monitor native** | **Open-source stack (Grafana LGTM or similar)** | **Third-party SaaS (Datadog, New Relic, Dynatrace, Honeycomb, Elastic, Grafana Cloud…)** |
|---|---|---|---|
| Components | Application Insights + Log Analytics + managed Prometheus + Grafana dashboards | Loki (logs), Grafana (UI), Tempo (traces), Mimir/Prometheus (metrics), Pyroscope (profiles) — self-run or Grafana Cloud | Vendor platform |
| Strengths | Integrated with Azure resources, RBAC, Entra, private links, policy; platform metrics and resource logs "for free"; one bill | Open formats and query languages; strong cost control at scale; no per-host licensing | Polished APM, strong correlation UX, fast time to value, multicloud |
| Weaknesses | Log Analytics ingestion pricing at high volume; some features tied to distros; KQL-specific assets | You operate it (or pay Grafana Cloud); more assembly | Cost at scale (per host, per GB, per span); data residency; another vendor |
| Fits | Azure-first estates, regulated environments wanting one cloud boundary | Kubernetes-heavy, cost-sensitive, multicloud platform teams | Polyglot multicloud companies valuing APM experience |

**The reversible pattern:** instrument with **OpenTelemetry APIs and semantic conventions**, export **OTLP to a Collector tier**, and let the Collector route to one or more backends. Then:

- evaluating a new backend is a Collector config change (dual-export for a month);
- migrating is a query/dashboard/alert rewrite, not a re-instrumentation;
- you can split by signal (metrics to managed Prometheus, traces to Application Insights, logs partly to Basic tables or a lake) and by cost tier.

**Questions that decide it:** Where does most of the estate run? What's the expected telemetry volume (GB/day) and its cost under each model? Data residency and access-control requirements? Which features are non-negotiable (APM UX, profiling, Live Metrics, security analytics with Sentinel)? Who will operate it? What skills does the team already have (KQL vs PromQL)?

**The interview-grade sentence:** *"I'd choose between Azure Monitor native — integrated with Entra, RBAC, private links and every resource's platform metrics, but expensive at high log volume — an open-source Grafana-style stack with open query languages and strong cost control that someone has to operate, or a SaaS vendor with polished APM at a per-host or per-GB price. Whatever the choice, I make it reversible: OpenTelemetry APIs and conventions in code, OTLP to a Collector tier that routes by signal and can dual-export, so changing backends is a query and dashboard migration rather than a re-instrumentation."*

---

## Concept 70 — Hidden costs

| Hidden cost | Triggered by | Mitigation |
|---|---|---|
| **Duplicate collection** | Console logs collected by the platform *and* exported via OTel; auto-instrumentation plus an in-code distro; two HTTP instrumentations | One path per signal per service (Concept 54) |
| **Health-check and probe telemetry** | Probes every few seconds × replicas × services, each a request span | Filter at instrumentation |
| **Verbose framework logging** | `Microsoft.*` at Information, EF Core command logging in production | Category filters |
| **Default sampling that's too high** | Unsampled traces on high-traffic services | Explicit sampling; tail sampling for quality |
| **Container Insights stdout** | Every pod's stdout ingested to Analytics tables | Cost-optimized profiles, namespace filters, Basic tables |
| **Resource diagnostic logs** | Diagnostic settings "all categories" on every resource (APIM, Front Door, Cosmos request logs, Key Vault) | Enable categories you query; resource-specific tables; Basic plans |
| **High-cardinality metrics** | Instance IDs, tenant IDs, raw routes | Budgets and views (Concept 28) |
| **Long interactive retention** | Workspace default retention applied to every table | Per-table retention; long-term tier |
| **Log query scans on Basic/Auxiliary** | Dashboards or alerts querying cheap tables frequently | Keep queried data in Analytics; use summary rules |
| **Alert rule sprawl** | Thousands of log alerts at 1-minute frequency | Consolidate; metric alerts where possible |
| **Profilers and snapshots** | Always-on at high frequency | Sampled, on-demand or triggered |
| **Collector compute** | Gateway tiers sized for peak with tail-sampling buffers | Autoscale; sampling at the edge |
| **Egress and private endpoints** | Cross-region telemetry export; private link per workspace and region | Regional workspaces; consolidate endpoints |
| **People time** | Unowned dashboards, noisy alerts, manual cost reviews | Ownership, archiving, budgets, showback |

**The interview-grade sentence:** *"The hidden telemetry costs are usually duplicates — platform console collection plus OTel logs, auto-instrumentation plus a distro — health-check spans, verbose framework categories, unsampled high-traffic traces, Container Insights stdout, every diagnostic-log category on every resource, high-cardinality metrics, workspace-wide long retention, frequent queries against cheap tables, alert sprawl, always-on profilers and Collector compute — plus the people time of unowned dashboards and noisy alerts, which ownership and showback address."*

---

## Concept 71 — Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Logging instead of tracing** | Per-method "entering/exiting" logs cost more and explain less than spans | Spans for flow; logs for decisions and failures |
| **String-interpolated log messages** | No templates, no properties, allocations when disabled | Message templates; source-generated `LoggerMessage` |
| **Log-and-rethrow at every layer** | One failure counted five times | Log once at the handling boundary |
| **High-cardinality metric attributes** | Series explosion, cost, backend instability | Bounded attributes; detail on spans |
| **Averaging percentiles** | Meaningless numbers on dashboards | Histograms; percentiles at query time |
| **Alerting on causes (CPU 80%)** | Noise; misses user-visible failures with healthy CPU | Page on SLO burn; ticket on causes |
| **Alerts without runbooks or owners** | Pages nobody can act on | Runbook link and owner per alert |
| **SLOs set by aspiration** | Permanently violated, ignored | Historical baseline, tighten over time |
| **SLIs computed from sampled traces** | Wrong error rates | Metrics from 100% of traffic |
| **Inconsistent service names** | Correlation and Application Map break | Shared resource configuration |
| **Ignoring propagation at async boundaries** | Orphan spans, logs without trace IDs | Capture and restore context; root activities for jobs |
| **Trusting external `traceparent` and baggage** | Cost attacks, forged context | New trace at the edge; allow-list baggage |
| **PII and secrets in telemetry** | Compliance exposure, credential leaks | Don't emit; classify and redact; scrub centrally |
| **Debug logging in production by default** | Cost and noise | Off by default; per-category on demand; buffering |
| **One giant dashboard** | Nobody finds anything during an incident | Hierarchy: journeys → services → components |
| **Daily cap as cost control** | Blind during the incident that caused the spike | Budgets, anomaly alerts, source reduction |
| **Vendor SDK calls in business code** | Lock-in in thousands of files | OTel APIs in code; vendor choice in startup |
| **Diagnostic logs as the audit trail** | Incomplete, mutable, expiring | Transactional, append-only audit store |
| **Telemetry that can block requests** | Observability outage becomes an application outage | Batching exporters, bounded queues, fail open |

---

## Concept 72 — The observability design review, and when not to

**A checklist to narrate** for any design:

**Objectives**
1. Which user journeys matter, and what are their SLIs, SLOs and windows (Concepts 43–44)?
2. What's the error budget policy, and who agreed to it (Concept 46)?
3. Which alerts page (burn rate), which create tickets, and do they all have runbooks (Concepts 47–48)?

**Instrumentation**
4. OpenTelemetry APIs and semantic conventions everywhere; vendor choice isolated to startup (Concepts 9, 11, 17)?
5. Consistent resource identity — service name, version, environment (Concept 10)?
6. Traces cross every boundary, including messages and background work, and stop at the trust boundary (Concepts 13, 21, 24, 64)?
7. Business metrics and business identifiers where they explain behavior (Concepts 23, 62)?

**Data model and cost**
8. Cardinality budget per metric; histograms with SLO-aligned buckets (Concepts 27–28)?
9. Logging policy: templates, levels, once-per-exception, no PII (Concepts 33–38)?
10. Sampling strategy, with SLIs computed from unsampled data (Concept 16)?
11. Telemetry volume estimate, table plans, retention and a per-service budget (Concepts 7, 58)?

**Pipeline and operations**
12. Export path — distro or OTLP via a Collector — with one collection path per signal (Concepts 15, 52–54)?
13. Pipeline health monitored; telemetry fails open (Concept 6)?
14. Dashboards hierarchy, deploy markers, and an incident process that uses them (Concepts 57, 68)?
15. Telemetry contract tests for what alerts depend on (Concept 66)?

**When not to:**

- **Don't build a Collector gateway tier for three services.** A distro exporting directly is fine until you need policy, multiple backends or tail sampling.
- **Don't tail-sample before you need to.** Rate-limited or ratio head sampling plus complete metrics covers most systems; tail sampling is infrastructure to run.
- **Don't write SLOs for everything.** A handful of journeys and critical dependencies; internal components get dashboards, not SLOs.
- **Don't chase five nines in telemetry either.** Losing a fraction of a percent of spans is acceptable; an SLI computed from metrics shouldn't be.
- **Don't add custom spans for every method or custom metrics for every number.** Instrumentation is code to maintain; add it where it answers questions you'll actually ask.
- **Don't build your own observability platform** when a managed one fits — the value is in using telemetry, not in operating storage engines.
- **Don't let observability become the product's biggest cloud line item** without a conscious decision.

**The meta-rule, echoing Modules 13, 26 and 27:** *every signal, attribute, dashboard and alert you add is also a cost, a maintenance burden and something that can mislead.* The best observability design answers the questions your incidents actually ask — detection from complete, cheap metrics tied to SLOs; diagnosis from correlated, richly attributed traces and logs — at a deliberate price, with every page earning its right to wake someone up.

**The interview-grade sentence:** *"I review observability from objectives down: journeys with SLIs, SLOs and an agreed budget policy, burn-rate pages with runbooks; then instrumentation — OTel APIs and conventions, consistent resource identity, traces across messages and background work but not across the trust boundary, business metrics; then the data model and cost — cardinality budgets, SLO-aligned histograms, a logging policy, sampling with SLIs from unsampled data and a volume budget; then the pipeline, dashboards, incident process and contract tests. And I hold back: no gateway tier or tail sampling before they're needed, SLOs only for journeys that matter, and no span or metric that doesn't answer a question we'll actually ask."*

---
# Putting it together

## Worked example 1 — "Design the observability for an order platform"

*Same platform as Module 27's Worked example 1: 2 million orders a day, peaks of 250 orders/s, services for catalog, orders, payments, inventory, shipping and notifications on Container Apps and AKS, Service Bus Premium, Event Hubs for clickstream, Cosmos DB. Users mainly in the EU. The CTO asks: "How will we know it's working — and what will it cost?"*

Narrate in this order:

**1. Journeys and SLOs first** (Concepts 43–46):

| Journey | SLI | SLO (28-day rolling) |
|---|---|---|
| Browse catalog | Availability of `GET /products*` (valid = non-4xx-client), latency ≤ 300 ms | 99.9% / 99% |
| Checkout | Availability of `POST /checkout` (my 429s count as bad), latency ≤ 800 ms | 99.95% / 99% |
| Order confirmation | % of orders whose confirmation email is sent within 2 min of `OrderPlaced` event time | 99.5% |
| Payment capture | % of authorized payments captured within 10 min | 99.9% |
| Personalization stream | Consumer lag ≤ 60 s for the `personalize` consumer group, time-based (minutes) | 99% of minutes |

Sanity check: checkout depends on orders-api, Cosmos (99.99% Business Critical with PPAF), payments provider (contractually 99.9%). The payments provider caps synchronous checkout below 99.95% — so checkout **authorizes** synchronously but **captures** asynchronously via Service Bus (the payment capture journey), and the authorization call has a circuit breaker and a "pay later" fallback for wallets that support it. Say this aloud: *the SLO changed the architecture*.

**2. Instrumentation** (Concepts 14, 52, 60–64):

- One shared `AddObservability()` in a ServiceDefaults project: resource (`service.name`, `service.namespace = shop`, version from the build, environment), ASP.NET Core and HttpClient tracing, Azure SDK sources (Service Bus, Cosmos, Event Hubs), runtime and hosting metrics, `ILogger` bridge with scopes, health checks filtered.
- Exporter: **OTLP to a Collector** (agent DaemonSet on AKS, Collector app in the Container Apps environment), so the services stay backend-neutral.
- Manual instrumentation: `Contoso.Orders`, `Contoso.Payments` sources for domain operations; business metrics — `contoso.orders.placed`, `contoso.payments.authorized` / `.declined` by provider, `contoso.checkout.duration` with buckets at 0.3, 0.5, 0.8, 1, 2 s.
- Propagation: HTTP and Service Bus automatic; the Cosmos change-feed outbox relay (Module 27, Concept 41) starts a consumer span per batch with links and puts `contoso.order.id` on spans; notification workers compute `now − OrderPlaced.time` into `contoso.confirmation.e2e_latency` (seconds, bucket at 120).
- Edge: Front Door → APIM → services; APIM starts a new trace for external callers, strips baggage, logs to Application Insights at 5% sampling.

**3. Pipeline and backends** (Concepts 15, 53, 69):

- Collector gateway: `memory_limiter`, `k8sattributes`, a `transform` redaction step (authorization headers, query-string tokens), `spanmetrics` (RED from 100% of spans), then **tail sampling**: keep errors, traces > 1 s, all payments traces, 5% baseline.
- Exports: traces and logs to **Azure Monitor** (Application Insights with OTLP support), metrics to an **Azure Monitor workspace** (managed Prometheus); Grafana dashboards in the portal.
- Collector health: refused spans, queue size, exporter failures alerted as tickets.

**4. Alerts** (Concepts 47–48, 56): Sloth-generated Prometheus rule groups per SLO — fast burn (14.4 over 1 h/5 min) and slow burn (6 over 6 h/30 min) page; 10%-in-3-days tickets. Additional pages: DLQ growth on `orders-events` subscriptions, Cosmos 429 rate > 5% sustained on the orders container (saturation of a page-worthy dependency), certificate expiry < 7 days. Business anomaly: orders placed per 5 min < 50% of the same window last week → page (silent checkout failure).

**5. Logging policy** (Concepts 33–39): source-generated `LoggerMessage` with EventIds; `Information` for business events and lifecycle only; `Microsoft.*` at `Warning`; per-request log buffering at `Information`, flushed on failure; redaction with data classifications on customer fields; console log collection by the platform **disabled** for services that export logs via OTel.

**6. Cost estimate** (Concept 7):

```
Traces: peak ~3,000 req/s across services × ~8 spans × ~1 KB ≈ 24 MB/s at peak; daily average ~40% of peak
        → ~830 GB/day unsampled; tail sampling keeps ~8% → ~65 GB/day
Logs:   ~0.5 records per request after policy, ~0.6 KB → ~30 GB/day; verbose job logs → Basic plan ~20 GB/day
Metrics: ~150k active series in managed Prometheus (sample-based pricing)
Log Analytics: ~95 GB/day Analytics ≈ $6,500/month at list (less with a 100 GB/day commitment tier)
             + ~20 GB/day Basic ≈ $300/month
```

Per-service ingestion budgets with an 80% alert, showback by `service.name`.

**7. Close with what you're not doing:** no SLOs on internal services like pricing (they get dashboards), no profiler always-on (on-demand with `dotnet-monitor` triggers), no custom spans per repository method, no daily cap.

---

## Worked example 2 — "p99 latency doubled after a deploy. Walk me through it."

*Checkout p99 went from 600 ms to 1.3 s twenty minutes after version 2.8.0 rolled out to 50% of replicas. Error rate is unchanged.*

1. **Confirm and scope** (Concept 2): the checkout latency SLI is burning at ~4× — a slow-burn ticket, not yet a fast page — but trending up. Split the latency histogram by `service.version`: 2.7.x p99 is 610 ms, **2.8.0 p99 is 1.9 s**. Strong signal it's the deploy.
2. **Mitigate first** (Concept 68): halt the rollout and shift traffic back to 2.7.x (Container Apps revision weights, or the canary controller). Watch the SLI recover within minutes. Now diagnose without pressure.
3. **Localize**: in Application Insights' Performance blade (or a TraceQL/KQL query), compare the dependency breakdown of `POST /checkout` between versions. 2.8.0 spends an extra ~900 ms in `Cosmos Query orders`.
4. **Diagnose with traces** (Concept 22): open an exemplar from the slow bucket. The waterfall shows a **ladder** — 24 sequential `Cosmos ReadItem` spans where 2.7.x had one query. That's an N+1 introduced in the new loyalty-points code (Module 19).
5. **Confirm with attributes**: slow traces have `contoso.order.line_count` > 10; RU per checkout tripled (Cosmos `RequestCharge` logged at Debug, or the Cosmos diagnostics for slow operations).
6. **Fix and verify**: batch the reads (a single query with `IN`, or a point-read fan-out with bounded concurrency), add a test asserting at most N dependency spans per checkout (Concept 66), redeploy as a canary, compare versions on the same histogram.
7. **Postmortem action items**: deploy markers on dashboards (they existed — good), an automated canary analysis gate comparing p99 by version, and a performance test with large carts.

---

## Worked example 3 — "Our Application Insights bill is $40,000 a month"

1. **Measure** (Concept 58): `Usage` by table for 30 days → `AppTraces` 55%, `AppDependencies` 25%, `AppRequests` 10%, `ContainerLogV2` 8%. By `AppRoleName`: two services produce 60% of `AppTraces`.
2. **Find the duplicates** (Concept 70): `ContainerLogV2` contains the same records as `AppTraces` — the services log to console *and* through OTel. Disable console collection for those namespaces (or route `ContainerLogV2` to Basic). Saving: ~8%.
3. **Fix the logging policy**: the two noisy services log every Cosmos call at `Information` and run `Microsoft.EntityFrameworkCore.Database.Command` at `Information`. Move to `Warning`; adopt per-request buffering. `AppTraces` drops ~70%.
4. **Fix sampling**: services set `SamplingRatio = 1.0` long ago; dependencies include health checks and Redis `GET`s at 4,000/s. Filter health checks; set rate-limited or 10% ratio sampling; keep standard metrics for the Performance blade (computed pre-sampling). `AppDependencies` and `AppRequests` drop ~85%.
5. **Tier and retain**: verbose job logs to a Basic table via a DCR transformation; retention 30 days for traces, 90 for requests/exceptions, long-term only for an audit-relevant custom table.
6. **Commitment tier** once volume is stable.
7. **Govern**: per-service ingestion budgets, monthly showback, an anomaly alert at 3× the 7-day average per table.
8. **Result to quote**: typically 60–85% reduction, with *better* diagnosis because the signal-to-noise ratio improved. Don't touch: error logs, exceptions, metrics used by SLOs.

---

## Worked example 4 — "Define the SLO and the page for a payment API"

*`POST /payments/authorize`, ~120 req/s average, ~400 req/s peak, called by checkout.*

1. **SLIs** (Concept 44):
   - availability: valid = authorize requests excluding 400/401/403/422 (client errors) and synthetic; good = status < 500 and not a 429 from our own limiter and not a 504 from APIM;
   - latency: good = server duration ≤ 1.0 s (the provider call dominates; histogram bucket at 1.0).
2. **Targets**: history shows 99.97% availability and 99.4% under 1 s over the last six months → SLO **99.95% availability**, **99% ≤ 1 s**, rolling 28 days.
3. **Budget**: 120 req/s × 86,400 s ≈ 10.4 M requests/day, × 28 days ≈ 290 M per window → **~145,000 failed authorizations** allowed per window at 99.95%.
4. **Burn-rate alerts** (Concept 47, scaled to a 28-day window, 672 h):
   - fast page: 2% of budget in 1 h → burn = 0.02 × 672 / 1 ≈ **13.4**; error ratio threshold = 13.4 × 0.0005 ≈ **0.67%** over 1 h *and* 5 min;
   - slow page: 5% in 6 h → burn = 0.05 × 672 / 6 = **5.6** → 0.28% over 6 h and 30 min;
   - ticket: 10% in 3 days → burn ≈ 0.93 → 0.047% over 3 days and 6 h.
5. **Detection time** for a full outage: 0.67% of 1 h ≈ 24 seconds of 100% errors, plus evaluation interval → under 2 minutes with metric-based rules.
6. **Dependency attribution**: a per-provider SLI (`contoso.payment.provider` on the metric — a bounded set of four) so the incident review knows whether the provider or our service burned the budget.
7. **Policy**: below 25% budget remaining, provider-integration changes need review; exhausted → freeze on payments features; repeated provider-caused exhaustion → escalate to add a second provider with failover.

---

## Worked example 5 — "We're on Application Insights SDK 2.x. What's the migration?"

1. **State the landscape** (Concept 52): 2.x is legacy; **SDK 3.x (GA February 2026)** keeps `TelemetryClient` on top of OpenTelemetry; the **Azure Monitor** and **Microsoft OpenTelemetry** distros are the OpenTelemetry-native paths; never mix them in one app.
2. **Inventory**: packages (`Microsoft.ApplicationInsights.*`, `DependencyCollector`, `PerfCounterCollector`, `Microsoft.Extensions.Logging.ApplicationInsights`, `Serilog.Sinks.ApplicationInsights`), custom `ITelemetryInitializer`s and `ITelemetryProcessor`s, adaptive sampling settings, `TrackEvent`/`TrackMetric`/`TrackDependency` call sites, `TrackPageView` use, tests mocking `ITelemetryChannel`, KQL alerts and workbooks that depend on property names.
3. **Choose per service**: heavy `TelemetryClient` usage and little time → **SDK 3.x** (remove discontinued packages, upgrade all AI packages together, connection strings, `SamplingRatio`/`TracesPerSecond`, port initializers to OTel processors); otherwise → **distro** and replace `Track*` with `ActivitySource`, `Meter` and `ILogger` (custom events via `microsoft.custom_event.name`).
4. **Port extensibility**: initializers → span processors or resource attributes (role name via `service.name`); filtering processors → instrumentation filters or sampler; `GlobalProperties` for per-item constants.
5. **Validate side by side**: run the new version in a canary sending to the same resource (or a parallel one), compare request counts, dependency counts, exception counts, role names and sampling; rewrite alerts and workbooks where names changed (metric names must now follow OTel instrument naming).
6. **Tests**: replace channel mocks with the in-memory exporter (Concept 66).
7. **Then improve**: once on OpenTelemetry, decide whether to move export to a Collector for vendor neutrality.

---

## Common interview questions, with model answers

**"What's the difference between monitoring and observability?"** Monitoring answers questions you predicted — thresholds and dashboards for known failure modes. Observability is being able to answer new questions about production from existing telemetry without shipping code, which you need because distributed-system failures are emergent. You need both: cheap metrics and SLO alerts for detection, rich correlated traces and logs for diagnosis (Concepts 1–2).

**"Why can't I put the user ID on this metric?"** Metric cost is the product of every attribute's distinct values, so millions of user IDs multiply the series count by millions — memory, storage, query time and the bill explode. Put user and tenant IDs on spans and log records, which are paid per record; keep metrics to bounded dimensions; bucket tenants into tiers if you need them on metrics (Concepts 4, 28).

**"How do you trace a request through a Service Bus queue?"** The producer's send span injects trace context into message application properties — the Azure SDK writes `Diagnostic-Id`/`traceparent` automatically when tracing is enabled. The consumer's process span uses it: as a child for one-to-one, short-delay delivery, otherwise a new trace with a link — always links for batch receives. I add the business correlation ID to every span and measure queue time separately from the enqueued timestamp (Concepts 13, 21).

**"Head or tail sampling?"** Head sampling decides at the root — cheap, consistent via parent-based sampling, but blind to outcomes. Tail sampling decides after the trace completes in a Collector gateway — keeps errors and slow traces — but needs buffering and trace-ID-affine routing. Common answer: modest head sampling or none at the edge, tail sampling at the gateway, and SLIs from unsampled metrics either way (Concept 16).

**"Our p99 went up after a deploy — what do you do?"** Confirm with the SLI and split the latency histogram by `service.version`; mitigate first by halting or rolling back; then localize by comparing dependency breakdowns between versions, open an exemplar trace from the slow bucket, read the waterfall shape, confirm with attributes across many traces, fix, canary and compare (Worked example 2).

**"Define an SLO for this API and the alert you'd page on."** Good-over-valid availability and latency SLIs with explicit status-code rules, a target from historical performance, a rolling 28-day window, and multi-window multi-burn-rate alerts — page at 2% of budget in an hour (burn ~14) confirmed over 5 minutes and 5% in 6 hours (burn ~6) confirmed over 30 minutes, ticket at 10% in 3 days (Concepts 44, 47, Worked example 4).

**"Why not just alert when CPU is above 80%?"** Because CPU is a cause, not a symptom — users can suffer with healthy CPU, and CPU can be high with happy users. Page on SLO burn; use saturation signals like CPU, thread-pool queue length and connection-pool waits for tickets and dashboards, or as pages only when they predict user impact faster than a human can respond (Concept 48).

**"What's an error budget for, really?"** It turns reliability-vs-velocity into arithmetic: within budget, ship and take risk; exhausted, the pre-agreed policy freezes features and prioritizes reliability; consistently unspent, you can move faster or tighten the SLO (Concepts 41, 46).

**"How do you correlate logs with traces in .NET?"** Use the OpenTelemetry `ILogger` provider: records written inside an `Activity` carry its trace and span IDs automatically. Make sure background work starts an activity per unit of work, and add business IDs as scope properties (Concept 36).

**"Application Insights SDK, Azure Monitor distro, or the Microsoft distro?"** New ASP.NET Core work: one of the distros — the Microsoft OpenTelemetry Distro if I want OTLP alongside Azure Monitor or AI-agent sources, the Azure Monitor distro otherwise. Existing heavy `TelemetryClient` code: SDK 3.x, which runs on OpenTelemetry. Never two in one app. Vendor-neutral estate: upstream SDK, OTLP to a Collector, Azure Monitor OTLP ingestion (Concepts 52–53).

**"How do you keep telemetry costs under control?"** Estimate volume like capacity; don't emit noise (health checks, debug, duplicates); aggregate into metrics; sample traces explicitly; buffer or sample logs; table plans and per-table retention; commitment tiers; per-service budgets and showback; no daily cap as the primary control (Concepts 7, 39, 58, Worked example 3).

**"How do you make sure telemetry doesn't take the service down?"** Batching exporters with bounded queues that drop, never block; flush on graceful shutdown; low-allocation logging; processors that can't throw; a Collector with `memory_limiter`; and monitoring of dropped telemetry — observability is best-effort and fail-open (Concept 6).

**"What would you put on a span?"** What explains behavior and what you'd filter by in an incident: tenant, order, feature-flag and dependency details, decisions and retry counts, under namespaced names; never secrets, tokens, payloads or unneeded PII; mind the 128-attribute default (Concept 23).

**"How do SLOs work for an event-driven system?"** Time-to-effect SLIs measured from the event time carried in the message — e.g. 99.5% of confirmations sent within 2 minutes — plus oldest-message age or consumer lag in time, dead-letter rate as correctness, and time-based SLOs for batch deadlines; queue depth is a scaling signal, not an SLI (Concept 49).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "We'll add logging and dashboards" | Starts with user journeys, SLIs and SLOs, then derives what to emit and what to page on |
| Treats logs, metrics and traces as three tools | Treats them as data structures with cost models, joined by trace IDs, resource identity and exemplars |
| Puts user or tenant IDs on metrics | Gives the cardinality product, keeps metrics bounded, puts identifiers on spans |
| Averages percentiles | Records histograms, merges buckets, computes percentiles at query time, aligns buckets with SLO thresholds |
| Computes error rates from sampled traces | Computes SLIs from unsampled metrics; samples traces for diagnosis only |
| "We use Application Insights" | Names the instrumentation path in 2026 — distro, Microsoft distro or SDK 3.x — and why |
| Vendor SDK calls throughout business code | OTel APIs and semantic conventions in code; vendor choice isolated in startup or a Collector |
| Unaware of semantic-convention churn | Builds on stable conventions, pins versions, uses `OTEL_SEMCONV_STABILITY_OPT_IN` during renames |
| Assumes traces "just work" through queues | Explains producer/consumer spans, parents vs links, batch links and background-work propagation |
| Trusts incoming `traceparent` and baggage | Starts a new trace at the edge, ignores external sampling flags, allow-lists baggage |
| Interpolated log strings, log-and-rethrow | Message templates, source-generated `LoggerMessage` with EventIds, log once at the handling boundary |
| Logs as the audit trail | Separate transactional, append-only audit store |
| PII "will be cleaned up later" | Doesn't emit it; classification and redaction in-process; scrubbing in the pipeline as a backstop |
| SLO targets from aspiration ("five nines") | Targets from history and dependencies, with the composition arithmetic aloud |
| Threshold alert on error rate over 5 minutes | Multi-window, multi-burn-rate alerts with the derivation and detection time |
| Pages on CPU and disk | Pages on symptoms, tickets on causes, every page actionable with a runbook |
| Queue depth as the async SLI | Event-time-to-effect latency, oldest-message age and lag in time |
| Ignores the telemetry bill | Estimates GB/day, picks table plans, sampling and budgets, and knows the levers |
| Daily cap as cost control | Source reduction, transformations, budgets and anomaly alerts; caps only as last resort |
| Diagnoses before mitigating | Rolls back or flags off first, verifies on fast metrics, then diagnoses |
| "We'd look at the logs" for performance | Reads trace shapes (ladders, staircases, retries, gaps), compares attribute distributions, then profiles |
| No idea what .NET emits natively | Knows ASP.NET Core, Kestrel, HttpClient and `System.Runtime` meters and `dotnet-counters` |
| Instruments everything | Instruments what answers questions; knows when a Collector tier, tail sampling or an SLO isn't worth it |

---

## Practice exercises

**Exercise 1 — Instrument a two-service system end to end (half a day).** Create an ASP.NET Core API that calls a second API and publishes a message to the Service Bus emulator (or a `Channel<T>` consumed by a `BackgroundService`). Wire OpenTelemetry with `UseOtlpExporter()` and run the Aspire dashboard standalone (`aspire dashboard run` or the container). Verify one trace spans both APIs and the consumer. Then break propagation in the background consumer, observe the orphan spans and logs without trace IDs, and fix it with explicit context capture (Concept 64).

**Exercise 2 — Feel cardinality (1 hour).** Add a `Counter<long>` with `contoso.user.id` as a tag; generate 10,000 distinct users with a load script; watch SDK memory and the `otel.metric.overflow` series appear at the cardinality limit. Replace the tag with a bounded tier and move the user ID to the span. Write the cardinality budget for three metrics.

**Exercise 3 — Percentiles that lie (1 hour).** Record durations from two simulated pods with different distributions into histograms. Compute the fleet p99 from merged buckets, then compute the average of the two pods' p99s, and explain the difference. Repeat with default millisecond buckets for second-valued data and observe the failure; fix with `InstrumentAdvice` boundaries aligned to a 300 ms SLO threshold.

**Exercise 4 — Burn-rate alerts from scratch (2 hours).** For a 99.9% SLO over 30 days, derive the 14.4/6/1 burn-rate thresholds. Write the Prometheus recording and alerting rules (or generate them with Sloth), and the equivalent KQL log alert. Simulate a full outage, a 5% partial outage and a 0.2% slow burn with a fault-injection flag; record detection time and reset time for each alert.

**Exercise 5 — Logging under load (1–2 hours).** Benchmark (BenchmarkDotNet) a classic `LogInformation` with three value-type arguments vs a source-generated `[LoggerMessage]` method, with the level enabled and disabled. Then add per-request log buffering and verify that only failed requests ship their Information-level trail.

**Exercise 6 — Redaction (1 hour).** Define a `[PersonalData]` classification, apply it to a logging method parameter, enable redaction with an erasing redactor, and prove with `FakeLogger` that the email never appears. Swap to an HMAC redactor and show the same email produces the same pseudonym.

**Exercise 7 — Tail sampling with a Collector (half a day).** Run the Collector (contrib or a custom build) with `spanmetrics` before `tail_sampling` (errors, > 500 ms, 5% baseline). Generate mixed traffic. Show that RED metrics from `spanmetrics` match the true request count while stored traces are ~8% of the volume and include every error.

**Exercise 8 — KQL drills (1 hour, against any Application Insights resource or the demo data).** Write: rate/failure/p95 per route in 5-minute bins; top 20 dependencies by p99; a union of all telemetry for one `operation_Id`; slow-vs-total share by a custom dimension; `Usage` by table for 30 days.

**Exercise 9 — An SLO document (1 hour).** For a service you know, write the one-page SLO document: journeys, SLI queries with explicit status-code rules, targets with historical justification, window, exclusions, budget policy, alerts and runbooks, owner, review date. Check the target against the product of its hard dependencies' availability.

**Exercise 10 — A cost review (1 hour).** Using Worked example 3's method on a real or sample workspace, produce a table of the top five ingestion sources, the action for each, and the expected saving — and list what you deliberately won't cut.

**Exercise 11 — Telemetry contract tests (1 hour).** Write tests with `MetricCollector<T>`, `FakeLogger` and the in-memory trace exporter that would fail if someone renamed `contoso.checkout.duration`, changed its unit, removed EventId 1299, or dropped `contoso.order.id` from the `PriceOrder` span.

---

## Free resources and learning material

### Foundations — SRE, SLOs and alerting

| Resource | What it covers |
|---|---|
| [Google SRE Book — table of contents](https://sre.google/sre-book/table-of-contents/) (free) | The origin of SLOs, error budgets and the golden signals |
| [SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) | SLI/SLO/SLA definitions and choosing targets (Concepts 41–44) |
| [SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) | Symptoms vs causes, the four golden signals (Concepts 29, 48) |
| [SRE Book — Embracing Risk](https://sre.google/sre-book/embracing-risk/) | Why 100% is the wrong target; error budgets (Concept 41) |
| [The Site Reliability Workbook — contents](https://sre.google/workbook/table-of-contents/) (free) | The practical companion |
| [Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/) | The SLI menu, measurement points, SLO documents (Concepts 43–44, 50) |
| [Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) | Multi-window, multi-burn-rate alerts and the 14.4/6/1 derivation (Concept 47) |
| [Workbook — Example error budget policy](https://sre.google/workbook/error-budget-policy/) | A real policy template (Concept 46) |
| [The Art of SLOs (workshop materials)](https://sre.google/resources/practices-and-processes/art-of-slos/) | Free workshop slides and exercises |
| [Rob Ewaschuk — My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit) | The classic on actionable pages (Concept 48) |
| [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) | Utilization, saturation, errors (Concept 29) |
| [Tom Wilkie — The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) | Rate, errors, duration (Concept 29) |
| [OpenSLO specification](https://openslo.com/) | A vendor-neutral SLO definition format |
| [Sloth](https://github.com/slok/sloth) · [Pyrra](https://github.com/pyrra-dev/pyrra) | Generate Prometheus burn-rate rules from SLO specs |
| [Azure Cloud Adoption Framework — service level objectives](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/manage/monitor/service-level-objectives) | Microsoft's SLO guidance for Azure estates |
| [Azure Well-Architected — reliability targets](https://learn.microsoft.com/en-us/azure/well-architected/reliability/metrics) | Defining reliability targets and composite SLAs on Azure |
| [Azure Well-Architected — mission-critical health modeling](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-health-modeling) | Health models layered from components to user flows |

### Foundational papers and talks

| Resource | What it covers |
|---|---|
| [Dapper, a Large-Scale Distributed Systems Tracing Infrastructure (Google, 2010)](https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/) | The paper behind every modern tracer (Concept 18) |
| [Gil Tene — How NOT to Measure Latency (talk)](https://www.youtube.com/watch?v=lJ8ydIuPFeU) | Percentiles, coordinated omission (Concept 27) |
| [Martin Fowler — Domain-Oriented Observability](https://martinfowler.com/articles/domain-oriented-observability.html) | Keeping instrumentation clean in domain code (Concepts 61–62) |
| [Charity Majors — Is it time to version observability?](https://charity.wtf/2024/08/07/is-it-time-to-version-observability-signs-point-to-yes/) | The wide-events / "observability 2.0" argument (Concept 3) |
| [Uber Engineering — Evolving Distributed Tracing at Uber](https://www.uber.com/blog/distributed-tracing/) | Building and scaling Jaeger in practice |

### OpenTelemetry — concepts and specification

| Resource | What it covers |
|---|---|
| [OpenTelemetry documentation](https://opentelemetry.io/docs/) | Entry point |
| [Signals overview](https://opentelemetry.io/docs/concepts/signals/) · [Traces](https://opentelemetry.io/docs/concepts/signals/traces/) | Spans, kinds, status, events, links (Concepts 18–20) |
| [Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/) | Propagators, baggage (Concept 13) |
| [Sampling](https://opentelemetry.io/docs/concepts/sampling/) | Head vs tail, consistency (Concept 16) |
| [Profiles signal](https://opentelemetry.io/docs/concepts/signals/profiles/) · [Profiles enters public Alpha (2026)](https://opentelemetry.io/blog/2026/profiles-alpha/) | The fourth signal (Concept 3) |
| [Specification](https://opentelemetry.io/docs/specs/otel/) | The normative reference |
| [Metrics supplementary guidelines](https://opentelemetry.io/docs/specs/otel/metrics/supplementary-guidelines/) | Instrument choice, temporality, cardinality (Concepts 25–26) |
| [OTLP specification](https://opentelemetry.io/docs/specs/otlp/) | Protocol, retries, partial success (Concept 12) |
| [Semantic conventions](https://opentelemetry.io/docs/specs/semconv/) | The registry (Concept 11) |
| [HTTP semconv](https://opentelemetry.io/docs/specs/semconv/http/) · [Database semconv](https://opentelemetry.io/docs/specs/semconv/database/) · [Messaging semconv](https://opentelemetry.io/docs/specs/semconv/messaging/) | Attribute names and span rules for the domains you use most |
| [W3C Trace Context](https://www.w3.org/TR/trace-context/) · [W3C Baggage](https://www.w3.org/TR/baggage/) | The header formats (Concept 13) |
| [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) | A polyglot microservices app fully instrumented — great for exploring |
| [Grafana — database semantic conventions become stable](https://grafana.com/blog/2025/06/06/database-observability-how-opentelemetry-semantic-conventions-improve-consistency-across-signals) | Why stable conventions matter, with examples |

### The OpenTelemetry Collector

| Resource | What it covers |
|---|---|
| [Collector documentation](https://opentelemetry.io/docs/collector/) | Components and configuration (Concept 15) |
| [Collector deployment patterns](https://opentelemetry.io/docs/collector/deployment/) | Agent, gateway, no-Collector |
| [Scaling the Collector](https://opentelemetry.io/docs/collector/scaling/) | Load balancing, tail-sampling topology |
| [Tail sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor) | Policies and memory considerations (Concept 16) |
| [Transform processor (OTTL)](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor) | Redaction and reshaping (Concepts 15, 38) |
| [Span metrics connector](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/spanmetricsconnector) | RED metrics from spans before sampling |
| [Collector contrib repository](https://github.com/open-telemetry/opentelemetry-collector-contrib) | All community components, including Azure exporters |

### .NET — OpenTelemetry and diagnostics

| Resource | What it covers |
|---|---|
| [OpenTelemetry .NET docs](https://opentelemetry.io/docs/languages/dotnet/) | Getting started, instrumentation, exporters |
| [open-telemetry/opentelemetry-dotnet](https://github.com/open-telemetry/opentelemetry-dotnet) · [docs folder](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/docs) | SDK source and in-depth guides for traces, metrics and logs |
| [opentelemetry-dotnet-contrib](https://github.com/open-telemetry/opentelemetry-dotnet-contrib) | Instrumentation libraries and resource detectors |
| [opentelemetry-dotnet-instrumentation](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation) | Zero-code instrumentation (Concept 14) |
| [.NET observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel) | Microsoft's overview for .NET |
| [Distributed tracing concepts](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing-concepts) · [instrumentation walkthroughs](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing-instrumentation-walkthroughs) | `Activity`/`ActivitySource` in depth (Concepts 59, 61) |
| [Creating metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/metrics-instrumentation) | `IMeterFactory`, instruments, testing with `MetricCollector` (Concepts 62, 66) |
| [Built-in metrics in .NET](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics) | ASP.NET Core, Kestrel, System.Net, runtime meters (Concept 32) |
| [ASP.NET Core metrics](https://learn.microsoft.com/en-us/aspnet/core/log-mon/metrics/metrics) | Enrichment with `IHttpMetricsTagsFeature`, testing |
| [Logging in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging) | The `ILogger` pipeline (Concept 34) |
| [Compile-time logging source generation](https://learn.microsoft.com/en-us/dotnet/core/extensions/logger-message-generator) · [High-performance logging](https://learn.microsoft.com/en-us/dotnet/core/extensions/high-performance-logging) | `[LoggerMessage]` (Concept 35) |
| [Log buffering in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging/log-buffering) | Global and per-request buffering (Concept 39) |
| [Data redaction in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/data-redaction) · [Compliance libraries](https://learn.microsoft.com/en-us/dotnet/core/extensions/compliance) | Classification and redactors (Concept 38) |
| [dotnet-counters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters) · [dotnet-trace](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) · [dotnet-dump](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump) · [dotnet-monitor](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-monitor) | The diagnostics toolbox (Concept 65) |
| [Grafana — OpenTelemetry support for .NET 10](https://grafana.com/blog/opentelemetry-support-for-net-10-a-behind-the-scenes-look) | How .NET and OTel releases line up |
| [Aspire — standalone dashboard](https://aspire.dev/dashboard/standalone/) · [What's new in Aspire 13.3](https://aspire.dev/whats-new/aspire-13-3/) | Local OTLP dashboard for any app (Concept 57) |
| [Aspire 13.2 — dashboard export and telemetry](https://devblogs.microsoft.com/aspire/aspire-dashboard-improvements-export-and-telemetry/) | Telemetry export, GenAI visualizer |

### Azure Monitor and Application Insights

| Resource | What it covers |
|---|---|
| [Azure Monitor documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/) | Entry point (Concept 51) |
| [OpenTelemetry with Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/opentelemetry-options) | The two approaches: native OTLP and the Microsoft distro (Concepts 52–53) |
| [OTLP ingestion options](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/opentelemetry-summary) · [via the Collector](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/opentelemetry-protocol-ingestion) · [AKS with OTLP](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-open-protocol) | The OTLP-native path (Concept 53) |
| [Enable Azure Monitor OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable) · [Configuration](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration) · [Add and modify telemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-add-modify) | The distro: setup, role names, sampling, custom telemetry |
| [Azure.Monitor.OpenTelemetry.AspNetCore README](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/monitor/Azure.Monitor.OpenTelemetry.AspNetCore) | Options, sampling, custom events, troubleshooting |
| [Microsoft OpenTelemetry Distro for .NET](https://github.com/microsoft/opentelemetry-distro-dotnet) | `UseMicrosoftOpenTelemetry`, export targets, AI-agent sources |
| [Migrate from Application Insights classic SDKs](https://learn.microsoft.com/en-us/azure/azure-monitor/app/migrate-to-opentelemetry) · [ApplicationInsights-dotnet (3.x)](https://github.com/microsoft/ApplicationInsights-dotnet) | SDK 3.x and the 2.x→3.x breaking changes (Worked example 5) |
| [Live Metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/app/live-stream) · [Dashboards with Grafana in Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/grafana-dashboards) | Investigation and dashboards (Concept 57) |
| [Use OpenTelemetry with Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/opentelemetry-howto) · [Container Apps OpenTelemetry agents](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents) | Compute-specific collection (Concept 54) |
| [Azure Monitor managed Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview) · [Azure Managed Grafana](https://learn.microsoft.com/en-us/azure/managed-grafana/overview) | Metrics store and visualization (Concept 31) |
| [Alerts overview](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview) · [Workbooks](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview) | Alerting and reports (Concepts 56–57) |
| [KQL reference](https://learn.microsoft.com/en-us/kusto/query/) · [Get started with log queries](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/get-started-queries) | Query language (Concept 55) |
| [Azure Monitor Logs cost calculations and options](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs) · [Azure Monitor pricing](https://azure.microsoft.com/pricing/details/monitor/) | Table plans, commitment tiers, retention (Concept 58) |
| [Troubleshoot high data ingestion in Application Insights](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/troubleshoot-high-data-ingestion) | The cost-review playbook (Worked example 3) |
| [Application Insights Profiler and Code Optimizations](https://learn.microsoft.com/en-us/azure/azure-monitor/profiler/profiler-overview) | Code-level CPU and memory bottleneck analysis (Concept 65) |
| [Azure Well-Architected — observability (operational excellence)](https://learn.microsoft.com/en-us/azure/well-architected/operational-excellence/observability) | Designing a monitoring system on Azure |

### Open-source backends and the metrics model

| Resource | What it covers |
|---|---|
| [Prometheus — overview](https://prometheus.io/docs/introduction/overview/) · [Querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/) | Pull model and PromQL (Concept 31) |
| [Prometheus — Histograms and summaries](https://prometheus.io/docs/practices/histograms/) | Why histograms aggregate and summaries don't (Concept 27) |
| [Prometheus — Metric and label naming](https://prometheus.io/docs/practices/naming/) · [Alerting practices](https://prometheus.io/docs/practices/alerting/) | Naming and symptom-based alerting |
| [Jaeger documentation](https://www.jaegertracing.io/docs/) · [Grafana Tempo documentation](https://grafana.com/docs/tempo/latest/) | Open-source trace backends |
| [Grafana Loki documentation](https://grafana.com/docs/loki/latest/) | Label-indexed logs and its cardinality rules |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Monitoring vs observability | Known questions vs new questions without shipping code; emergent failures need the latter |
| The signals | Data structures: metrics aggregate (cost ∝ series), traces record causality (cost ∝ traffic, sampled), logs record events (biggest bill), profiles record code cost (alpha in OTel since March 2026) |
| Cardinality | Series = product of attribute values; bounded attributes on metrics, identifiers on spans |
| Correlation | Trace/span IDs on every log, same resource identity everywhere, exemplars |
| OpenTelemetry | Spec, API, SDK, semconv, OTLP, Collector — not a backend; .NET API is `System.Diagnostics` |
| Missing custom spans | `AddSource` / `AddMeter` — nothing is collected unless subscribed |
| Semantic conventions | HTTP stable since 1.23, DB since 1.33, messaging and GenAI not yet; `OTEL_SEMCONV_STABILITY_OPT_IN` |
| OTLP | gRPC 4317, HTTP 4318, `OTEL_EXPORTER_OTLP_*`, `UseOtlpExporter()` |
| Propagation | `traceparent` = version-traceid-parentid-flags; baggage goes everywhere, never secrets; `Activity.Current` via `AsyncLocal` |
| Edge security | New trace for external callers, ignore their sampled flag, allow-list baggage |
| Sampling | Head (parent-based ratio, rate-limited 5/s default in Azure distro) vs tail (Collector, keep errors/slow); SLIs from unsampled data |
| The Collector | Receivers → processors (`memory_limiter` first) → exporters; spanmetrics before tail sampling; agent + gateway |
| Span status | 4xx: Unset on server, Error on client; 5xx/exceptions Error; record exceptions once |
| Messaging traces | Producer injects context; consumer child for 1:1, new trace + link otherwise; batches = links; measure queue time |
| Trace shapes | Ladder = N+1, staircase = sequential awaits, repeated spans = retries, client/server gap = pools/starvation |
| Metric instruments | Counter, UpDownCounter, Histogram, Gauge, Observable*; histogram for anything you'd percentile |
| Temporality | Cumulative (Prometheus) vs delta (Azure Monitor); derive rates |
| Percentiles | Can't average; merge histogram buckets; boundary at the SLO threshold; seconds need seconds buckets |
| Golden signals | RED for services, USE for resources, saturation leads: thread-pool queue, GC pause, pool waits |
| .NET built-ins | `Microsoft.AspNetCore.Hosting`, Kestrel, `System.Net.Http`, `System.Runtime` (.NET 9+) |
| Structured logging | Constant templates + typed properties; no interpolation; consistent names |
| Fast logging | `[LoggerMessage]` source generator, EventIds, no boxing when disabled |
| What to log | Boundaries, decisions, failures — once; Debug off, switchable per category |
| PII | Don't emit; classify + redact (`Microsoft.Extensions.Compliance`); scrub in Collector/DCR |
| Log volume | Filters → sampling → buffering (.NET 9 per-request) → pipeline filters → table plans |
| Audit | Separate, complete, tamper-evident, transactional — never diagnostic logs |
| SLI / SLO / SLA | Good/valid; target + window; contract looser than SLO; budget = 1 − SLO |
| Nines | 99.9% ≈ 43 min/30 days; 99.99% ≈ 4.3 min; 99.999% ≈ 26 s |
| Composition | Serial dependencies multiply; SLO ≤ product of hard dependencies |
| Error budget policy | Agreed in advance with product; freeze when exhausted; unspent budget = permission |
| Burn-rate alerts | 14.4 over 1 h & 5 min (page), 6 over 6 h & 30 min (page), 1 over 3 d & 6 h (ticket) |
| Alert design | Page on symptoms, ticket on causes; actionable, urgent, novel; runbook + owner; alert on absent data |
| Async SLOs | Event-time-to-effect latency, oldest-message age, lag in time; depth isn't an SLI |
| Azure Monitor map | Platform metrics, Log Analytics (KQL), Azure Monitor workspace (PromQL), App Insights on top, DCRs |
| .NET on Azure 2026 | Azure Monitor distro (`UseAzureMonitor`), Microsoft distro (`UseMicrosoftOpenTelemetry`, 1.0 May 2026), SDK 3.x (GA Feb 2026) — never two |
| OTLP into Azure | Collector ≥ 0.132 + Entra; AMA and AKS paths in preview; traces/logs → Log Analytics (semconv), metrics → Azure Monitor workspace |
| Container Apps agent | App Insights / Datadog / OTLP; gRPC only; App Insights destination: traces + logs, no metrics |
| Telemetry cost | Analytics ~$2.30/GB, Basic ~$0.50, Auxiliary ~$0.05; measure with `Usage`; one path per signal; no daily cap as primary control |
| Incident flow | Detect (burn) → triage (who, since when, how bad) → mitigate (rollback) → diagnose → blameless postmortem |
| Architecture choice | OTel in code, OTLP to a Collector, backend as a routing decision |
| When not to | No gateway tier, tail sampling or SLOs before they earn their keep |

---

## Progress

Module 28 complete. It closes the loop on Phase 6's operational story:

- **Module 13** (reliability patterns) and **Module 25** (Polly) are now measurable: resilience events, retries and breaker states become spans, metrics and burn-rate inputs.
- **Module 26** (compute) and **Module 27** (messaging and data) each gained their collection paths and the metrics that predict incidents.
- **Module 29** (security architecture) comes next and picks up threads opened here: Entra-authenticated telemetry ingestion, PII in telemetry, audit trails, and security signals routed to Sentinel.
- **Module 30** (design documents) and **Module 31** (ADRs and C4) will use the SLO document and the observability design review as standard sections of every design.
