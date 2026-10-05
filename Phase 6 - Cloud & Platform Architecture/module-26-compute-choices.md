# Module 26 — Compute Choices on Azure: App Service, Functions, Container Apps, AKS — and When Each Wins
*Phase 6: Cloud & Platform Architecture · Senior/Architect Interview Prep for .NET & C#*

> **Platform state verified on September 29, 2026.** This module's facts move faster than any other module's, so the dates matter:
>
> - **Azure Functions:** the **in-process .NET model reaches end of support on November 10, 2026** (six weeks from today) — the isolated worker model is the only supported .NET model after that. The **Consumption plan is now labelled "legacy"**; **Flex Consumption** (Linux-only, code-only) is the recommended serverless plan. Hosting on **Linux Consumption retires September 30, 2028**, gets no new language versions (.NET 9 was the last), and apps still on the v3 runtime there **stop running after September 30, 2026**.
> - **Durable Task Scheduler** is GA (Dedicated SKU since November 2025, Consumption SKU since March 2026) and is now Microsoft's recommended Durable Functions backend; the **Durable Task SDKs** run the same orchestration model on Container Apps, AKS or App Service.
> - **App Service:** **Premium v4** has been GA since September 1, 2025; **Isolated v4** on ASEv3 went GA at Build 2026 in limited regions; **Managed Instance** (Windows, Pv4/Pmv4 only) is GA in select regions; **Aspire deployment to App Service** is GA.
> - **Container Apps:** the **workload-profiles environment** is the default and the **Consumption-only environment is "legacy"**; Functions-on-Container-Apps is GA.
> - **AKS:** **AKS Automatic** is Microsoft's stated production default; **managed system node pools** reached GA in May 2026 with a **pod readiness SLA**; **Node Auto Provisioning** (Karpenter) is the modern node autoscaler. Kubernetes **1.36** is GA on AKS and **1.37** is expected in October 2026. Upstream **ingress-nginx maintenance ended in March 2026**; Microsoft patches the NGINX-based application routing add-on **only through November 2026**, and the successor is the **application routing Gateway API implementation** (default for new AKS Automatic clusters on 1.36+). **kubenet** networking retires March 31, 2028.
> - **.NET 10** (LTS) is current; its container images default to **Ubuntu 24.04** and **no Debian images ship for .NET 10**. **.NET 8 and .NET 9 both reach end of support on November 10, 2026.**
> - **Aspire 13.x:** `aspire publish` / `aspire deploy` are GA (13.4); Kubernetes/AKS deployment via Helm arrived in 13.3.
> - **Retirements to know:** Cloud Services (extended support) on **March 31, 2027**; Azure Spring Apps on **March 31, 2028**.
>
> Prices quoted in this module are US-region pay-as-you-go list prices at the time of writing, used for **arithmetic**, not procurement. Verify in the pricing calculator before you quote a number to anyone who will hold you to it.

## Orientation

Here is the sentence to carry through the whole module: **a compute platform is a bundle of decisions someone else has already made for you — placement, scaling, networking, rollout, patching, identity. Choosing one is choosing which decisions you delegate, which you must still make, and which constraints you accept in return. The right choice is the highest-abstraction platform whose constraints this workload can live with for its whole life — and the architect's job is to know those constraints before the workload hits them.**

Every earlier module in this curriculum assumed there was *somewhere* for your code to run. Module 6 talked about autoscaling and statelessness in the abstract; Module 13 about health probes, graceful degradation and zone redundancy; Module 14 about Server vs Workstation GC; Module 17 about Native AOT and startup; Module 18 about Kestrel and the hosting model; Module 21 about how many deployable units you should have; Module 25 about Aspire service defaults that quietly add resilience handlers. This module is where all of that lands on a real platform — and where you discover that the platform changes the answers. A health endpoint means something different to App Service, to Container Apps and to a Kubernetes kubelet. Server GC in a 0.5-vCPU container is a different animal from Server GC on a 16-core VM. "Scale out" means adding VMs on App Service, adding replicas in seconds on Container Apps, adding pods *and then* nodes on AKS, and adding whole host processes on Functions.

Why this matters in an interview: compute choice is the most common "cloud architecture" question asked of .NET candidates, and the most commonly answered badly. Weak answers are brand loyalty ("we always use Kubernetes") or feature lists ("Functions has triggers, AKS has pods"). Strong answers are **structured trade-offs with numbers**: what the workload's shape is, which scaling signal fits it, what cold start costs at the p99, what the monthly bill looks like at 10% and 80% utilization, how many engineer-hours per month the platform eats, and which constraint would force you off it later. Current loops ask questions like *"Why not just put everything on AKS?"*, *"Your Functions app has a p99 of 9 seconds at 3 a.m. — why?"*, *"When does Container Apps stop being enough?"*, *"The in-process model goes out of support in November — what's your plan?"*, and *"We have four engineers and an AKS cluster; the upgrades are eating us alive. What do you recommend?"*

This module has seven jobs:

1. **Build a first-principles model of what a compute platform is** — the delegated decisions, the responsibility stack, workload shapes, scaling models, cold start, cost models and operational burden — so every platform after that is a *point in a space* rather than a product to memorize.
2. **Teach App Service precisely** — the plan as the unit of compute and billing, tiers, three kinds of scaling, slots and swap mechanics, networking, the free operational features, and the traps (230 seconds, SNAT, ARR affinity, `/home`).
3. **Teach Azure Functions precisely** — the host/trigger/binding model, the 2026 hosting-plan lineup, how scaling decisions are made, the isolated worker model and the November deadline, cold start, timeouts, Durable orchestration and delivery semantics.
4. **Teach Container Apps precisely** — Kubernetes you don't operate: environments, workload profiles, revisions, KEDA scaling with its real defaults, jobs, networking, Dapr and the limits that eventually push teams elsewhere.
5. **Teach AKS precisely** — the Kubernetes API as a platform for building platforms, AKS Automatic vs Standard, two-layer scaling and its timing math, the post-ingress-nginx networking story, and the day-2 operational tax you sign up for.
6. **Make .NET behave on all of them** — container images, CPU and memory limits, GC and DATAS, startup engineering, probes, graceful shutdown, identity, and Aspire as the deployment front door.
7. **Make the decision defensible** — decision axes, a default ladder with explicit step-down triggers, cost arithmetic with break-even utilization, hidden costs, lock-in and migration paths, mixing platforms, reliability, isolation, lifecycle risk — and when not to over-engineer.

Seven framings to carry through:

1. **Every platform is an opinion about who owns which layer.** Read a platform by asking what it decides for you and what you can no longer decide.
2. **Choose the highest abstraction whose constraints you can live with.** Step down the ladder only for a *named* constraint — never for flexibility you can't name.
3. **The scaling model is the platform's personality.** What is the scale unit (VM, replica, pod, host process), what is the signal (CPU, concurrency, queue depth), and how long does a new unit take to be useful?
4. **Cold start is latency you pay for money you save.** It is a knob, not a defect — you buy it down with minimum instances, smaller startup, and faster images.
5. **Operational burden is a recurring cost** measured in engineer-hours per month, forever — and it belongs in the same spreadsheet as the Azure bill.
6. **The container is the portability unit; the programming model is the lock-in.** A Docker image moves between platforms in a day; a Durable Functions orchestration or a pile of bindings does not.
7. **Decide per workload, but cap the number of platforms.** Each platform you add brings its own deployment, networking, identity, observability and on-call story.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What a compute platform does | It makes seven decisions for you: placement, scaling, networking, rollout, patching, identity, observability |
| 2 | The responsibility stack | IaaS → CaaS → PaaS → FaaS: each step up hands a layer to the provider and removes a knob from you |
| 3 | The price of convenience | Constraints are the invoice for delegation; choose the highest abstraction whose constraints you can live with |
| 4 | Workload shapes | APIs, web apps, event processors, jobs, workers, workflows, stateful and GPU workloads each have a natural home |
| 5 | Scaling models | Scale unit × scale signal × time-to-useful — the three numbers that describe any autoscaler |
| 6 | Cold start from first principles | Placement + pull + process + runtime + app init + first-request JIT + dependency warm-up |
| 7 | Cost models | Provisioned vs consumption; consumption wins when average ÷ peak < provisioned price ÷ consumption price |
| 8 | Operational burden | Engineer-hours per month are a line item; platforms differ by an order of magnitude |
| 9 | The App Service model | The plan is the unit of compute and billing; apps are tenants on its workers |
| 10 | App Service tiers | Basic → Standard → Premium v3/v4 → Isolated v2/v4; Managed Instance for legacy Windows |
| 11 | Scaling App Service | Manual, rule-based autoscale, or HTTP-driven automatic scaling; 3/10/30/100-instance ceilings |
| 12 | Density and noisy neighbors | Many apps per plan is the cost advantage and the blast-radius problem; per-app scaling helps |
| 13 | Slots and swaps | Warm-up, sticky settings, swap-with-preview; the zero-downtime story most teams underuse |
| 14 | App Service networking | Inbound (access restrictions, private endpoints) and outbound (VNet integration, NAT) are separate problems; SNAT is the classic trap |
| 15 | Free operational features | Health check, auto-heal, Easy Auth, sidecars, WebJobs, managed certs — capability you'd otherwise build |
| 16 | App Service traps | The 230-second limit, ARR affinity, `/home` semantics, platform restarts, Windows/Linux gaps |
| 17 | When App Service wins | Web apps and APIs with steady traffic, small teams, and a need for boring |
| 18 | The Functions model | A host that turns events into method calls via triggers and bindings |
| 19 | Functions hosting plans in 2026 | Flex Consumption (default), Premium, Dedicated, Container Apps; Consumption is legacy |
| 20 | How Functions scale | Target-based scaling per trigger; per-function scaling groups on Flex; concurrency is the lever |
| 21 | The isolated worker model | Out-of-process, ASP.NET Core integration, any .NET version — and in-process ends Nov 10, 2026 |
| 22 | Cold start in Functions | Always-ready instances, smaller packages, fewer startup calls; measure the p99 not the average |
| 23 | Timeouts and long work | `functionTimeout`, the 230-second HTTP ceiling, scale-in grace periods; long work goes async |
| 24 | Durable orchestration | Replay-based workflows with determinism rules; Durable Task Scheduler as the backend |
| 25 | Triggers and delivery semantics | At-least-once everywhere; idempotency is your job; checkpoints and poison handling differ by trigger |
| 26 | Functions traps | Connection exhaustion, host.json defaults, storage dependency, Flex specifics |
| 27 | When Functions wins | Event-driven glue with bursty, spiky or idle traffic, and orchestration without a workflow server |
| 28 | What Container Apps is | Serverless containers on managed Kubernetes, KEDA, Envoy and Dapr — without the Kubernetes API |
| 29 | Environments, plans, profiles | The environment is the network and log boundary; Consumption and Dedicated profiles coexist |
| 30 | Revisions and traffic | Immutable revisions, single vs multiple mode, weighted traffic and labels for blue/green and canary |
| 31 | KEDA scaling in Container Apps | 0–10 replicas and 10 concurrent requests by default; 30 s polling, 300 s cool-down; 1→4→8→16 steps |
| 32 | Container Apps jobs | Manual, scheduled and event-driven executions — batch without a warm replica |
| 33 | Container Apps networking | Internal vs external ingress, environment DNS, VNet integration, UDR/NAT egress |
| 34 | Dapr, sidecars and platform extras | Dapr building blocks, init/sidecar containers, dynamic sessions, serverless GPUs, Functions hosting |
| 35 | Container Apps limits and traps | No Kubernetes API, 4 vCPU/8 GiB Consumption ceiling, cold starts at zero, idle billing, environment IP planning |
| 36 | When Container Apps wins | Containerized microservices and event processors that want scale-to-zero without a platform team |
| 37 | What Kubernetes gives you | A declarative API plus reconciliation loops — a platform for building platforms |
| 38 | AKS architecture | Managed control plane, node pools, tiers; Automatic as the default, Standard for control |
| 39 | Two-layer scaling | Pods scale in seconds, nodes in minutes; headroom = demand growth rate × node provisioning time |
| 40 | AKS networking after ingress-nginx | CNI Overlay + Cilium, Gateway API via app routing or App Gateway for Containers, mesh when justified |
| 41 | Identity and supply chain on AKS | Workload identity, Azure RBAC, safeguards/policy, signed images from ACR |
| 42 | The day-2 tax | Minor upgrades every few months, weekly node images, add-on lifecycles, deprecated APIs |
| 43 | Delivery on AKS | Helm + GitOps (Flux/Argo CD) + progressive delivery; the cluster is desired state |
| 44 | Bin packing and cost | Requests drive scheduling and cost; namespaces, quotas, spot and consolidation |
| 45 | When AKS wins | You need the Kubernetes API, a platform team, or workloads the managed tiers can't host |
| 46 | The rest of the menu | VMs/VMSS, Container Instances, Batch, Service Fabric, Static Web Apps — and the retirements |
| 47 | Containerizing .NET 10 | SDK container publish, chiseled Ubuntu images, non-root on 8080, small images |
| 48 | .NET inside a container | CPU limits set `ProcessorCount`; the GC hard limit is 75% of memory; DATAS; CFS throttling |
| 49 | Startup engineering | ReadyToRun, Native AOT, trimming, and removing startup network calls |
| 50 | Probes per platform | Liveness, readiness, startup mapped onto App Service, Container Apps and AKS |
| 51 | Graceful shutdown per platform | SIGTERM → stop accepting → drain → exit, within each platform's grace window |
| 52 | Identity, config and secrets | Managed/workload identity, Key Vault references, App Configuration — no secrets in images |
| 53 | Aspire as the deployment front door | One AppHost; Container Apps, App Service, AKS and Compose as targets |
| 54 | Observability parity | One OpenTelemetry pipeline regardless of platform (Module 28 preview) |
| 55 | The decision axes | Eleven axes that separate the platforms, and which ones are usually decisive |
| 56 | The default ladder | Start high, step down only on a named trigger — with the triggers listed |
| 57 | Cost modeling, worked | Break-even utilization, per-replica arithmetic, and the Functions concurrency effect |
| 58 | Hidden costs | Logs, NAT, egress, idle minimums, private endpoints, dedicated-plan fees — and people |
| 59 | Lock-in and migration paths | Containers move; programming models don't; the ladder is walkable in both directions |
| 60 | Mixing platforms, capping the count | Two or three platforms per estate, with shared identity, network, delivery and telemetry |
| 61 | Reliability by platform | Zones, instance minimums, SLAs, multi-region stamps — what each platform gives and needs |
| 62 | Isolation and compliance | Multi-tenant vs dedicated compute, network isolation, single-tenant options per platform |
| 63 | Lifecycle risk | Retirements are architecture inputs: track them, date them, budget for them |
| 64 | Anti-patterns | The mistakes that make a platform the problem |
| 65 | The compute decision review | A checklist you can narrate for any hosting proposal |
| 66 | When not to | Don't pick AKS for a CV, Functions for everything, or a new platform per team |

---

# Part A — First principles: what a compute platform actually is

## Concept 1 — What a compute platform does: the seven delegated decisions

Strip away the product names and every way of running code in the cloud answers the same seven questions. A platform is defined by **which of these it answers for you** and **how much say you keep**.

| Decision | The question | Who answers on a VM | Who answers on a high-level PaaS |
|---|---|---|---|
| **1. Placement** | On which physical machine does this process run, next to what? | You (VM size, availability set/zone) | The platform |
| **2. Scaling** | How many copies, and when do we add or remove one? | You (scale set rules, scripts) | The platform, from a signal you choose |
| **3. Networking** | How does traffic reach the process, how does it reach others, what is it allowed to talk to? | You (NSGs, load balancer, TLS, DNS) | The platform (ingress, TLS, service discovery), with your policy |
| **4. Rollout** | How does version N+1 replace version N without dropping requests? | You (scripts, blue/green by hand) | The platform (slots, revisions, rolling updates) |
| **5. Patching** | Who updates the OS, the runtime, the web server — and when do they reboot? | You | The platform (on its schedule, not yours) |
| **6. Identity** | How does the code prove who it is to other services, without secrets? | You (managed identity on the VM, cert rotation) | The platform (managed identity, workload identity) |
| **7. Observability** | Where do logs, metrics and traces go, and what does the platform itself report? | You | Shared: the platform emits its own signals; you instrument your code |

Two things follow immediately.

**First, "managed" is a spectrum, not a binary.** App Service makes all seven decisions for you and lets you tune several. Container Apps makes most of them and lets you tune more. AKS makes placement and control-plane patching decisions and hands you a toolkit for the rest. A VM makes almost none. When someone says "we want a managed service," the useful follow-up is *"managed with respect to which of the seven?"*

**Second, every decision the platform makes is one you can no longer make differently.** App Service patches your OS on its schedule; if your app can't survive a restart at 3 p.m. on a Tuesday, that's your problem to fix, not a setting to change. Functions decides how many instances run your code; if your downstream database can take 50 connections and the platform scales you to 200 instances, you must cap it explicitly. Delegation is a trade, and senior answers name both sides.

There's also an eighth, quieter decision — **the programming model**. App Service, Container Apps and AKS all run *processes* you write however you like. Functions runs *functions* inside a host it controls, with triggers and bindings it defines. That difference matters more for lock-in and testability than any of the seven above (Concept 59).

**The interview-grade sentence:** *"I read any compute platform as the set of decisions it makes for me — placement, scaling, networking, rollout, patching, identity and observability — plus whether it imposes a programming model. The more it decides, the less I operate, and the more constraints I accept, so the question is always which decisions this workload can afford to delegate."*

---

## Concept 2 — The responsibility stack and the abstraction ladder

The classic way to picture this is a stack of layers, from hardware at the bottom to your business logic at the top. Each service model hands a slice of that stack to the provider:

```
                     VMs / VMSS    AKS           Container Apps   App Service     Functions
                     (IaaS)        (managed CaaS) (serverless CaaS) (PaaS)          (FaaS)
Business logic       you           you           you              you             you
Programming model    you           you           you              you             PLATFORM (host, triggers)
Runtime (.NET)       you           you (image)   you (image)      platform or you platform or you
Web server/ingress   you           you (choose)  PLATFORM (Envoy) PLATFORM         PLATFORM
Scaling logic        you           you (HPA/KEDA) PLATFORM (KEDA)  PLATFORM         PLATFORM
Orchestration        you           PLATFORM*     PLATFORM         PLATFORM         PLATFORM
Node OS & patching   you           shared**      PLATFORM         PLATFORM         PLATFORM
Virtualization, HW   PLATFORM      PLATFORM      PLATFORM         PLATFORM         PLATFORM

*  AKS manages the control plane; you own what runs on it and how it's configured.
** You own node image upgrades and their timing (automatable); AKS Automatic manages more of it.
```

Mapped onto Azure's compute menu, it becomes an **abstraction ladder**. Moving *up* removes operational work and flexibility; moving *down* adds both:

| Rung | Azure services | You write | You operate | Typical scale unit |
|---|---|---|---|---|
| **FaaS** | Azure Functions | Functions against triggers/bindings | Configuration | Host instance |
| **PaaS (web)** | App Service | A web app or API (code or container) | Plan size, slots, settings | VM instance in a plan |
| **Serverless containers** | Container Apps, Container Instances | A container image | Revisions, scale rules, environment | Replica (≈ a pod) |
| **Managed Kubernetes** | AKS (Automatic or Standard) | Images + Kubernetes manifests | Cluster config, upgrades, add-ons, policies | Pod, then node |
| **IaaS** | VMs, VM Scale Sets, Batch pools | Anything | Everything above the hypervisor | VM |

Three observations that interviewers like:

1. **The ladder isn't strictly linear.** Functions can run *on* App Service plans (Dedicated) or *on* Container Apps; Container Apps runs *on* AKS under the hood; App Service runs *on* VM scale sets you never see. What you buy is not the machinery but the **contract** — what you're allowed to change and what's guaranteed.
2. **Containers cut across the ladder.** The same OCI image runs on App Service (custom container), Container Apps, AKS, Container Instances and even Functions Premium. The container is the *packaging* decision; the rung is the *operations* decision. Keep them separate (framing 6).
3. **"Serverless" is a billing and scaling property, not a rung.** It means *scale to zero and pay per use*. Functions Flex, Container Apps Consumption and Container Instances are serverless; App Service and AKS are not (though AKS can approximate it with KEDA and Node Auto Provisioning, you still pay for the control plane tier and the system nodes).

**The interview-grade sentence:** *"I think of Azure compute as a ladder — VMs, AKS, Container Apps, App Service, Functions — where each rung up hands another layer to Microsoft and takes away a knob. Containers cut across it as the packaging choice, and 'serverless' is a billing and scaling property that several rungs offer, not a rung of its own."*

---

## Concept 3 — The price of convenience: constraints, and choosing the highest rung you can live with

Every delegated decision comes back as a **constraint**. That's not a flaw — it's how the provider makes the decision cheaply for millions of tenants. A platform that let you change everything would be a VM. The skill is knowing each platform's constraints *before* you commit, because the costly ones show up months later, under load, in production.

A catalogue of the constraint types, with examples you'll meet in this module:

| Constraint type | Examples |
|---|---|
| **Duration** | HTTP requests capped at 230 s by the front-end load balancer on App Service and Functions (Concept 16); legacy Consumption functions capped at 10 min |
| **Size** | Container Apps Consumption replicas max out at 4 vCPU / 8 GiB; Flex Consumption instances at 4,096 MB |
| **Scale ceilings** | App Service Premium plans stop at 30 instances; Flex function groups at 1,000; regional quotas cap everything |
| **Startup** | Flex Consumption apps must initialize within 30 s; Functions language workers within 60 s |
| **Runtime and OS** | Flex is Linux-only and code-only; Functions supports no Windows containers; Container Apps runs only `linux/amd64` |
| **Kernel/host access** | No privileged containers on Container Apps; no DaemonSets, no custom CRDs, no operators |
| **Networking** | Some features require specific tiers or environment types (private endpoints, UDR, NAT gateway) |
| **Lifecycle** | The platform restarts you for patching; the platform retires plans and models on its timeline (Concept 63) |
| **Programming model** | Functions' triggers, bindings, host.json and Durable determinism rules |

From this follows the rule the whole module hangs on:

> **Choose the highest-abstraction platform whose constraints this workload can live with for its whole expected life. Step down a rung only for a *named* constraint you actually hit — never for flexibility you can't name.**

Why "highest"? Because every rung down costs you something continuous: upgrades to run, YAML to maintain, on-call knowledge to keep, security baselines to enforce. Those costs don't show up in a proof of concept; they show up in month eight. And why "for its whole life"? Because some constraints are cheap to discover early and expensive to escape late — the 230-second limit is trivial to design around on day one and a rewrite on day 400.

The flip side is equally important: **a constraint you *will* hit is a reason to start lower.** If you know the workload needs a GPU, a custom kernel module, 64 GiB per process, a Kubernetes operator, or a Windows container with COM components, don't start on a rung that forbids it just to "keep it simple." Simple that fails is not simple.

**The interview-grade sentence:** *"Every delegated decision comes back as a constraint — duration, size, scale ceiling, OS, networking, lifecycle, programming model. My rule is to take the highest-abstraction platform whose constraints the workload can live with for its whole life, and step down only for a constraint I can name — but if I already know it'll hit one, I start lower rather than migrate later."*

---

## Concept 4 — Workload shapes and their natural homes

"Where should this run?" is unanswerable until you've said **what shape the work is**. Eight shapes cover almost everything a .NET team builds:

| Shape | Characteristics | Natural first home | Why |
|---|---|---|---|
| **Request/response API** | Synchronous, latency-sensitive, stateless, steady-ish traffic | App Service or Container Apps | Always-warm instances, built-in ingress/TLS, simple scaling on concurrency |
| **Server-rendered web app** (Razor Pages, Blazor Server, MVC) | Same as API, plus sessions/WebSockets, affinity concerns | App Service | Slots, Easy Auth, WebSockets, custom domains/certs, the most "boring" option |
| **Event processor** (queue/topic/stream consumer) | Asynchronous, throughput-oriented, bursty, tolerates latency | Functions (Flex) or Container Apps with a KEDA rule | Scale on queue depth or lag, including to zero |
| **Scheduled / batch job** | Runs to completion, periodic or on demand | Container Apps jobs, Functions timer trigger, Batch for HPC scale | Pay only while running; no warm replica |
| **Long-running worker** | Continuous background processing (a `BackgroundService`) | Container Apps (min replicas ≥ 1) or App Service WebJobs | A process that must stay alive and drain on shutdown |
| **Workflow / orchestration** | Multi-step, long-lived, needs durable state and retries | Durable Functions or Durable Task SDKs + Durable Task Scheduler | Durable execution without running a workflow server |
| **Stateful / clustered service** | In-memory state, leader election, Orleans silos, databases | AKS (or VMs); Service Fabric for legacy | Needs stable identity, storage, and membership control |
| **GPU / ML inference** | Accelerators, large images, model loading time | Container Apps (serverless or dedicated GPU profiles), AKS GPU node pools | Specialized hardware and long cold starts |

Three rules help when a workload doesn't fit neatly:

1. **Classify by the *dominant* constraint.** An API that also processes a queue is two workloads that share a codebase; they can share a container image and still run as two apps with different scaling rules (a very common Container Apps pattern).
2. **Duration and statefulness decide more than technology.** Anything that must hold an HTTP request longer than a few minutes, or keep state in memory across requests, rules out the serverless rungs as the *synchronous* path — and should usually be redesigned as asynchronous anyway (Module 11).
3. **Traffic shape decides the billing model.** Spiky or mostly-idle work is where consumption billing shines; flat, high utilization is where provisioned capacity wins (Concept 7).

**The interview-grade sentence:** *"Before naming a service I name the workload shape — synchronous API, web app, event processor, scheduled job, long-running worker, workflow, stateful service, or GPU inference — because each has a natural home. When a component mixes shapes I split it into separately scaled apps, often from the same image."*

---

## Concept 5 — Scaling models: unit, signal, and time-to-useful

Module 6 introduced autoscaling as a control loop. Every autoscaler on every platform can be described by three quantities, and comparing platforms on these three is far more useful than comparing marketing pages.

**1. The scale unit — what gets added.**

| Platform | Unit | Granularity |
|---|---|---|
| App Service | A VM instance in the plan (all apps on it scale together unless per-app scaling is on) | Coarse — a whole worker |
| Functions (Flex) | A host instance for a *function group* (Concept 20) | Fine — 0.25 to 2 cores each |
| Container Apps | A replica of a revision | Fine — from 0.25 vCPU |
| AKS | A pod — and, separately, a node when pods don't fit | Two levels (Concept 39) |
| VMSS | A VM | Coarse |

**2. The scale signal — what triggers the decision.**

- **Resource utilization** (CPU %, memory %) — the classic App Service autoscale and Kubernetes HPA signal. Lagging: CPU rises *after* the load arrives, and for I/O-bound .NET services CPU may never rise at all while latency explodes.
- **Concurrency** (in-flight requests per instance) — Container Apps' HTTP rule, Functions HTTP scaling, App Service automatic scaling. Leading and directly tied to Little's Law: concurrency = throughput × latency (Module 6).
- **Backlog** (queue length, consumer lag, pending events) — KEDA scalers and Functions target-based scaling. The best signal for asynchronous work, because it measures work *waiting*, not work *being done*.
- **Schedule** — predictable peaks (a 9 a.m. login storm) are better served by scheduled pre-scaling than by any reactive signal.

**3. Time-to-useful — how long until the new unit serves traffic.** This is the number most designs ignore, and it's the sum of *decision delay* + *provisioning delay* + *startup delay*:

```
time-to-useful = detection (polling interval, metric window)
               + provisioning (VM or node boot, or nothing if capacity is pre-warmed)
               + image pull (seconds to minutes; cached or streamed images help)
               + process start + runtime init + app init (Concept 6)
               + readiness probe passing
```

Orders of magnitude (vary by region, size and image — measure your own):

| Scenario | Rough time-to-useful |
|---|---|
| New Container Apps replica on existing capacity, small warm-cached .NET image | seconds |
| Functions Flex scale-out with always-ready baseline already serving | near zero for the baseline; seconds for on-demand instances |
| App Service rule-based autoscale adding an instance | minutes (metric window + instance allocation + app start) |
| AKS pod on an existing node with spare room | seconds |
| AKS pod that needs a new node (cluster autoscaler or NAP) | typically one to several minutes |

**The design consequence.** A reactive autoscaler can only absorb load growth that's slower than its time-to-useful. If demand can rise faster than that, you need **headroom**: spare capacity equal to (rate of demand growth) × (time-to-useful). That's why minimum replicas, always-ready instances, prewarmed instances and over-provisioned node pools exist — they are *pre-paid time-to-useful*. Concept 39 does this arithmetic for AKS.

**The interview-grade sentence:** *"I compare autoscalers on three numbers: the scale unit, the signal, and time-to-useful. Concurrency and backlog are leading signals, CPU is lagging; and since a reactive scaler can only follow demand that grows slower than its time-to-useful, I size headroom as growth rate times that delay — which is exactly what minimum and always-ready instances buy."*

---

## Concept 6 — Cold start from first principles

A **cold start** is the extra latency a request pays when no warm instance is available to serve it. It's not one delay but a chain, and each link has a different fix:

| Stage | What happens | Typical cause of slowness | Mitigation |
|---|---|---|---|
| **Placement** | The platform picks a host with capacity | Capacity shortage, large instance size | Pre-provisioned/always-ready capacity |
| **Image or package fetch** | Pull the container image or download the code package | Large images, cold registry caches, cross-region pulls | Small images (chiseled, trimmed), regional registry, artifact streaming |
| **Process start** | Start the container and the `dotnet` process | — | — |
| **Runtime init** | CLR starts, loads assemblies, tiered JIT begins | Many assemblies, JIT of hot paths | ReadyToRun, Native AOT (Concept 49) |
| **App init** | DI container built, configuration loaded, EF model built, hosted services started | Network calls at startup (Key Vault, App Configuration, token acquisition), eager warm-ups | Remove or parallelize startup I/O; lazy init; cache config |
| **First-request path** | First request JITs the pipeline, opens DB/HTTP connections, completes TLS handshakes, acquires Entra tokens | Connection establishment + TLS + token fetch | Warm-up endpoint hit by the platform; connection pooling; readiness probes that wait for warm-up |
| **Readiness** | Platform routes traffic only after probe/health passes | Probes too slow or too eager | Probes that reflect genuine readiness (Concept 50) |

For a typical ASP.NET Core service, the *platform* stages dominate when scaling from zero (seconds, occasionally much more for large images), while the *application* stages dominate when an instance is already placed — and the application stages are the ones **you control**. It's common to find that half of a .NET service's cold start is a synchronous call to Key Vault plus an Entra token acquisition plus the first TLS handshake to SQL — none of which have anything to do with the platform.

Why .NET gets a reputation for slow cold starts: JIT compilation and assembly loading are real costs compared with Go or Rust binaries, and ASP.NET Core apps tend to do a lot at startup through DI. But modern .NET has strong answers — ReadyToRun images, Native AOT for minimal APIs and workers, trimming — and Module 17 covered their trade-offs. The architect's version of the answer is: **cold start is a budget you allocate**, not a fixed tax.

**Microsoft's own data explains why cold start exists at all.** The *Serverless in the Wild* study (USENIX ATC 2020) characterized Azure Functions production traffic and found that most applications are invoked rarely — about 45% of function apps at most once per hour, and 81% at most once per minute — while the remaining 19% account for more than 99.5% of all invocations. About half of functions ran for under a second on average. Keeping every rarely used app warm would waste enormous memory; unloading idle apps is what makes consumption pricing possible. Cold start is the latency side of that economic bargain (framing 4). The same paper proposed adaptive keep-alive and pre-warming policies driven by each app's own invocation history — the idea that a platform can learn *when* to keep you warm.

**Measuring it honestly.** Cold starts hide in averages. Measure **the p99 and p99.9 of the first request per new instance**, and separately **the fraction of requests that land on a cold instance** — a 4-second cold start that hits 0.01% of requests is a different problem from a 700 ms cold start that hits 5% of them during every scale-out.

**The interview-grade sentence:** *"Cold start is a chain — placement, image pull, process start, runtime init, app init, first-request connections and tokens, readiness — and for a .NET service the app stages I control are often half of it: Key Vault calls, token acquisition, TLS handshakes. I buy it down with always-ready capacity, small images, ReadyToRun or AOT, and startup without network I/O, and I measure the first-request p99 and the fraction of requests that hit a cold instance, not the average."*

---

## Concept 7 — Cost models: provisioned vs consumption, and break-even utilization

There are two ways to pay for compute, and every Azure plan is one, the other, or a hybrid:

- **Provisioned** — you pay for capacity whether or not it's used: App Service plans (per instance-hour), Container Apps Dedicated workload profiles (per profile instance plus a management fee), AKS nodes (per VM-hour) plus the cluster tier, Functions Premium (at least one instance always billed).
- **Consumption** — you pay for resources only while work runs: Functions Flex on-demand (memory GB-seconds while executing plus executions), Container Apps Consumption (vCPU-seconds and GiB-seconds while replicas run, plus requests), Container Instances.

Consumption prices are **higher per unit** — you pay the provider to hold spare capacity for you and to absorb the risk of idle hardware. Provisioned prices are **lower per unit but charged for all capacity**, including the headroom you keep for peaks.

Derive the break-even from first principles. Let:

- `c_p` = provisioned price per unit of capacity per hour,
- `c_c` = consumption price per unit of capacity per hour of *use*,
- `P` = the capacity you must provision to meet **peak** demand (plus headroom),
- `A` = the **average** capacity actually used over the month.

Then, over a period T:

```
provisioned cost  ≈ c_p × P × T
consumption cost  ≈ c_c × A × T

consumption is cheaper when   A / P  <  c_p / c_c
```

In words: **consumption wins when your average-to-peak ratio is lower than the ratio of provisioned to consumption unit prices.** If consumption costs 2× provisioned per unit, it wins whenever average use is below 50% of provisioned peak capacity — which, for most business APIs with daily cycles and weekend troughs, it is. If your service runs flat at 80% of its provisioned capacity around the clock, provisioned wins easily.

Three refinements make this realistic:

1. **Minimums break pure consumption.** Always-ready instances (Flex), minimum replicas (Container Apps) and "at least one instance" (Functions Premium) turn part of a consumption bill into a provisioned one — at a discounted *idle* rate in Container Apps, at the baseline rate in Flex. That's the price of killing cold start (Concept 58).
2. **Granularity and per-request charges matter at the extremes.** Flex bills a minimum of 1,000 ms per execution, then rounds up to 100 ms; Container Apps charges per million external HTTP requests beyond the free grant. Millions of 5 ms requests look very different under each model.
3. **Utilization of provisioned capacity is usually worse than you think.** Teams provision for peak, add headroom for zone failure (Module 13: N+1 across zones), add a second environment for slots or blue/green — and end up with 15–30% average CPU. That's a strong argument for consumption or for denser packing (Concept 44).

Concept 57 works the numbers for a concrete service across all four platforms.

**The interview-grade sentence:** *"Provisioned capacity is cheaper per unit but you pay for your peak; consumption is dearer per unit but you pay for your average. So consumption wins whenever average-over-peak is below the ratio of the unit prices — at a 2× premium, whenever average load is under half of what I'd provision — with the caveats that minimum instances, billing granularity and per-request charges move the line."*

---

## Concept 8 — Operational burden as a recurring cost

The Azure invoice is the smaller half of the cost of a platform for most teams. The larger half is **people-time**, and it recurs every month for as long as the platform exists. Make it explicit by listing the work each platform creates:

| Recurring work | App Service | Functions | Container Apps | AKS Standard | AKS Automatic |
|---|---|---|---|---|---|
| OS and runtime patching | Platform | Platform | Platform (you rebuild images for base-image CVEs) | Node images: you schedule (automatable) | Managed defaults |
| Orchestrator upgrades | — | Functions host: platform; extension/worker packages: you | Platform | **You**: Kubernetes minor versions several times a year, deprecated APIs, add-on compatibility | Automatic upgrade channel; you still validate workloads |
| Ingress, certificates, DNS | Platform (managed certs) | Platform | Platform | **You** choose and run ingress/Gateway, cert-manager or equivalent | Defaults provided (Gateway API) |
| Autoscaling configuration | Rules or automatic | Mostly automatic | Scale rules | HPA/KEDA + node autoscaling | NAP preconfigured |
| Security baseline | Tier settings, access restrictions | Same | Environment networking | Policies, pod security, RBAC, network policies, image admission | Safeguards enforced by default |
| Observability plumbing | Built-in + App Insights | Built-in + App Insights | Log Analytics + OTel agent | Prometheus/Grafana/Container Insights: you assemble | More preconfigured |
| Cost management | Plan sizing | Mostly none | Replica sizing, min replicas | Requests/limits, node pools, spot, consolidation | Consolidation via NAP |
| Knowledge to hire for | Common | Common + Functions model | Containers + KEDA basics | Kubernetes expertise | Kubernetes fundamentals |

A useful, defensible rule of thumb (quote it as such, not as a law): a production AKS estate run well — upgrades, security, networking, observability, cost and developer support — typically needs **a meaningful fraction of at least one or two engineers' time on an ongoing basis**, and grows into a platform team as tenants multiply. App Service, Functions and Container Apps estates typically need a small fraction of one person for the platform itself. At fully loaded engineering costs, that difference routinely **exceeds the entire compute bill** of a small or mid-sized system — which is why "AKS is cheaper per vCPU" is frequently true and frequently irrelevant.

The honest caveats:

- **Operational burden has economies of scale.** A platform team running AKS for 60 services amortizes its cost; the same team for 6 services does not. That's the core of the "platform as a product" argument (Concept 60).
- **Burden shifts, it doesn't vanish.** Moving from AKS to Container Apps removes cluster upgrades but adds platform limits you must design around. Moving from App Service to Functions removes instance sizing but adds a programming model.
- **AKS Automatic narrows the gap** substantially — managed system node pools, node auto-provisioning, default ingress, safeguards, automatic upgrades — without closing it: you still own workload manifests, Kubernetes API deprecations, and a much larger configuration surface.

**The interview-grade sentence:** *"I put engineer-hours on the same spreadsheet as the Azure bill. A well-run AKS estate needs ongoing platform engineering time that for a small system often costs more than all the compute; App Service, Functions and Container Apps need a fraction of a person. AKS Automatic narrows that gap, and a platform team amortizes it across many services — which is why the right answer depends on how many workloads will share the platform."*

---

# Part B — Azure App Service

## Concept 9 — The App Service model: the plan is the unit of compute and billing

App Service's mental model has exactly two nouns, and most confusion comes from mixing them up:

- **An App Service plan** is a set of VM instances (workers) of one size, in one region, running one OS. It is the unit of **compute**, **scaling** and **billing**. You pay per instance-hour for the plan, regardless of how many apps run on it or whether they're busy.
- **An app** (web app, API app, function app on a Dedicated plan, or a custom container) is a tenant *on* the plan. By default, **every app runs on every instance of its plan**.

```
App Service plan "orders-prod-plan"  (P1v4, Linux, West Europe, 3 instances)
 ├─ instance 1:  orders-api   orders-api/staging (slot)   admin-portal   webjob
 ├─ instance 2:  orders-api   orders-api/staging (slot)   admin-portal   webjob
 └─ instance 3:  orders-api   orders-api/staging (slot)   admin-portal   webjob
                 ▲ Every app and every slot shares these three VMs' CPU and memory.
```

Consequences you should be able to state without looking anything up:

1. **Apps in one plan scale together.** Scale the plan to 5 instances and every app now runs on 5 — unless you enable **per-app scaling** or **automatic scaling**, which let an app use only a subset of the plan's instances (Concepts 11–12).
2. **Slots are apps too.** A staging slot with a load test running against it steals CPU from production on the same instances (Concept 13).
3. **WebJobs, backups, diagnostic logging and Kudu** also run on the plan's instances and consume its resources.
4. **Isolation is per plan.** "Dedicated compute" tiers are dedicated *to your plan* — not to your app. To isolate an app's compute from another app's, put it in a separate plan.
5. **The plan pins the OS.** Windows and Linux apps can't share a plan; Windows gives you IIS-hosted code (including .NET Framework); Linux gives you built-in stacks (.NET, Node, Python, Java, PHP) and custom Linux containers.

**Code vs container.** On Linux you can deploy either **code** onto a Microsoft-maintained runtime image (App Service keeps the .NET runtime patched) or a **custom container** (you own the image and its patching). Code deployment is the more "PaaS" option: Microsoft patches the runtime within a major version. A container gives you control of dependencies and a portable artifact (framing 6), and it makes the move to Container Apps or AKS a configuration change rather than a rebuild.

**The interview-grade sentence:** *"An App Service plan is a set of same-sized VMs that is the unit of compute, scaling and billing; apps and their slots are tenants that by default run on every instance. So apps in a plan share fate and CPU, isolation means a separate plan, and scaling the plan scales every app on it unless I turn on per-app or automatic scaling."*

---

## Concept 10 — Tiers, and what each one unlocks

App Service tiers bundle **VM hardware**, **scale ceilings** and **features**. Know the progression rather than the price list:

| Tier family | Compute | Scale-out ceiling | Unlocks | Use for |
|---|---|---|---|---|
| **Free, Shared** | Shared VMs with other customers; CPU-minute quotas per app | No scale-out | Basics only | Demos, experiments |
| **Basic** | Dedicated VMs | 3 instances (manual) | Custom domains, TLS | Dev/test, small internal tools |
| **Standard** | Dedicated VMs (older generation) | 10 instances | Rule-based autoscale, deployment slots, backups | Legacy; mostly superseded by Premium v3/v4 |
| **Premium v3 / v4** | Dedicated, newer hardware; memory-optimized variants | 30 instances | Automatic (HTTP) scaling, zone redundancy, more slots, larger sizes, VNet features | **The default production tier** |
| **Isolated v2 / v4** (in an App Service Environment v3) | Dedicated VMs in *your* virtual network, single-tenant front ends | 100 instances | Network isolation, internal load balancer, very high scale, compliance | Regulated or network-isolated estates |

**Premium v4** (GA since September 2025) runs on AMD-based Dadsv6/Eadsv6 hardware with NVMe temporary storage, in four general sizes (**P0v4–P3v4**) and five memory-optimized sizes (**P1mv4–P5mv4**), from 1 vCPU/4 GB up to 32 vCPU/256 GB. Microsoft's launch benchmarks claimed roughly a quarter better price-performance than Premium v3, and Windows pay-as-you-go prices were cut at the same time. It's available for Windows code and Linux code or containers — **not for Windows containers**. For a new production plan in a supported region, Pv4 is the default choice; Pv3 remains where Pv4 isn't available yet.

**Isolated v4** on **App Service Environment v3** went GA at Build 2026 in a limited set of regions. An ASE is the "single-tenant App Service" option: the whole stamp lives in your subnet, with a flat infrastructure charge plus per-worker charges. You choose it for network isolation and compliance, not for performance (Concept 62).

**Managed Instance** is the newest and most unusual option: a **plan-scoped Windows hosting mode for legacy apps that need OS-level customization** — PowerShell configuration scripts that persist OS and middleware setup, registry values backed by Key Vault, storage mounts (Azure Files, UNC paths), plan-level VNet integration and managed identity, just-in-time RDP through Azure Bastion for diagnostics, with .NET Framework 3.5/4.8 and .NET 8 preinstalled. It's GA in select regions, **Windows only, Pv4/Pmv4 only, no containers**. Its purpose is exactly the classic lift-and-shift blocker: the ASP.NET Framework app that needs a COM component, a registry key, a mapped drive or a font installed — previously a reason to fall all the way down to VMs.

**Scaling up vs out.** *Scale up* changes the instance size or tier (more CPU/memory per instance, more features); *scale out* changes the instance count. Scaling up across hardware generations (e.g., Pv3 → Pv4) can require a plan in a different deployment "stamp" — sometimes a new resource group — which is why the docs describe redeploy paths for unsupported region/resource-group combinations.

**The interview-grade sentence:** *"App Service tiers bundle hardware, scale ceilings and features: Basic for dev at 3 instances, Standard at 10 with autoscale and slots, Premium v3/v4 at 30 with automatic scaling and zone redundancy — Pv4 is my default for new production plans — and Isolated in an ASE v3 at 100 for network isolation. Managed Instance on Pv4 is the new answer for legacy Windows apps that need registry, COM or drive access without dropping to VMs."*

---

## Concept 11 — Scaling App Service: manual, rule-based autoscale, and automatic scaling

App Service offers three ways to decide the instance count, and choosing between them is a real design decision.

**1. Manual.** You set the instance count. Right for dev/test and for steady production loads where predictability beats efficiency.

**2. Rule-based autoscale (Azure Monitor autoscale).** Available from Standard up. You define a profile with minimum, maximum and default counts, and rules such as "if average CPU > 70% over 10 minutes, add 1 instance; cool down 5 minutes." Profiles can be **scheduled** (e.g., a higher minimum on weekdays 08:00–18:00). The properties to know:

- It's **plan-level**: it scales the plan, so all apps on it scale.
- It's **lagging**: it reacts to aggregated metrics over a window (Concept 5), then allocates and starts an instance. Minutes, not seconds.
- It's easy to **flap**: a scale-out rule without a matching, more conservative scale-in rule (different threshold, longer window) oscillates. Azure Monitor autoscale tries to prevent flapping by estimating the post-scale-in metric, but asymmetric rules are still your job.
- **CPU is the wrong signal for many .NET APIs.** An I/O-bound API waiting on a slow database can have terrible latency at 25% CPU. Use HTTP queue length, request rate per instance, or response time where available — or use automatic scaling.

**3. Automatic scaling.** Available on Premium v2, v3 and v4 plans. The platform scales on **HTTP traffic** — the same style of decision Functions Premium makes — rather than on your rules. Its knobs:

| Setting | Scope | Meaning |
|---|---|---|
| **Maximum burst** | Plan | The most instances any app in the plan can scale to (up to **30** on Premium v2/v3/v4); must be ≥ the plan's current worker count |
| **Always ready instances** | App | The minimum number of instances this app always runs on (per-app minimum) |
| **Prewarmed instances** | App | Buffer instances kept warm ahead of need (default **1**); billed only once the always-ready instances are serving traffic |
| **Maximum scale limit** | App | Caps this app below the plan's burst — useful when a database behind it can't keep up |

Behavior worth knowing from the docs: instances are added roughly every few seconds to a minute depending on the app's startup time and load pattern; scale-in review begins about 5–10 minutes after load stops increasing. Because it's per-app, automatic scaling also gives you much of what per-app scaling offered, without writing rules. Two restrictions: a plan using automatic scaling **can't also host function apps**, and it replaces rule-based autoscale for that plan.

**Scale ceilings are hard.** Basic stops at 3 instances, Standard at 10, Premium at 30, Isolated (ASE) at 100. If a single app might need more than 30 instances of the largest size you're willing to pay for, App Service is the wrong rung — or you need multiple plans behind Front Door (the deployment stamps pattern, Concept 61).

**Which to choose:** automatic scaling for HTTP APIs and web apps with variable traffic on Premium; scheduled profiles plus a conservative CPU/memory rule for predictable business-hours loads; manual for flat loads. Always keep a **minimum of 2 instances in production** (3 with zone redundancy) so a single instance restart (Concept 16) isn't an outage.

**The interview-grade sentence:** *"App Service scales three ways: manually, with Azure Monitor autoscale rules — plan-level, lagging, prone to flapping, and often keyed on CPU which is the wrong signal for I/O-bound APIs — or with automatic scaling on Premium, which scales per app on HTTP load with always-ready and prewarmed instances and a per-app cap. Ceilings are hard: 3, 10, 30, or 100 in an ASE."*

---

## Concept 12 — Density and noisy neighbors: many apps per plan

Running many apps on one plan is App Service's cost superpower. Ten small internal APIs, each averaging 3% CPU, can share a two-instance P1v4 plan for the price of two VMs instead of twenty. Microsoft's own guidance gives rough density limits by size — on the order of **8 apps on P0v3/P0v4, 16 on P1v3/P1v4, 32 on P2, 64 on P3**, with memory-optimized and Isolated large sizes bounded by vCPU usage — and counts an **active slot as an app**.

The same property is the risk:

- **Noisy neighbor.** One app's memory leak, CPU-bound loop, or thread-pool starvation (Module 15) degrades every app on the plan. One app's crash-loop restarts don't isolate the others from the CPU it burns.
- **Shared blast radius.** A plan-level operation — a scale-down, a platform upgrade, a bad configuration of the plan — hits every tenant.
- **Coupled scaling.** Without per-app or automatic scaling, a traffic spike on one app scales the plan for all of them (and you pay for all of them).
- **Coupled sizing.** The plan's instance size must fit the most demanding app.

The architecture guidance is simple:

| Put apps in the same plan when | Separate them when |
|---|---|
| They're low-traffic, similar criticality, same team or same product | One is resource-intensive or bursty |
| They can tolerate each other's failures | One is customer-facing and critical and the others are internal |
| They have similar scaling needs | They need different scaling or different instance sizes |
| Cost matters more than isolation | Compliance or data classification requires isolation |

**Per-app scaling** (and automatic scaling's per-app always-ready and maximum settings) lets a plan with, say, 10 instances run a busy app on all 10 and a quiet admin portal on 2. Capacity and billing remain at plan level; the app's replica count can't exceed the plan's instance count.

A useful framing for interviews: **a shared plan is a small, manual bin-packing cluster.** AKS and Container Apps Dedicated profiles do the same job with finer granularity and scheduler-enforced resource requests (Concept 44); App Service does it with coarse, whole-VM tenancy and no per-app resource limits beyond the plan's size.

**The interview-grade sentence:** *"A shared plan is a coarse bin-packing cluster: great for cost with many small apps, but every app shares CPU, memory and blast radius with its neighbors, and there are no per-app resource limits. I group apps of similar criticality and scaling needs, isolate anything busy or critical into its own plan, and use per-app or automatic scaling so one spike doesn't scale everything."*

---

## Concept 13 — Deployment slots and swap mechanics

**Deployment slots** are live apps with their own hostnames (`orders-api-staging.azurewebsites.net`), sharing the production app's plan. Standard and above support them; Premium supports more. They give App Service the best zero-downtime deployment story of any rung for teams that don't want to build one — if you understand what a swap actually does.

**What a swap does**, step by step (source = staging, target = production):

1. **Apply target settings to source.** The *slot-specific* ("sticky", or "deployment slot setting") app settings and connection strings of the **production** slot are applied to the **staging** slot's instances. This **restarts** the staging app's instances.
2. **Warm up.** The platform waits for the staging instances to restart and warms them by sending requests to the site root, or to your custom warm-up path (`applicationInitialization` in `web.config` on Windows; `WEBSITE_SWAP_WARMUP_PING_PATH` and `WEBSITE_SWAP_WARMUP_PING_STATUSES` app settings on Linux and containers). If an instance fails to warm up, the swap is aborted.
3. **Switch routing.** Once every instance is warm, the platform swaps the routing rules between slots. Production traffic now reaches the already-warmed instances — no restart on the production hostname.
4. **The old production code** now lives in the staging slot, with the staging slot's sticky settings, ready for an instant **swap back** as a rollback.

**Swap with preview** splits this into two phases: phase 1 applies the target settings and stops, so you can validate the staging slot *with production configuration* before completing the swap (or cancelling it).

**Sticky vs swapped settings** is where most slot bugs live:

| Should be sticky (stay with the slot) | Should swap (travel with the code) |
|---|---|
| Connection strings to environment-specific resources (a staging database) | Feature flags that belong to the code version |
| Monitoring settings that label the environment | Settings the new code requires |
| Scale settings, certificates, custom domains, IP restrictions (these are always slot-specific) | — |

A classic failure: the new code needs a new app setting, it's added to the staging slot but not marked sticky, the swap moves it into production (good) — but the *old* code now in staging still has it (usually harmless); or the reverse — a sticky setting was meant to swap, so production runs new code with old configuration.

**Other slot features:**

- **Auto-swap**: deploy to staging and swap automatically after warm-up — good for continuous deployment of low-risk apps.
- **Traffic routing (testing in production)**: route a percentage of production traffic to a slot, with an `x-ms-routing-name` cookie pinning users. It's a crude canary: no automatic analysis or rollback — you watch the metrics and decide.
- **Slots share the plan.** A slot under a heavy load test competes with production for CPU. For performance testing, use a separate plan.
- **Database schema changes** are not swapped. Slots make *code* rollback instant; they don't roll back migrations. Expand/contract migrations (Module 19) are what make a swap-back safe.

**The interview-grade sentence:** *"A slot swap applies production's sticky settings to the staging instances, restarts and warms them via the warm-up path, and only then swaps routing — so production never sees a cold start and the old version sits in staging as an instant rollback. The failure modes are sticky-setting mistakes, slots competing with production on the same plan, and forgetting that schema changes don't swap back."*

---

## Concept 14 — App Service networking: inbound, outbound, and SNAT

App Service networking confuses people because **inbound and outbound are completely separate features**. Keep them apart:

**Inbound — who can reach the app:**

| Feature | What it does | Notes |
|---|---|---|
| Public endpoint (default) | `*.azurewebsites.net` on shared front ends | Multi-tenant front ends; TLS terminated by the platform |
| **Access restrictions** | Allow/deny rules by IP range, VNet subnet (service endpoint) or **service tag** | Put Front Door in front and allow only the `AzureFrontDoor.Backend` tag **plus** a check of the `X-Azure-FDID` header value, so only *your* Front Door profile can reach the app |
| **Private endpoints** | A private IP for the app in your VNet; public access can be disabled | The standard way to make an app internal-only |
| **ASE with internal load balancer** | The whole environment is private | Isolated tier (Concept 10) |

**Outbound — what the app can reach:**

| Feature | What it does | Notes |
|---|---|---|
| Default outbound | Through shared outbound IPs of the stamp (listed on the app) | Outbound IPs can change when you change tiers or move stamps |
| **Regional VNet integration** | The app's outbound traffic enters a delegated subnet in your VNet | Lets the app reach private endpoints, on-premises (via VPN/ExpressRoute) and private DNS; enable "route all" to send all outbound through the VNet |
| **NAT gateway** on the integration subnet | Deterministic, static outbound IP(s) and a large SNAT port pool | The answer to "the partner needs to allow-list our IP" and to SNAT exhaustion |
| **Hybrid Connections** | Relay-based access to a specific host:port on-premises | Niche, Windows-centric |

**SNAT port exhaustion — the classic App Service production incident.** When an app makes outbound connections to public endpoints through the default path, each connection to the same destination IP and port consumes a **source NAT port** from a limited, pre-allocated pool per instance (historically 128). Code that creates a new `HttpClient` per request, disables connection pooling, or opens a new `SqlConnection` without pooling can exhaust the pool; new connections then **hang until timeout**, producing intermittent failures that don't correlate with CPU or memory. Fixes, in order:

1. **Reuse connections**: `IHttpClientFactory` or long-lived `SocketsHttpHandler` with `PooledConnectionLifetime` (Module 25, Concept 44), a singleton `CosmosClient`, one `ConnectionMultiplexer` for Redis, ADO.NET pooling left on.
2. **Avoid public paths for Azure services**: private endpoints or service endpoints via VNet integration don't consume SNAT ports on the load balancer.
3. **NAT gateway** on the integration subnet for the remaining public destinations — tens of thousands of ports per public IP.

This is a good example of a platform constraint that looks like an application bug: the same code runs happily on a VM with its own public IP and fails intermittently on App Service at scale.

**The interview-grade sentence:** *"On App Service inbound and outbound are separate: access restrictions, Front Door service tags plus the FDID header, and private endpoints control who reaches the app; VNet integration and a NAT gateway control what it reaches and from which IP. The classic incident is SNAT port exhaustion from not reusing connections, and I fix it with IHttpClientFactory and pooled clients, private endpoints for Azure services, and a NAT gateway for the rest."*

---

## Concept 15 — The operational features you get for free

A large part of App Service's value is capability you'd otherwise build, configure and operate yourself on Container Apps or AKS. Know them, because "we'd have to build that" is a legitimate argument for staying on App Service.

| Feature | What it does | The part to remember |
|---|---|---|
| **Health check** | Pings a path on every instance (about once a minute); instances that keep failing are taken out of load-balancer rotation, and ones that stay unhealthy are eventually replaced | Needs **at least 2 instances** to matter; limits how many instances it will exclude at once; the path should check the instance, not its dependencies (Module 13, Concept 41) |
| **Always On** | Keeps the app loaded instead of unloading it after idle time | Required for continuous WebJobs, in-process timers and `BackgroundService` work; without it, the first request after idle is a cold start |
| **Auto-heal** | Recycles the process on triggers: memory above a threshold, too many slow requests, specific status codes | A pragmatic mitigation for leaks while you fix them — record it as debt, not as a fix |
| **Easy Auth** (built-in authentication) | Entra ID and other identity providers at the platform edge, before your code | Good for internal apps; for APIs you usually want the ASP.NET Core authentication pipeline (Module 29) |
| **Managed certificates** | Free TLS certificates for custom domains, auto-renewed | Not wildcard; for wildcard use App Service Certificate or Key Vault |
| **Backups** | Scheduled app content (and, until 2028, linked DB) backups | Linked database backups are being retired — back up databases natively |
| **WebJobs** | Continuous or triggered background jobs in the same app | Share the app's instances; continuous WebJobs run on every instance unless set to singleton |
| **Sidecars** (Linux) | Additional containers alongside the main app — OpenTelemetry collectors, agents, local caches | Brings a slice of the container-app model to App Service |
| **Diagnose and solve problems**, Kudu/SCM, log streaming, profilers | Built-in diagnostics and a console on the instance | Strong for Windows .NET apps (memory dumps, profilers) |
| **Deployment Center / run-from-package** | GitHub Actions, zip deploy, `WEBSITE_RUN_FROM_PACKAGE` (mount a read-only package; atomic, faster cold start) | Run-from-package is the recommended code deployment mode |
| **Zone redundancy** | Spread plan instances across availability zones (Premium and Isolated) | Requires enough instances — plan for at least 3 (Concept 61) |

**The interview-grade sentence:** *"App Service ships a lot of operations for free — health-check instance replacement, Always On, auto-heal, Easy Auth, managed certificates, WebJobs, Linux sidecars, run-from-package, zone redundancy and deep diagnostics. When I compare it with Container Apps or AKS, I count what we'd have to rebuild, because that's often the real cost of moving down the ladder."*

---

## Concept 16 — App Service traps

These are the constraints that show up in production, not in demos.

**1. The 230-second request limit.** Requests that take longer than **230 seconds** to produce a response are cut off by the front-end load balancer's idle timeout — regardless of any timeout you set in Kestrel, IIS or your code. The client gets an error; your code may keep running. The fix is architectural: long work goes asynchronous — accept the request, return `202 Accepted` with a status URL, do the work in a queue-driven worker or a Durable orchestration, and let the client poll or receive a callback (Module 11). The same 230-second ceiling applies to HTTP-triggered Azure Functions on every plan (Concept 23).

**2. ARR affinity is on by default.** App Service's front end sets an `ARRAffinity` cookie that pins a client to an instance. For stateless APIs this causes **uneven load** (long-lived clients pile onto old instances after scale-out) and **sticky failures** (a client keeps hitting an unhealthy instance). Turn it off for stateless apps. If you need it, you have server-side session state — which also means data loss on instance restarts; fix that with a distributed cache (Module 10) rather than affinity. Blazor Server and SignalR need affinity or Azure SignalR Service.

**3. File system semantics.** On Windows, `D:\home` (and on Linux, `/home` for built-in stacks) is **persistent, shared storage across all instances**, backed by Azure Storage — slow for heavy I/O and a shared-state hazard. Local disks (`D:\local`, `/tmp`) are **per-instance and ephemeral**. For custom Linux containers, `/home` is only persistent if `WEBSITES_ENABLE_APP_SERVICE_STORAGE` is enabled. Never write application state to the file system; use Blob Storage.

**4. The platform restarts you.** Platform updates, host maintenance, scale operations, app setting changes and slot swaps all **restart app instances** — at times you don't choose. Survive them with: at least two instances (three for zones), health check, fast startup (Concept 49), graceful shutdown (Concept 51), and no in-memory state that matters.

**5. Outbound IPs aren't forever.** Default outbound IPs can change when you scale across tiers or when your app moves to another stamp. Anything a partner allow-lists should go through a NAT gateway with a static IP (Concept 14).

**6. Windows and Linux aren't the same product.** Feature availability, deployment mechanics, diagnostics and some settings differ. Windows has IIS, `web.config`, .NET Framework, Windows-specific profilers and Managed Instance; Linux has sidecars, custom containers, and generally lower prices. Check the feature per OS before designing around it. Windows containers are supported in some tiers, but not in Premium v4.

**7. Plan-level resource sharing is invisible in app metrics.** An app's latency can degrade because *another* app on the plan is consuming CPU. Monitor the **plan's** CPU and memory, not just each app's.

**8. Autoscale rules and the database.** Scaling the web tier out to 30 instances multiplies connection pools and query load on a database that doesn't scale with it. Cap app-level maximums (automatic scaling's "maximum scale limit") and size pools against the database's connection limit (Module 12).

**The interview-grade sentence:** *"The App Service traps I check for are the 230-second front-end timeout — long work goes async — ARR affinity left on for stateless APIs, state written to the shared or ephemeral file system, assuming instances aren't restarted by the platform, allow-listed outbound IPs without a NAT gateway, Windows/Linux feature gaps, noisy neighbors hidden in plan-level metrics, and scale-outs that overwhelm the database."*

---

## Concept 17 — When App Service wins

App Service is the **"boring on purpose"** choice, and boring is a feature. It wins when:

- **The workload is a web app or HTTP API** with steady or business-hours traffic — the shape it was built for.
- **The team is small** or has no appetite for platform work; App Service has the smallest operational surface of any always-on option.
- **You want batteries included**: slots with warm swap, health-check replacement, Easy Auth, managed certificates, WebJobs, deep .NET diagnostics.
- **You have many small apps** that can share plans for density (Concept 12).
- **You're modernizing .NET Framework** on Windows — the only managed PaaS that runs it, now with Managed Instance for apps that need OS customization.
- **Predictable billing** matters more than scale-to-zero.

It stops winning when:

- The system is **many containerized services** that need service discovery, per-service scaling on backlog, scale-to-zero, Dapr, or revision-based traffic splitting — Container Apps does that natively.
- Work is **event-driven and mostly idle** — paying for always-on instances to wait for a queue is waste; use Functions Flex or a Container Apps KEDA rule.
- A single app needs **more than 30 instances** (100 in an ASE) of your largest acceptable size.
- You need **Kubernetes-level control**, sidecar-heavy architectures beyond the supported sidecar feature, GPUs, or non-HTTP protocols (raw TCP/UDP, gRPC streaming with special needs).
- You need **requests longer than 230 seconds** — though that's an argument for an async design more than for a different platform.

**The interview-grade sentence:** *"App Service wins for web apps and APIs with steady traffic, small teams, many small apps sharing plans, and .NET Framework modernization — it's the smallest operational surface for always-on HTTP. It stops winning for many independently scaled containers, mostly-idle event-driven work, single apps beyond 30 instances, or anything needing Kubernetes-level control."*

---

# Part C — Azure Functions

## Concept 18 — The Functions model: a host that turns events into method calls

Azure Functions is the only rung on the ladder that imposes a **programming model**. You don't write a process; you write **functions**, and a **host** you don't control decides when to call them.

The moving parts:

- **The Functions host** — a process Microsoft ships (the Functions runtime, v4 today). It loads configuration from `host.json`, talks to event sources through **extensions**, decides when a function should run, handles retries and checkpoints for some triggers, and reports to the scale controller.
- **The language worker** — for .NET, your **isolated worker process**: a normal .NET application with its own `Program.cs`, DI container and middleware, talking to the host over gRPC (Concept 21).
- **Triggers** — exactly one per function: what starts it. HTTP, timer, Storage queue, Service Bus, Event Hubs, Event Grid, Blob (via Event Grid on Flex), Cosmos DB change feed, Kafka, SQL change tracking, Durable orchestration/activity/entity, and more.
- **Bindings** — declarative inputs and outputs: "give me this blob," "write my return value to this queue." They remove boilerplate at the cost of control.
- **The scale controller** — a platform component *outside* your app that watches event sources and decides how many host instances you need (Concept 20).

```csharp
// Isolated worker: a Service Bus-triggered function with a queue output binding.
public sealed class OrderPlacedHandler(IInventoryService inventory, ILogger<OrderPlacedHandler> log)
{
    [Function(nameof(OrderPlacedHandler))]
    [QueueOutput("shipping-requests", Connection = "Storage")]      // output binding: return value → queue
    public async Task<ShippingRequest?> Run(
        [ServiceBusTrigger("order-placed", Connection = "ServiceBus")] OrderPlaced message,
        FunctionContext context,
        CancellationToken ct)
    {
        bool reserved = await inventory.ReserveAsync(message.OrderId, message.Lines, ct);
        if (!reserved)
        {
            log.LogWarning("Reservation failed for {OrderId}", message.OrderId);
            return null;                                              // no output message
        }
        return new ShippingRequest(message.OrderId, message.Address);
    }
}
```

What this model buys you: **no host to write**, event-source integration with checkpointing and scaling built in, and billing that can reach zero. What it costs you: a runtime between you and the event source whose behavior (batching, concurrency, retries, checkpoints) you must learn; configuration spread across `host.json`, app settings and attributes; and code shaped around triggers — which is the lock-in (Concept 59).

A useful way to position Functions in an interview: **it's an event-processing framework with a hosting plan attached.** The framework (triggers, bindings, Durable) is the reason to choose it; the hosting plan decides cost and scaling. Since Functions can now be hosted on Container Apps, those two decisions are more separable than they used to be (Concept 19).

**The interview-grade sentence:** *"Functions is the one rung with a programming model: a host I don't control loads my isolated worker, listens to event sources through triggers, moves data through bindings, and a scale controller outside my app decides instance counts. It's an event-processing framework with a hosting plan attached — and the framework is both the reason to choose it and the lock-in."*

---

## Concept 19 — The Functions hosting plans in 2026

The lineup changed materially in the last two years. As of September 2026:

| Plan | Status | OS / packaging | Scale | Max instances | Cold start | Timeout (default / max) | VNet | Billing |
|---|---|---|---|---|---|---|---|---|
| **Flex Consumption** | GA; **the recommended serverless plan** | Linux, **code only** (no containers) | Event-driven, **per-function scaling** | **1,000** per function group (plus always-ready) | Improved; **always-ready** instances optional | 30 min / unbounded | ✅ | Executions + GB-s while executing (on-demand); baseline for always-ready |
| **Premium** (Elastic Premium) | GA | Windows code; Linux code or container | Event-driven with **prewarmed** workers | Windows 100; Linux 20–100 by region | None for always-ready; ≥ 1 instance always billed | 30 min / unbounded | ✅ | Core-seconds and memory of instances (min 1) |
| **Dedicated** (App Service plan) | GA | Windows code; Linux code or container | Manual/autoscale (Concept 11) | 10–30 (100 in ASE) | None with Always On | 30 min / unbounded | ✅ | App Service plan rates |
| **Container Apps** | GA | Linux **containers** | Event-driven via auto-generated KEDA rules | Up to 1,000 replicas (300 via portal); default 10 | Depends on min replicas | 30 min / unbounded (min replicas ≥ 1) | ✅ | Container Apps plan billing |
| **Consumption** | **Legacy**. Windows: GA. Linux: retiring **Sept 30, 2028** | Code only | Event-driven | Windows 200; Linux 100 | Yes (placeholders help) | 5 min / 10 min | ❌ | Executions + GB-s |

Key facts behind the table:

- **Flex Consumption** instance sizes are **512 MB (≈0.25 core), 2,048 MB (≈1 core), 4,096 MB (≈2 cores)**; 2,048 MB is the recommended default. The platform adds a 272 MB buffer for host processes that you don't pay for. It supports **.NET 8, 9 and 10 on the isolated worker model only** — the in-process model can't run there.
- **Flex has a regional quota**: by default **250 cores per subscription per region** across all Flex apps (including always-ready instances). A 2,048 MB app at 250 instances, or a 512 MB app at 1,000, consumes it all — and then every Flex app in that region stops scaling. Raise it via support before you need it.
- **Flex constraints** you must design for: **one app per plan**; **no deployment slots** (rolling-update "site update strategy" is in preview for zero-downtime deployments); **app initialization must finish within 30 seconds**; the Blob trigger must use the **Event Grid source**; `WEBSITE_TIME_ZONE` isn't supported; and you **can't migrate in place** into or out of Flex — you create a new app and redeploy.
- **Premium** workers come in three fixed sizes (1 vCPU/3.5 GB, 2 vCPU/7 GB, 4 vCPU/14 GB). It's the plan for near-continuous load with no cold starts, larger instances, Linux custom containers, deployment slots (up to 3), or multiple function apps sharing one elastic plan.
- **Dedicated** runs Functions on an App Service plan you already pay for — useful when you have spare capacity or need predictable billing, but you lose event-driven scale-to-zero. Turn on **Always On**, or timer and queue triggers stop firing when the app idles.
- **Functions on Container Apps** runs your function app as a container in a Container Apps environment, next to your other microservices, with KEDA rules generated from your triggers (you can override them). It's the path for custom images, GPUs, or putting functions inside the same network and environment as the rest of a containerized system — without deployment slots or custom domains on the function app itself.
- **Consumption is legacy.** Windows Consumption still works and is still the only *serverless Windows* option (for .NET Framework or Windows-only dependencies), but new apps should use Flex. **Linux Consumption gets no new language versions — .NET 9 was the last** — and apps still on the retired v3 runtime there **stop running after September 30, 2026**.

**The interview-grade sentence:** *"In 2026 Flex Consumption is the default serverless plan — Linux, code-only, isolated .NET 8–10, per-function scaling to a thousand instances, VNet support and optional always-ready instances — within a 250-core regional quota and with one app per plan, no slots and a 30-second init limit. Premium is for near-continuous load without cold starts, Dedicated for spare App Service capacity, Container Apps for containerized functions next to other services, and the old Consumption plan is legacy, with Linux Consumption retiring in 2028."*

---

## Concept 20 — How Functions scale: the scale controller, targets, and concurrency

The **scale controller** runs outside your app. For each trigger it reads the event source — queue length, Service Bus active messages, Event Hubs unprocessed events per partition, Cosmos change-feed lag, HTTP concurrency — and computes how many instances you need.

**Target-based scaling** (the model for Service Bus, Storage queues, Event Hubs, Cosmos DB, Kafka and others) uses a simple formula:

```
desired instances = ceil( event-source backlog / target executions per instance )
```

where the *target per instance* comes from your concurrency settings in `host.json` (for example, Service Bus `maxConcurrentCalls`, or a dedicated target setting per extension). Two important bounds:

- **Partitioned sources cap parallelism.** Event Hubs and Kafka can't use more instances than partitions — a 16-partition hub scales to at most 16 useful instances, however large the backlog (Module 11).
- **Your maximum instance count** caps everything — set it to protect downstream systems, because the scale controller doesn't know your database can only take 200 connections.

**HTTP scaling** is concurrency-based: each instance handles up to a target number of concurrent requests (on Flex the default depends on instance memory size; you can set it), and the platform adds instances as concurrency exceeds that target.

**Per-function scaling (Flex).** Flex groups functions by trigger type into **scale groups**: all **HTTP** (and SignalR) triggers scale together, all **Blob (Event Grid)** triggers together, all **Durable** triggers together — and **every other function scales on its own instances** (named `function:<name>`). So a Service Bus handler flooded with 100,000 messages scales out on its own instances without dragging the HTTP API along, and the HTTP API's always-ready instances aren't consumed by queue work. This is the biggest architectural change from the old Consumption model, where every function in an app scaled as one unit.

**Scale-out rate.** Flex adds instances in bursts that follow a **scale curve**: fast when an app runs few instances, progressively more measured at very high counts, and subject to three limits — the curve, your maximum instance count, and the regional core quota. Brief scale-out throttling is normal. If you know a burst is coming (a marketing email at 09:00), **always-ready instances** are pre-provisioned capacity that bypasses the on-demand curve.

**Concurrency is the most important knob you have**, because on Flex you're billed for **instance memory while it's executing** — not per execution. One instance handling 16 concurrent I/O-bound executions costs the same GB-seconds as one handling 1. Raising per-instance concurrency for I/O-bound .NET handlers (which spend most of their time awaiting) means fewer instances, fewer cold starts, fewer connections and a lower bill. Lower it for CPU-bound or memory-heavy work, where concurrency causes contention.

```json
// host.json — concurrency for a Service Bus handler (isolated worker, Service Bus extension v5)
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "maxConcurrentCalls": 32,            // per instance: raise for I/O-bound handlers
      "prefetchCount": 0,                  // prefetch + slow handlers = lock expiry; measure before raising
      "maxAutoLockRenewalDuration": "00:05:00"
    }
  },
  "concurrency": {
    "dynamicConcurrencyEnabled": false     // dynamic concurrency exists; start with explicit, measured values
  }
}
```

**The interview-grade sentence:** *"The Functions scale controller runs outside my app and computes desired instances as backlog divided by the per-instance target from my concurrency settings — bounded by partition counts, my maximum instance count and, on Flex, a 250-core regional quota. Flex scales per function group, so queue handlers scale independently of HTTP, and because Flex bills instance memory while executing, per-instance concurrency is my main cost and scaling lever for I/O-bound .NET handlers."*

---

## Concept 21 — The isolated worker model, and the November 10, 2026 deadline

.NET functions run in one of two models, and only one has a future:

| | **In-process** | **Isolated worker** |
|---|---|---|
| Where your code runs | Inside the Functions host process | In a separate .NET worker process, talking to the host over gRPC |
| .NET versions | Only LTS — **ending with .NET 8** | .NET 8, 9, 10 (LTS and STS), and .NET Framework 4.8.x |
| Dependency conflicts | Your assemblies share the host's (classic `Newtonsoft.Json` / SDK version clashes) | None — separate process |
| Startup, DI, middleware | Limited (`FunctionsStartup`) | Full control: your own host builder, DI, **middleware**, `IOptions`, OpenTelemetry |
| HTTP | Functions-specific `HttpRequest` via the host | **ASP.NET Core integration** (`HttpRequest`, `IActionResult`, minimal-API-like programming) |
| Plans | Not supported on Flex | All plans, including Flex |
| Support | **Ends November 10, 2026** — no security updates or bug fixes after that | Supported |

**The deadline is six weeks away.** After November 10, 2026, in-process apps keep running but receive **no security updates, no bug fixes and no new features** — and since .NET 8 itself also reaches end of support that same day, an in-process app will be running an unsupported host model on an unsupported runtime. That is a compliance finding in most organizations, not just a technical nicety. If you're asked "what would you do about our in-process functions?", the answer is a prioritized migration now, starting with internet-facing and business-critical apps.

A current isolated worker `Program.cs`:

```csharp
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.Hosting;

var builder = FunctionsApplication.CreateBuilder(args);

builder.ConfigureFunctionsWebApplication();          // ASP.NET Core integration for HTTP triggers

builder.UseMiddleware<CorrelationMiddleware>();       // function-level middleware (runs for every trigger)

builder.Services
    .AddApplicationInsightsTelemetryWorkerService()   // or OpenTelemetry (Module 28)
    .ConfigureFunctionsApplicationInsights()
    .AddSingleton(_ => new CosmosClient(builder.Configuration["Cosmos:Endpoint"], new ManagedIdentityCredential(
        ManagedIdentityId.FromUserAssignedClientId(builder.Configuration["AZURE_CLIENT_ID"]!))))
    .AddHttpClient<PricingClient>();                  // IHttpClientFactory: pooled connections (Concept 26)

builder.Build().Run();
```

The migration checklist, compressed:

1. Target a supported .NET (move straight to **.NET 10**, the current LTS) and set `FUNCTIONS_WORKER_RUNTIME=dotnet-isolated`.
2. Replace `Microsoft.NET.Sdk.Functions` and `Microsoft.Azure.WebJobs.*` packages with `Microsoft.Azure.Functions.Worker`, `Microsoft.Azure.Functions.Worker.Sdk` and the `Microsoft.Azure.Functions.Worker.Extensions.*` equivalents.
3. `[FunctionName]` → `[Function]`; inject `ILogger<T>` through the constructor; `FunctionsStartup` → the host builder.
4. Rework bindings that differ: several in-process binding types (`IAsyncCollector<T>`, SDK-typed bindings) have different isolated equivalents; for dynamic targets, Microsoft's guidance is to **use the service SDKs directly** instead of imperative bindings.
5. **Durable Functions**: move to `Microsoft.Azure.Functions.Worker.Extensions.DurableTask` (`TaskOrchestrationContext`, class-based orchestrators and activities) — and consider moving the backend to **Durable Task Scheduler** at the same time (Concept 24).
6. Deploy side by side (a new app or slot), compare behavior and metrics, then switch — and use the migration as the moment to decide whether this app should move to **Flex** too.

**The interview-grade sentence:** *"For .NET Functions I use only the isolated worker model: a separate process with my own host builder, DI, middleware and ASP.NET Core integration, on any .NET version and every plan including Flex. In-process support ends November 10, 2026 — the same day .NET 8 does — so any remaining in-process app is a prioritized migration now, and I'd move it to .NET 10 and consider Flex and Durable Task Scheduler in the same change."*

---

## Concept 22 — Cold start in Functions, and how to buy it down

Concept 6's chain applies in full, with some Functions-specific links: allocating an instance, mounting or downloading your package, starting the host, starting your **worker process**, loading extensions, running your `Program.cs`, then the first invocation's JIT and connection setup. Two hard limits bound it: the language worker must start within **60 seconds**, and on Flex the app must initialize within **30 seconds** (you'll see gRPC-related `TimeoutException`s if it doesn't).

What each plan offers:

| Plan | Cold-start mitigation |
|---|---|
| Flex Consumption | **Always-ready instances** per scale group or function (billed as baseline); faster platform scale-out |
| Premium | Always-ready instances plus **prewarmed** buffer instances; at least one instance always running |
| Dedicated | No cold start with **Always On** |
| Container Apps | **Minimum replicas ≥ 1** |
| Consumption (legacy) | Prewarmed "placeholder" hosts; nothing configurable |

What *you* control, in order of payoff:

1. **Startup I/O.** Don't fetch secrets from Key Vault synchronously in `Program.cs` if Key Vault references in app settings will do; don't warm up caches or run migrations at startup; create clients lazily or as singletons that connect on first use.
2. **Package size.** Smaller deployments download and load faster; trim unused dependencies; for large binaries or models, Flex can **mount an Azure Files share** instead of shipping them in the package.
3. **ReadyToRun.** `<PublishReadyToRun>true</PublishReadyToRun>` precompiles your assemblies and reduces JIT at startup (Module 17). Native AOT support for the isolated worker is limited — check current status before planning around it.
4. **Instance size.** Larger Flex instances get more CPU, which shortens JIT-heavy startup.
5. **Keep the HTTP path warm with always-ready**, and let queue/event functions — which tolerate a few seconds of latency — scale from zero.

**The mindset:** decide *which* functions need warm capacity. A payment webhook with a 2-second partner timeout needs always-ready instances; a nightly report generator doesn't. Flex's per-function scaling makes this choice granular.

**The interview-grade sentence:** *"Functions cold start is the platform chain plus my worker's startup, bounded by a 60-second worker limit and a 30-second init limit on Flex. The plan gives me always-ready, prewarmed or minimum instances; I add small packages, ReadyToRun, no network I/O in Program.cs, and always-ready capacity only for the functions whose callers can't wait — queue handlers can scale from zero."*

---

## Concept 23 — Timeouts and long-running work

Three different clocks govern how long a function can run, and confusing them causes production incidents.

**1. `functionTimeout` (host.json)** — the execution limit for any trigger. Defaults are **30 minutes** on Flex, Premium, Dedicated and Container Apps (with no enforced maximum when configured as unbounded), and **5 minutes (max 10)** on legacy Consumption. When an execution exceeds it, the worker process is restarted — all other in-flight executions on that instance fail too.

**2. The 230-second HTTP ceiling** — regardless of `functionTimeout`, an **HTTP-triggered** function must respond within **230 seconds**, because that's the idle timeout of the Azure load balancer in front of it (the same limit as App Service, Concept 16). A long HTTP-triggered job will see its client disconnect while it keeps running.

**3. Platform grace periods** — "unbounded" doesn't mean "never interrupted." On Flex and Premium, an execution gets a **60-minute grace period during scale-in** and **10 minutes during platform updates**; on Dedicated, 10 minutes during platform updates. After that, the instance goes away.

The design consequences:

- **HTTP functions should be short.** For anything longer, use the **async request-reply** pattern: accept, enqueue or start an orchestration, return `202 Accepted` with a status endpoint. Durable Functions implements this pattern out of the box (Concept 24).
- **Long work should be resumable.** Break it into steps with checkpoints (Durable activities, or a queue message per chunk), so a scale-in or platform update costs you one step, not the whole job.
- **Honor cancellation.** The isolated worker passes a `CancellationToken` that fires on host shutdown; long loops must check it and stop cleanly (Concept 51) — otherwise shutdown becomes a hard kill.
- **If the work is genuinely a multi-hour batch**, consider a **Container Apps job** (Concept 32) or Azure Batch rather than bending Functions around it.

**The interview-grade sentence:** *"Functions have three clocks: functionTimeout — 30 minutes by default on modern plans, 10 minutes max on legacy Consumption — a hard 230-second limit for HTTP responses from the load balancer, and grace periods of 60 minutes on scale-in and 10 on platform updates even when 'unbounded'. So HTTP functions stay short and hand off via async request-reply, long work is chunked and resumable, and multi-hour batch goes to a Container Apps job."*

---

## Concept 24 — Durable orchestration: replay, determinism, and Durable Task Scheduler

**Durable Functions** turns Functions from a stateless event handler into a **durable execution** engine: you write a workflow as ordinary `async` code, and the framework makes it survive process restarts, scale-in and failures.

**How it works — event-sourced replay.** The orchestrator function doesn't keep running while it waits. When it calls an activity, it **records** the call in a history log and **unloads**. When the activity completes, the framework **replays** the orchestrator from the beginning, feeding it the recorded results, until it reaches the next new step. It's event sourcing (Module 24) applied to control flow: the history is the source of truth; the orchestrator's local variables are a projection rebuilt on every replay.

```csharp
[Function(nameof(FulfilOrder))]
public static async Task<FulfilmentResult> FulfilOrder([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var order = ctx.GetInput<Order>()!;
    DateTime startedAt = ctx.CurrentUtcDateTime;                       // NOT DateTime.UtcNow: replay-safe clock

    await ctx.CallActivityAsync(nameof(ReserveInventory), order);

    // Fan-out / fan-in: one activity per line, awaited together.
    var tasks = order.Lines.Select(l => ctx.CallActivityAsync<Price>(nameof(PriceLine), l));
    Price[] prices = await Task.WhenAll(tasks);

    // Durable timer + external event: human approval for large orders, with a deadline.
    if (prices.Sum(p => p.Amount) > 10_000m)
    {
        using var cts = new CancellationTokenSource();
        Task timeout  = ctx.CreateTimer(ctx.CurrentUtcDateTime.AddHours(24), cts.Token);   // NOT Task.Delay
        Task<bool> approval = ctx.WaitForExternalEvent<bool>("Approval");
        if (await Task.WhenAny(approval, timeout) != approval || !approval.Result)
            return FulfilmentResult.Rejected;
        cts.Cancel();
    }

    await ctx.CallActivityAsync(nameof(CapturePayment), order, new TaskOptions(
        new TaskRetryOptions(new RetryPolicy(maxNumberOfAttempts: 3, firstRetryInterval: TimeSpan.FromSeconds(5)))));
    return FulfilmentResult.Completed;
}
```

**The determinism rules** follow from replay: the orchestrator must make the same decisions every time it replays the same history. So inside orchestrators — never in activities — you must not:

| Don't | Use instead |
|---|---|
| `DateTime.UtcNow` | `ctx.CurrentUtcDateTime` |
| `Guid.NewGuid()`, `Random` | `ctx.NewGuid()`, or generate in an activity |
| I/O: HTTP calls, database, file, environment variables | An activity |
| `Task.Delay`, `Thread.Sleep` | `ctx.CreateTimer(...)` |
| `Task.Run`, `ConfigureAwait(false)`, custom threads | Plain `await` on framework tasks |
| Non-deterministic iteration (unordered dictionaries, hash sets) | Deterministic ordering |

**The operational realities an interviewer will probe:**

- **Activities are at-least-once.** A crash after an activity did its work but before the result was recorded means it runs again. Activities must be idempotent (Module 11).
- **Versioning is the hard part.** Changing an orchestrator's code while instances are in flight can make replay diverge from history and fail. Strategies: keep old orchestrator versions side by side, use the framework's orchestration versioning support, drain before deploying breaking changes, or keep orchestrations short.
- **History grows.** Long loops should use `ContinueAsNew` to restart with fresh history (the eternal-orchestration pattern).
- **The backend matters.** Durable state lives in a **storage provider**. The recommended one is now **Durable Task Scheduler** — a fully managed backend (Dedicated SKU GA since November 2025, **Consumption SKU GA since March 2026**, billed per action dispatched) with a built-in dashboard. Bring-your-own providers (Azure Storage, MSSQL) remain supported. On Flex, only Azure Storage and Durable Task Scheduler are supported.

**Durable execution is no longer tied to Functions.** The **Durable Task SDKs** (.NET, Python, Java, JavaScript) run the same orchestration model **on any compute** — Container Apps, AKS, App Service, VMs — against Durable Task Scheduler. This is an important architectural decoupling: you can choose Durable-style workflows *and* a container platform, rather than choosing Functions to get workflows (Concept 59). The older Durable Task Framework (DTFx) library still exists but comes without official Microsoft support; new work should use Durable Functions or the Durable Task SDKs.

**When to use it:** multi-step business processes with waits, fan-out/fan-in batch work, human approvals with deadlines, sagas with compensation (Module 12), and async HTTP APIs. **When not to:** simple one-step event handling, very high-throughput stream processing, or workflows that are really just a queue and a handler.

**The interview-grade sentence:** *"Durable Functions is event-sourced control flow: the orchestrator records each step in a history, unloads, and is replayed from the start with recorded results — so orchestrator code must be deterministic, using the context's clock, GUIDs and timers and pushing all I/O into idempotent activities. Versioning in-flight instances is the real operational problem. Durable Task Scheduler is now the recommended managed backend, and the Durable Task SDKs let me run the same model on Container Apps or AKS without Functions."*

---

## Concept 25 — Triggers and delivery semantics: idempotency is your job

Every non-HTTP trigger is **at-least-once**. The host may deliver the same message or event more than once — after a crash, a lock expiry, a scale-in, a checkpoint that didn't land. Module 11's rule applies unchanged: **exactly-once processing = at-least-once delivery + idempotent handler.** What differs per trigger is *how failures are retried* and *where poison ends up*:

| Trigger | Retry mechanism | Where failures end up | The trap |
|---|---|---|---|
| **Storage queue** | Message becomes visible again; the host retries up to `maxDequeueCount` (default 5) | Moved to `<queue>-poison` | Nobody watches the poison queue |
| **Service Bus** | Abandon → redelivery until the entity's `MaxDeliveryCount` (default 10) | The entity's dead-letter queue | Lock expiry on slow handlers with high prefetch; long in-process retries holding locks (Module 25, Concept 64) |
| **Event Hubs** | The host **checkpoints after the batch completes — whether or not your code threw** | Nowhere: failed events are simply passed | Unhandled exceptions silently skip events; use a function retry policy and handle per-event failures yourself |
| **Cosmos DB change feed** | Lease-based checkpoints; retry policies available | Nowhere by default | Same as Event Hubs: catch and park failures |
| **Timer** | Singleton via a blob lease; missed schedules can run on startup (`RunOnStartup` is dangerous in production) | — | Overlapping runs if work exceeds the interval |
| **HTTP** | None — the caller retries | The caller | Non-idempotent POSTs retried by clients (Module 13) |

**Function-level retry policies** (`fixedDelay`, `exponentialBackoff`) are supported for a subset of triggers — notably Event Hubs, Kafka, Timer and Cosmos DB — and hold the instance while they wait. For Service Bus and Storage queues, rely on the broker's redelivery and dead-lettering instead, with at most a tiny in-process retry (Module 25, Concept 64).

**Bindings don't make multi-output work atomic.** An output binding writes after your function returns; if the function wrote to a database *and* returns a queue message, a failure between the two leaves them inconsistent. The dual-write problem (Module 11) exists inside Functions too — use an outbox, or make the second step idempotent and retryable.

**Checkpoints on streams are batch-level.** For Event Hubs, a batch of 100 events is checkpointed as a unit. If event 57 fails and you let the exception escape, the host moves on after the batch and events 57–100's failure is lost. Process each event in a `try/catch`, park failures (a dead-letter blob or queue), and emit a metric.

**The interview-grade sentence:** *"Every non-HTTP Functions trigger is at-least-once, so handlers must be idempotent. Queues and Service Bus retry via redelivery into poison or dead-letter queues; Event Hubs and the Cosmos change feed checkpoint per batch even if my code throws, so I catch per event and park failures myself. Output bindings don't make multi-system writes atomic — the outbox still applies."*

---

## Concept 26 — Functions traps

**1. Connection exhaustion from per-invocation clients.** Creating an `HttpClient`, `CosmosClient`, `ServiceBusClient` or `SqlConnection` per invocation multiplies sockets by concurrency and by instance count. Legacy Consumption limits instances to 600 active outbound connections; everywhere, you'll hit SNAT (Concept 14) and server limits. Register clients as singletons or through `IHttpClientFactory` in the isolated worker's DI.

**2. Scale-out overwhelms downstream systems.** The scale controller will happily run hundreds of instances against a database sized for ten. Always set a **maximum instance count** (per function group on Flex) and size per-instance concurrency against the downstream's connection and throughput limits. The effective concurrency against a dependency is `instances × per-instance concurrency` — multiply before you deploy.

**3. `host.json` defaults you didn't choose.** Batch sizes, prefetch, concurrency, lock renewal, retry, logging levels and sampling all have defaults tuned for "reasonable," not for your workload. Review `host.json` explicitly for every production app — it's configuration, and it's change-managed like code (Module 25, Concept 48).

**4. The storage account dependency.** Every function app needs `AzureWebJobsStorage` for host state, leases (timer singletons, blob receipts), keys and — on Flex — the deployment package. If that storage account is throttled, deleted, firewalled incorrectly or its keys rotated without updating the app, **the whole app stops**. Use managed identity for it, keep it in the same region, don't share it across many busy apps, and treat it as a hard dependency in your reliability model.

**5. Flex specifics.** One app per plan, no deployment slots, 30-second initialization limit, Event Grid-sourced Blob triggers only, several App Service-era settings deprecated, no in-place plan migration, and the regional core quota (Concept 19). Don't discover these during a migration weekend.

**6. Logging cost.** Application Insights ingestion for a high-throughput function can cost more than the compute. Configure sampling, log levels in `host.json`, and avoid logging per-message payloads (Concept 58).

**7. Hidden cold starts on rarely used functions.** A webhook invoked twice a day will cold-start almost every time on a scale-to-zero plan; if the caller has a short timeout, you'll see sporadic failures that "can't be reproduced." Always-ready for that function group, or an HTTP API on a warm platform.

**8. Testing against the host.** Business logic inside function classes is hard to unit test and couples it to triggers. Keep function classes thin: bind, validate, call an application service (Module 20), map the result.

**The interview-grade sentence:** *"The Functions traps I look for are per-invocation clients exhausting connections, scale-out that multiplies instances times concurrency against a database, unreviewed host.json defaults, the AzureWebJobsStorage account as an unacknowledged hard dependency, Flex's one-app-per-plan, no-slots and 30-second-init rules, Application Insights costs exceeding compute, cold starts on rarely called webhooks, and business logic trapped inside function classes."*

---

## Concept 27 — When Functions wins

Functions wins when the **programming model** earns its keep and the **traffic shape** suits consumption billing:

- **Event-driven glue** — reacting to queue messages, blobs, Event Grid events, Cosmos change feed, timers — where triggers, bindings and target-based scaling replace a lot of hand-written plumbing.
- **Bursty or mostly idle traffic** — the *Serverless in the Wild* profile: many apps invoked rarely. Scale-to-zero plus per-execution billing is dramatically cheaper than any always-on option.
- **Orchestrations** without running a workflow server — Durable Functions with Durable Task Scheduler.
- **Small, independently deployable units** owned by small teams, where operational simplicity matters more than control.
- **Fan-out processing** — thousands of parallel short tasks — on Flex's fast scale-out.

It stops winning when:

- **Traffic is high and flat.** Continuous load on consumption pricing is more expensive than provisioned capacity (Concept 7); Premium or a container platform wins.
- **You need control of the process** — custom middleware at the HTTP server level, long-lived connections (WebSockets for a real-time app), non-HTTP protocols, specific OS packages (Flex is code-only), or large memory.
- **The logic is a full web API** with dozens of endpoints — ASP.NET Core on App Service or Container Apps gives you the full framework with less ceremony; the ASP.NET Core integration in Functions narrows the gap but doesn't remove the host.
- **Portability matters.** Trigger-and-binding code is Azure-specific; a container running a `BackgroundService` isn't.
- **Latency SLOs are strict at the tail** and you aren't willing to pay for always-ready capacity.

**The interview-grade sentence:** *"Functions wins for event-driven glue, bursty or mostly idle traffic, Durable orchestrations, and small independently deployed units — the programming model replaces plumbing and scale-to-zero replaces idle capacity. It loses for high flat traffic, full web APIs, workloads needing process-level control or special OS packages, and anywhere portability or strict tail latency matters more than convenience."*

---

# Part D — Azure Container Apps

## Concept 28 — What Container Apps is: Kubernetes you don't operate

**Azure Container Apps** runs your containers on Microsoft-operated Kubernetes infrastructure, with a curated set of open-source components baked in — and **without exposing the Kubernetes API**:

| Component | Role in Container Apps | What you'd do yourself on AKS |
|---|---|---|
| Kubernetes (on AKS, operated by Microsoft) | Scheduling, restarts, rolling updates | Run and upgrade the cluster (Concept 42) |
| **KEDA** | Event-driven autoscaling, including to zero (Concept 31) | Install and operate KEDA |
| **Envoy** | Ingress, TLS termination, traffic splitting, session affinity | Choose and run an ingress/Gateway (Concept 40) |
| **Dapr** (optional) | Service invocation, pub/sub, state, bindings, secrets, actors | Install and operate the Dapr control plane |
| Managed OpenTelemetry agent (optional) | Export logs, metrics and traces | Deploy collectors |

The resulting abstraction sits between App Service and AKS on the ladder, and its sweet spot is precise: **containerized microservices, APIs and event processors that want per-service scaling — including to zero — revisions, internal service discovery and background jobs, without a platform team.**

What you describe is an **app** (image, CPU/memory, environment variables, secrets, ingress, scale rules, probes), not Deployments, Services, Ingresses, HPAs and ScaledObjects. What you give up is everything the Kubernetes API would let you do beyond that description: custom resources and operators, DaemonSets, privileged containers, arbitrary admission controllers, node-level access, fine-grained scheduling (affinities, taints, topology spread), and the ecosystem of Helm charts that assume a cluster.

That trade — **Kubernetes semantics, not Kubernetes operations** — is the reason Microsoft points Azure Spring Apps customers (retiring March 31, 2028) at Container Apps as their primary landing zone, and why Container Apps is now a hosting option for Azure Functions too.

**The interview-grade sentence:** *"Container Apps is serverless containers on Microsoft-operated Kubernetes with KEDA, Envoy and optional Dapr built in: I describe apps — image, resources, ingress, scale rules, probes — and get revisions, scale-to-zero, service discovery and jobs, but no Kubernetes API, so no operators, DaemonSets, privileged pods or custom scheduling. Kubernetes semantics without Kubernetes operations."*

---

## Concept 29 — Environments, plans, and workload profiles

**The environment** is the most important boundary in Container Apps. It's a secure boundary around a group of apps and jobs that:

- share **one virtual network** (created for you, or your own for fine-grained control);
- share **one logging destination** (a Log Analytics workspace, or other destinations/OTel);
- can call each other **by name** over the environment's internal network (Concept 33) and share **Dapr** configuration;
- are upgraded, scaled and balanced by the Container Apps runtime as a unit.

Use **one environment per "system that talks to itself"** — typically per product and per stage (prod, test). Use **separate environments** when apps must never share compute, must not communicate via built-in Dapr service invocation, or belong to different teams or trust levels. One operational gotcha: an environment that stays **idle, failed, or blocked from infrastructure updates for 90 days is automatically deleted** — don't leave an empty prod environment waiting for a launch.

**Two environment types:**

| Type | Status | Plans | Notes |
|---|---|---|---|
| **Workload profiles environment** | **Default** | Consumption **and** Dedicated | Supports UDR, NAT gateway, private endpoints, smaller subnets, larger Consumption replicas (4 vCPU / 8 GiB) |
| **Consumption-only environment** | **Legacy** | Consumption only | Replicas capped at 2 vCPU / 4 GiB; fewer networking features |

**Two plans, mixable inside one workload-profiles environment:**

- **Consumption** — serverless: replicas billed **per second** for allocated vCPU and memory while they exist, plus HTTP requests; scale to zero; a monthly free grant per subscription (**180,000 vCPU-seconds, 360,000 GiB-seconds, 2 million requests**). Replicas sitting at a non-zero minimum while doing nothing bill at a reduced **idle** rate (Concept 57).
- **Dedicated** — **workload profiles**: pools of dedicated, single-tenant VMs of a chosen type — general purpose (D-series), memory-optimized (E-series), and GPU profiles — that scale in and out between a minimum and maximum instance count. You pay **per profile instance** plus a **fixed plan-management fee** per environment, and pack many apps onto each profile. Dedicated wins for steady, high-utilization workloads, bigger replicas, compute isolation, or specialized hardware.

A common, sensible layout: latency-sensitive APIs with steady traffic on a small **Dedicated D-profile** (predictable cost, no cold starts), event processors and jobs on **Consumption** (scale to zero), GPU inference on **serverless GPU** or a GPU profile — all in one environment, calling each other by name.

**The interview-grade sentence:** *"In Container Apps the environment is the network, logging and service-discovery boundary, and the workload-profiles environment is the default — the Consumption-only type is legacy. Inside it I can mix the Consumption plan — per-second, scale to zero, free monthly grant — with Dedicated workload profiles, which are single-tenant VM pools billed per instance plus a management fee, so steady APIs can sit on dedicated capacity next to scale-to-zero processors and jobs."*

---

## Concept 30 — Apps, revisions, replicas, and traffic

Container Apps' deployment model is built on **revisions** — immutable snapshots of an app's *template* (image, containers, resources, scale rules, probes, volumes).

- **Revision-scope changes** (anything in `properties.template`) **create a new revision**.
- **Application-scope changes** (ingress, secrets, registries, Dapr settings, traffic weights) apply to all revisions **without** creating one. A secret value change doesn't restart replicas by itself — you restart a revision (or create one) to pick it up.
- **Replicas** are the running instances of a revision (conceptually, pods).

**Two revision modes:**

| Mode | Behavior | Use for |
|---|---|---|
| **Single** (default) | One active revision; a new revision replaces the old once it's ready — an automatic zero-downtime rollout when readiness probes are meaningful | Most apps; **required with non-HTTP event scale rules** |
| **Multiple** | Several active revisions at once, with **traffic weights** and **labels** | Blue/green, canary, A/B, and testing a new revision at its own URL before it takes traffic |

Blue/green in multiple mode, concretely:

```bash
# Deploy v2 as a new revision with zero traffic, labelled "green"
az containerapp update -n orders-api -g rg-prod --image acr.azurecr.io/orders-api:2.4.0 \
  --revision-suffix v240
az containerapp revision label add -n orders-api -g rg-prod --label green --revision orders-api--v240
# Test at the label URL (orders-api---green.<env-domain>), then shift traffic in steps
az containerapp ingress traffic set -n orders-api -g rg-prod \
  --revision-weight orders-api--v230=90 orders-api--v240=10
# ...observe error rate and p99 per revision... then 100% and deactivate the old revision
```

What revisions **don't** give you: automated canary analysis. Shifting weights is manual or scripted; if you want "roll back automatically when the error rate for the new revision exceeds X," your pipeline must query the metrics and act. Weighted traffic also splits per request, so a user can bounce between versions unless you enable session affinity — which matters for incompatible API or UI changes.

Compared with App Service slots (Concept 13), revisions are cheaper (no separate slot app, no shared-instance warm-up dance, old revisions can scale to zero) and more flexible (many revisions, arbitrary weights), but lack the swap-time configuration choreography of sticky settings.

**The interview-grade sentence:** *"A Container Apps revision is an immutable snapshot of the template: template changes create revisions, app-level settings like secrets and ingress don't. Single mode gives automatic zero-downtime replacement; multiple mode gives labelled revisions and weighted traffic for blue/green and canary — but the analysis and rollback decisions are my pipeline's job, and without affinity users can bounce between versions."*

---

## Concept 31 — KEDA scaling in Container Apps: the real defaults

Container Apps scales each revision **horizontally** between a minimum and maximum replica count, using **scale rules** implemented with KEDA. (There is **no vertical scaling** — a replica's CPU and memory are fixed per revision.)

**Limits and defaults:**

| Setting | Default | Range |
|---|---|---|
| Minimum replicas | **0** | 0 – 1,000 |
| Maximum replicas | **10** | 1 – 1,000 (subject to core quota) |
| HTTP rule `concurrentRequests` | **10** per replica | ≥ 1 |
| TCP rule `concurrentConnections` | **10** per replica | ≥ 1 |
| Default rule if you define none | HTTP, min 0, max 10 | — |

**Behavior** (these are the numbers that explain real-world scaling curves):

| Behavior | Value |
|---|---|
| Polling interval (custom/event rules) | **30 s** |
| Cool-down before scaling from the last replica to zero | **300 s** |
| Scale-up stabilization | **0 s** |
| Scale-down stabilization | **300 s** |
| Scale-up step | **1 → 4 → 8 → 16 → 32 …** up to max |
| Scaling algorithm | `desiredReplicas = ceil(currentMetricValue / targetMetricValue)` |

HTTP concurrency is computed every 15 seconds as requests in the last 15 seconds divided by 15. For event rules, each step scales to `min(maxReplicas, desiredReplicas, max(4, 2 × currentReplicas))` — so from 1 replica with a large backlog you go to 4, then 8, then 16: **doubling per decision**, not an instant jump.

**Rule types:**

- **HTTP** — concurrent requests per replica. Tune `concurrentRequests` from Little's Law: if a replica handles 400 req/s at 25 ms average latency, it has ~10 requests in flight; set the target somewhat below saturation (Module 6). The default of 10 is sensible only by coincidence.
- **TCP** — concurrent connections, for non-HTTP protocols.
- **Custom (any KEDA `ScaledObject` scaler)** — Service Bus (active message count), Event Hubs (unprocessed events, bounded by partitions), Storage queues, Kafka lag, Redis lists, Cosmos DB, and **CPU/memory** utilization. Authenticate scalers with **managed identity** where supported (Service Bus, Event Hubs, Storage queues) instead of connection-string secrets.

Multiple rules combine as **"first condition met wins"** for scale-out.

**Three consequences worth saying out loud:**

1. **CPU and memory rules can't scale to zero** — there's no utilization to measure with no replicas. An app scaled only on CPU needs `minReplicas ≥ 1`.
2. **An app with no ingress and no event rule scales to zero and never comes back.** The docs call this out: without ingress, you must define an event rule or set a minimum ≥ 1.
3. **Scale-to-zero has two costs**: the cold start on the next request (Concept 6), and the 300-second cool-down *before* reaching zero — so "idle" apps with sporadic traffic may never actually reach zero, and bill at the idle or active rate in between.

A Service Bus-driven processor in Bicep:

```bicep
scale: {
  minReplicas: 0
  maxReplicas: 30                      // cap: 30 replicas × 16 concurrent handlers = 480 against the DB
  rules: [
    {
      name: 'orders-queue'
      custom: {
        type: 'azure-servicebus'
        metadata: {
          namespace: 'sb-orders-prod'
          queueName: 'order-placed'
          messageCount: '50'           // target backlog per replica: desired = ceil(backlog / 50)
        }
        identity: userAssignedIdentityId   // managed identity, no secret
      }
    }
  ]
}
```

**The interview-grade sentence:** *"Container Apps scales horizontally only, from 0 to 10 replicas by default with 10 concurrent requests per replica, via KEDA: desired replicas is metric over target, event sources are polled every 30 seconds, scale-up doubles per step from 1 to 4 to 8, scale-down waits 300 seconds, and reaching zero takes a 300-second cool-down. CPU and memory rules can't scale to zero, and an app without ingress or an event rule scales to zero and never wakes."*

---

## Concept 32 — Container Apps jobs: batch without a warm replica

A **Container Apps job** is a containerized task that **runs to completion** and exits. Unlike an app, a job has no ingress and no always-on replicas; each **execution** starts one or more replicas, runs them until they exit, and bills at the active rate only while they run.

| Trigger type | Starts when | Typical use |
|---|---|---|
| **Manual** | You (or a pipeline, or an API call) start an execution | Migrations, one-off backfills, admin tasks |
| **Scheduled** | A cron expression fires (UTC) | Nightly reports, cleanup, periodic syncs |
| **Event-driven** | A KEDA scaler (e.g., queue length) indicates work — one execution per unit of work, as a `ScaledJob` | Queue-driven processing where each message is a long, isolated task |

Settings that matter:

- **Replica timeout** — the maximum duration of one execution's replicas; a runaway job is stopped.
- **Retry limit** — failed replicas (non-zero exit code) retry up to this count.
- **Parallelism** and **replica completion count** — how many replicas run per execution, and how many must succeed.

**Jobs vs apps vs Functions for background work:**

| Need | Choose |
|---|---|
| Continuous consumer of a stream or queue with many small messages | **App** with a KEDA rule (keeps a consumer loop hot while there's backlog) |
| Each unit of work is long (minutes to hours) and should be isolated | **Event-driven job** (one execution per message) |
| Periodic batch | **Scheduled job** (or a Functions timer trigger if it's short and you already use Functions) |
| Multi-step, stateful process | **Durable orchestration** (Concept 24) — possibly running on Container Apps via the Durable Task SDKs |

For .NET, a job is simply a console app (or a generic-host app whose `BackgroundService` completes and calls `IHostApplicationLifetime.StopApplication()`), exiting with **0 for success and non-zero for failure** so the platform's retry logic works. Make executions **idempotent** — a retry may re-run work partially done.

**The interview-grade sentence:** *"Container Apps jobs run to completion with no ingress and bill only while running: manual for migrations and backfills, scheduled by cron for batch, and event-driven via KEDA to start one execution per unit of long-running work. Exit codes drive retries, so the .NET job returns non-zero on failure and does idempotent work."*

---

## Concept 33 — Container Apps networking and service-to-service calls

**Ingress** is per app and handled by the environment's Envoy-based edge:

| Ingress setting | Meaning |
|---|---|
| **Disabled** | No inbound traffic (workers, processors) |
| **Internal** | Reachable only from inside the environment (and the VNet, for custom-VNet environments) |
| **External** | Reachable from the internet (or from the VNet if the environment itself is internal) |
| Transport | HTTP/1.1, HTTP/2 (incl. gRPC), or **TCP** (requires a custom VNet) |
| Extras | Custom domains with managed certificates, IP restrictions, CORS, session affinity, client certificates, weighted traffic (Concept 30) |

**Service discovery** is by app name: inside an environment, `http://orders-api` (or the fully qualified internal FQDN) reaches the app's internal ingress, load-balanced across replicas. No service mesh, no DNS configuration. Environment-level **peer-to-peer encryption** can encrypt this traffic with mTLS between apps.

**Custom VNets** give you control of the environment's network:

- The environment gets a **delegated subnet**; with workload profiles a **/27** is the minimum, but plan for growth — IP usage depends on replica and node counts. Undersized subnets are a common scale ceiling nobody noticed.
- An **internal environment** has no public endpoint; put Application Gateway or Front Door (with Private Link) in front for public traffic.
- **Egress control** — user-defined routes to Azure Firewall/NVA and **NAT gateway** for static outbound IPs — requires the **workload-profiles environment** type.
- **Private endpoints** to the environment are supported, but, like planned maintenance windows, they trigger the **Dedicated plan management charge** even if every app is on Consumption.

**For .NET apps behind Container Apps ingress**, three configuration items recur:

1. **Forwarded headers** — TLS terminates at Envoy, so enable `UseForwardedHeaders` (with known proxies/networks configured appropriately) to get correct scheme and client IP (Module 18).
2. **Data Protection** — with multiple replicas, ASP.NET Core Data Protection keys must be shared (Blob Storage + Key Vault key wrapping) or antiforgery tokens and auth cookies break across replicas. The Container Apps docs call this out as required for .NET apps that scale.
3. **Port** — .NET 8+ images listen on **8080** by default (Concept 47); set the ingress target port to match.

**The interview-grade sentence:** *"Container Apps ingress is per app — disabled, internal or external, HTTP/2 and gRPC, or TCP on a custom VNet — and services call each other by app name through the environment's Envoy edge, optionally with mTLS. Custom VNets need a properly sized delegated subnet; UDR and NAT egress require the workload-profiles environment; and .NET apps need forwarded headers, shared Data Protection keys and the 8080 target port."*

---

## Concept 34 — Dapr, sidecars, and the platform extras

**Dapr** (Distributed Application Runtime) can be enabled per app. A Dapr sidecar runs next to your container and exposes building blocks over HTTP/gRPC:

| Building block | What it gives you | When it's worth it |
|---|---|---|
| **Service invocation** | Name-based calls with retries, mTLS and tracing | Mostly redundant with Container Apps' own discovery; useful for portability |
| **Pub/sub** | Broker-agnostic publish/subscribe (Service Bus, Event Hubs, Kafka, Redis…) | Portability across brokers, or polyglot teams |
| **State management** | Key/value state with a pluggable store | Simple state needs; not a database replacement |
| **Bindings** | Input/output connectors to external systems | Similar niche to Functions bindings |
| **Secrets / configuration** | Pluggable secret and config stores | Portability |
| **Actors** | Virtual actors with turn-based concurrency | Actor workloads — **but actor apps can't scale to zero** |
| **Workflows** | Durable workflows | An alternative to Durable Task SDKs |

The honest assessment for a .NET team: **Dapr is a portability and polyglot abstraction.** If the whole system is .NET on Azure, the native SDKs (Azure.Messaging.ServiceBus, MassTransit, Wolverine) plus Container Apps' built-in discovery usually give you more control, better diagnostics and fewer moving parts. Dapr earns its place for multi-language systems, hybrid or multi-cloud portability requirements, or teams that want a uniform abstraction over infrastructure — and it's another component whose semantics (retries, delivery guarantees, component configuration) you must learn.

**Other container features:**

- **Init containers** run to completion before the app starts (schema checks, config fetch). Note: on the Dedicated plan and in Consumption-only environments, init containers can't use managed identity at run time.
- **Sidecar containers** share the replica's network and volumes (log shippers, local proxies). Use sparingly; most microservices should be one container per app.
- **Volumes**: ephemeral (replica-scoped scratch), **Azure Files** (shared, persistent — for legacy needs, not a database), and **secret volumes**.
- **Secrets** at app scope, including **Key Vault references** resolved with managed identity.
- **Managed identity** for image pulls (AcrPull), scale-rule authentication, and your code's Azure calls.
- **Health probes** (liveness, readiness, startup) — Concept 50.

**Platform extras that change the menu:**

- **Dynamic sessions** — pools of pre-warmed, Hyper-V-isolated sandboxes allocated per session in milliseconds, for running **untrusted or AI-generated code** (code-interpreter pools billed per session-hour; custom-container pools on dedicated E16 capacity).
- **Serverless GPUs** — GPU-backed replicas on the Consumption plan, billed per second with scale to zero (no idle rate); dedicated GPU workload profiles for steady inference.
- **Functions on Container Apps** — function apps as containers alongside other apps (Concept 19).
- **Durable Task SDK workers** — orchestrations with Durable Task Scheduler as the backend, on Container Apps (Concept 24).

**The interview-grade sentence:** *"Dapr in Container Apps gives broker-agnostic pub/sub, state, bindings, actors and workflows via a sidecar — valuable for polyglot or portability requirements, usually unnecessary for an all-.NET, all-Azure system that can use native SDKs, and actor apps can't scale to zero. Beyond that I get init and sidecar containers, Azure Files and secret volumes, Key Vault references, dynamic sessions for sandboxed code, serverless GPUs, and Functions and Durable workers on the same environment."*

---

## Concept 35 — Container Apps limits and traps

**1. No Kubernetes API.** No CRDs, operators, DaemonSets, privileged containers, host networking, node selection, pod affinity/anti-affinity or custom admission. If a vendor product ships only as a Helm chart with an operator, it won't run here. This is *the* line between Container Apps and AKS (Concept 45).

**2. Size ceilings.** On the Consumption profile a replica's containers must total one of the fixed combinations from **0.25 vCPU / 0.5 GiB up to 4 vCPU / 8 GiB** (1 vCPU per 2 GiB); Consumption-only environments stop at 2 vCPU / 4 GiB. Container images on the Consumption profile can total **up to 8 GB per replica**. Images must be `linux/amd64`. Bigger replicas mean a Dedicated profile.

**3. Cold starts at zero.** Scale-to-zero + a .NET image pull + app startup can be several seconds; for latency-sensitive APIs set `minReplicas: 1` (billed at the idle rate when truly idle) or use a Dedicated profile.

**4. Idle is not free, and "idle" has a strict definition.** A replica bills at the reduced idle rate only when it's at the configured minimum, all containers are running, it's processing **no HTTP requests**, using **under 0.01 vCPU**, and receiving **under 1,000 bytes/s** of network traffic. A chatty health check, a metrics scraper, a `BackgroundService` polling every second, or a Dapr sidecar can keep a "quiet" replica at the active rate all month.

**5. Subnet sizing and IP exhaustion** in custom VNets (Concept 33) — plan the environment subnet for peak replicas plus upgrade surges.

**6. Replica counts are targets, not guarantees**, and during platform upgrades you may temporarily see *more* replicas than expected as new ones are pre-warmed before traffic shifts.

**7. Log Analytics cost.** Every app in an environment writes to one destination; verbose console logging from dozens of replicas can make logging the largest line on the bill (Concept 58). Use structured logs, sensible levels, and consider OpenTelemetry export with sampling.

**8. No vertical scaling and no in-place resize** — changing CPU/memory creates a new revision.

**9. Revision-scoped vs app-scoped surprises** — updating a secret doesn't restart replicas; scale rules live on revisions, so in multiple-revision mode an old revision keeps its old rules.

**10. Quotas.** Regional core quotas cap total replicas across environments; the default can be lower than a large burst needs. Check before launch day.

**The interview-grade sentence:** *"The Container Apps limits I design around are no Kubernetes API, Consumption replicas capped at 4 vCPU and 8 GiB with 8 GB images, cold starts from zero, idle billing that only applies below 0.01 vCPU and 1 KB/s with no requests, subnet sizing in custom VNets, horizontal-only scaling, secrets that don't restart replicas, core quotas, and Log Analytics costs that can outgrow compute."*

---

## Concept 36 — When Container Apps wins

Container Apps wins when:

- The system is **several containerized services** — APIs, workers, processors — that need **independent scaling**, including to zero, and **name-based service discovery**, without anyone operating Kubernetes.
- Work is **event-driven** and you want a **process you own** (a normal .NET host with `BackgroundService`, MassTransit or Wolverine consumers) rather than a function-shaped programming model — KEDA rules give you Functions-like scaling for ordinary processes.
- You need **revision-based rollouts** — blue/green, canary with weighted traffic.
- You have **batch and scheduled work** alongside services (jobs), **GPU inference**, or **sandboxed code execution** (dynamic sessions).
- You want **portability**: the same images run on AKS later if you outgrow the platform (Concept 59).
- You use **Aspire**, whose default Azure deployment target is Container Apps (Concept 53).

It stops winning when:

- You need the **Kubernetes API** — operators, CRDs, DaemonSets, custom networking (service mesh with custom policies), node-level control, Windows containers.
- You need **replicas larger than 4 vCPU / 8 GiB** on Consumption and don't want to manage Dedicated profile capacity — or very large, stateful, clustered workloads.
- **Flat, very high utilization** makes AKS's denser bin packing and reserved VMs materially cheaper *and* you have the people to run it (Concept 57).
- The workload is **one or two simple web apps** — App Service gives you slots, Easy Auth and diagnostics with even less to think about.

**The interview-grade sentence:** *"Container Apps wins for a handful to dozens of containerized services and event processors that need independent, scale-to-zero scaling, service discovery, revisions and jobs without anyone running Kubernetes — and it keeps the AKS exit open because the artifact is an image. It loses when I need the Kubernetes API, very large or stateful clustered workloads, Windows containers, or when high flat utilization plus an existing platform team makes AKS cheaper."*

---

# Part E — Azure Kubernetes Service (AKS)

## Concept 37 — What Kubernetes actually gives you: an API and control loops

Before AKS, understand what Kubernetes *is*, because "container orchestrator" undersells it and leads to bad decisions.

**Kubernetes is a declarative API backed by reconciliation loops.** You write the **desired state** of the world as objects — "3 replicas of this pod template," "route this hostname to that service," "this job must complete 10 times" — and store them in the API server (backed by etcd). **Controllers** continuously compare desired state with observed state and act to close the gap: the ReplicaSet controller creates pods, the scheduler assigns them to nodes, the kubelet on each node starts containers, the endpoints controller updates service routing. Nothing is a one-shot command; everything is a loop converging on what you declared.

Three properties follow, and they're why Kubernetes won:

1. **Self-healing is the default.** A crashed container restarts, a lost node's pods reschedule, a deleted pod is replaced — not by a script someone wrote, but by the same loops that created them.
2. **Everything is an API object, and the API is extensible.** **Custom Resource Definitions (CRDs)** let anyone add new object types, and **operators** — controllers for those types — encode operational knowledge: "a `PostgresCluster` with 3 members, backups every hour, failover on primary loss." Certificate managers, service meshes, GitOps engines, KEDA, Karpenter and database operators are all just controllers watching the API.
3. **It's a platform for building platforms.** Kubernetes itself gives you primitives — Pods, Deployments, Services, ConfigMaps, Jobs, StatefulSets, DaemonSets, NetworkPolicies. The *developer platform* (golden paths, templates, self-service, guardrails) is something your organization builds on top of it (Concept 60). Container Apps is, quite literally, such a platform built on Kubernetes by Microsoft.

The corollary is the thing interviews probe: **Kubernetes' flexibility is a cost, not just a benefit.** Every capability comes with configuration choices — CNI, ingress or Gateway implementation, autoscaler, policy engine, secrets integration, observability stack, deployment tooling — and every choice is a component to version, upgrade and debug. Borg, Kubernetes' ancestor at Google, was run by a large dedicated SRE organization; the Kubernetes API makes those ideas available to everyone, but not the SRE organization.

**The interview-grade sentence:** *"Kubernetes is a declarative API plus reconciliation loops: I declare desired state as objects and controllers converge the cluster toward it, which gives self-healing by default and, through CRDs and operators, an extensible platform for building platforms. That extensibility is exactly what I pay for in configuration choices and upgrades — the API is free, the operating organization isn't."*

---

## Concept 38 — AKS architecture: control plane, node pools, tiers, Automatic vs Standard

**AKS** is Microsoft's managed Kubernetes. Microsoft runs the **control plane** (API server, etcd, scheduler, controller manager) in its own infrastructure; you get **node pools** of VMs in your subscription where your pods run.

**Node pools:**

- **System node pools** host critical system pods (CoreDNS, metrics server, konnectivity, some add-ons). At least one is required — unless it's **managed** (below).
- **User node pools** host your workloads. Use separate pools for different VM sizes (general, memory-optimized, GPU), OS (Linux, Windows), pricing (regular, **spot**), or isolation needs — and steer pods with node selectors, taints and tolerations.
- Pools are VM Scale Sets (or the newer Virtual Machines pool type that allows mixed sizes), optionally spread across **availability zones** (automatic zone placement is now available).
- **Node OS:** Ubuntu is the default on AKS Standard; **Azure Linux 3** is the default for AKS Automatic's system pool and a strong choice generally (Azure Linux 2.0 lost support on November 30, 2025); **Windows Server 2025** node pools are GA and become the default Windows SKU from Kubernetes 1.37.

**Pricing tiers** (control plane):

| Tier | For | Notes |
|---|---|---|
| **Free** | Dev/test | No financially backed API-server SLA |
| **Standard** | Production | Uptime SLA on the API server; higher control-plane scale |
| **Premium** | Production needing **Long-Term Support** | Adds LTS: an extended support window for selected Kubernetes versions |

**Two cluster modes — the most important AKS decision since 2025:**

| | **AKS Automatic** | **AKS Standard** |
|---|---|---|
| Positioning | Microsoft's **recommended production default** | Maximum flexibility and control |
| Nodes | **Node Auto Provisioning** (Karpenter) chooses VM sizes from pending pods; **managed system node pools** (GA since May 2026) run core components on Microsoft-owned infrastructure | You design node pools (or enable NAP yourself) |
| Security | **Deployment safeguards** enforce Kubernetes Pod Security Standards and default resource requests; Entra ID + Azure RBAC; workload identity; locked-down system pool | You choose and configure |
| Networking | Azure CNI Overlay with Cilium; **application routing with Gateway API** by default on 1.36+ | You choose CNI, ingress/Gateway |
| Upgrades | Automatic cluster and node OS upgrades with planned maintenance windows; deprecated-API checks | You choose channels and timing |
| Observability | Managed Prometheus, Container Insights, Grafana preconfigured | You enable and assemble |
| SLA | API server uptime **plus a pod readiness SLA** (with managed system node pools) | API server uptime (Standard/Premium tier) |
| Trade-off | Some knobs locked; opinionated defaults | Everything is your decision — and your work |

The **pod readiness SLA** is a notable shift: rather than only promising that the API server is up, Microsoft financially backs that qualifying pods actually reach readiness and serve traffic — which is much closer to what an application team cares about.

**How to choose between them:** start with **Automatic** unless you can name a requirement it blocks — a CNI or ingress you must use, node-level configuration its safeguards forbid, specialized node pool designs, or existing tooling that assumes full control. The ladder rule (Concept 3) applies *inside* AKS too.

**The interview-grade sentence:** *"AKS runs the control plane; I run node pools — system and user, by VM size, OS, spot or isolation. Tiers are Free for dev, Standard for the uptime SLA and Premium for long-term support. AKS Automatic is Microsoft's production default — node auto-provisioning, managed system node pools with a pod readiness SLA, enforced safeguards, Gateway API ingress, automatic upgrades — and I'd pick Standard only for a requirement Automatic blocks."*

---

## Concept 39 — Two-layer scaling, and the timing math

AKS scales at **two independent layers**, and most scaling incidents on Kubernetes come from forgetting the second one.

**Layer 1 — pods.**

- **Horizontal Pod Autoscaler (HPA)** adds or removes replicas of a Deployment to keep a metric near a target — by default CPU or memory utilization **as a percentage of the pod's resource requests**, evaluated on a roughly 15-second loop, with a scale-down stabilization window to prevent flapping.
- **KEDA** (an AKS add-on, and preinstalled on Automatic) extends HPA with event sources — queue length, Event Hubs lag, Prometheus queries, cron — and **scale to zero** for Deployments (Concept 31's model, with you owning the configuration).
- **Vertical Pod Autoscaler (VPA)** adjusts requests and limits per pod — useful as a recommender for right-sizing; don't let VPA and HPA fight over the same CPU metric.

**Layer 2 — nodes.**

- **Cluster autoscaler** watches for **pending pods** that can't be scheduled on existing nodes and adds nodes to the node pool that can fit them (within the pool's min/max), and removes underutilized nodes after a delay. It scales *pools you designed*.
- **Node Auto Provisioning (NAP)** — Microsoft's managed Karpenter — watches pending pods' requirements (CPU, memory, GPU, zone, architecture, spot tolerance) and **provisions right-sized VMs directly**, choosing from many VM sizes, then **consolidates** workloads onto fewer or cheaper nodes when possible (recent versions add a "balanced" consolidation policy to reduce node churn). It's preconfigured in AKS Automatic and optional in Standard.

**The critical interaction:** HPA creates pods *instantly*; if there's no room, they sit **Pending** until a node arrives. Node provisioning — create VM, boot, join the cluster, pull images, start pods, pass readiness — takes **on the order of one to several minutes**. During that window, demand is served by whatever capacity already existed.

**The timing math** (Concept 5, made concrete). Suppose:

- a service handles 100 req/s per pod at target utilization,
- traffic can ramp at **+300 req/s per minute** during a flash sale,
- pod start on an existing node takes **20 s**, and a new node takes **3 min** end to end.

Then while waiting for new nodes, traffic grows by `300 req/s/min × 3 min = 900 req/s` — **9 pods' worth** — that must fit on nodes that already exist. So the cluster needs **spare node capacity for ~9 pods** (plus a margin), permanently, or the ramp turns into queueing and timeouts. Three ways to provide it:

1. **Overprovisioning with low-priority "placeholder" pods** that reserve space and get preempted by real pods — the classic cluster-autoscaler pattern; the preempted placeholders go Pending and trigger new nodes *before* real demand needs them.
2. **Scheduled scaling** — raise HPA minimums (KEDA cron scaler) before known peaks.
3. **Faster nodes** — smaller images, **artifact streaming** from ACR (GA on AKS in 2026, pulls only the layers needed to start), pre-pulled images on node images, and NAP choosing available sizes quickly.

**Requests drive everything.** HPA percentages are relative to requests; the scheduler places pods by requests; the cluster autoscaler and NAP provision capacity by requests. A pod with no requests is invisible to all three — it neither triggers scale-out nor gets protected. That's why AKS Automatic's safeguards **inject default requests** when you forget them.

**The interview-grade sentence:** *"AKS scales in two layers: HPA or KEDA add pods in seconds, but pods that don't fit stay Pending until the cluster autoscaler or Node Auto Provisioning brings a node, which takes minutes. So I size spare capacity as the demand ramp rate times node provisioning time — with placeholder pods, scheduled pre-scaling or faster images — and I make sure every pod has resource requests, because requests are what HPA, the scheduler and the node autoscaler all reason about."*

---

## Concept 40 — AKS networking after ingress-nginx: CNI, Gateway API, and mesh

AKS networking is where Kubernetes' flexibility turns into decisions. In 2026 there are three layers to decide, and one of them just changed under everyone.

**1. Pod networking (CNI).**

| Option | Model | Status / when |
|---|---|---|
| **Azure CNI Overlay** | Pods get IPs from a private overlay CIDR; only nodes consume VNet IPs | **The default choice** — scales without exhausting VNet address space |
| **Azure CNI powered by Cilium** | eBPF data plane (with Overlay), built-in network policy, better observability | Recommended data plane; used by AKS Automatic; required for NAP self-hosted mode |
| Azure CNI (flat / VNet) | Every pod gets a VNet IP | When pods must be directly addressable from the VNet; plan IPs carefully |
| kubenet | Legacy overlay | **Retires March 31, 2028** — migrate |

**2. North-south traffic (ingress).** The long-time default — the community **ingress-nginx** controller — was retired: Kubernetes SIG Network and the Security Response Committee announced its retirement in November 2025, and **upstream maintenance ended in March 2026**. On AKS:

- The **application routing add-on in NGINX mode** receives Microsoft **critical security patches only through November 2026**. After that, managed NGINX is unsupported.
- The supported successor is the **application routing Gateway API implementation**: a lightweight, Istio-based control plane that reconciles **only Kubernetes Gateway API resources** (its own `approuting-istio` GatewayClass) — no sidecar injection, no Istio CRDs. It's the **default for new AKS Automatic clusters on Kubernetes 1.36+**. It can't be enabled at the same time as the Istio service mesh add-on.
- **Application Gateway for Containers** — Azure's managed L7 load balancer for AKS — supports both the Ingress API and Gateway API, with the data plane outside the cluster.
- The migration pattern is **parallel run**: the NGINX and Gateway API data planes get separate load-balancer IPs; there's no in-place flip. Deploy Gateway/HTTPRoute resources alongside existing Ingress objects, validate, then move DNS.

If you're asked about ingress on AKS today, the senior answer mentions all of this: **Gateway API is the long-term standard** (role-oriented: platform owns `Gateway`, app teams own `HTTPRoute`), Ingress still works but is where development stopped, and any cluster still on ingress-nginx has a dated migration.

```yaml
# Gateway API: platform team owns the Gateway; the orders team owns its route.
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-api
  namespace: orders
spec:
  parentRefs:
    - name: public-gateway          # Gateway managed by the platform team
      namespace: gateway-system
  hostnames: ["api.contoso.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /orders } }]
      backendRefs:
        - name: orders-api-v1        # weighted backends: Gateway API's built-in canary primitive
          port: 8080
          weight: 90
        - name: orders-api-v2
          port: 8080
          weight: 10
```

**3. East-west traffic (service mesh) — only when justified.** The **Istio-based service mesh add-on** gives mTLS between services, fine-grained traffic policy, retries/timeouts at the proxy, and telemetry. It also adds sidecars (or ambient-mode components), revision upgrades, and a control plane to operate. Module 25 (Concept 69) framed the rule: a mesh is worth it for fleet-wide policy, zero-trust mTLS requirements and outlier ejection across many services — and each failure kind still needs exactly one retry owner (mesh *or* application, not both). Cilium network policies give you L3/L4 (and some L7) segmentation without a mesh.

**Egress** is the last piece: default load-balancer outbound, **NAT gateway** for scale and static IPs, or **UDR to Azure Firewall** for inspection — plus Cilium FQDN policies or a static egress gateway where specific pods need specific IPs.

**The interview-grade sentence:** *"On AKS I default to Azure CNI Overlay with Cilium. Ingress-nginx is retired upstream and Microsoft's managed NGINX add-on is patched only through November 2026, so new clusters use Gateway API — the app-routing Gateway API implementation, default on AKS Automatic 1.36+, or Application Gateway for Containers — migrated by running both data planes in parallel and switching DNS. A service mesh comes only with a concrete mTLS or fleet-policy requirement, and never as a second retry owner."*

---

## Concept 41 — Identity, security, and the supply chain on AKS

On Container Apps and App Service, most of the security baseline is the platform's. On AKS Standard, it's **yours** — and it's where many clusters fail audits.

**Identity:**

- **Microsoft Entra Workload ID** — a Kubernetes service account is federated with a user-assigned managed identity; pods get Entra tokens without secrets. The .NET side is just `DefaultAzureCredential` or `WorkloadIdentityCredential` (Concept 52). It replaces the retired pod-managed identity.
- **Entra ID authentication + Azure RBAC for Kubernetes authorization** for humans and pipelines; **disable local accounts** so there's no static admin credential.
- **Kubelet identity** for image pulls from ACR with `AcrPull`.

**Workload security:**

- **Pod Security Standards** (baseline/restricted) enforced — AKS Automatic's **deployment safeguards** do this by default; on Standard, use Azure Policy (Gatekeeper) or built-in Pod Security Admission.
- **Resource requests/limits required**, containers **non-root**, **read-only root filesystem** where possible, no privilege escalation, dropped capabilities.
- **Network policies** — default-deny per namespace, then allow what's needed.
- **Secrets** from Key Vault via the **Secrets Store CSI driver** (or better, workload identity and direct SDK access), never baked into images or plain ConfigMaps.

**Supply chain:**

- **ACR** as the only allowed registry (policy-enforced), private endpoint, geo-replication for multi-region.
- **Image signing and verification** (Notary Project / Notation with Ratify, enforced at admission) and **vulnerability scanning** (Defender for Containers).
- **Minimal base images** — chiseled or distroless .NET images (Concept 47) shrink the CVE surface dramatically.
- **SBOMs** from your build.

**Cluster hardening:** private API server (or authorized IP ranges), regular node image upgrades (Concept 42), audit logs to Log Analytics or a SIEM.

The architect's framing: **every one of these is a decision and a component to maintain** — which is precisely the operational burden of Concept 8, and precisely what AKS Automatic and the higher rungs bundle for you. Module 29 covers identity and threat modeling in depth.

**The interview-grade sentence:** *"On AKS the security baseline is mine: Entra Workload ID so pods get tokens without secrets, Entra authentication with Azure RBAC and local accounts disabled, Pod Security Standards and network policies enforced — which AKS Automatic's safeguards do by default — Key Vault instead of Kubernetes secrets, ACR-only images that are signed, scanned and built on chiseled bases, and a private API server. Each item is a component to own, which is the real cost of the rung."*

---

## Concept 42 — The day-2 tax: upgrades, versions, node images, add-ons

"Day 2" is everything after the cluster is created. On AKS, it's a **continuous, calendar-driven workload**, and this concept is the one most teams underestimate.

**Kubernetes versions move fast.** Upstream ships roughly **three minor versions a year**. AKS supports a rolling window of recent minors, each for about a year; older versions become deprecated and then unsupported. As of September 2026: **1.36 is GA on AKS, 1.37 is expected in October 2026, 1.33 is available only under Long-Term Support, and 1.30 is deprecated.** Staying supported means **a minor-version upgrade several times a year**, forever. **LTS** (Premium tier) extends a version's life to buy time — it doesn't remove the upgrade, it delays it.

**Each minor upgrade can break you** through:

- **Removed or deprecated APIs** (the classic: manifests or Helm charts using API versions that were removed) — AKS checks for deprecated API usage and can block upgrades that would break workloads;
- **Add-on and extension compatibility** (Istio revisions, KEDA, CSI drivers, policy agents);
- **Behavior changes** in components like CoreDNS, kube-proxy (nftables vs iptables) or the CNI;
- **PodDisruptionBudgets** that block node drains — or no PDBs, so drains take down all replicas at once.

**Node images update weekly-ish** with OS security patches. You choose a **node OS upgrade channel** (node image or security-patch cadence) and a **planned maintenance window**; surge settings control how many extra nodes are added during rolling replacement. Newer safety nets help: **control-plane-only upgrades** (upgrade the control plane, validate, then node pools — now also into LTS), and **node pool rollback** (GA in 2026).

**Add-ons have their own lifecycles** — the ingress-nginx retirement (Concept 40) is a live example of an add-on end-of-life forcing a migration on a date you didn't choose. Istio revisions, Azure Linux major versions (2.0 → 3.0), Windows Server versions and kubenet are others.

**A realistic day-2 calendar** for a production AKS Standard estate:

| Cadence | Work |
|---|---|
| Weekly | Node image/security patch rollouts (automated, in maintenance windows); CVE triage for images |
| Monthly | Review deprecations and AKS release notes; patch add-ons; cost and capacity review |
| Every few months | Kubernetes minor upgrade: test in a staging cluster, check deprecated APIs, upgrade control plane, then node pools, validate |
| Yearly | Major component migrations (ingress → Gateway API, OS version, CNI), DR rehearsal: **rebuild a cluster from code** |

Two practices make this survivable: **treat clusters as cattle** — the whole cluster (and its add-ons and configuration) defined in Bicep/Terraform and GitOps, so you can build a new cluster and move traffic rather than upgrading in place under pressure (blue/green clusters) — and **automate upgrades with guardrails** (auto-upgrade channels plus maintenance windows plus PDBs plus good readiness probes).

**The interview-grade sentence:** *"The AKS day-2 tax is calendar-driven: roughly three Kubernetes minors a year, each supported about a year — 1.36 is current, 1.37 lands in October — so minor upgrades several times a year with deprecated-API checks, add-on compatibility and PDB-aware drains, plus weekly node image patches and add-on end-of-lifes like ingress-nginx. I make it survivable with auto-upgrade channels in maintenance windows, control-plane-first upgrades, and clusters defined entirely as code so I can replace rather than repair."*

---

## Concept 43 — Delivery on AKS: Helm, GitOps, and progressive delivery

On App Service you deploy a package to a slot; on Container Apps you create a revision. On AKS, **deployment is changing desired state in the API** — and the tooling you choose shapes your whole delivery process.

**Packaging:**

- **Helm** — templated manifests with values per environment; the dominant packaging format, and what most vendors ship. Aspire 13.3+ can generate Helm charts from an AppHost (Concept 53).
- **Kustomize** — overlay-based patching of plain YAML; built into `kubectl`.

**Applying — push vs pull:**

| Model | How | Trade-off |
|---|---|---|
| **Push** (CI/CD runs `helm upgrade` or `kubectl apply`) | Pipeline has cluster credentials | Simple; drift goes unnoticed; credentials in CI |
| **Pull / GitOps** (Flux — available as an AKS extension — or Argo CD) | An in-cluster agent syncs the cluster to a Git repository continuously | Git is the source of truth; drift is corrected; audit trail is the Git history; more moving parts |

GitOps is the idiomatic AKS model for multi-team clusters: application teams change their repo, platform teams change the platform repo, and the cluster converges — the reconciliation-loop idea (Concept 37) applied to delivery.

**Progressive delivery.** Kubernetes Deployments do rolling updates out of the box (`maxSurge`, `maxUnavailable`, gated by readiness probes). For **canary with automated analysis** — shift 5% of traffic, compare error rate and latency to baseline, promote or roll back automatically — add **Argo Rollouts** or **Flagger**, using Gateway API weighted `backendRefs` (Concept 40) or a mesh for traffic splitting. This is more capable than Container Apps' manual traffic weights or App Service's slot routing, and it's another controller to operate.

**Configuration and secrets** — ConfigMaps for non-secret config, Key Vault via workload identity for secrets, **Azure App Configuration** (with its Kubernetes provider) for shared, dynamic configuration and feature flags.

**The interview-grade sentence:** *"On AKS a deployment is a change to desired state: Helm or Kustomize for packaging, and GitOps with Flux or Argo CD so Git is the source of truth and drift gets reconciled. Rolling updates come free with good readiness probes; automated canary analysis needs Argo Rollouts or Flagger on Gateway API weights — more capable than slots or revision weights, and one more controller to own."*

---

## Concept 44 — Bin packing, multi-tenancy, and cost on AKS

AKS's cost advantage comes from **density**: many workloads packed onto shared nodes, reserved-instance pricing on those nodes, and spot capacity for interruptible work. The cost *risk* comes from the same place — you pay for **nodes**, whether or not your pods use them.

**How Kubernetes packs:**

- The scheduler places pods by **requests** (reserved CPU/memory), not by actual usage.
- **Limits** cap what a container may use: exceeding the memory limit gets it OOM-killed; hitting the CPU limit gets it **throttled** (Concept 48).
- **QoS classes** follow: *Guaranteed* (requests = limits), *Burstable* (requests < limits), *BestEffort* (none) — which determines eviction order under node pressure.

**So the unit of cost is the request.** Cluster cost ≈ Σ(node cost), and nodes exist to satisfy Σ(requests) + system overhead + headroom. Overstated requests (a service that requests 2 vCPU and uses 0.2) waste money exactly as surely as an oversized App Service plan — just less visibly. The rightsizing loop is: measure actual usage (Prometheus, VPA in recommendation mode), set requests near p90–p95 of real usage, set memory limits at a safe ceiling, and let NAP consolidate.

**Multi-tenancy inside a cluster:**

- **Namespaces per team or service**, with **ResourceQuotas** (cap total requests per namespace) and **LimitRanges** (defaults and bounds per container).
- **Node pools per workload class** — general, memory-optimized, GPU, **spot** (up to large discounts, but evictable with short notice — for batch, stateless workers with PDBs, dev environments).
- **Priority classes** so critical workloads preempt less critical ones.
- **Network policies** for isolation between tenants (Concept 41).
- Hard multi-tenancy (untrusted tenants) needs **separate clusters** or sandboxed runtimes; namespaces are a soft boundary.

**Visibility:** AKS **cost analysis** (built on OpenCost) attributes node costs to namespaces and workloads, including **idle cost** — capacity you pay for but no pod requests. Idle cost is the number to drive down.

**Commitment discounts** (reserved instances, savings plans) apply to node VMs — and Azure savings plans also cover App Service Premium v3/v4, Container Apps, and Functions Premium, so commitments aren't unique to AKS.

**The interview-grade sentence:** *"On AKS I pay for nodes, and nodes exist to satisfy resource requests, so the request is the unit of cost: I rightsize requests from measured usage, let Node Auto Provisioning consolidate, use namespaces with quotas and limit ranges for soft multi-tenancy, spot pools for interruptible work, and watch idle cost in AKS cost analysis. Density is AKS's cost advantage only if someone actually manages it."*

---

## Concept 45 — When AKS wins, and the honest "do you need it?" test

AKS wins when you need what only the **Kubernetes API** provides, or when your **scale and team** make its economics work:

- **Workloads the higher rungs can't host**: operators and CRDs (databases, Kafka via Strimzi, Orleans or Akka clusters with custom membership, ML platforms), DaemonSets (node agents), privileged or host-level access, Windows containers at scale, GPU scheduling across many models, custom schedulers, unusual protocols.
- **A platform team** that builds an internal developer platform on Kubernetes for **many** services and teams — where the operational cost is amortized and the uniformity is the product (Concept 60).
- **Portability requirements** across clouds or on-premises (AKS Arc, other Kubernetes distributions) — with the caveat that portability of manifests is real but portability of the *platform around them* (identity, storage, networking, managed services) is much weaker than it sounds.
- **High, steady utilization at large scale**, where dense bin packing on reserved nodes beats per-second consumption pricing by a wide margin *and* the people cost is already paid.
- **Fine-grained control** of networking, security policy, rollout automation or scheduling that the managed rungs don't expose.

**The honest test** — ask these before choosing AKS; if the answers are "no," you probably want Container Apps or App Service:

1. Can you name a **specific capability** you need that Container Apps (or App Service) doesn't provide? ("Flexibility" and "industry standard" aren't capabilities.)
2. Do you have — or will you fund — **people who will run it** for years: upgrades several times a year, networking, security baselines, observability, developer support?
3. Will **enough workloads** share the platform to amortize that cost?
4. Have you priced the **people** alongside the nodes (Concept 8)?
5. If you chose it for portability, is the **rest of the system** (data, identity, messaging) portable too?

A useful middle position to offer in interviews: **Container Apps first, AKS when a named requirement forces it, and AKS Automatic before AKS Standard.** Because Container Apps and AKS both run OCI images with Kubernetes-like semantics (probes, replicas, KEDA scaling, revisions vs Deployments), the move is a manageable migration rather than a rewrite — especially if the services were built twelve-factor style (Concept 59).

**The interview-grade sentence:** *"AKS wins when I need the Kubernetes API itself — operators, DaemonSets, privileged or Windows workloads, custom scheduling — or when a platform team serves many services and high steady utilization makes dense packing pay. Before choosing it I ask for a named capability, the people to run it, enough tenants to amortize them, and a total cost including engineers; otherwise it's Container Apps first, and AKS Automatic before Standard when a real requirement forces the step down."*

---

# Part F — The rest of the menu

## Concept 46 — VMs, Container Instances, Batch, Service Fabric, Static Web Apps — and the retirements

The four platforms above cover most .NET workloads, but a senior answer knows the rest of the menu and when each item is the right one.

| Service | What it is | Choose it when | Don't choose it when |
|---|---|---|---|
| **Virtual Machines / VM Scale Sets** | IaaS: you own the OS and everything above | Software that needs a full OS (legacy Windows services, licensed products, agents), special hardware, lift-and-shift as a first step | You'd be re-creating what App Service or Container Apps already gives you |
| **Azure Container Instances (ACI)** | Run a container or container group on demand, per-second billing, no orchestrator | Short-lived isolated tasks, CI agents, burst capacity for AKS via virtual nodes, simple sidecar groups | Long-running services needing scaling, discovery, rollouts — use Container Apps |
| **Azure Batch** | Managed pools of VMs for large-scale parallel and HPC jobs, with spot/low-priority support | Rendering, simulations, massively parallel compute with task scheduling | Ordinary background jobs — Container Apps jobs are simpler |
| **Service Fabric (managed clusters)** | Microsoft's older orchestrator with **stateful** Reliable Services and Reliable Actors | Existing Service Fabric estates; stateful services built on its programming model; the documented target for Cloud Services (extended support) migrations | Greenfield services — AKS or Container Apps are the mainstream choices |
| **Static Web Apps** | Global static hosting with integrated serverless APIs, auth and preview environments | SPAs and static front ends (Blazor WebAssembly, React) with a light API | Server-rendered apps or heavy APIs |
| **Azure Red Hat OpenShift / Arc-enabled Kubernetes** | OpenShift as a managed service; Kubernetes clusters anywhere managed from Azure | Organizations standardized on OpenShift; hybrid/edge Kubernetes | A single Azure-native estate |

**Retirements that shape migration work right now:**

- **Cloud Services (extended support)** is deprecated and **retires on March 31, 2027**. Microsoft's documented migration target is **Service Fabric managed clusters**, but the right destination depends on the workload: web roles often fit App Service or Container Apps, worker roles fit Container Apps (apps or jobs) or Functions, and only genuinely stateful Service Fabric-shaped designs belong on Service Fabric.
- **Azure Spring Apps** (all plans) **retires on March 31, 2028**; Microsoft recommends Container Apps (with its Java features) or AKS.
- **Linux Consumption for Functions** retires on **September 30, 2028** (Concept 19); **kubenet** on AKS on **March 31, 2028** (Concept 40); the Functions **in-process model** loses support on **November 10, 2026** (Concept 21).

These dates are architecture inputs, not IT trivia (Concept 63): a design review that proposes a platform should state its lifecycle position.

**The interview-grade sentence:** *"Beyond the big four: VMs and scale sets for software that needs a full OS, Container Instances for short isolated tasks and AKS burst, Batch for HPC-scale parallel jobs, Service Fabric managed clusters for existing stateful SF estates, and Static Web Apps for front ends. And I track retirements as inputs — Cloud Services extended support ends March 2027, Spring Apps March 2028 — choosing each migration target by workload shape rather than by the documented default."*

---

# Part G — .NET on these platforms

## Concept 47 — Containerizing .NET 10

Whatever platform you choose (except code-deployed App Service or Functions), the artifact is an OCI image. Getting it right pays off on every rung: smaller images pull faster (cold start, node scale-out), fewer packages mean fewer CVEs, and correct defaults avoid platform surprises.

**Build without a Dockerfile.** The .NET SDK can produce images directly:

```bash
dotnet publish src/Orders.Api -c Release -t:PublishContainer \
  -p:ContainerRepository=orders-api \
  -p:ContainerImageTags='"2.4.0;sha-3f9c2e1"' \
  -p:ContainerFamily=noble-chiseled               # distroless Ubuntu 24.04 base
```

or in the project file:

```xml
<PropertyGroup>
  <ContainerRepository>orders-api</ContainerRepository>
  <ContainerFamily>noble-chiseled</ContainerFamily>
  <InvariantGlobalization>true</InvariantGlobalization>   <!-- chiseled has no ICU; or use -extra images -->
  <PublishReadyToRun>true</PublishReadyToRun>             <!-- faster startup (Concept 49) -->
</PropertyGroup>
```

**Know the .NET 10 image defaults** — several changed in recent releases:

| Default | Value | Why it matters |
|---|---|---|
| Base distro for default tags (`10.0`) | **Ubuntu 24.04 "noble"** — and **no Debian images ship for .NET 10** | Scripts and Dockerfiles assuming Debian package names or `-bookworm` tags break |
| Listening port | **8080** (`ASPNETCORE_HTTP_PORTS=8080`) since .NET 8 | Ingress target ports, App Service `WEBSITES_PORT`, probes must use 8080, not 80 |
| User | Non-root **`app`** user available (`USER $APP_UID`); SDK container publish uses it by default | Required by pod security policies and AKS Automatic safeguards; can't bind to ports < 1024 |
| Chiseled (distroless) variants | `-noble-chiseled`, `-noble-chiseled-extra` (adds ICU and time zone data) | No shell, no package manager: much smaller attack surface, harder `docker exec` debugging |
| Other families | Alpine, **Azure Linux 3.0** (including distroless), AOT-oriented SDK images | Azure Linux aligns with AKS node OS and Microsoft's supply chain |

**A multi-stage Dockerfile** when you need one (native dependencies, custom build steps):

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["src/Orders.Api/Orders.Api.csproj", "src/Orders.Api/"]
RUN dotnet restore "src/Orders.Api/Orders.Api.csproj"
COPY . .
RUN dotnet publish "src/Orders.Api/Orders.Api.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled-extra AS final
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

**Image hygiene rules:**

1. **Immutable, unique tags** (version plus git SHA); never deploy `latest` — Container Apps' docs warn about caching and traceability problems.
2. **Rebuild regularly** even without code changes, to pick up base-image security fixes — on container platforms, *you* own base-image patching.
3. **One process per image**; one image can back several apps (an API and a worker with different entry points or arguments).
4. **Push to ACR in the same region** as the compute, pulled with managed identity.
5. **For the smallest possible image**, publish **self-contained + trimmed** or **Native AOT** onto `runtime-deps` chiseled images (Concept 49).

**The interview-grade sentence:** *"I build .NET images with SDK container publish onto chiseled Ubuntu 24.04 bases — .NET 10 ships no Debian images — running as the non-root app user on port 8080, with ReadyToRun, immutable version-plus-SHA tags, regular rebuilds for base-image CVEs, and ACR in-region pulled with managed identity. Smaller images mean faster cold starts and node scale-out, and fewer packages mean fewer vulnerabilities."*

---

## Concept 48 — The .NET runtime inside a container: CPU, memory, GC, and throttling

Module 14 covered GC modes and Module 15 the thread pool on a normal machine. In a container, the runtime reads the **cgroup limits** and adapts — and several defaults that are fine on a 16-core VM are wrong in a 0.5-vCPU replica.

**CPU.** `Environment.ProcessorCount` reflects the container's **CPU limit** (rounded up to a whole number), not the host's cores. A replica limited to 0.5 vCPU reports 1 processor; a 2-vCPU limit reports 2. The thread pool's minimum threads and Server GC's heap count derive from it. You can override with `DOTNET_PROCESSOR_COUNT`.

**Memory.** When a container memory limit is set, the GC's **heap hard limit defaults to 75% of it** (minimum 20 MB). The GC will work hard (and eventually throw `OutOfMemoryException`) to stay under that — leaving the remaining 25% for native memory: the runtime itself, JIT'd code, thread stacks, native libraries, and buffers outside the managed heap. Tune with `GCHeapHardLimit` or `GCHeapHardLimitPercent` when you know the native footprint. Exceeding the *container* limit gets the process **OOM-killed** by the kernel — no exception, no log line from your app, just a restart and exit code 137.

**GC mode.** ASP.NET Core defaults to **Server GC**: one heap and one GC thread per logical processor, optimized for throughput with more memory. In small containers that can be wasteful, so:

- **.NET 9 and later enable DATAS** (Dynamic Adaptation To Application Sizes) by default with Server GC: it starts with a small heap count and adapts heap count and budgets to the actual workload — making Server GC reasonable across container sizes and substantially reducing memory in lightly loaded replicas.
- For very small replicas (≤ 1 vCPU) with tight memory, **Workstation GC** (`<ServerGarbageCollector>false</ServerGarbageCollector>`) remains a valid choice — measure both.

**CPU throttling — the latency trap.** Linux enforces CPU limits with the **CFS quota**: a 1-vCPU limit means the container may use 100 ms of CPU time per 100 ms period, *summed across all its threads*. A .NET process that bursts on 4 threads (a GC, JIT, a parallel request spike) can burn its whole quota in 25 ms and then be **frozen for the remaining 75 ms** of the period. The symptom is p99 latency spikes at moderate average CPU — and CPU dashboards that show "only 40%."

What to do about it, per platform:

| Platform | Guidance |
|---|---|
| **AKS** | Set CPU **requests** accurately; for latency-sensitive services consider **no CPU limit** (or a generous one) so pods can burst into idle node capacity; always set **memory limits** (memory isn't compressible) — typically equal to the memory request |
| **Container Apps / App Service / Functions** | The allocation *is* the limit — you can't burst beyond it. Size replicas so steady-state CPU leaves headroom for GC and bursts, and prefer **fewer, larger** replicas over many 0.25-vCPU replicas for latency-sensitive .NET services |
| **All** | Watch throttling metrics (on AKS, `container_cpu_cfs_throttled_periods_total`) and the .NET `System.Runtime` counters (GC pause time, thread pool queue length) |

**A sizing starting point** — then measure:

| Replica size | GC suggestion | Notes |
|---|---|---|
| ≤ 0.5 vCPU, ≤ 1 GiB | Workstation GC or Server GC with DATAS; test both | Tiny replicas are prone to throttling; fine for low-traffic workers |
| 1–2 vCPU, 2–4 GiB | Server GC with DATAS (the default) | A good default size for .NET APIs on Container Apps |
| ≥ 4 vCPU | Server GC | Throughput-oriented services; watch heap count vs memory |

**The interview-grade sentence:** *"Inside a container .NET reads the cgroup limits: ProcessorCount follows the CPU limit, and the GC heap hard limit defaults to 75% of the memory limit, leaving room for native memory — exceed the container limit and the kernel OOM-kills you silently. Server GC with DATAS, on by default since .NET 9, adapts to small containers; and CFS throttling means a 1-vCPU limit can freeze a bursting process for most of each 100 ms period, so on AKS I set requests and often skip CPU limits for latency-sensitive services, and on the managed platforms I size replicas with burst headroom."*

---

## Concept 49 — Startup engineering: making .NET fast to start

Cold start (Concept 6) and scale-out speed (Concept 5) both depend on how quickly an instance becomes useful. Module 17 covered the mechanisms; here's how they apply to platform choice.

| Technique | Effect | Cost / constraint |
|---|---|---|
| **ReadyToRun** (`PublishReadyToRun`) | Precompiled native code for your assemblies; far less JIT at startup; tiered compilation still re-optimizes hot paths | Larger binaries; platform-specific builds |
| **Composite / framework-precompiled images** | Framework and app compiled together for faster startup | Larger images; check availability for your image family |
| **Native AOT** | No JIT at all; fast startup, small memory footprint, tiny images on `runtime-deps` | Only AOT-compatible code: minimal APIs, gRPC, worker services — no reflection-heavy libraries; MVC and some EF Core features aren't supported; test with the AOT analyzers |
| **Trimming** | Smaller self-contained deployments | Reflection breakage risk; requires trim-safe libraries |
| **`WebApplication.CreateSlimBuilder`** | Fewer default services and features | Opt back in what you need |
| **Removing startup I/O** | Often the biggest single win | Requires discipline: no synchronous Key Vault/App Configuration/DB calls blocking startup |

The last row deserves emphasis because it's the one with no trade-off. Typical startup offenders in .NET services:

- Loading configuration from Key Vault or App Configuration **synchronously** in `Program.cs` — prefer platform Key Vault references (Concept 52), or load in parallel with a timeout and cache.
- **EF Core migrations at startup** — run them as a separate job (a Container Apps job, a pipeline step), never in every replica's startup (Module 19).
- **Eager warm-up** of caches or clients before the host starts — move to a background warm-up gated by the readiness probe.
- **Token acquisition and TLS handshakes** on the first request — acceptable if the readiness probe waits for a warm-up call, otherwise the first user pays.

**Where it matters most:** scale-to-zero platforms (Functions Flex, Container Apps Consumption with `minReplicas: 0`), aggressive autoscaling (every new replica is a cold start), and AKS node scale-out (image size matters). On always-on App Service with warm-swap slots, startup time matters mainly for restarts and scale-out.

**Measure it**: log a timestamp at process start and at "ready," track the platform's replica start metrics, and trace the first request per instance. Optimize the slowest link first.

**The interview-grade sentence:** *"For .NET startup I first remove startup I/O — synchronous Key Vault and config loads, EF migrations, eager warm-ups — then use ReadyToRun or composite images, and Native AOT where the code is AOT-compatible, like minimal APIs and workers. It matters most on scale-to-zero platforms and aggressive autoscaling, where every new replica is a cold start, and I measure process-start-to-ready plus first-request latency per instance."*

---

## Concept 50 — Health probes, mapped onto each platform

Module 13 (Concept 41) established the probe semantics: **liveness** = "restart me if this fails" (process-level only, never dependencies); **readiness** = "don't send me traffic right now"; **startup** = "I'm still booting; don't judge liveness yet." Each platform implements a different subset:

| Platform | What exists | What it does on failure | .NET mapping |
|---|---|---|---|
| **App Service** | **Health check** (one path) + slot swap **warm-up** path | Removes instance from rotation; replaces instances that stay unhealthy | One endpoint that reflects *this instance's* ability to serve — closer to readiness |
| **Functions** | Platform-managed; no user probes on Flex/Consumption/Premium | Host restarts on failures/timeouts | Keep startup fast; nothing to configure |
| **Container Apps** | **Liveness, readiness, startup** probes (HTTP or TCP), per container; default TCP probes when ingress is on | Liveness: restart container; readiness: remove from ingress; startup: delay other probes | Map to `/health/live`, `/health/ready`, `/health/startup` |
| **AKS** | Liveness, readiness, startup probes, plus **PodDisruptionBudgets** for voluntary disruptions | Same as Kubernetes | Same; PDBs protect against drains taking all replicas |

The ASP.NET Core side, once, used on every platform:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddCheck<WarmupCompletedCheck>("warmup", tags: ["ready", "startup"])      // caches primed, first token acquired
    .AddCheck<LocalQueueDepthCheck>("backpressure", tags: ["ready"]);         // shed load by going not-ready

var app = builder.Build();

app.MapHealthChecks("/health/live",    new() { Predicate = r => r.Tags.Contains("live") });
app.MapHealthChecks("/health/ready",   new() { Predicate = r => r.Tags.Contains("ready") });
app.MapHealthChecks("/health/startup", new() { Predicate = r => r.Tags.Contains("startup") });
// Dependency checks (database, downstream APIs) go on a separate, un-probed endpoint for dashboards —
// failing readiness on a shared dependency takes EVERY replica out at once (Module 13, Concept 41).
```

Two platform-specific cautions:

1. **Probe traffic affects billing and scaling on Container Apps**: probes aren't billed as requests, but a replica that does heavy work in a probe handler, or a sidecar that stays busy, may never qualify for the idle rate (Concept 35).
2. **App Service's single health path** can't distinguish "restart me" from "don't route to me." Keep it instance-local, and keep dependency health out of it — otherwise a database blip marks every instance unhealthy.

**The interview-grade sentence:** *"I write one set of ASP.NET Core health endpoints — live, ready, startup — with dependency checks kept off the probed paths, and map them per platform: full liveness/readiness/startup probes on Container Apps and AKS plus PDBs on AKS, App Service's single health-check path as an instance-local readiness signal plus the swap warm-up path, and nothing to configure on Functions except fast startup."*

---

## Concept 51 — Graceful shutdown, per platform

Every platform stops instances — during scale-in, deployments, node drains, platform maintenance and slot swaps. **Graceful shutdown** means no request is dropped and no message is half-processed when that happens. The sequence is the same everywhere; the timings differ.

**The universal sequence:**

1. The platform decides to stop an instance and **removes it from load balancing** (endpoints, ingress, front end).
2. The platform sends **SIGTERM** to the process.
3. The process **stops accepting new work**, **finishes in-flight work**, and **exits**.
4. After a **grace period**, the platform sends **SIGKILL**.

**The .NET side.** The generic host translates SIGTERM into `IHostApplicationLifetime.ApplicationStopping`; Kestrel stops accepting new connections and waits for in-flight requests; hosted services' `StopAsync` runs and each `BackgroundService`'s `stoppingToken` is cancelled — all bounded by **`HostOptions.ShutdownTimeout`** (default 30 seconds in current .NET).

```csharp
builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(25);     // must be LESS than the platform's grace period
    o.ServicesStopConcurrently = true;                // stop hosted services in parallel
});

public sealed class OrderConsumer(ServiceBusClient client, ILogger<OrderConsumer> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await using ServiceBusProcessor processor = client.CreateProcessor("order-placed");
        processor.ProcessMessageAsync += HandleAsync;
        processor.ProcessErrorAsync   += args => { log.LogError(args.Exception, "Processor error"); return Task.CompletedTask; };

        await processor.StartProcessingAsync(stoppingToken);
        try { await Task.Delay(Timeout.Infinite, stoppingToken); }
        catch (OperationCanceledException) { /* shutting down */ }

        // Stop pulling new messages; in-flight handlers complete (or their locks expire and the broker redelivers).
        await processor.StopProcessingAsync(CancellationToken.None);
    }

    private static Task HandleAsync(ProcessMessageEventArgs args) => /* idempotent handler (Module 11) */ Task.CompletedTask;
}
```

**The race on Kubernetes.** Steps 1 and 2 happen **concurrently**, not in order: the kubelet sends SIGTERM while endpoint removal propagates to kube-proxy, ingress controllers and Gateways. For a few seconds, new requests can still arrive at a pod that has stopped accepting them → connection errors during every deployment. The standard fix is a **`preStop` sleep** that delays SIGTERM until routing has converged:

```yaml
spec:
  terminationGracePeriodSeconds: 45           # total budget: preStop + app shutdown + margin
  containers:
    - name: orders-api
      lifecycle:
        preStop:
          sleep: { seconds: 10 }              # native sleep action (no shell needed — works on chiseled images)
```

Budget: `preStop (10 s) + ShutdownTimeout (25 s) + margin < terminationGracePeriodSeconds (45 s)`. The native `sleep` action matters for chiseled images, which have no shell for the older `exec: sleep 10` pattern.

**Per platform:**

| Platform | Grace window | Notes |
|---|---|---|
| **AKS** | `terminationGracePeriodSeconds` (default 30 s) | preStop sleep for the endpoint race; PDBs for drains |
| **Container Apps** | Configurable termination grace period on the app (default 30 s) | SIGTERM → drain; same `HostOptions` discipline |
| **App Service** | Short and platform-controlled | Slots avoid most deployment-time shutdown pain; keep requests short; `ApplicationStopping` for cleanup |
| **Functions** | Cancellation token on host shutdown; **60 min on scale-in (Flex/Premium), 10 min on platform updates** | Long executions must check the token and checkpoint (Concept 23) |

**Message consumers** add one more rule: on shutdown, **stop pulling first**, then finish or abandon in-flight messages; anything interrupted will be redelivered, so handlers must be idempotent anyway.

**The interview-grade sentence:** *"Graceful shutdown is the same sequence everywhere — remove from routing, SIGTERM, stop accepting, drain, exit before SIGKILL — with HostOptions.ShutdownTimeout set below the platform's grace period. On Kubernetes, routing removal races SIGTERM, so I add a preStop sleep using the native sleep action because chiseled images have no shell; consumers stop pulling before draining, and Functions code honors the cancellation token within the scale-in and platform-update grace periods."*

---

## Concept 52 — Identity, configuration, and secrets across platforms

The goal on every platform is the same: **no secrets in code, images or plain configuration; identities instead of keys; configuration layered and changeable without rebuilds.** The mechanisms differ:

| Concern | App Service / Functions | Container Apps | AKS |
|---|---|---|---|
| Workload identity | System- or user-assigned **managed identity** | Managed identity | **Entra Workload ID** (service account ↔ user-assigned MI) |
| Secrets into config | **Key Vault references** in app settings: `@Microsoft.KeyVault(SecretUri=…)` | **Secrets with `keyVaultUrl`** + identity, referenced by env vars | **Secrets Store CSI driver** (Key Vault provider) — or read Key Vault directly via SDK |
| Shared/dynamic config & feature flags | **Azure App Configuration** (all platforms) | Same | Same, plus the App Configuration Kubernetes provider |
| Image pull | n/a (code) / MI with AcrPull (containers) | MI with AcrPull | Kubelet identity with AcrPull |

**Prefer user-assigned managed identities** for workloads: you can create them and grant RBAC roles *before* the app exists (no chicken-and-egg in IaC), they survive app recreation, and one identity can be shared by an app's slots or revisions. System-assigned identities are fine for simple, single-resource cases.

**`DefaultAzureCredential` in production.** It's convenient because it walks a chain (environment, workload identity, managed identity, developer tools…) — but in production that chain adds latency on first token acquisition, can pick an unexpected credential, and makes failures confusing. Microsoft's guidance is to **use a specific credential in production** (`ManagedIdentityCredential` with the client ID of a user-assigned identity, or `WorkloadIdentityCredential` on AKS) — or constrain `DefaultAzureCredential` (recent Azure.Identity versions support an `AZURE_TOKEN_CREDENTIALS` environment variable to limit it to production credentials) — and keep the full chain for local development.

```csharp
TokenCredential credential = builder.Environment.IsDevelopment()
    ? new DefaultAzureCredential()                                        // dev: VS, Azure CLI, azd...
    : new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(
          builder.Configuration["AZURE_CLIENT_ID"]!));                    // prod: one explicit identity
                                                                          // (on AKS: WorkloadIdentityCredential)
builder.Configuration.AddAzureAppConfiguration(o => o
    .Connect(new Uri(builder.Configuration["AppConfig:Endpoint"]!), credential)
    .ConfigureKeyVault(kv => kv.SetCredential(credential))                // resolves Key Vault references
    .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
    .ConfigureRefresh(r => r.RegisterAll().SetRefreshInterval(TimeSpan.FromMinutes(5))));
```

**Configuration layering** works the same on every platform because ASP.NET Core's configuration system sits above them: `appsettings.json` → environment-specific file → **environment variables** (set by the platform — `Section__Key` naming) → App Configuration / Key Vault. Platform settings are just environment variables — which is what makes a service portable between rungs (Concept 59).

**The interview-grade sentence:** *"On every platform: identities instead of keys — managed identity on App Service, Functions and Container Apps, Entra Workload ID on AKS — preferring user-assigned identities so RBAC exists before the app; secrets via Key Vault references, Container Apps Key Vault secrets or the CSI driver; App Configuration for shared config and flags. In production I use an explicit ManagedIdentityCredential or WorkloadIdentityCredential rather than the full DefaultAzureCredential chain, and platform settings arrive as environment variables so the service stays portable."*

---

## Concept 53 — Aspire as the deployment front door

Module 25 met **Aspire** through its service defaults. For compute choice, Aspire matters because it separates **what your system is** (the AppHost model: projects, containers, databases, queues and their connections) from **where it runs** (a *compute environment* you attach at publish time).

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Where things run in Azure. One environment can host all compute; several can coexist.
var aca = builder.AddAzureContainerAppEnvironment("aca");
var web = builder.AddAzureAppServiceEnvironment("web");

var sql      = builder.AddAzureSqlServer("sql").AddDatabase("orders");
var bus      = builder.AddAzureServiceBus("bus");

var api = builder.AddProject<Projects.Orders_Api>("orders-api")
    .WithReference(sql).WithReference(bus)
    .WithExternalHttpEndpoints()
    .WithComputeEnvironment(aca);                 // → Azure Container Apps

builder.AddProject<Projects.Orders_Worker>("orders-worker")
    .WithReference(sql).WithReference(bus)
    .WithComputeEnvironment(aca);

builder.AddProject<Projects.Storefront>("storefront")
    .WithReference(api)
    .WithComputeEnvironment(web);                 // → Azure App Service

builder.Build().Run();
```

Then:

- `aspire run` — local orchestration and the dashboard.
- `aspire publish` — generate deployment artifacts (Bicep for Azure, Docker Compose, Helm charts for Kubernetes).
- `aspire deploy` — provision and deploy (GA in Aspire 13.4): builds images, pushes them to an ACR that the platform pulls with **managed identity and AcrPull** (no registry passwords), provisions resources, deploys.
- `aspire destroy` — tear down deployments (Azure Container Apps, App Service, Kubernetes, Compose) — useful for ephemeral environments.

**Targets as of 2026:** **Azure Container Apps** (the long-standing default), **Azure App Service** (GA), **Kubernetes and AKS via Helm** (added in 13.3, with Ingress and Gateway API routing), and Docker Compose. `azd` workflows remain supported for teams already using them.

**What Aspire does and doesn't do for the compute decision:**

- ✅ It makes the platform a **late, swappable decision** for services modeled in the AppHost — moving a service from Container Apps to App Service is a line of code plus a redeploy, as long as you haven't used platform-specific features.
- ✅ It standardizes **service defaults** (OpenTelemetry, health checks, resilience, service discovery) across all targets (Concept 54).
- ❌ It doesn't **choose** the platform for you — the constraints in Parts B–E still apply.
- ❌ It doesn't replace your organization's **production infrastructure governance** where that exists (landing zones, network topology, policy); generated resource names are normalized and may not match conventions, so teams often use `aspire publish` output as input to their own IaC pipeline, or customize the generated infrastructure.

**The interview-grade sentence:** *"Aspire separates what the system is — the AppHost model of services and their dependencies — from where it runs: I attach compute environments like Container Apps, App Service or AKS via Helm, and aspire publish generates Bicep, Compose or Helm while aspire deploy builds, pushes to ACR with managed-identity pulls and deploys. It makes the platform a late, swappable decision and standardizes service defaults, but it doesn't make the decision for me or replace enterprise infrastructure governance."*

---

## Concept 54 — Observability parity across platforms

Module 28 covers observability in depth; the compute-choice point is narrower: **your telemetry should not change shape when your platform does.**

- **Instrument the application once** with OpenTelemetry (traces, metrics, logs) — Aspire service defaults do this — and export via OTLP to Azure Monitor (the Azure Monitor OpenTelemetry distro) or a collector.
- **Use standard resource attributes** — `service.name`, `service.version`, `deployment.environment` — and let the platform detectors add `cloud.platform`, region and instance identifiers, so dashboards and queries work regardless of where a service runs.
- **Know what each platform adds** on top of your telemetry:

| Platform | Platform signals you get | Where they go |
|---|---|---|
| App Service | HTTP logs, platform restarts, health-check events, **Diagnose and solve problems** analyzers | App Service logs, Azure Monitor, App Insights |
| Functions | Host logs, execution counts/durations, **scale controller** decisions, Flex deployment/quota diagnostics | App Insights, Azure Monitor |
| Container Apps | System logs (revision provisioning, scaling events, probe failures), replica metrics; optional **managed OpenTelemetry agent** | Log Analytics / OTel destinations |
| AKS | Node and pod metrics, Kubernetes events, control-plane logs | **Managed Prometheus**, Container Insights, Grafana — you assemble (Automatic preconfigures) |

- **Budget telemetry costs per platform** (Concept 58): high-volume logs from many replicas or function executions can cost more than the compute.

**The interview-grade sentence:** *"I instrument once with OpenTelemetry — service defaults in Aspire — export OTLP to Azure Monitor or a collector, and rely on standard resource attributes so dashboards don't care which platform a service runs on; each platform then adds its own signals, from App Service's analyzers to the Functions scale controller, Container Apps system logs and AKS's Prometheus stack. And I budget telemetry cost, because at scale it can exceed compute."*

---

# Part H — Deciding

## Concept 55 — The decision axes

With the four platforms understood, the decision reduces to a small number of **axes**. Most real decisions are settled by two or three of them; the skill is knowing which ones are decisive for *this* workload.

| Axis | Question | App Service | Functions (Flex) | Container Apps | AKS |
|---|---|---|---|---|---|
| **1. Workload shape** | API, web app, event processor, job, workflow, stateful, GPU? | Web/API | Events, glue, workflows | APIs, workers, processors, jobs, GPU | Anything, incl. stateful/operators |
| **2. Traffic pattern** | Steady, diurnal, spiky, mostly idle? | Steady/diurnal | Spiky/idle | Any; scale to zero | Steady at scale |
| **3. Latency & cold-start tolerance** | Can the first request after idle take seconds? | Always warm | Needs always-ready for strict SLOs | `minReplicas ≥ 1` for strict SLOs | Always warm (you pay for nodes) |
| **4. Execution duration** | Seconds, minutes, hours? | HTTP ≤ 230 s; WebJobs for longer | HTTP ≤ 230 s; async/Durable for long | Jobs for long batch | Anything |
| **5. Protocol & state** | HTTP only? WebSockets, gRPC streaming, TCP? In-memory state? | HTTP, WebSockets | Triggers; HTTP | HTTP/2, gRPC, TCP (VNet) | Anything; StatefulSets |
| **6. Size & scale ceiling** | Biggest instance? Most instances? | Up to 32 vCPU/256 GB (Pv4); 30 instances (100 ASE) | 4 GB instances; 1,000 per group; regional quota | 4 vCPU/8 GiB Consumption; larger on Dedicated; 1,000 replicas | Node-sized; thousands of pods |
| **7. OS & runtime needs** | Windows, .NET Framework, GPU, privileged, kernel? | Windows + .NET Framework; Managed Instance | Linux (Flex); Windows via legacy Consumption/Premium | Linux amd64; GPUs | Linux, Windows, GPU, privileged |
| **8. Networking & isolation** | Private only? Static egress? Single tenant? | VNet integration, private endpoints, ASE | VNet on Flex/Premium | VNet environments, UDR/NAT, Dedicated profiles | Full control |
| **9. Need for the Kubernetes API** | Operators, CRDs, DaemonSets, Helm-only vendors? | No | No | No | **Yes** |
| **10. Team & ops capacity** | Who runs it, for years? | Minimal | Minimal | Low | Significant (less with Automatic) |
| **11. Cost model & predictability** | Pay for peak or for use? Budget predictability? | Provisioned, predictable | Consumption; always-ready baseline | Both (Consumption + Dedicated) | Provisioned nodes + tier |
| **12. Portability & lifecycle** | Container portable? Programming-model lock-in? Retirements ahead? | Code or container | Programming model lock-in | Container; KEDA/Dapr portable to AKS | Most portable manifests; most upgrade work |

**Which axes are usually decisive?**

- **Axis 9 (Kubernetes API)** is binary and often settles AKS vs everything else by itself.
- **Axes 1 + 2 (shape and traffic)** usually decide between Functions/Container Apps (event-driven, spiky) and App Service/Dedicated profiles (steady HTTP).
- **Axis 10 (team capacity)** decides whether a technically justified AKS is organizationally viable.
- **Axis 7 (OS/runtime)** forces Windows workloads toward App Service, Managed Instance, AKS Windows pools or VMs.

The others usually *constrain* rather than *decide* — they rule options out (230 seconds, 8 GiB, 30 instances) or add cost.

**The interview-grade sentence:** *"I evaluate compute on about a dozen axes — shape, traffic pattern, cold-start tolerance, duration, protocol and state, size and scale ceilings, OS needs, networking and isolation, need for the Kubernetes API, team capacity, cost model, and portability and lifecycle. Usually the Kubernetes-API question, the shape-and-traffic pair and team capacity decide; the rest rule options out."*

---

## Concept 56 — The default ladder, and the triggers to step down

Here's a defensible default procedure — the kind you can draw on a whiteboard and justify line by line:

```
For each workload (not for the whole company):

 1. Is it event-driven glue, a scheduled task, or an orchestration — short executions,
    spiky or idle traffic, fits Linux code-only?                  ─── yes ──► Functions (Flex Consumption)
                                                                              (+ Durable Task Scheduler for workflows)
 2. Is it a web app or HTTP API — one or a few apps, steady traffic,
    small team, or .NET Framework / Windows dependencies?          ─── yes ──► App Service (Premium v4)
                                                                              (Managed Instance for legacy Windows needs)
 3. Is it one of several containerized services, workers or jobs that
    need independent scaling, scale-to-zero, revisions, jobs, GPUs?  ─── yes ──► Container Apps
                                                                              (Consumption; Dedicated profiles for steady load)
 4. Does it need the Kubernetes API, or is there a platform team
    serving many services with high steady utilization?             ─── yes ──► AKS Automatic
                                                                              (Standard only for what Automatic blocks)
 5. Does it need a full OS or special software/hardware?             ─── yes ──► VMs / VM Scale Sets
```

The procedure is ordered by **abstraction**, and the **triggers** for stepping down a rung are the important part — they're what you'll be asked to justify:

| From → To | Step-down triggers (name at least one) |
|---|---|
| **Functions → Container Apps** | Flat, high traffic where consumption pricing loses; need full process control (custom middleware, long-lived connections, OS packages, containers); portability requirement; cold-start SLO that always-ready can't meet economically; the app has become a full API |
| **App Service → Container Apps** | Several independently scaled services; scaling on backlog, not HTTP; scale-to-zero economics; revision-based canary; jobs alongside services; need beyond 30 instances without ASE; want one environment with service discovery |
| **Container Apps → AKS** | Need operators, CRDs, DaemonSets, privileged or Windows containers; vendor software shipped only as Helm charts with cluster-level components; custom networking or mesh policies; replica or scheduling needs beyond Container Apps' limits; a platform team already running Kubernetes for many services |
| **AKS Automatic → AKS Standard** | A specific CNI, ingress, node configuration or policy that Automatic's safeguards or defaults block |
| **Any → VMs** | Full OS control, licensed software tied to VMs, kernel modules, special hardware |

And the **step-up triggers** — moving *up* the ladder is equally legitimate and more often neglected:

| From → To | Step-up triggers |
|---|---|
| **AKS → Container Apps** | Manifests use only Deployments, Services, Ingress/HTTPRoute and KEDA; platform work is consuming the team; upgrades are routinely late |
| **VMs → App Service / Container Apps** | The "special" OS dependency turned out to be a config file, a font, or a registry key (consider Managed Instance) |
| **Container Apps → Functions or App Service** | The service is a single web app that would benefit from slots, Easy Auth and simpler operations; or a few event handlers that fit the Functions model |

**Why "per workload"?** Because a single company-wide platform decision is how you end up with a two-endpoint admin API on a Kubernetes cluster, or a 40-service system crammed into Functions. But — framing 7 — **cap the total**: two or three platforms, not one per workload (Concept 60).

**The interview-grade sentence:** *"My default, per workload, is Functions Flex for event-driven glue and orchestrations, App Service Premium v4 for a few steady web apps or anything .NET Framework, Container Apps for sets of containerized services and jobs, AKS Automatic when I need the Kubernetes API or have a platform team at scale, and VMs only for full-OS needs — and I step down a rung only on a named trigger, like operators or DaemonSets for AKS, while also stepping up when a cluster only runs Deployments and KEDA."*

---

## Concept 57 — Cost modeling, worked through

Interviewers rarely want exact prices; they want to see that you can **model** cost. Here is the method, with list prices where they're verified and symbols where they vary.

**Scenario — "Orders API":** a .NET 10 API, each replica 1 vCPU / 2 GiB. Business hours (≈ 10 h × 21.7 weekdays ≈ **217 h/month**) need **6 replicas**; the remaining **513 h** need **2 replicas** for availability, lightly used (assume active 30% of that time, idle 70%). **50 million requests/month.**

**Option A — Container Apps, Consumption plan** (US list prices: active $0.000024/vCPU-s and $0.000003/GiB-s; idle vCPU about $0.000003/vCPU-s; $0.40 per million requests; monthly free grant 180,000 vCPU-s, 360,000 GiB-s, 2 M requests):

```
Replica-seconds, peak:      6 × 217 h × 3,600          = 4,687,200
Replica-seconds, off-peak:  2 × 513 h × 3,600          = 3,693,600

Active vCPU-s:  4,687,200 + 0.3 × 3,693,600            = 5,795,280  − 180,000 free = 5,615,280 × $0.000024 = $134.77
Idle   vCPU-s:  0.7 × 3,693,600                         = 2,585,520               × $0.000003 = $  7.76
Memory GiB-s:   (4,687,200 + 3,693,600) × 2 GiB         = 16,761,600 − 360,000   × $0.000003 = $ 49.20
Requests:       50 M − 2 M free = 48 M                  × $0.40 / M                          = $ 19.20
                                                                                  Total ≈ $211 / month
```

Compare with running **6 replicas 24/7** on the same plan: 6 × 2,628,000 s × ($0.000024 + 2 × $0.000003) ≈ **$473/month** plus requests. Autoscaling with a sensible minimum saves over half — the diurnal shape is doing the work (Concept 7).

**Option B — provisioned (App Service Premium v4, or a Container Apps Dedicated profile, or AKS nodes).** You provision for the **peak** (6 replicas' worth) plus **zone headroom** (Module 13: survive one zone loss → typically +1 unit, or +50% with 3 zones and 2-of-3 capacity). Let `u` be the monthly price of one always-on unit equivalent to 1 vCPU / 2 GiB in your region and SKU. Then:

```
provisioned ≈ 7u      (6 for peak + 1 for zone headroom; autoscale can trim some of this)
break-even:  7u = $211  →  u ≈ $30 per vCPU/2 GiB-month
```

So the question becomes a lookup: **can you get an always-on 1 vCPU / 2 GiB unit for less than ~$30/month** (after reservations or savings plans)? Pay-as-you-go general-purpose compute is often around or above that; with 1- or 3-year commitments it's often below. That's the real shape of the answer: **for diurnal loads, autoscaled consumption and pay-as-you-go provisioned are close; commitments tip steady loads toward provisioned; scale-to-zero tips idle loads decisively toward consumption.** And you haven't yet added people (Concept 8) or hidden costs (Concept 58).

**Scenario — "Order events processor" on Functions Flex:** 10 million messages/month, 200 ms average execution, I/O-bound, on 2,048 MB instances. Flex bills **instance memory while executing**:

```
Concurrency 1 per instance:   10 M × 0.2 s                  = 2,000,000 instance-s × 2 GB = 4,000,000 GB-s
Concurrency 16 per instance:  10 M × 0.2 s / 16 (ideal)     =   125,000 instance-s × 2 GB =   250,000 GB-s
```

A **16× difference** in the compute meter from one `host.json` setting, before any price is applied — plus fewer instances, fewer cold starts and fewer connections. In practice packing isn't perfect and minimum billing granularity (1,000 ms minimum per execution, then 100 ms increments) inflates very short executions, so measure with the Flex billing metrics. The lesson generalizes: **on consumption plans, concurrency per instance is a cost lever; on provisioned plans, it's a density lever.**

**Scenario — AKS floor cost.** AKS has a **fixed floor** before your first pod: the control-plane tier (Standard/Premium for production SLAs), system node capacity (unless managed system node pools absorb it), load balancers and public IPs, NAT, disks, and the monitoring stack. For a small estate the floor dominates; for a large one it's noise, and dense packing on reserved nodes wins. Model it as:

```
AKS monthly ≈ floor (tier + system capacity + networking + monitoring)
            + Σ node-hours × node price   (sized from Σ requests ÷ target utilization + headroom)
            + people (Concept 8)
```

**The method, in four steps**, for any interview cost question:

1. **Shape**: peak, average, and idle hours; request and message volumes.
2. **Unit**: the smallest replica/instance that serves the workload; its per-second or per-hour price.
3. **Model**: consumption = Σ active + idle + requests − free grants; provisioned = peak units + headroom (minus commitments).
4. **Add** hidden costs and people, then compare — and state the break-even condition, not just the winner.

**The interview-grade sentence:** *"I model cost from the load shape: for a diurnal API on Container Apps Consumption, peak 6 replicas for business hours and 2 otherwise comes to about $211 a month versus $473 always-on, and provisioned only wins if an always-on 1-vCPU/2-GiB unit with zone headroom costs under roughly $30 — which commitments often achieve. On Functions Flex, concurrency per instance cuts the GB-second meter proportionally — 16× in my example — and on AKS I add the fixed floor and the people before comparing."*

---

## Concept 58 — Hidden costs

The compute meter is rarely the whole bill — and several "extras" are **triggered by architectural choices**, which is why architects must know them.

| Hidden cost | Triggered by | Mitigation |
|---|---|---|
| **Log ingestion** (Log Analytics, Application Insights) | Verbose logging × many replicas/executions; per-request payload logs | Structured logs at sensible levels, sampling, Basic/Auxiliary log tiers for high-volume tables, OpenTelemetry filtering, retention policies |
| **Minimum instances** | Always-ready (Flex), `minReplicas` (Container Apps idle rate), Premium's minimum instance, AKS system capacity | Keep minimums only where SLOs need them; per-function always-ready on Flex |
| **Idle that isn't idle** | Chatty sidecars, polling `BackgroundService`s, frequent health checks keeping Container Apps replicas at the active rate | Reduce background activity; measure active vs idle billing |
| **Dedicated plan management fee** (Container Apps) | Any Dedicated workload profile — **and** private endpoints or planned maintenance, even on Consumption | Know the trigger before enabling the feature |
| **NAT gateway** | Static egress IPs, SNAT mitigation | Needed often; budget it (hourly + per GB processed) |
| **Private endpoints** | Private access to PaaS services | Hourly + per-GB; consolidate where possible |
| **Egress bandwidth** | Cross-region replication, CDN origin traffic, external APIs | Keep chatty services co-located; CDN for static content |
| **Deployment slots / extra environments** | Staging slots share the plan (capacity), separate staging plans cost full price | Right-size non-prod; scale non-prod to zero off-hours |
| **ASE** | App Service Environment infrastructure | Only when isolation is required (Concept 62) |
| **AKS floor** | Control-plane tier, system pools, load balancers, disks, Managed Prometheus ingestion, Defender | Consolidate clusters; AKS Automatic managed system pools; right-size monitoring |
| **Container registry** | Premium tier for geo-replication/private endpoints | Share a registry per estate |
| **Regional quotas and headroom** | Flex 250-core default, core quotas | Not a cost, but a failure mode — raise before launch |
| **People** | Every platform, in different amounts | Concept 8 |

A good habit to show in interviews: **turn one cost line on and off in your head per design decision.** "Private endpoints on this Container Apps environment will add the Dedicated plan management charge and per-endpoint fees — the security requirement justifies it, but it's part of the decision."

**The interview-grade sentence:** *"Beyond the compute meter I budget log ingestion — often the biggest surprise — minimum and not-really-idle instances, the Container Apps Dedicated management fee that private endpoints trigger even on Consumption, NAT gateways, private endpoints, egress, non-prod environments, the AKS floor, and people. Several of these are triggered by architectural choices, so I attach each one to the decision that causes it."*

---

## Concept 59 — Lock-in and migration paths

"Lock-in" is not one thing. Separate it into layers, from cheapest to most expensive to escape:

| Layer | Examples | Lock-in | Escape cost |
|---|---|---|---|
| **Packaging** | OCI container image | Low | Days — images run everywhere |
| **Platform configuration** | Bicep/Terraform for App Service plans, Container Apps revisions, AKS manifests; slots vs revisions vs Deployments | Low–medium | Rewrite deployment definitions |
| **Platform features used by code** | Easy Auth headers, Functions `FunctionContext`, platform-specific environment variables, KEDA scalers (portable to AKS), Dapr components (portable where Dapr runs) | Medium | Replace with framework equivalents |
| **Programming model** | Functions triggers/bindings, `host.json` semantics, Durable orchestrations | **High** | Rewrite handlers as processes; move orchestrations to Durable Task SDKs or another engine |
| **Managed services the code depends on** | Cosmos DB, Service Bus, Event Hubs, Entra ID (Module 27) | Highest — and usually worth it | Data migration, semantic differences |

The key insight: **compute is usually the *least* locked-in layer of an Azure system** — if you keep the programming-model layer thin. The data and messaging services are where portability really gets decided, and moving compute between Azure rungs is far more common than moving clouds.

**Keep compute portable with twelve-factor discipline** — the same properties that make a service run well on any rung:

- configuration from **environment variables** (and App Configuration), never baked in;
- **stateless processes**, state in backing services;
- **port binding** (Kestrel on 8080), not a host-specific server;
- **disposability** — fast startup, graceful shutdown (Concepts 49, 51);
- **logs as streams** to stdout/OpenTelemetry;
- **dev/prod parity** — the same image everywhere.

**Keep the Functions layer thin** (Module 20's hexagonal idea): a function class is an *adapter* — bind, validate, call an application service, map the result. The application core then moves unchanged to a `BackgroundService` on Container Apps if the hosting decision changes.

**Common migration paths:**

| Path | What moves easily | What needs work |
|---|---|---|
| App Service (code) → App Service (container) | Everything | Dockerfile/SDK publish; port 8080 |
| App Service (container) → Container Apps | Image, env vars, health endpoints | Slots → revisions; Easy Auth → Container Apps auth or in-app auth; WebJobs → jobs/apps; data protection key sharing |
| Functions → Functions on Container Apps | The whole function app, as a container | Scale-rule overrides; no slots/custom domains on the function app |
| Functions → Container Apps process | Application core (if handlers were thin) | Rewrite triggers as consumers (Azure SDK, MassTransit/Wolverine) + KEDA rules; Durable → Durable Task SDKs + Durable Task Scheduler |
| Container Apps → AKS | Images, probes, env vars, KEDA rules (as ScaledObjects), Dapr | Write Deployments/Services/HTTPRoutes; own ingress, identity, upgrades |
| AKS → Container Apps | Workloads using only Deployments, Services, Ingress/HTTPRoute, KEDA, standard probes | Anything using CRDs, operators, DaemonSets, custom networking |
| Cloud Services → modern | Code that was already stateless | Roles re-shaped into apps and jobs (Concept 46) |

**The interview-grade sentence:** *"I split lock-in into packaging, platform config, platform features, programming model and managed services. Compute is usually the least locked-in layer if handlers stay thin and services follow twelve-factor rules — env config, stateless, port binding, fast start and shutdown, logs as streams — so moving between App Service, Container Apps and AKS is configuration work. The expensive lock-in is the Functions programming model and, above all, the data and messaging services."*

---

## Concept 60 — Mixing platforms, capping the count, and platform-as-a-product

Real estates are **mixed**, and that's healthy: a Static Web App front end, an App Service BFF, a dozen Container Apps services and workers, a handful of Functions for event glue, and — maybe — an AKS cluster for the few workloads that need the Kubernetes API. Each workload sits on the rung that fits it.

But every platform you add brings a full set of cross-cutting concerns:

| Concern | Must be solved *per platform* |
|---|---|
| Delivery | Pipeline templates, deployment strategy, rollback |
| Networking | VNet integration model, private endpoints, egress control |
| Identity | Managed identity patterns, RBAC assignments |
| Observability | Platform logs, dashboards, alerts |
| Security | Baselines, policy, vulnerability management |
| Operations | Runbooks, on-call knowledge, incident response |
| Cost | Tagging, budgets, rightsizing |
| Lifecycle | Retirement tracking, upgrades |

So: **decide per workload, but cap the estate at two or three compute platforms**, each with a named owner, a paved-road template and a runbook. "We use Container Apps for services and Functions for glue; App Service only for the legacy portal; AKS only for the stream-processing platform the data team runs" is a mature estate. "Each team picked its favorite" is not.

**Platform as a product.** When many teams share compute, a **platform team** that provides a **paved road** (Team Topologies calls this a *thinnest viable platform*) amortizes the operational burden (Concept 8): templates for a new service, a standard pipeline, identity and networking wired in, observability by default, and guardrails via policy. The CNCF platforms white paper frames the same idea: a platform is an internal product whose customers are developers. Two consequences for compute choice:

1. **The platform team's rung isn't the application team's rung.** A platform team might run AKS so that product teams see something closer to Container Apps — a "golden path" where a service is a short spec file. (Container Apps itself is Microsoft's version of this.)
2. **Don't build a platform team to justify a platform.** If Container Apps already *is* the thinnest viable platform for your needs, building an internal one on AKS is expensive duplication — unless there's a named requirement (Concept 45).

**The interview-grade sentence:** *"I decide compute per workload but cap the estate at two or three platforms, because each one needs its own delivery, networking, identity, observability, security and on-call story. When many teams share compute, a platform team offering a thinnest viable paved road amortizes that cost — but I don't build an internal Kubernetes platform that just recreates Container Apps without a named requirement."*

---

## Concept 61 — Reliability by platform: zones, minimums, and multi-region

Module 13 gave the principles (redundancy across failure domains, N+1 capacity, composite SLAs, active-active vs active-passive). Here's what each platform gives you and what it needs from you:

| Platform | Zone redundancy | What you must configure | Notes |
|---|---|---|---|
| **App Service** | Premium v3/v4 and Isolated plans can be zone-redundant | Enough instances to span zones (plan for ≥ 3), health check, ≥ 2 instances minimum | Instances restart during platform maintenance; health check + zones + slots cover most incidents |
| **Functions Flex** | Supported | With zone redundancy, **at least 2 always-ready instances** per function group | Storage account (`AzureWebJobsStorage`) must be zone-redundant too |
| **Functions Premium** | Supported | Minimum instance counts per zone | Same storage caveat |
| **Container Apps** | Zone-redundant environments | Enable at environment creation; set `minReplicas` so every zone has a replica for critical apps | Scale-to-zero apps have no zonal presence while idle — fine for async, not for SLO-bound APIs |
| **AKS** | Zonal node pools; Standard tier SLA is higher with zones | **Topology spread constraints** across zones, PDBs, ≥ 3 replicas for critical services, zone-redundant storage choices | Zonal disks don't follow pods across zones — stateful workloads need zone-aware storage design |

**Multi-region** is mostly *not* a compute decision — the data layer decides whether active-active is possible (Modules 7, 8, 12). Compute's contribution:

- **Deployment stamps** — identical copies of the compute (and often data) per region or per tenant group, behind **Azure Front Door**. Stamps are also the answer to platform **scale ceilings**: if one App Service plan stops at 30 instances or one Container Apps environment hits a quota, add stamps.
- **Stateless compute + global routing** is easy; **failover of state** is the hard part.
- **Rehearse regional failover** — a standby region that has never taken traffic is a hypothesis, not a DR plan (Module 13).

**Composite SLAs still multiply.** A Container Apps app calling a Functions app calling SQL has a composite availability below any of the three (Module 13). Each platform SLA covers *its* infrastructure — not your code, your dependencies, or your configuration.

**The interview-grade sentence:** *"Every platform can be zone-redundant, but each needs something from me: enough App Service instances to span zones, two always-ready Flex instances per group plus a zone-redundant storage account, zone-redundant Container Apps environments with minimum replicas per zone, and topology spread and PDBs on AKS. Multi-region is decided by the data layer; compute contributes stamps behind Front Door — which also solve platform scale ceilings — and rehearsed failover."*

---

## Concept 62 — Isolation and compliance

"We have compliance requirements" often gets translated into "we need AKS" or "we need an ASE." Usually it means something more specific — and several rungs can meet it.

| Requirement | App Service | Functions | Container Apps | AKS |
|---|---|---|---|---|
| **Private inbound only** | Private endpoints; ASE internal LB | Private endpoints (Flex, Premium, Dedicated) | Internal environment | Private cluster + internal ingress |
| **Controlled egress / inspection** | VNet integration + route all + firewall | VNet integration (Flex, Premium) | UDR to firewall (workload-profiles env) | UDR, egress gateway, network policies |
| **Dedicated compute** (no co-tenancy on VMs) | Dedicated tiers (per plan); **ASE v3** single-tenant incl. front ends | Dedicated/ASE plans | **Dedicated workload profiles** (single-tenant guarantee) | Nodes are VMs in your subscription; **dedicated hosts** for physical isolation |
| **Isolation for untrusted code** | Not a fit | Not a fit | **Dynamic sessions** (Hyper-V sandboxes) | Pod sandboxing (Kata-based), confidential containers, separate clusters |
| **Data residency** | Region choice | Region choice | Region choice | Region choice |
| **Audit & policy** | Azure Policy on resources | Same | Same | Azure Policy + in-cluster policy (Gatekeeper), audit logs |

Three distinctions to make explicit:

1. **Network isolation ≠ compute isolation.** Private endpoints and VNet integration control *traffic*; dedicated tiers and profiles control *who shares the hardware*. Most regulatory frameworks care about the former far more than the latter.
2. **Multi-tenant PaaS is not "less secure" by default.** Azure's multi-tenant services are certified under the same compliance programs; the question is which **controls** you must demonstrate (network, identity, encryption, logging), and whether a specific control requires single tenancy.
3. **Isolation costs.** ASE's infrastructure fee, Dedicated profiles' management fee, private endpoints and firewalls are real money (Concept 58) — tie each one to the requirement that justifies it.

**The interview-grade sentence:** *"I translate 'compliance' into specific controls: private inbound, controlled egress, dedicated compute, isolation for untrusted code, residency and audit. Most are met on every rung with private endpoints, VNet integration and policy; dedicated compute means App Service dedicated plans or an ASE, Container Apps Dedicated profiles, or AKS nodes and dedicated hosts; untrusted code means Container Apps dynamic sessions or sandboxed pods — and each isolation feature is tied to the requirement that pays for it."*

---

## Concept 63 — Lifecycle risk: retirements are architecture inputs

Cloud platforms evolve by **adding** a better option, **labelling** the old one legacy, and eventually **retiring** it. Every item below was, at some point, a reasonable default:

| What | Date | Replacement |
|---|---|---|
| Functions **v3 runtime apps on Linux Consumption** stop running | **Sept 30, 2026** | v4 runtime (and Flex) |
| Functions **in-process .NET model** loses support (and .NET 8 & 9 go out of support) | **Nov 10, 2026** | Isolated worker, .NET 10 |
| AKS **managed NGINX (application routing)** security patches end | **Nov 2026** | App routing Gateway API implementation, App Gateway for Containers |
| **Cloud Services (extended support)** retires | **Mar 31, 2027** | Service Fabric managed clusters (or App Service / Container Apps by shape) |
| **Azure Spring Apps** retires | **Mar 31, 2028** | Container Apps, AKS |
| AKS **kubenet** retires | **Mar 31, 2028** | Azure CNI Overlay |
| Functions **Linux Consumption** hosting retires | **Sept 30, 2028** | Flex Consumption |
| Functions **Consumption plan** (Windows) | Labelled **legacy**, still GA | Flex (Linux) — requires Windows → Linux move |
| Container Apps **Consumption-only environment** | Labelled **legacy** | Workload-profiles environment |

The architect's practices:

1. **Prefer the option the provider is investing in.** Flex over Consumption, isolated over in-process, Gateway API over Ingress, workload-profiles environments over Consumption-only, AKS Automatic defaults over bespoke choices. "Legacy" is a leading indicator of "retired."
2. **Track retirements systematically** — Azure Updates, the service retirement workbook in Azure Advisor, the AKS release notes — and review them monthly against your inventory.
3. **Record platform decisions in ADRs with review dates** (Module 31), including the lifecycle position of each choice and the trigger for revisiting it.
4. **Budget migration work** as a recurring line item — on a multi-year horizon, *some* part of any Azure estate is always migrating.
5. **Design for replaceability** (Concept 59) so a forced migration is configuration work, not a rewrite.

**The interview-grade sentence:** *"I treat retirements as architecture inputs: in the next two years alone, Functions in-process support ends in November, AKS managed NGINX loses patches the same month, Cloud Services extended support goes in March 2027, and Spring Apps, kubenet and Linux Consumption in 2028. So I prefer the option the provider is investing in, track retirements monthly against our inventory, record each platform choice in an ADR with a review date, and budget migration as recurring work."*

---

## Concept 64 — Anti-patterns catalogue

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **AKS by default** ("industry standard", CV-driven) | Operational burden without a named capability; small team drowns in upgrades | Container Apps first; AKS on a named trigger (Concept 45) |
| **Functions for everything** | Full APIs forced into triggers; flat traffic on consumption pricing; programming-model lock-in | Functions for glue and orchestration; processes on Container Apps/App Service |
| **One platform per team** | Five delivery, security and on-call stories | Cap at two or three platforms (Concept 60) |
| **Choosing by list price only** | Ignores people, logs, NAT, idle minimums | Total cost including hidden costs and engineers (Concepts 8, 57, 58) |
| **Synchronous long work on HTTP** | 230-second cut-offs, orphaned work | Async request-reply, queues, Durable, jobs |
| **Scale-to-zero on SLO-bound APIs** | Seconds-long cold starts at the tail | `minReplicas ≥ 1`, always-ready, or a warm platform |
| **Autoscaling without downstream caps** | 200 instances × 16 concurrency against a 100-connection database | Maximum instances/replicas sized against dependencies |
| **CPU-only autoscaling for I/O-bound .NET APIs** | Latency explodes at low CPU | Concurrency or backlog signals |
| **No resource requests on AKS** | Invisible to scheduler and autoscalers; noisy neighbors | Requests on every pod; safeguards |
| **Tight CPU limits on latency-sensitive .NET pods** | CFS throttling p99 spikes | Accurate requests; generous or no CPU limits; memory limits always |
| **Per-invocation clients** | SNAT/socket exhaustion | Singletons, `IHttpClientFactory` |
| **ARR affinity on stateless apps** | Uneven load, sticky failures | Turn it off; externalize state |
| **Dependency checks in liveness probes** | A database blip restarts every replica | Instance-local liveness; dependency checks off the probed path |
| **No graceful shutdown** | Dropped requests on every deploy and scale-in | SIGTERM handling, `ShutdownTimeout` < grace, preStop on AKS |
| **Migrations at startup in every replica** | Races, slow starts, failed rollouts | A separate job or pipeline step |
| **`latest` tags and stale base images** | Untraceable deployments, unpatched CVEs | Immutable tags, regular rebuilds |
| **Secrets in images or plain settings** | Leaks, rotation pain | Managed identity, Key Vault references |
| **Staying on legacy options** (in-process, Linux Consumption, ingress-nginx) | Forced migrations under deadline pressure | Move to the invested option early (Concept 63) |
| **Business logic inside function classes** | Untestable, unportable | Thin adapters over an application core |
| **One giant shared App Service plan for everything** | Noisy neighbors, coupled scaling, shared blast radius | Group by criticality and scaling needs (Concept 12) |
| **Building an internal Kubernetes platform that recreates Container Apps** | Cost without differentiation | Use the managed platform unless a requirement forces otherwise |

---

## Concept 65 — The compute decision review

A checklist you can walk through, in order, for any hosting proposal — and narrate in an interview when asked "where would you run this?"

**Workload**
1. What is the **shape** (API, web app, processor, job, worker, workflow, stateful, GPU) and the **traffic pattern** (steady, diurnal, spiky, idle)?
2. What are the **latency SLOs**, including tolerance for cold starts at the p99?
3. What's the longest **execution** and the longest **HTTP request**? Anything near 230 seconds?

**Fit**
4. Which **rung** is the highest whose **constraints** fit (size, scale ceiling, OS, protocol, networking)? Name the constraint that rules out the rung above.
5. Does it need the **Kubernetes API**? If AKS, why not Container Apps — and why not AKS Automatic?
6. What's the **scaling signal** and **time-to-useful**? Are **headroom**, minimum instances and **downstream caps** set?

**Cost and operations**
7. What's the **monthly cost** at peak, average and idle — including **hidden costs** — and what's the **break-even** against the alternative?
8. Who **operates** it, how many hours a month, and is the platform **shared** enough to amortize that?

**.NET readiness**
9. Image: chiseled base, non-root, port 8080, ReadyToRun/AOT where it helps, immutable tags?
10. Runtime: CPU/memory sizing, GC mode, throttling risk; startup without network I/O?
11. Probes and **graceful shutdown** mapped to the platform?
12. Identity: managed/workload identity, Key Vault, no secrets in images?

**Resilience and lifecycle**
13. **Zones**, minimum instances per zone, multi-region stamps if required?
14. **Lock-in** layers identified; handlers thin; migration path to the next rung understood?
15. **Lifecycle** position: legacy or invested option? Retirement dates? ADR with a review date?

You won't cover fifteen items in an interview. Lead with **shape and traffic**, the **decisive constraint**, and **cost including people** — and say which question would change your answer.

---

## Concept 66 — When not to

**Don't pick AKS to look senior.** Choosing Kubernetes for a handful of APIs is the most common over-engineering in cloud architecture interviews; the senior signal is the justification, not the logo.

**Don't pick Functions to avoid thinking about hosting.** A full API or a steady high-throughput service on consumption pricing costs more and constrains more than an ordinary ASP.NET Core process.

**Don't add a platform for one workload** unless its constraint is real and the operating cost is accepted. One GPU inference model can run on Container Apps serverless GPUs; it doesn't need a cluster.

**Don't chase scale-to-zero for services that are never idle,** or for SLO-bound APIs where the cold start matters more than the savings.

**Don't pre-optimize portability.** Containers and thin adapters give you most of it for free; abstracting away every Azure service "in case we switch clouds" costs more than it will ever save for most organizations.

**Don't migrate platforms without a trigger.** "Container Apps is newer than App Service" isn't a reason to move a healthy App Service estate; a named constraint, a cost break-even, or a retirement date is.

**Don't let the platform decision precede the design.** Shape, data, consistency and failure modes (Modules 6–13) come first; compute is where the design runs, not what it is.

**The meta-rule, echoing Module 13:** *every layer of infrastructure you add is also a new failure mode and a new recurring cost.* The best compute choice is the highest rung that meets the workload's constraints for its lifetime — with every step down justified by a named requirement, and every decision written down with a date to revisit it.

**The interview-grade sentence:** *"I don't pick AKS to look senior, Functions to avoid hosting decisions, a new platform for one workload, scale-to-zero for SLO-bound APIs, or cloud-agnostic abstractions nobody will use — and I don't migrate a healthy estate without a named trigger. The best choice is the highest rung that meets the workload's constraints for its lifetime, with each step down justified and each decision recorded with a review date."*

---

# Putting it together

## Worked example 1 — "Choose compute for a mid-size e-commerce platform"

*Twelve engineers in three product teams, no platform team. Components: a server-rendered storefront (Razor Pages/Blazor), a BFF API for mobile and web, catalog/orders/payments services, order-event processing, image processing on product upload, a nightly reconciliation job with a payment provider, a partner webhook receiver, a recommendations model (GPU inference), and an internal admin portal. Traffic is diurnal with sale-day spikes of 8×.*

Narrate it in this order:

1. **Classify each workload by shape and traffic** (Concept 4) before naming anything:

| Component | Shape | Traffic | Latency sensitivity |
|---|---|---|---|
| Storefront | Server-rendered web app | Diurnal, 8× spikes | High |
| BFF API | Request/response | Diurnal, 8× spikes | High |
| Catalog / Orders / Payments | Request/response services | Diurnal | High (internal) |
| Order-event processing | Event processor | Bursty | Low |
| Image processing | Event-driven, CPU-heavy per item | Bursty around catalog imports | Low |
| Reconciliation | Scheduled batch | Nightly | None |
| Partner webhook receiver | Tiny HTTP endpoint | Sporadic | Partner timeout: 5 s |
| Recommendations | GPU inference | Diurnal | Medium |
| Admin portal | Internal web app | Business hours, low | Low |

2. **Pick the platforms — and cap them at two plus one** (Concept 60):
   - **Container Apps** as the main platform: BFF, catalog, orders, payments (HTTP scaling, `minReplicas: 2` across zones for SLO-bound services), order-event processing (Service Bus KEDA rule, scale to zero), image processing (event-driven **jobs**, one execution per import batch), reconciliation (**scheduled job**), recommendations (**serverless GPU** replicas, `minReplicas: 1` during business hours via a cron scale rule). Everything in **one workload-profiles environment**, calling each other by name.
   - **App Service Premium v4** for the storefront and the admin portal: server-rendered apps with slots for warm swaps, Easy Auth for the admin portal (Entra ID), ARR affinity **off** with Data Protection keys and session state in Redis. Two apps, two plans (the storefront must not share a blast radius with the admin portal — Concept 12).
   - **Functions Flex** for the partner webhook — with **1 always-ready instance** for the HTTP group because the partner's 5-second timeout can't absorb a cold start — plus any small event glue (Event Grid → cache invalidation).
   - **No AKS**: no requirement needs the Kubernetes API, there's no platform team, and the ops cost would fall on product engineers (Concept 45). Write down the triggers that would change that.

3. **Scaling and headroom** (Concepts 5, 31): sale days are scheduled — raise `minReplicas` and App Service always-ready instances the morning before rather than trusting reactive scaling for an 8× ramp. Cap `maxReplicas` on orders and payments against the database's connection budget (Concept 26). Tune `concurrentRequests` from measured latency (Little's Law), not the default 10.

4. **.NET readiness** (Part G): chiseled .NET 10 images on 8080 as non-root; ReadyToRun; migrations as a manual Container Apps job in the pipeline; health endpoints split live/ready/startup; `ShutdownTimeout` below the termination grace period; user-assigned managed identities with Key Vault references; OpenTelemetry via Aspire service defaults.

5. **Deployment**: Aspire AppHost with a Container Apps environment and an App Service environment (Concept 53); multiple-revision mode with labels for the BFF and payments canaries; slots with swap-with-preview for the storefront.

6. **Cost sketch** (Concept 57): Container Apps Consumption for bursty processors and jobs (scale to zero), and consider moving the always-busy BFF and catalog to a small **Dedicated D-profile** once utilization data shows it's cheaper; App Service Pv4 with a savings plan for the storefront. Budget Log Analytics explicitly (Concept 58).

7. **Reliability** (Concept 61): zone-redundant Container Apps environment and App Service plans; Front Door in front; one region now, with the stamp pattern documented for a second region when the data tier supports it.

8. **Close with what you're not doing and why**: no AKS (no named trigger), no Dapr (all-.NET, native SDKs suffice), no Durable Functions yet (the order flow is a short saga already modeled with the outbox — Module 11), no multi-region active-active (the database is single-region primary).

---

## Worked example 2 — "We run AKS with four engineers and eight services. Upgrades are eating us alive."

*The cluster is AKS Standard on Kubernetes 1.31 (deprecated), using the community ingress-nginx chart, cert-manager, a self-installed KEDA, Prometheus and Grafana. Eight .NET services, two workers, one CronJob. The last minor upgrade slipped four months.*

1. **Name the actual problem.** The team is paying the day-2 tax (Concept 42) without the economies of scale that justify it (Concept 8): four engineers, eight services, a deprecated Kubernetes version and a **retired ingress controller** (upstream maintenance ended March 2026 — Concept 40). This is an operational-risk issue, not only a productivity one.

2. **Inventory the manifests** against the step-up trigger (Concept 56): does anything use CRDs beyond cert-manager and KEDA, operators, DaemonSets, privileged pods, custom networking or node affinity? Typically: Deployments, Services, Ingresses, ConfigMaps, Secrets, a CronJob and KEDA ScaledObjects — **all of which map directly onto Container Apps** (apps, ingress, secrets/Key Vault refs, scheduled jobs, KEDA rules).

3. **Offer the options honestly:**

| Option | Effort | Ongoing burden | Risk |
|---|---|---|---|
| A. Stay on AKS Standard, catch up upgrades, migrate ingress to Gateway API | Medium | Unchanged — the problem recurs | Low technical, high organizational |
| B. Rebuild on **AKS Automatic** (managed system pools, NAP, Gateway API, auto-upgrades, safeguards) | Medium | Much lower, not zero | Some manifests need safeguard fixes (requests, non-root) |
| C. Move to **Container Apps** | Medium | Lowest | Loses nothing they use today; KEDA rules and probes port directly |

4. **Recommend C** (with B as the fallback if the inventory finds a blocker), and plan it as a **service-by-service strangler** (Module 32): stand up a Container Apps environment in the same VNet, move stateless services one at a time behind the existing front door, switch DNS per service, keep the cluster until the last workload leaves. Fix the .NET basics on the way (Concepts 47–51) — they apply on both platforms.

5. **Quantify the win** in the language executives respond to (Module 33): engineer-hours per month on platform work before and after, plus the risk removed (unsupported Kubernetes version, retired ingress). Compute cost may be similar or slightly higher; people cost dominates.

6. **Record it in an ADR** (Module 31) with the triggers that would send them back to AKS — e.g., adopting a database operator or needing Windows containers.

---

## Worked example 3 — "Our Functions app's p99 is 9 seconds at night, and the bill tripled."

*An in-process .NET 8 function app on the (legacy) Linux Consumption plan: an HTTP API for a mobile app, an Event Hubs telemetry processor, and a timer that sends reminder emails. Logs go to Application Insights without sampling.*

Diagnose in layers:

1. **The 9-second p99 at night** is cold start (Concept 22): traffic drops to near zero, instances are reclaimed, the first requests pay placement, package mount, host and worker startup, and — in this code — a synchronous Key Vault call and a new `HttpClient` per request. The average looks fine; the p99 doesn't.
2. **The bill tripled** — check the meters: likely **Application Insights ingestion** (per-event telemetry from the Event Hubs processor, no sampling — Concept 58) and **execution GB-s** inflated by per-event processing with no batching.
3. **Hidden correctness issue**: the Event Hubs function throws on malformed events; the host **checkpoints anyway**, silently skipping them (Concept 25).
4. **Hidden deadline**: in-process support ends **November 10, 2026**, and Linux Consumption gets no .NET versions beyond 9 and retires in 2028 (Concepts 19, 21).

The plan:

1. **Migrate to the isolated worker on .NET 10 and Flex Consumption** in one change (a new app, deployed side by side — no in-place plan migration).
2. **Per-function scaling does the separation for you**: the HTTP API gets **always-ready instances** (1–2, or 2 with zone redundancy); the Event Hubs processor scales on its own instances from zero; the timer runs in its own group.
3. **Fix startup and clients**: Key Vault references in app settings instead of startup calls; `IHttpClientFactory`; singleton SDK clients.
4. **Process Event Hubs in batches** (`string[]` / `EventData[]` triggers), with per-event `try/catch` and a parking destination for bad events; set per-instance concurrency and maximum instances against downstream limits.
5. **Telemetry**: sampling, log levels in `host.json`, no per-event Information logs; move high-volume tables to cheaper log tiers.
6. **Verify with numbers**: first-request p99 per instance, fraction of cold requests, GB-s per million events, ingestion GB per day — before and after.

---

## Worked example 4 — "Lift a .NET Framework 4.8 monolith with a COM dependency"

*A WebForms + WCF application on IIS, using a COM component for PDF generation, a registry key for licensing, a mapped network drive for document storage, and Windows authentication for internal users.*

1. **Name the blockers** for classic App Service: COM registration, registry writes, mapped drives. Historically those forced **VMs** or **Windows containers on AKS**.
2. **App Service Managed Instance** (Concept 10) targets exactly this: PowerShell configuration scripts to install and register the COM component, Key Vault-backed registry values, Azure Files/UNC storage mounts, plan-level VNet integration to reach on-premises AD and file shares — on Pv4/Pmv4, Windows only, in supported regions. Prove it with a spike on the COM component first.
3. **Alternatives**, in order: Windows containers on **AKS** (heavier: Windows node pools, image sizes, the whole AKS day-2 tax — Concept 42), or **VMs/VMSS** (full control, full patching burden). Choose by what the spike shows.
4. **The real plan is modernization in steps** (Module 32): host the monolith as-is on Managed Instance; put **Front Door** or a reverse proxy in front; carve out new capabilities as .NET 10 services on Container Apps (strangler fig); replace the COM PDF component with a library or a small containerized service; move document storage to Blob Storage. The hosting choice buys time and removes the VM burden; the architecture work removes the constraints that forced it.

---

## Common interview questions, with model answers

**"Why not just put everything on AKS?"** Because AKS is the lowest managed rung: it gives me the Kubernetes API at the cost of a continuous operational burden — minor upgrades several times a year, networking choices like the post-ingress-nginx move to Gateway API, security baselines, observability assembly — that only pays off with a platform team and many workloads, or with a capability only Kubernetes has, like operators or DaemonSets. For most services, Container Apps gives Kubernetes semantics without Kubernetes operations, and the same images move to AKS later if a real trigger appears. If AKS is justified, I'd start with AKS Automatic (Concepts 8, 38, 45, 56).

**"App Service or Container Apps for a single ASP.NET Core API?"** Both are good. App Service if it's one or two steady web apps and the team wants slots with warm swaps, Easy Auth, health-check replacement and minimal thinking. Container Apps if it's the first of several services, if it needs scale-to-zero, backlog-based scaling, jobs or revisions-based canaries, or if we want the container as the portable artifact. The deciding axes are how many services there will be and the traffic shape (Concepts 17, 36, 56).

**"When would you choose Functions over a Container Apps worker with a KEDA rule?"** Functions when the programming model pays for itself — many small handlers, triggers and bindings replacing plumbing, Durable orchestrations, spiky or idle traffic that benefits from per-function scaling and execution billing. A Container Apps worker when I want a normal .NET process — MassTransit or Wolverine consumers, full control of batching and concurrency, portability — with KEDA giving similar scale-to-zero behavior. With Durable Task SDKs now running on Container Apps, even orchestration no longer forces the choice (Concepts 24, 27, 36).

**"A Container Apps API scales to zero at night and the first requests take several seconds. Why, and what do you do?"** Scaling from zero means placement, image pull, container and .NET startup, and first-request connection and token setup — a cold start. Options: `minReplicas: 1` (billed at the reduced idle rate when truly idle), smaller chiseled images with ReadyToRun, removing startup I/O, and a readiness probe that waits for warm-up. And I'd check whether the cool-down and polling behavior mean it's even reaching zero (Concepts 6, 31, 35, 49).

**"Walk me through an App Service slot swap."** Production's sticky settings are applied to the staging instances, which restart; the platform warms them via the site root or a configured warm-up path and aborts if they fail; only then are routing rules swapped, so production traffic lands on warm instances, and the old version sits in staging for an instant swap back. Watch sticky-setting mistakes, slots sharing the plan's capacity, and schema changes, which don't swap back (Concept 13).

**"We still have in-process .NET Functions. What's your plan?"** Support ends November 10, 2026 — the same day .NET 8 does — after which there are no security fixes. I'd inventory apps, prioritize internet-facing and critical ones, and migrate to the isolated worker on .NET 10, using the change to move to Flex Consumption where it fits (in-process can't run there) and to consider Durable Task Scheduler for Durable apps. Deploy side by side, compare, switch (Concepts 19, 21).

**"How do you size .NET containers?"** From measurements, with the runtime's container behavior in mind: ProcessorCount follows the CPU limit, the GC heap hard limit is 75% of the memory limit, DATAS adapts Server GC to small containers, and CFS throttling punishes bursty processes under tight CPU limits. On AKS I set accurate requests and memory limits and usually skip CPU limits for latency-sensitive services; on the managed platforms I pick replica sizes with burst headroom and prefer fewer, larger replicas for latency-sensitive APIs (Concept 48).

**"Why does AKS need spare capacity if it autoscales?"** Because it scales in two layers: pods in seconds, nodes in minutes. While a node provisions, extra pods sit Pending, so existing nodes must absorb demand growth for that whole window: headroom ≈ ramp rate × node provisioning time. I provide it with placeholder pods, scheduled pre-scaling, or faster nodes via smaller images and artifact streaming (Concept 39).

**"What would you use for ingress on a new AKS cluster in 2026?"** Gateway API. Ingress-nginx is retired upstream and Microsoft's managed NGINX add-on only gets security patches through November 2026. On AKS Automatic 1.36+, the app-routing Gateway API implementation is the default; Application Gateway for Containers is the managed alternative with an out-of-cluster data plane. For existing NGINX clusters I'd run both data planes in parallel and move DNS (Concept 40).

**"Estimate the monthly compute cost of this API."** I'd start from the load shape — peak, average, idle hours, requests — then pick a unit, compute consumption as active plus idle plus requests minus free grants, and provisioned as peak plus zone headroom, then state the break-even. E.g., peak 6 replicas for business hours and 2 otherwise on Container Apps Consumption is about $211 a month versus $473 always-on; provisioned wins only below about $30 per vCPU/2 GiB-month. Then I add hidden costs and people (Concepts 7, 57, 58).

**"A report endpoint takes six minutes. What breaks, and how do you fix it?"** On App Service and HTTP-triggered Functions, the front-end load balancer cuts responses after 230 seconds regardless of my timeouts, and the work may keep running orphaned. I'd make it async: accept, return 202 with a status URL, do the work in a queue-driven worker, a Container Apps job or a Durable orchestration, and let the client poll or get a callback (Concepts 16, 23).

**"How do you get zero-downtime deployments on each platform?"** App Service: slots with warm-up and swap. Container Apps: single-revision mode with meaningful readiness probes, or multiple-revision mode with labels and weighted traffic. AKS: rolling updates gated by readiness, PDBs, preStop sleeps for the endpoint race, and Argo Rollouts or Flagger for automated canaries. Functions Flex: the rolling-update site strategy (in preview), since slots aren't supported. Everywhere: graceful shutdown and backward-compatible schema changes (Concepts 13, 30, 43, 51).

**"How does Durable Functions work, and what bites people?"** It's event-sourced control flow: the orchestrator records each step in a history, unloads, and is replayed with recorded results. So orchestrator code must be deterministic — context clock, GUIDs and timers, no I/O — and activities must be idempotent because they're at-least-once. The hard parts are versioning in-flight orchestrations and unbounded history (use ContinueAsNew). Durable Task Scheduler is now the recommended backend, and the Durable Task SDKs run the same model outside Functions (Concept 24).

**"How do you avoid lock-in on Azure compute?"** I separate the layers: containers and twelve-factor services make compute the least locked-in part; I keep Functions handlers thin adapters over an application core; I prefer portable mechanisms like KEDA rules and standard probes. The real lock-in is the managed data and messaging services, which is usually a good trade — so I make it deliberately rather than abstracting everything (Concept 59).

**"Compliance says 'isolated'. Do we need an App Service Environment?"** First translate "isolated" into controls: private inbound, controlled egress, dedicated compute, audit. Private endpoints, VNet integration and policy meet most of them on any rung. Only a genuine single-tenant or network-isolated compute requirement justifies an ASE v3 (Isolated v2/v4), Container Apps Dedicated profiles, or AKS with dedicated hosts — and each has a cost I'd tie to the requirement (Concept 62).

**"How would you move services from AKS to Container Apps?"** Inventory manifests: anything using only Deployments, Services, Ingress/HTTPRoute, ConfigMaps/Secrets, CronJobs and KEDA maps directly to apps, secrets, scheduled jobs and scale rules. Anything with CRDs, operators, DaemonSets or custom networking stays. Then strangle service by service in the same VNet behind the existing entry point, switching DNS per service, and keep the cluster until the last workload leaves (Concept 59, worked example 2).

**"Where does Aspire fit into this?"** Aspire separates the application model from the hosting target: the AppHost declares services and dependencies, and compute environments — Container Apps, App Service, AKS via Helm, Compose — are attached at publish time. `aspire publish` generates artifacts and `aspire deploy` provisions and deploys with managed-identity registry pulls. It makes the platform a late, swappable decision and standardizes telemetry and resilience, but it doesn't make the decision or replace enterprise infrastructure governance (Concept 53).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Names a service before describing the workload | Classifies shape, traffic, latency tolerance and duration first |
| "We always use Kubernetes" | Names the capability that requires the Kubernetes API — or picks a higher rung |
| Treats "managed" as binary | Lists which of the seven decisions each platform makes |
| Compares list prices only | Models load shape, break-even utilization, hidden costs and engineer-hours |
| Ignores cold start or treats it as a defect | Treats it as a budget; measures first-request p99; buys it down deliberately |
| Autoscaling on CPU for I/O-bound APIs | Scales on concurrency or backlog; knows time-to-useful and sizes headroom |
| Forgets the node layer on AKS | Explains Pending pods, node provisioning time and headroom math |
| Leaves autoscaling uncapped | Caps instances against database and dependency limits |
| Unaware of the Functions deadlines | Knows in-process ends Nov 10, 2026, Consumption is legacy, Flex is the default |
| Thinks Flex is "Consumption but better" with no constraints | Names one app per plan, no slots, 30 s init, Linux code-only, regional core quota |
| Designs long synchronous HTTP work | Knows the 230-second limit and designs async request-reply |
| Recommends ingress-nginx on AKS | Knows it's retired; uses Gateway API (app routing or App Gateway for Containers) |
| Uses default `concurrentRequests` and replica sizes | Derives targets from Little's Law and measured latency |
| Tight CPU limits on .NET pods "for safety" | Explains CFS throttling; sets requests, memory limits, and CPU limits deliberately |
| Doesn't mention graceful shutdown | Describes SIGTERM handling, ShutdownTimeout vs grace period, and the preStop race |
| Puts dependency checks in liveness probes | Keeps liveness instance-local, readiness meaningful, dependencies on dashboards |
| Secrets in app settings or images | Managed/workload identity, Key Vault references, explicit production credentials |
| "Functions scale infinitely" | Knows partition limits, max instance counts and regional quotas |
| Treats portability as all-or-nothing | Separates packaging, config, programming model and data-service lock-in |
| One platform per team | Decides per workload, caps the estate at two or three, owns each with a paved road |
| Proposes an ASE for any compliance requirement | Translates compliance into controls and ties each isolation cost to one |
| Ignores retirements | Tracks lifecycle dates, prefers invested options, writes ADRs with review dates |
| Migrates platforms because something is newer | Migrates on a named trigger: constraint, break-even or retirement |

---

## Practice exercises

**Exercise 1 — Classify and place (45 min).** Take a system you know (or eShop). List every deployable component; classify shape, traffic pattern, latency tolerance and longest execution (Concept 4). Place each on the ladder (Concept 56) and write the **one constraint** that rules out the rung above it. Cap the result at three platforms and justify the cap.

**Exercise 2 — Build the cost model (1–2 hours).** In a spreadsheet, model one service across Container Apps Consumption, Container Apps Dedicated, App Service Pv4 and AKS using your region's current prices from the pricing pages. Inputs: replica size, peak/off-peak hours and replica counts, active fraction, requests, free grants, zone headroom, commitments. Output: monthly cost per option and the **break-even unit price** (Concept 57). Then add log ingestion and NAT (Concept 58) and see whether the ranking changes.

**Exercise 3 — Cold-start lab (half a day).** Deploy a .NET 10 minimal API to Container Apps with `minReplicas: 0`. Measure first-request latency after scale-to-zero ten times. Then, one change at a time: chiseled image, ReadyToRun, remove a synchronous Key Vault call at startup, add a warm-up readiness check. Record each delta. Which link in Concept 6's chain dominated?

**Exercise 4 — .NET inside a container (2 hours).** Run an API locally with `docker run --cpus=1 --memory=512m`. Log `Environment.ProcessorCount` and `GC.GetGCMemoryInfo().TotalAvailableMemoryBytes`. Load-test it (e.g., with k6 or bombardier), compare Server GC (DATAS) vs Workstation GC for p99 and memory, then try `--cpus=0.5` and watch the p99 under bursty load (Concept 48).

**Exercise 5 — Graceful shutdown under load (2 hours).** On a local Kubernetes cluster (kind or k3d) or AKS, run a rolling update of a .NET deployment while generating steady traffic. Count failed requests with no `preStop`, then with a 10-second `preStop` sleep and `ShutdownTimeout` set below `terminationGracePeriodSeconds` (Concept 51). Explain the difference.

**Exercise 6 — Functions migration (half a day).** Port an in-process function app (or a sample) to the isolated worker on .NET 10 and Flex Consumption. Split an HTTP function and a queue function into separate scale groups, set always-ready for HTTP only, tune `maxConcurrentCalls`, and compare instance counts and GB-s at concurrency 1 vs 16 under the same load (Concepts 20, 21, 57).

**Exercise 7 — The AKS-to-Container Apps audit (1 hour).** Take a Helm chart or manifest set (your own, or an open-source .NET sample's). Classify every Kubernetes object by portability to Container Apps (Concept 59). Write the list of blockers and the step-up/step-down recommendation.

**Exercise 8 — Write the ADR (1 hour).** Write an ADR (Module 31) for the compute choice of one system: context, decision, alternatives with the decisive axis for each, cost estimate, lifecycle position of the chosen options with dates, and the **triggers and review date** that would reopen the decision (Concepts 63, 65).

**Exercise 9 — Mock "where would you run this?" (20 min, twice).** Have someone give you a system description; answer in 10 minutes using Concept 65's order — shape and traffic, decisive constraint, cost including people, then .NET readiness and lifecycle. Score yourself against the mistakes-vs-signals table.

---

## Free resources and learning material

### Foundational papers and essays

| Resource | What it covers |
|---|---|
| [Large-scale cluster management at Google with Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/) (EuroSys 2015) | The system Kubernetes descends from: declarative jobs, bin packing, the economics of shared clusters |
| [Borg, Omega, and Kubernetes](https://queue.acm.org/detail.cfm?id=2898444) (ACM Queue, 2016) | Lessons from three generations of container management — why Kubernetes is an API plus control loops (Concept 37) |
| [Autopilot: workload autoscaling at Google](https://research.google/pubs/autopilot-workload-autoscaling-at-google-scale/) (EuroSys 2020) | Vertical and horizontal autoscaling at scale; why requests are the unit of cost (Concept 44) |
| [Serverless in the Wild](https://arxiv.org/abs/2003.03423) (USENIX ATC 2020) | Microsoft's characterization of Azure Functions production traffic and keep-alive policies — the data behind cold starts (Concept 6) |
| [Serverless in the Wild — a readable summary](https://mikhail.io/2020/05/serverless-in-the-wild-azure-functions-usage-stats/) | Mikhail Shilkov's walkthrough of the paper's numbers |
| [Cloud Programming Simplified: A Berkeley View on Serverless Computing](https://arxiv.org/abs/1902.03383) (2019) | Serverless's promise and its limitations, from first principles |
| [Serverless Computing: One Step Forward, Two Steps Back](https://arxiv.org/abs/1812.03651) (CIDR 2019) | The sharpest critique of FaaS: data shipping, limited execution, no specialized hardware |
| [Peeking Behind the Curtains of Serverless Platforms](https://www.usenix.org/conference/atc18/presentation/wang-liang) (USENIX ATC 2018) | Measured cold starts, placement and isolation across providers |
| [Firecracker: Lightweight Virtualization for Serverless Applications](https://www.usenix.org/conference/nsdi20/presentation/agache) (NSDI 2020) | How serverless isolation is built — the placement and startup links of the cold-start chain |
| [The Twelve-Factor App](https://12factor.net/) | The portability contract that makes compute choices reversible (Concept 59) |

### Azure decision guides and architecture

| Resource | What it covers |
|---|---|
| [Choose an Azure compute service](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/compute-decision-tree) | Microsoft's decision tree — compare it with Concept 56 |
| [Choose an Azure container service](https://learn.microsoft.com/en-us/azure/architecture/guide/choose-azure-container-service) · [general considerations](https://learn.microsoft.com/en-us/azure/architecture/guide/container-service-general-considerations) | Container Apps vs AKS vs App Service vs others, by operational model |
| [Compare Container Apps with other Azure container options](https://learn.microsoft.com/en-us/azure/container-apps/compare-options) | The Container Apps team's own positioning |
| [Choose a compute option for microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/compute-options) | Microservices-specific trade-offs |
| [Well-Architected service guides](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/) · [AKS service guide](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-kubernetes-service) | Reliability, security, cost and operations checklists per service |
| [Deployment Stamps pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) | Scaling past platform ceilings and going multi-region (Concept 61) |
| [Mission-critical workloads](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/) | Zone and region design for the highest availability tiers |
| [Container Apps landing zone accelerator](https://github.com/Azure/aca-landing-zone-accelerator) · [App Service landing zone accelerator](https://github.com/Azure/appservice-landing-zone-accelerator) · [AKS landing zone accelerator](https://github.com/Azure/AKS-Landing-Zone-Accelerator) | Enterprise-ready reference implementations in Bicep/Terraform |

### Azure App Service

| Resource | What it covers |
|---|---|
| [App Service plans](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans) | Plans, tiers, density guidance, Managed Instance (Concepts 9–12) |
| [Automatic scaling](https://learn.microsoft.com/en-us/azure/app-service/manage-automatic-scaling) | Maximum burst, always-ready and prewarmed instances (Concept 11) |
| [Deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots) | Swap mechanics, warm-up, sticky settings, swap with preview (Concept 13) |
| [Health check](https://learn.microsoft.com/en-us/azure/app-service/monitor-instances-health-check) | Instance removal and replacement behavior (Concept 15) |
| [Networking features](https://learn.microsoft.com/en-us/azure/app-service/networking-features) · [Intermittent outbound connection errors (SNAT)](https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-intermittent-outbound-connection-errors) | Inbound vs outbound, VNet integration, NAT, SNAT exhaustion (Concept 14) |
| [Availability and performance FAQ](https://learn.microsoft.com/en-us/azure/app-service/faq-availability-performance-application-issues) | Including why requests time out after 230 seconds (Concept 16) |
| [Managed Instance on App Service](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-instance) | OS customization for legacy Windows apps (Concept 10, worked example 4) |
| [Premium v4 GA announcement](https://techcommunity.microsoft.com/blog/appsonazureblog/announcing-general-availability-of-premium-v4-for-azure-app-service/4446204) · [App Service at Build 2026](https://azure.github.io/AppService/2026/06/08/AppService-Build2026.html) · [Continued investment in App Service](https://azure.github.io/AppService/2026/03/31/continued-investment.html) | Current platform direction: Pv4, Isolated v4, Managed Instance, Aspire on App Service |

### Azure Functions and Durable

| Resource | What it covers |
|---|---|
| [Functions hosting options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale) | The plan comparison, timeouts, limits and the 230-second note (Concepts 19, 23) |
| [Flex Consumption plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan) | Instance sizes, per-function scaling, always-ready, billing, regional quota, considerations |
| [Event-driven scaling](https://learn.microsoft.com/en-us/azure/azure-functions/event-driven-scaling) · [Target-based scaling](https://learn.microsoft.com/en-us/azure/azure-functions/functions-target-based-scaling) · [Concurrency](https://learn.microsoft.com/en-us/azure/azure-functions/functions-concurrency) | How the scale controller decides (Concept 20) |
| [Isolated worker guide](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide) · [In-process vs isolated differences](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-in-process-differences) · [Migrate to isolated](https://learn.microsoft.com/en-us/azure/azure-functions/migrate-dotnet-to-isolated-model) | The November 10, 2026 migration (Concept 21) |
| [Migrate Consumption apps to Flex](https://learn.microsoft.com/en-us/azure/azure-functions/migration/migrate-plan-consumption-to-flex) | The Linux Consumption retirement path |
| [Functions best practices](https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices) · [Reliable event processing](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reliable-event-processing) · [Error handling and retries](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-error-pages) | Delivery semantics, checkpoints, retries (Concepts 25–26) |
| [What is Durable Task?](https://learn.microsoft.com/en-us/azure/durable-task/common/what-is-durable-task) · [Durable Functions code constraints](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints) | Replay, determinism rules (Concept 24) |
| [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler) · [Consumption SKU GA](https://techcommunity.microsoft.com/blog/appsonazureblog/the-durable-task-scheduler-consumption-sku-is-now-generally-available/4506682) · [Durable Task SDKs overview](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-overview) | The managed backend and running orchestrations outside Functions |
| [Azure Functions cold starts (older, explanatory)](https://mikhail.io/serverless/coldstarts/azure/) | Measurements and explanation of the cold-start chain — dated numbers, durable ideas |

### Azure Container Apps

| Resource | What it covers |
|---|---|
| [Environments](https://learn.microsoft.com/en-us/azure/container-apps/environment) · [Plans](https://learn.microsoft.com/en-us/azure/container-apps/plans) · [Workload profiles](https://learn.microsoft.com/en-us/azure/container-apps/workload-profiles-overview) | Environment types, Consumption vs Dedicated (Concept 29) |
| [Billing](https://learn.microsoft.com/en-us/azure/container-apps/billing) · [Understanding idle usage](https://techcommunity.microsoft.com/blog/appsonazureblog/understanding-idle-usage-in-azure-container-apps/4419197) | Active vs idle rates, free grants, management fees (Concepts 35, 57–58) |
| [Scaling](https://learn.microsoft.com/en-us/azure/container-apps/scale-app) | Defaults, KEDA rules, scale behavior table (Concept 31) |
| [Revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions) · [Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs) | Rollouts, traffic splitting, batch (Concepts 30, 32) |
| [Networking](https://learn.microsoft.com/en-us/azure/container-apps/networking) · [Health probes](https://learn.microsoft.com/en-us/azure/container-apps/health-probes) · [Containers](https://learn.microsoft.com/en-us/azure/container-apps/containers) | Ingress, VNets, probes, sizes, limits (Concepts 33, 35, 50) |
| [Dapr in Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview) · [Dynamic sessions](https://learn.microsoft.com/en-us/azure/container-apps/sessions) · [Serverless GPUs](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview) · [Functions on Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/functions-overview) | The platform extras (Concept 34) |
| [Deploying and scaling ASP.NET Core on Container Apps](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/scaling-aspnet-apps/scaling-aspnet-apps) | Data Protection keys and other multi-replica .NET requirements (Concept 33) |

### Azure Kubernetes Service

| Resource | What it covers |
|---|---|
| [AKS core concepts](https://learn.microsoft.com/en-us/azure/aks/core-aks-concepts) · [AKS Automatic](https://learn.microsoft.com/en-us/azure/aks/intro-aks-automatic) | Control plane, node pools, Automatic vs Standard (Concept 38) |
| [Managed system node pools and the pod readiness SLA](https://blog.aks.azure.com/2025/11/26/aks-automatic-managed-system-node-pools) | Why Automatic changes the operational equation |
| [Node Auto Provisioning](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning) · [Karpenter provider for Azure](https://github.com/Azure/karpenter-provider-azure) | Node-layer scaling (Concept 39) |
| [Supported Kubernetes versions](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions) · [Upgrade options](https://learn.microsoft.com/en-us/azure/aks/upgrade-options) · [AKS release notes](https://github.com/Azure/AKS/releases) | The day-2 calendar (Concept 42) |
| [Ingress concepts and the ingress-nginx retirement](https://learn.microsoft.com/en-us/azure/aks/concepts-network-ingress) · [App routing with Gateway API](https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api) · [Migrating from NGINX to Gateway API](https://learn.microsoft.com/en-us/azure/aks/app-routing-nginx-to-gateway-api-migration) | The 2026 ingress story (Concept 40) |
| [Workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) · [AKS best practices](https://learn.microsoft.com/en-us/azure/aks/best-practices) | Identity and baseline practices (Concept 41) |
| [AKS baseline architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks) | The reference production cluster, with a GitHub implementation |
| [AKS cost analysis](https://learn.microsoft.com/en-us/azure/aks/cost-analysis) | Namespace and idle cost visibility (Concept 44) |
| [AKS engineering blog](https://blog.aks.azure.com/) | Where AKS changes are explained first |

### Kubernetes and cloud-native fundamentals

| Resource | What it covers |
|---|---|
| [Kubernetes overview](https://kubernetes.io/docs/concepts/overview/) · [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) | What the API and components are; building a cluster by hand to understand it |
| [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) · [KEDA concepts](https://keda.sh/docs/latest/concepts/) · [Karpenter](https://karpenter.sh/docs/) | Pod- and node-layer scaling |
| [Resource management for pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Setting the right requests and limits](https://learnkube.com/setting-cpu-memory-limits-requests) | Requests, limits, QoS, throttling (Concepts 44, 48) |
| [Liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) · [Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) · [Graceful shutdown in Kubernetes](https://learnkube.com/graceful-shutdown) | Probes and the shutdown race (Concepts 50–51) |
| [Gateway API](https://gateway-api.sigs.k8s.io/) · [Ingress NGINX retirement: what you need to know](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/) | The ingress transition (Concept 40) |
| [Dapr documentation](https://docs.dapr.io/) | Building blocks and when they help (Concept 34) |
| [Kubernetes failure stories](https://k8s.af/) | Real incident write-ups — the day-2 tax in practice |
| [CNCF Platforms white paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/) · [Team Topologies key concepts](https://teamtopologies.com/key-concepts) | Platform-as-a-product and the thinnest viable platform (Concept 60) |

### .NET in containers and on these platforms

| Resource | What it covers |
|---|---|
| [.NET containers overview](https://learn.microsoft.com/en-us/dotnet/core/containers/overview) · [Containerize with `dotnet publish`](https://learn.microsoft.com/en-us/dotnet/core/containers/sdk-publish) | SDK container builds (Concept 47) |
| [dotnet-docker repository](https://github.com/dotnet/dotnet-docker) · [Ubuntu Chiseled images](https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md) · [.NET 10 container images](https://github.com/dotnet/dotnet-docker/discussions/6801) · [Default images now use Ubuntu](https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/10.0/default-images-use-ubuntu) | Image families and .NET 10 changes |
| [GC configuration settings](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/garbage-collector) · [DATAS](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/datas) | Heap hard limits, Server vs Workstation, DATAS (Concept 48) |
| [ReadyToRun](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run) · [Native AOT](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) · [ASP.NET Core and Native AOT](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot) | Startup engineering (Concept 49) |
| [Health checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) · [.NET Generic Host (shutdown)](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host) | Probes and graceful shutdown (Concepts 50–51) |
| [Credential chains in the Azure Identity library](https://learn.microsoft.com/en-us/dotnet/azure/sdk/authentication/credential-chains) | DefaultAzureCredential vs explicit production credentials (Concept 52) |
| [Deploy Aspire apps to Azure](https://aspire.dev/deployment/azure/) · [What's new in Aspire 13.3](https://devblogs.microsoft.com/aspire/whats-new-aspire-13-3/) · [Aspire roadmap update (June 2026)](https://github.com/microsoft/aspire/discussions/18023) | Aspire as the deployment front door (Concept 53) |

### Pricing, lifecycle and free training

| Resource | What it covers |
|---|---|
| [App Service pricing](https://azure.microsoft.com/pricing/details/app-service/linux/) · [Functions pricing](https://azure.microsoft.com/pricing/details/functions/) · [Container Apps pricing](https://azure.microsoft.com/pricing/details/container-apps/) · [AKS pricing](https://azure.microsoft.com/pricing/details/kubernetes-service/) · [Pricing calculator](https://azure.microsoft.com/pricing/calculator/) | Inputs for Exercise 2 and Concept 57 |
| [Azure Updates](https://azure.microsoft.com/updates/) · [Advisor: plan for service retirements](https://learn.microsoft.com/en-us/azure/advisor/advisor-how-to-plan-migration-workloads-service-retirement) | Tracking lifecycle risk (Concept 63) |
| [Cloud Services (extended support) retirement Q&A](https://learn.microsoft.com/en-us/answers/questions/5844312/cloud-services-extended-support-will-be-retired-on) · [Azure Spring Apps retirement](https://learn.microsoft.com/en-us/azure/spring-apps/basic-standard/retirement-announcement) | Two retirements driving migrations (Concept 46) |
| [.NET Microservices: Architecture for Containerized .NET Applications](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/) · [Architecting Cloud-Native .NET Apps for Azure](https://learn.microsoft.com/en-us/dotnet/architecture/cloud-native/) | Free Microsoft e-books — older platform details, still-valid principles |
| [Azure Developer Associate (AZ-204)](https://learn.microsoft.com/en-us/credentials/certifications/azure-developer/) · [Azure Solutions Architect Expert (AZ-305)](https://learn.microsoft.com/en-us/credentials/certifications/azure-solutions-architect/) | Free Microsoft Learn paths covering these services hands-on |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What a platform is | Seven delegated decisions + optional programming model; constraints are the invoice |
| The rule | Highest rung whose constraints the workload can live with for its whole life; step down on a named trigger |
| The ladder | Functions → App Service → Container Apps → AKS (Automatic, then Standard) → VMs |
| Autoscaling | Unit × signal × time-to-useful; headroom = ramp rate × time-to-useful |
| Cold start | Placement, pull, process, runtime, app init, first-request connections; measure first-request p99 |
| Cost | Consumption wins when average/peak < provisioned price / consumption price; add hidden costs and people |
| App Service | Plan = unit of compute and billing; 3/10/30/100 instances; Pv4 default; slots warm-swap; 230 s; SNAT; ARR affinity off |
| Managed Instance | Windows Pv4/Pmv4 for COM, registry, drives — legacy .NET Framework without VMs |
| Functions plans | Flex default (Linux, code-only, isolated .NET 8–10, 1,000/group, always-ready, 250-core quota); Premium; Dedicated; on Container Apps; Consumption legacy |
| Functions deadlines | In-process ends Nov 10, 2026; v3 on Linux Consumption stops Sept 30, 2026; Linux Consumption retires Sept 30, 2028 |
| Functions scaling | Backlog ÷ per-instance target; partitions cap; per-function groups on Flex; concurrency = cost lever |
| Functions timeouts | 30 min default (Flex/Premium/Dedicated/ACA), 10 min max legacy; HTTP 230 s; 60 min scale-in grace |
| Durable | Replay-based; deterministic orchestrators; idempotent activities; versioning; Durable Task Scheduler; SDKs run anywhere |
| Delivery semantics | At-least-once; Event Hubs/Cosmos checkpoint per batch even on exceptions; poison/dead-letter queues |
| Container Apps | Kubernetes semantics without Kubernetes operations; workload-profiles environment; Consumption + Dedicated |
| Container Apps scaling | 0–10 default, 10 concurrent requests, 30 s poll, 300 s cool-down, 1→4→8 steps; CPU rules can't reach zero |
| Container Apps limits | No K8s API; 4 vCPU/8 GiB Consumption; 8 GB images; idle < 0.01 vCPU & < 1 KB/s; private endpoints trigger management fee |
| AKS | API + reconciliation loops; Automatic is the default; managed system pools + pod readiness SLA; NAP (Karpenter) |
| AKS scaling | Pods in seconds, nodes in minutes; requests drive HPA, scheduler and node autoscaling |
| AKS ingress | ingress-nginx retired March 2026; managed NGINX patched to Nov 2026; Gateway API (app routing / AGC) |
| AKS day-2 | ~3 minors/year; 1.36 GA, 1.37 Oct 2026; deprecated APIs; weekly node images; clusters as code |
| .NET images | .NET 10: Ubuntu 24.04 default, no Debian; chiseled; non-root `app`; port 8080; immutable tags |
| .NET runtime in containers | ProcessorCount = CPU limit; GC hard limit 75% of memory; DATAS default (.NET 9+); CFS throttling |
| Probes | live (instance-only), ready (can serve), startup (booting); dependencies off probed paths |
| Shutdown | SIGTERM → drain → exit; ShutdownTimeout < grace; preStop sleep on Kubernetes |
| Identity | Managed/workload identity, user-assigned preferred; Key Vault references; explicit prod credential |
| Aspire | App model vs compute environment; publish/deploy/destroy; ACA, App Service, AKS (Helm), Compose |
| Lock-in | Packaging < config < platform features < programming model < managed data services |
| Estate | Per-workload decisions, two or three platforms max, paved roads, lifecycle ADRs with review dates |
| When not to | No AKS for CVs, no Functions for everything, no platform per team, no migration without a trigger |
