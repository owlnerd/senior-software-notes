# Module 32 — Brownfield Thinking: Incremental Migration and the Strangler Fig
*Phase 7: The Architect-Specific Track · Senior/Architect Interview Prep for .NET & C#*

> **State of practice verified on October 7, 2026.** The ideas in this module are old and stable — Feathers' *seams* (2004), Fowler's *strangler* metaphor (2004), branch by abstraction (2007), parallel change (2014) — but the **.NET migration toolchain and the support clock moved a lot in the last year**, and in a brownfield conversation those dates are often the business case:
>
> - **The support clock.** **.NET 8 (LTS) and .NET 9 (STS) both reach end of support on November 10, 2026 — five weeks from today.** STS releases are now supported for 24 months, which is why the two dates coincide. **.NET 10** (released November 11, 2025) is the current LTS, supported to **November 2028**. On the Framework side, **.NET Framework 4.6.2 reaches end of support on January 12, 2027**, **3.5 SP1 on January 9, 2029**; **4.7 – 4.8.1 have no announced end date** and follow the lifecycle of the Windows version they ship with. "We're on .NET 8" is, as of next month, also a brownfield problem.
> - **The .NET Upgrade Assistant is officially deprecated** (late 2025). Its replacements are AI agents: **GitHub Copilot upgrade** (the upgrade agent — .NET Framework → .NET 8+, SDK-style conversion, Newtonsoft → System.Text.Json, `System.Data.SqlClient` → `Microsoft.Data.SqlClient`, Functions in-process → isolated, **Web Forms → Blazor**, adding Aspire), available in Visual Studio, VS Code, GitHub Copilot CLI and GitHub.com, which keeps its assessment, plan and task state as Markdown under `.github/upgrades/{scenarioId}/` and commits each step; and **GitHub Copilot modernization**, which handles the Azure-migration scenarios (identity, storage, messaging, databases, deployment). **AWS Transform for .NET** (GA May 2025) is the AWS equivalent for porting .NET Framework to Linux-ready .NET.
> - **The Microsoft-endorsed strangler fig for ASP.NET** is still **YARP + System.Web adapters**: a new ASP.NET Core app becomes the front door and forwards unmigrated routes to the ASP.NET Framework app. **`Microsoft.AspNetCore.SystemWebAdapters` is at 2.3.0** (February 2026); the docs moved under `aspnet/core/migration/fx-to-core/`; there is now an **Aspire integration** (`Aspire.Hosting.IncrementalMigration` with `AddIISExpressProject` and `WithIncrementalMigrationFallback`, plus `MapRemoteAppFallback()` in the Core app) — **still in preview**. **YARP is at 2.3.0**, moved to the `dotnet` GitHub organization, and ships a minimal container image.
> - **CoreWCF 1.9** (April 2026; 1.9.1 in June) is current; **CoreWCF 1.8 support ends October 24, 2026.**
> - **Managed Instance on Azure App Service went GA on August 18, 2026**: a plan-scoped Windows hosting option for apps that need COM components, registry keys, MSI installers or OS customization (Premium v4 plans, select regions, Entra ID / managed identity — no domain join). It is the new "lift and improve" landing zone for legacy .NET Framework web apps that the standard App Service sandbox can't host.
> - **BizTalk Server 2020 is the final BizTalk version** (announced December 2025): mainstream support ends **April 11, 2028**, end of support **April 10, 2030**; **Azure Logic Apps** is the named successor. BizTalk 2016 is already out of mainstream support.
> - **SQL Server 2025 *change event streaming*** (also in Azure SQL Database and Managed Instance) pushes row-level changes as CloudEvents to Azure Event Hubs or Fabric Eventstream — useful for coexistence-phase synchronization — but it is **still in preview**, and its AMQP transport was deprecated on August 15, 2026. Classic **CDC**, **change tracking** and **Debezium** remain the production options.
> - **Martin Fowler rewrote the strangler fig entry** (August 2024) and pointed it at **Patterns of Legacy Displacement** (Ian Cartwright, Rob Horn, James Lewis), whose catalogue — *Transitional Architecture, Legacy Mimic, Event Interception, Divert the Flow, Revert to Source, Critical Aggregator, Extract Product Lines, Feature Parity* (an anti-pattern) — is now the shared vocabulary in serious modernization discussions.
> - **Microsoft's "enterprise web app patterns" were retired.** The **Reliable Web App** (replatform) and **Modern Web App** (refactor with the strangler fig) guidance pages for .NET now **redirect to the App Service baseline architecture**, and their GitHub reference implementations are **archived (read-only)**. You will still hear the names — they shaped a lot of Azure migration thinking from 2023 to 2025 — but cite them as history. The **Strangler Fig** and **Anti-corruption Layer** pattern pages in the Azure Architecture Center are current (Strangler Fig last updated June 2026).
> - **Library versions used in code below:** Microsoft.FeatureManagement 4.8.0 (September 2026), Scientist.NET 2.2.0, YARP 2.3.0, SystemWebAdapters 2.3.0.
>
> Tool and lifecycle facts are "verified on this date"; the principles are stable.

## Orientation

Here is the sentence to carry through the whole module: **in brownfield work the running system is both the constraint and the asset — you change it through a sequence of small, reversible, value-delivering transitions in which old and new coexist behind a seam you control, each transition with a way back, until the old part can be switched off; you never bet the business on a single cutover.**

The curriculum entry for this module reads: *incremental migration, the strangler fig pattern.* Module 31 ended with the promise to show *how to move a running system from the as-is diagram to the to-be diagram one reversible transition at a time.* That is the whole job.

How this connects to earlier modules:

- **Module 31** gave you the documentation tools — as-is / to-be / transition diagrams and retroactive ADRs. This module is what you *do* with them.
- **Modules 20–22** (layering, modular monolith vs microservices, DDD) gave you the *target* shapes and the boundaries — bounded contexts are usually the best seams.
- **Modules 11–12** (messaging, outbox, CDC, data storage, sagas) are the data-synchronization toolkit you need during coexistence.
- **Module 13 and 25** (reliability, Polly) and **Module 28** (observability, SLOs) are how you know each transition is safe.
- **Module 26** (compute choices) and **Module 29** (identity) are where most Azure-migration decisions land.
- **Module 30** (design docs) — a migration plan is a design doc whose "design" is a sequence of states.
- **Module 33** (cost, build-vs-buy, technical debt) — the next module — supplies the business language for *why* to migrate at all.

Why it matters in interviews:

1. **Architect loops include a brownfield exercise explicitly** (Module 1's table: *"a brownfield/incremental-redesign exercise"*). The prompt is some version of: *"Here's a 12-year-old monolith with a shared database. The business wants X. What do you do?"*
2. **It separates architects from designers.** Anyone can draw a target architecture. The architect is the person who can get there *without stopping the business*, explain the transitional states, and say what happens when step 4 goes wrong.
3. **Senior candidates are expected to have scars** — a migration that stalled at 80%, a dual-write bug, a "temporary" proxy still running three years later — and to have learned mechanism-level lessons from them.
4. **It tests honesty about trade-offs.** Incremental migration is not free: the coexistence period costs money and complexity. Interviewers listen for whether you price that in or pretend it away.

This module has six jobs:

1. **Build a first-principles model** of why big-bang rewrites fail and what incremental change buys and costs.
2. **Teach how to understand a legacy system** well enough to change it — discovery, seams, characterization tests, choosing the first slice.
3. **Teach the strangler fig thoroughly** — mechanics, the interception layer, slicing strategies, transitional architecture and failure modes.
4. **Teach the companion patterns** — anti-corruption layer, branch by abstraction, feature flags, parallel run, expand/contract, and the legacy-displacement catalogue.
5. **Solve the data problem** — ownership transfer, synchronization, splitting shared databases, cutover and rollback.
6. **Ground it in .NET and Azure** and prepare you to lead and narrate it under interview conditions.

Seven framings to carry through:

1. **The legacy system is a specification nobody wrote down.** Every observable behaviour is depended on by someone; assume it matters until proven otherwise.
2. **Incremental beats big-bang because risk and value are both front-loaded.** Small steps fail small and pay early; a rewrite fails late and pays at the end — if it pays.
3. **Control the seam, control the migration.** Whoever owns the interception point — a proxy, an interface, a queue, a table — decides where each request, call or event goes.
4. **Transitional architecture is real architecture.** It needs design, ownership, monitoring, ADRs and a removal date.
5. **Data is the hard part.** Moving code is easy to reverse; moving the source of truth is not.
6. **Every step has a way back — until a deliberate point of no return.** Name it, prepare for it, and cross it on purpose.
7. **You're not done until the old thing is off.** Decommissioning is a deliverable with a date, not a hope.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What "brownfield" means | Changing a running system whose behaviour is depended on — the normal case |
| 2 | Why big-bang rewrites fail | Moving target, hidden requirements, value deferred, one giant risky cutover |
| 3 | The economics of incremental change | Front-loaded value and learning, bounded risk — paid for with a coexistence tax |
| 4 | What the legacy system knows | Hyrum's law, Chesterton's fence, the Feature Parity trap |
| 5 | Outcomes before technology | Agree *why* (cost of change, process, retirement, disruption) before *how* |
| 6 | Discovery: mapping the as-is | Climb a tree: structure, runtime, data, people, shadow IT |
| 7 | Seams | Places to change behaviour without editing in place — in code, at runtime, in the business |
| 8 | Safety nets | Characterization tests, golden masters, telemetry baselines before you touch anything |
| 9 | Choosing the first slice | Valuable, low-coupling, learnable, reversible — not the trivial edge, not the core |
| 10 | The strangler fig pattern | Intercept → divert slice by slice → retire; coexistence by design |
| 11 | The interception layer | A dumb, observable, highly available router you own — YARP, APIM, Front Door, a queue |
| 12 | Slicing and routing strategies | By route, capability, product line, tenant/cohort, read vs write, event type |
| 13 | Transitional architecture | Temporary components that buy options — designed, owned and scheduled for removal |
| 14 | How strangler figs fail | The 80% stall, the god facade, double maintenance, no decommissioning |
| 15 | Anti-corruption layer | Translate at the boundary so the legacy model doesn't leak into the new one |
| 16 | Branch by abstraction | The in-process strangler: abstraction, two implementations, switch, delete |
| 17 | Feature flags for migration | Release, ops and permission toggles; kill switches; cohort rollout; toggle debt |
| 18 | Parallel run and dark launching | Run both, compare, use the old result until the new one is proven |
| 19 | Expand and contract | Change an interface or schema in three reversible steps |
| 20 | The legacy-displacement catalogue | Legacy Mimic, Event Interception, Divert the Flow, Revert to Source, Critical Aggregator |
| 21 | Why data is the hardest part | Shared databases are integration contracts; data has gravity and a single source of truth |
| 22 | Transferring data ownership | Read old → sync → flip the writer → retire; one writer at a time |
| 23 | Synchronization during coexistence | CDC, outbox, change event streaming — never naive dual writes; idempotency; reconciliation |
| 24 | Splitting a shared database | Views, wrapping services, schema-per-owner, moving joins and FKs into code |
| 25 | Cutover, rollback and the point of no return | Reversible flips, reverse sync, reconciliation, a named one-way door |
| 26 | .NET Framework → modern .NET: the landscape | Support dates, what didn't come across, the dependency graph decides the order |
| 27 | Incremental ASP.NET → ASP.NET Core | YARP fallback + System.Web adapters, remote session and auth, Aspire |
| 28 | Porting libraries and the tooling | .NET Standard 2.0 and multi-targeting bridges; Copilot upgrade; AWS Transform; analyzers |
| 29 | Rehost, replatform, refactor on Azure | The Rs; Managed Instance; replatform first, then strangle |
| 30 | The hard Windows-era technologies | WCF, Web Forms, WF, Remoting, MSMQ, BizTalk, Windows services, COM |
| 31 | Planning and measuring a migration | A roadmap of states, value per step, leading indicators, a stop-adding-to-legacy rule |
| 32 | Decommissioning as a deliverable | Traffic to zero, data archived, dependencies removed, costs gone — proven |
| 33 | Organization, people and Conway | Who owns the legacy, knowledge holders, sponsorship, corporate antibodies |
| 34 | AI in brownfield work, 2026 | Comprehension and mechanical porting are cheap; behavioural equivalence still must be proven |
| 35 | Brownfield in the interview | Understand → stabilize → seam → first slice → coexist → data → cut over → decommission |

---
# Part A — First principles: why rewrites fail and incremental change wins

## Concept 1 — What "brownfield" means

The terms come from construction. A **greenfield** site has never been built on; a **brownfield** site has existing structures — and existing contamination — that constrain what you can build and how. In software:

| | Greenfield | Brownfield |
|---|---|---|
| Starting point | Nothing runs yet | A system is running and people depend on it |
| Requirements | Written (more or less) | Embodied in behaviour, mostly unwritten |
| Freedom | Choose any architecture | Constrained by data, contracts, integrations, skills, contracts with vendors |
| Risk of change | Low — nobody uses it yet | High — every change can break someone |
| Main skill | Design | Design *plus* sequencing, coexistence and risk management |

**Brownfield is the default, not the exception.** Most engineering effort in most companies goes into changing systems that already exist. Even a "new" service usually integrates with, replaces part of, or takes data from an existing one. A team that has shipped for two years is already brownfield.

**What "legacy" really means.** It is not "old" and it is not "written in a language I don't like". Useful definitions:

1. **Michael Feathers** (*Working Effectively with Legacy Code*, 2004): *legacy code is code without tests* — because without tests you can't change it with confidence.
2. A pragmatic variant: **legacy is valuable code that you are afraid to change.** Both words matter — *valuable* (it earns money or runs the business; otherwise just delete it) and *afraid* (the cost and risk of change have grown out of proportion to the change).
3. In platform terms: a system is legacy when **its platform, skills, or vendor support are disappearing** — an unsupported runtime, a framework nobody hires for, a product at end of life (BizTalk, .NET Framework 4.6.2, .NET 8 next month).

These definitions point to different remedies: the first to tests and seams, the second to reducing the cost of change, the third to platform migration. A real brownfield problem usually has all three.

**Brownfield ≠ big ball of mud.** Many brownfield systems are reasonably designed for the world they were built in. The problem is usually that the *world* changed — scale, team size, business model, regulation, cloud — not that the original engineers were careless. Starting from respect for the existing design is both kinder and more accurate, and interviewers notice the difference.

**The interview-grade sentence:** *"Brownfield means changing a system that's running and depended on, which is the normal case in software: the requirements are embodied in behaviour rather than written down, and every change can break someone. I use Feathers' definition — legacy code is code without tests — together with a more economic one: valuable code you're afraid to change, or whose platform is disappearing. Those point to different remedies — tests and seams, lowering the cost of change, or platform migration — and real systems usually need all three. I also assume the original design was reasonable for its time; usually the world changed, not the engineers."*

---

## Concept 2 — Why big-bang rewrites fail

The tempting plan: *"We know what the old system does. Let's build a new one that does the same thing, properly, then switch over."* Experience — Fowler's rewritten strangler fig article says such plans go down in flames *most of the time* — shows why it usually fails. The mechanisms:

| Mechanism | What happens |
|---|---|
| **Moving target** | The business can't stop for 18 months. New requirements land in the old system; the new one has to chase them. Either the business freezes (and loses), or the rewrite never catches up |
| **Hidden requirements** | Much of the old behaviour is undocumented: edge cases, data-quality fixes, regulatory quirks, workarounds other systems rely on. They're discovered late — usually in UAT or after cutover |
| **Feature parity trap** | "Do exactly what the old one does" means rebuilding behaviour nobody wants any more, *and* agreeing what "exactly" means — itself a huge effort. Patterns of Legacy Displacement lists **Feature Parity** as a pattern that is usually an anti-pattern |
| **Value deferred** | Nothing is delivered until the end. The investment is all cost for years; sponsorship erodes; the project gets cut or "descoped" just before it would have paid |
| **One giant cutover** | All risk concentrates in one weekend. If it fails, the rollback is also huge — and often impossible once new data has been written |
| **Second-system effect** | Fred Brooks' observation: the second system an architect designs tends to be over-engineered, packed with every idea deferred from the first |
| **Knowledge discontinuity** | The rewrite team is often new, and the people who understand the old system are busy keeping it alive |
| **The treadmill** | Patterns of Legacy Displacement describes organizations stuck in a cycle of 3–5-year modernization programmes, each abandoned when business needs overtook its tech strategy, leaving another layer on top of an unchanged legacy |

**The classic reference** is Joel Spolsky's *"Things You Should Never Do, Part I"* (2000), about Netscape's decision to rewrite its browser from scratch: years without a release while competitors shipped. His core argument holds for most systems: *old code isn't ugly by accident — it contains years of bug fixes, each encoding knowledge you'll lose by throwing it away.*

**When a rewrite *is* reasonable.** "Never rewrite" is a slogan, not a law. A replacement can be the right call when:

- The system is **small** — weeks, not years, to replace — and well understood.
- The **business process itself is being redesigned**, so parity isn't the goal (the new system is a different product).
- The old platform **cannot be incrementally evolved** and **cannot coexist** with a new one (e.g., an appliance, a vendor product with no seams), and the business can accept a stop-the-world cutover — Patterns of Legacy Displacement includes **Stop the World cutover** for exactly these cases.
- A **commercial product** replaces it (Module 33's build vs buy) — even then, migrate *users and data* incrementally where possible.

Even in those cases, the *delivery* can usually be incremental: replace functionality behind a seam, migrate cohort by cohort. The real distinction is not "rewrite vs refactor" but **"one cutover vs many small ones."**

**The interview-grade sentence:** *"Big-bang rewrites usually fail for structural reasons: the business can't freeze, so the target moves; the old system's real requirements are undocumented and surface late; 'feature parity' means rebuilding behaviour nobody needs and arguing about what the old system really does; nothing is delivered until the end, so sponsorship erodes; and all the risk lands in one cutover whose rollback is often impossible once new data exists. A rewrite can be right for a small, well-understood system, or when the business process itself is changing — but even then I'd deliver it as many small cutovers rather than one."*

---

## Concept 3 — The economics of incremental change

Incremental migration isn't better because it's fashionable; it changes the shape of the risk and value curves.

```text
 value
 delivered        incremental: value starts early, grows with each slice
   │                    ____________________
   │               ____/
   │          ____/
   │     ____/                              big bang: nothing until cutover,
   │ ___/                              ┌─── then everything (if it works)
   │/                                  │
   └───────────────────────────────────┴──────────────► time
                                   cutover
```

**What incremental buys:**

| Benefit | Why |
|---|---|
| **Early value** | Each slice delivers something — a faster page, a new feature, a retired licence — so the investment pays as it goes |
| **Bounded risk per step** | A failed slice affects one capability or one cohort, and can be rolled back |
| **Learning** | Each slice teaches you about the legacy, the new platform and the organization; later slices are planned with better information |
| **Option value** | You can stop, re-prioritize or change direction after any slice and still keep what's done. A rewrite has no useful intermediate states |
| **Continuous business delivery** | New features can be built in the new system from day one — the "moving target" becomes an advantage |
| **Sustained sponsorship** | Visible, measurable progress keeps funding alive |

**What it costs — the coexistence tax:**

| Cost | Example |
|---|---|
| **Transitional architecture** | A routing proxy, adapters, sync jobs, reconciliation reports — all eventually thrown away (Concept 13) |
| **Running two systems** | Two deployments, two monitoring setups, two sets of on-call knowledge, often two licences for a while |
| **Data synchronization** | CDC pipelines, conflict rules, reconciliation (Part E) |
| **Cognitive load** | Engineers and support must know which system handles what, for whom, this week |
| **Elapsed time** | Total calendar time may be longer than an optimistic rewrite estimate (but optimistic rewrite estimates are rarely met) |

**The senior position is to make this trade explicit.** Patterns of Legacy Displacement ends its worked example on exactly this point: transitional architecture is an *investment in risk mitigation*, and the team should be explicit about the value the business places on that mitigation versus the cost of the transitional pieces. Don't hide the tax; price it.

**Reversibility as the organizing principle.** Amazon's "one-way vs two-way doors" (Module 30) maps directly: incremental migration converts one huge one-way door into many two-way doors plus a few small, deliberate one-way doors (usually around data ownership). The architect's job is to keep as many steps as possible two-way and to make the remaining one-way steps small, late and well-prepared.

**Sacrificial architecture.** Fowler's *SacrificialArchitecture* entry argues it's often right to build something knowing it will be replaced when circumstances change. Brownfield work is the other side of that coin: the old system was perhaps a reasonable sacrificial architecture, and the transitional pieces you build now are deliberately sacrificial too.

**The interview-grade sentence:** *"Incremental migration changes the shape of the curves: value starts with the first slice instead of at cutover, each step's risk is bounded and reversible, every slice teaches us something, and we keep option value — we can stop or change direction after any slice and keep what's done. It isn't free: we pay a coexistence tax of transitional components, two systems to run, data synchronization and cognitive load. I make that trade explicit with the business, and I organize the plan around reversibility — many two-way doors and a few small, deliberate one-way doors, usually around data ownership."*

---

## Concept 4 — What the legacy system knows

Before replacing anything, internalize that the running system is the most complete specification you have — and that most of it is unwritten.

**Hyrum's law** (Hyrum Wright, Google): *with a sufficient number of users of an API, all observable behaviours of your system will be depended on by somebody* — regardless of what the contract says. For legacy systems this applies to everything: response timings, field ordering in a CSV export, the fact that a report runs at 02:00, a rounding quirk, an error message a partner parses, a column that "isn't used" but feeds the data warehouse.

**Chesterton's fence** (Module 31): don't remove a fence until you know why it was put up. Every strange behaviour gets a "why" before it gets deleted — and if nobody knows why, the default is to *preserve and observe*, not to delete.

**Where the hidden requirements live:**

| Source | Example |
|---|---|
| **Edge-case code** | `if (customer.Country == "NO" && order.Date < new DateTime(2019, 1, 1))` — a regulatory rule from a past audit |
| **Data-quality fixes** | Trimming, defaulting and deduplication applied on write that downstream systems now assume |
| **Batch jobs and schedules** | A nightly job that recalculates balances; reports that must exist by 06:00 |
| **Integration side effects** | A partner polls a table; the data warehouse replicates the database (PoLD's worked example found KPI reports fed from the middleware's own database) |
| **Implicit contracts** | Message ordering, at-most-once assumptions, timeouts other systems are tuned to |
| **Shadow IT** | The Access database or the versioned spreadsheet a team runs alongside the system — Patterns of Legacy Displacement repeatedly finds that these *actually* run parts of the business |
| **User workarounds** | Users enter data in a specific order because the old system breaks otherwise; a "notes" field used to store structured data |
| **Operational knowledge** | "Restart the service on Mondays"; "never run the import during month-end" |

**The Feature Parity trap, reframed.** Parity isn't wrong because the old behaviour doesn't matter; it's wrong because it treats *all* behaviour as equally important and defers the question of which behaviour is still needed. The better posture:

1. **Find out what's actually used** — telemetry, logs, database access patterns, interviews. A typical legacy system has features nobody has used for years.
2. **Classify behaviour:** *must preserve exactly* (contracts, regulatory, money), *must preserve in outcome but may change in form* (UX, internal flows), *may drop* (unused), *must fix* (known bugs people work around — but beware Hyrum: fixing a bug can break someone who depends on it).
3. **Preserve the must-preserve with tests and parallel runs** (Concepts 8, 18); redesign the rest.

**The interview-grade sentence:** *"I treat the running system as the most complete specification we have, and mostly an unwritten one: by Hyrum's law every observable behaviour — timings, field orders, a 2 a.m. batch, a rounding quirk, a table the warehouse replicates — is depended on by someone, and the hidden requirements live in edge-case code, data fixes, batch schedules, integration side effects, shadow-IT spreadsheets and user workarounds. So rather than promising feature parity, I find out what's actually used, classify behaviour into must-preserve-exactly, preserve-in-outcome, drop and fix, and protect the must-preserve set with characterization tests and parallel runs."*

---

## Concept 5 — Outcomes before technology

"Modernize the platform" is not an outcome. Patterns of Legacy Displacement's first activity is **understand the outcomes you want to achieve**, and its observation is that different parts of an organization often want different things from the same programme. Agree the priority first, because it determines the strategy.

| Outcome | What it sounds like | What it implies for the migration |
|---|---|---|
| **Reduce the cost of change** | "A small change to the website takes weeks and costs six figures" | Start where change is most frequent and most expensive (hotspots, Concept 7); measure lead time and change failure rate |
| **Improve the business process** | "The system forces staff through a 12-step flow; mistakes mean starting over" | Redesign the process with the business; parity is explicitly *not* the goal for these slices |
| **Retire an old system** | "Licence renewal in 18 months"; "out of support next year" | Decommissioning is the deliverable; sequence by dependency, track remaining traffic and data |
| **Respond to imminent disruption** | "New regulation in 9 months"; "a competitor launched X" | Build the new capability in the new world first, integrated with legacy (Divert the Flow, Concept 20) |
| **Reduce operational risk** | "Only Dragan knows how to deploy it"; "we buy spare parts on eBay" | Stabilize first — automate deployment, add observability, document — before moving anything |
| **Cost** | "Hosting and licences are too expensive" | Rehost/replatform may be enough (Concept 29); don't refactor what you only need to move |
| **Enable scale or new markets** | "Can't onboard enterprise tenants"; "can't run in the EU region" | Target the specific constraint (tenancy, data residency), not the whole system |

**"Newer technology" is not an outcome.** PoLD warns specifically against *Netflix envy* — choosing technologies because famous companies use them — and recommends not making choices that can't be "done over" within 2–3 years, because technology lifetimes keep shrinking. The technology choice should fall out of the outcome.

**Make outcomes measurable** and put them in the migration's design doc (Module 30):

| Outcome | Measure |
|---|---|
| Cost of change | Lead time for changes, deployment frequency, change failure rate (DORA metrics) in the migrated area |
| Process | Task completion time, error/rework rate, support tickets |
| Retirement | % of traffic/functions/data on the legacy; date the last server is switched off; licence cost removed |
| Risk | Bus factor, mean time to restore, unsupported components count |
| Cost | Monthly run cost per environment |

**The interview-grade sentence:** *"Before touching technology I get agreement on the outcome, because different stakeholders usually want different things from the same 'modernization': lowering the cost of change, redesigning a business process, retiring a system before its licence or support ends, responding to a regulatory or competitive deadline, reducing operational risk, or cutting run cost. Each implies a different first slice and a different measure of success — lead time, process error rates, percentage of traffic still on the legacy, run cost — and newer technology on its own is not an outcome; it should fall out of the outcome."*

---
# Part B — Understanding before changing

## Concept 6 — Discovery: mapping the as-is

Patterns of Legacy Displacement's advice for starting: you begin in a forest, so **climb a tree** — get a good-enough view of the landscape quickly, without getting lost in detail. Module 31, Concept 31 covered drawing the as-is structure and recovering decisions; here is the fuller discovery checklist for a migration.

**Five lenses, each with its own sources:**

| Lens | Question | Sources on .NET/Azure |
|---|---|---|
| **Structure** | What are the deployables, data stores and integrations? | Solution and project files; deployment pipelines; IIS configuration; Bicep/ARM/Terraform; Azure Resource Graph; connection strings in config, Key Vault, App Configuration |
| **Runtime** | Who actually calls whom, how often, how slowly? | Application Insights application map and dependency telemetry; IIS logs; SQL Server Query Store and DMVs; Service Bus/MSMQ metrics; firewall logs |
| **Data** | What data exists, who writes it, who reads it, where does it flow? | Schema; `sys.dm_db_index_usage_stats`; SQL Server Audit or Extended Events on table access; SSIS packages; replication and linked servers; data-warehouse ETL; file shares and SFTP drops |
| **Business** | What capabilities does it support, for whom, through which processes? | Event Storming; business capability mapping; value stream mapping; customer journeys; Wardley maps |
| **People** | Who knows what? Who depends on it? | Interviews with the longest-serving engineers, support staff and power users; on-call history; Slack/Teams search for "why does…" questions |

**Outputs you want after one to two weeks:**

1. **As-is C4 context and container diagrams**, marked "verified on <date>" (Module 31).
2. **A dependency map**: inbound and outbound integrations, with protocol and owner — including the ones nobody remembered.
3. **A data map**: which component writes which tables; which components and external consumers read them.
4. **A capability map** overlaid on the containers: which parts of the system serve which business capabilities.
5. **A usage heatmap**: which endpoints, pages, reports and jobs are used, how often, by whom — and which aren't used at all.
6. **A risk register**: unsupported components with end-of-support dates (PoLD suggests an explicit calendar of end-of-life dates), single points of knowledge, known fragile areas.
7. **A short list of surprises** — every discovery that contradicted what people believed. These are the most valuable findings.

**Cross the boundary.** PoLD notes a common failure: discovery stops at the edge of the legacy system — *"here be dragons"* — so teams never learn how the legacy actually supports or hinders the business processes. Go inside: read the code paths behind the most important capabilities, and watch users work.

**Find the shadow IT.** Ask users what they do *outside* the system: exports to Excel, Access databases, manual re-keying, email-based approvals. Requirements like "must support CSV import/export" often turn out to be symptoms of such workarounds.

**The interview-grade sentence:** *"Discovery for a migration looks through five lenses: structure from solutions, pipelines and infrastructure definitions; runtime from Application Insights dependency data, IIS and Query Store; data flows from schema, index-usage stats, auditing and ETL; business capabilities from Event Storming and capability or value-stream mapping; and people from interviews with long-serving engineers, support and power users. In one to two weeks I want as-is context and container diagrams, a dependency map, a data map of who writes and reads what, a capability overlay, a usage heatmap, an end-of-support calendar and a list of surprises — and I deliberately cross into the legacy and look for the spreadsheets and Access databases that actually run part of the business."*

---

## Concept 7 — Seams

A **seam**, in Michael Feathers' definition, is *a place where you can alter behaviour in your program without editing in that place*. Each seam has an **enabling point** — where you decide which behaviour is used. Fowler's 2024 *LegacySeam* entry extends the idea from code to whole systems: in a well-designed system the seams already exist; in legacy systems you usually have to *find* or *insert* them before you can split anything.

**Seams at different scales:**

| Scale | Seam | Enabling point | .NET example |
|---|---|---|---|
| **Object** | An interface or virtual method | Dependency injection / construction | Extract `IPriceCalculator` from a static `PriceHelper`; register the implementation in DI |
| **Link / assembly** | Which assembly or package is loaded | Project references, build configuration | Swap a legacy data-access assembly for one with the same public surface |
| **Process / HTTP** | A URL or route | Reverse proxy, gateway, DNS | YARP route `/orders/**` → new service, everything else → legacy |
| **Messaging** | A queue or topic | Broker configuration, routing rules | Insert a router between producer and consumer queue (PoLD's Event Router) |
| **Data** | A table, view or stored procedure | Database permissions, synonyms, views | Replace a table with a view over the new store while the old code keeps reading it |
| **File / batch** | A drop folder, an SFTP path, a schedule | Job scheduler, file paths | The nightly import reads a file produced by the new system instead of the old one |
| **UI** | A page, a menu item, a frame | Navigation, the front-end shell | Menu link points to the new Blazor page instead of the old Web Forms page |
| **Business** | A product line, customer segment, region, channel | Business routing rules | New product type handled only by the new system (PoLD's worked example: *segment by product*) |

**Good seams have three properties:** you can **route** at them (decide old vs new per request/call/event), you can **observe** them (see what flows through), and switching is **cheap and reversible** (configuration, not redeployment).

**Creating seams in legacy .NET code** (Feathers' techniques, applied):

- **Extract interface** from a concrete class, then inject it.
- **Wrap static calls** (`DateTime.Now`, `HttpContext.Current`, `ConfigurationManager.AppSettings`, static helpers) behind an injectable abstraction — `TimeProvider` exists for exactly this since .NET 8.
- **Subclass and override** a virtual method in tests when you can't change the constructor yet.
- **Introduce a facade** in front of a tangle of calls so callers depend on one surface you can redirect.
- **Sprout method / sprout class**: put new behaviour in a new, tested unit called from the old code rather than growing the old method.

```csharp
// Before: a hidden dependency on static time and a static data helper — no seam.
public class InvoiceService
{
    public decimal LateFee(int invoiceId)
    {
        var invoice = DataHelper.GetInvoice(invoiceId);          // static, hits the DB
        var daysLate = (DateTime.Now - invoice.DueDate).Days;     // static clock
        return daysLate > 30 ? invoice.Amount * 0.02m : 0m;
    }
}

// After: two object seams; behaviour unchanged, now testable and redirectable.
public interface IInvoiceReader { Invoice Get(int invoiceId); }

public class InvoiceService(IInvoiceReader invoices, TimeProvider clock)
{
    public decimal LateFee(int invoiceId)
    {
        var invoice = invoices.Get(invoiceId);
        var daysLate = (clock.GetLocalNow().DateTime - invoice.DueDate).Days;
        return daysLate > 30 ? invoice.Amount * 0.02m : 0m;
    }
}

// The legacy implementation keeps calling the old helper — later swapped for the new service's client.
public sealed class LegacyInvoiceReader : IInvoiceReader
{
    public Invoice Get(int invoiceId) => DataHelper.GetInvoice(invoiceId);
}
```

**Finding where to cut: hotspots and coupling.** Adam Tornhill's *Your Code as a Crime Scene* (and the open-source `code-maat` tool) uses version-control history to find **hotspots** — files with both high change frequency and high complexity — and **temporal coupling** — files that always change together even though they look unrelated. Hotspots are where reducing the cost of change pays most; temporal coupling reveals hidden seams (or the absence of them) better than static dependency graphs. A quick version needs only `git log`:

```bash
# Change frequency per file over the last two years (top 30) — combine with a complexity metric
git log --since="2 years ago" --name-only --pretty=format: -- '*.cs' \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -30
```

**Domain seams.** The best long-term seams follow **bounded contexts** (Module 22). Event Storming on the legacy's business processes often reveals contexts that the code tangles together — "Billing" logic spread across Orders, Customers and Reporting. The migration is the chance to cut along the domain seam, using the technical seams above as the mechanism.

**The interview-grade sentence:** *"A seam, in Feathers' sense, is a place where behaviour can change without editing in place, with an enabling point where we choose which behaviour runs — and they exist at every scale: an interface injected through DI, an assembly reference, an HTTP route at a proxy, a queue, a view or synonym in the database, a drop folder, a menu link, or a business segment like product line or region. Good seams let me route, observe and switch cheaply. In legacy .NET code I create them with extract-interface, wrapping statics like DateTime.Now behind TimeProvider, facades and sprout classes, and I choose where to cut with hotspot and temporal-coupling analysis from git history, aiming for seams that follow bounded contexts."*

---

## Concept 8 — Safety nets: characterization tests and baselines

You cannot safely change what you cannot verify. Before moving anything, build a net that tells you when behaviour changes.

### Characterization tests

A **characterization test** (Feathers) records what the code *actually does* today — not what it should do — so that any change in behaviour shows up. You don't judge the output; you pin it.

**Golden master / approval testing** is the scalable form: run a wide set of inputs through the legacy code, capture the outputs into files, and compare future runs against them. In .NET, **Verify** (VerifyTests) is the standard tool: it serializes the result to a `.verified.txt` file, and a test fails with a diff when the output changes.

```csharp
using VerifyXunit;
using Xunit;

public class LateFeeCharacterizationTests
{
    // Pin current behaviour across a grid of inputs, including the odd ones.
    public static TheoryData<decimal, int> Cases => new()
    {
        { 100m, 0 }, { 100m, 30 }, { 100m, 31 }, { 0m, 365 }, { -50m, 45 }, { 99_999_999.99m, 31 },
    };

    [Theory]
    [MemberData(nameof(Cases))]
    public Task Late_fee_matches_legacy(decimal amount, int daysLate)
    {
        var clock = new FakeTimeProvider(new DateTimeOffset(2026, 10, 7, 12, 0, 0, TimeSpan.Zero));
        var invoices = new StubInvoiceReader(amount, dueDate: clock.GetLocalNow().DateTime.AddDays(-daysLate));
        var fee = new InvoiceService(invoices, clock).LateFee(invoiceId: 1);

        return Verifier.Verify(new { amount, daysLate, fee })
                       .UseParameters(amount, daysLate);   // one .verified file per case
    }
}
```

Rules that make characterization tests useful:

1. **Include the weird cases** — negative amounts, nulls, boundary dates, huge values, Unicode, time zones. The weird cases are where the hidden requirements live.
2. **Record bugs as they are.** If the legacy returns a fee for a negative amount, the test pins that. Whether to *fix* it is a separate, explicit decision (Hyrum's law).
3. **Make outputs deterministic** — inject clocks (`FakeTimeProvider`), random seeds, culture (`CultureInfo.InvariantCulture`), and scrub volatile values (Verify scrubs GUIDs and dates by default).
4. **Use production-shaped inputs** — sampled, anonymized real requests are far better than invented ones. Recorded HTTP traffic replayed against both systems is the black-box version of the same idea.
5. **Delete or convert them later.** Once the new implementation is the source of truth, replace characterization tests with intention-revealing specification tests.

### Black-box characterization at the boundary

When the code is too tangled to test from inside, characterize from outside:

- **Contract snapshots** of HTTP responses (status, headers that matter, body shape) for a catalogue of requests.
- **Database end-state snapshots** after running a scenario: which rows changed, to what.
- **Message snapshots**: what was published, in which order, with which headers.
- **Report and file outputs**: byte-for-byte or normalized comparisons of generated CSVs, PDFs, EDI files.

### Observability baselines

Characterization tests catch behavioural differences in test; **baselines** catch them in production. Before the first transition, capture for the affected area:

| Baseline | Why |
|---|---|
| Request rate, latency percentiles and error rate per endpoint | Module 28's RED metrics — so "the new service is slower" is a fact, not a feeling |
| Business KPIs (orders/hour, payment success rate, report row counts) | Many migration bugs show up as business anomalies before technical alerts |
| Data volumes and row counts per table per day | Reconciliation (Concept 25) needs a baseline |
| Batch job durations and output sizes | Silent truncation shows up as a smaller file |
| Support ticket volume and categories | Users notice subtle changes first |

Add distributed tracing through the **interception layer** from day one, with an attribute that records **which side served the request** (`migration.target = legacy|new`). Every dashboard can then be split by old vs new.

**The interview-grade sentence:** *"Before moving anything I build a safety net: characterization tests that pin what the legacy actually does — including its bugs — usually as golden-master approval tests with Verify over a grid of production-shaped inputs, made deterministic with FakeTimeProvider, invariant culture and scrubbing; black-box snapshots of HTTP responses, database end states, messages and generated files where the code is too tangled to test from inside; and production baselines for latency, errors, business KPIs, data volumes and batch outputs. I tag every trace at the interception point with which side served it, so every dashboard can be split old versus new."*

---

## Concept 9 — Choosing the first slice

The first slice sets the pattern — technically and politically. Pick badly and the programme either learns nothing (too trivial) or fails publicly (too hard).

**Score candidates on five axes:**

| Axis | Prefer | Avoid |
|---|---|---|
| **Value** | A visible improvement someone cares about (speed, a new feature, a retired cost) | Something nobody will notice |
| **Coupling** | Few inbound dependencies; owns (or can own) its data; clear seam | The hub every other module calls; shared tables everywhere |
| **Learning** | Exercises the full path — the proxy, deployment, auth, observability, data sync — so later slices reuse it | A purely static page that teaches nothing about data or auth |
| **Risk** | Failure is survivable and reversible; not the money path on day one | Payments, regulatory reporting, or the core transaction as slice one |
| **Frequency of change** | A hotspot the business keeps asking to change | A stable corner nobody touches (moving it saves nothing) |

**Good first-slice archetypes:**

1. **A new feature built in the new world** — no parity problem, immediate value, establishes the new platform and the integration path back to the legacy (PoLD: new behaviour first; Fowler: the fig *begins with small additions, often new features*).
2. **A read-only capability with high traffic** — e.g., product catalogue or order history reads served from a new read model synchronized from the legacy database (CQRS-style, Module 23). Reversible, measurable, and it builds the data-sync pipeline.
3. **A well-bounded capability with its own data** — notifications, document generation, search — where ownership can move cleanly.
4. **An edge capability on a hotspot** — something changed every sprint, whose extraction immediately reduces lead time.

**The "walking skeleton" first.** Before the first real slice, make the end-to-end path exist: the interception layer in production routing 100% to the legacy, the new application deployed with CI/CD, telemetry with the `migration.target` attribute, shared authentication working. This is a pure refactoring (no behaviour change) — PoLD's worked example began exactly this way, inserting an event router that initially forwarded everything unchanged — and it de-risks every later step.

**Thin slices, not layers.** Migrate a *vertical* slice — UI, logic and data for one capability — rather than a horizontal layer ("first the data layer, then the business layer"). A horizontal migration delivers no user-visible value and creates a coexistence nightmare across every capability at once.

**The interview-grade sentence:** *"I pick the first slice on five axes — real value someone will notice, low coupling with a clear seam and ideally its own data, maximum learning because it exercises the proxy, deployment, auth, observability and data sync end to end, survivable and reversible risk, and high change frequency so extraction actually lowers lead time. Good candidates are a new feature built directly in the new world, a high-traffic read path served from a synchronized read model, or a bounded capability like notifications. Before that I ship a walking skeleton — the proxy in production routing everything to the legacy — and I always slice vertically, never by layer."*

---
# Part C — The strangler fig pattern

## Concept 10 — The strangler fig pattern

**Origin.** Martin Fowler saw strangler figs in the Queensland rainforest in 2001: a vine that germinates high in a host tree, grows down to the ground and up to the light, draws on the host until it is self-sustaining, and may eventually leave the host dead inside a fig-shaped shell. He wrote it up as *"Strangler Application"* in 2004 and later renamed it **"Strangler Fig Application"** — to emphasize the botanical metaphor over the violent connotation — and rewrote the entry in August 2024. The name is now the standard term for gradual legacy replacement.

**The pattern, in one sentence:** build the new system *around* the old one, **intercept** the traffic (or calls, or events) at a seam you control, **divert** it to new implementations one slice at a time while the rest continues to the legacy, and **retire** each legacy part once nothing reaches it — until the legacy can be switched off.

The Azure Architecture Center and many teams describe the same lifecycle as **transform → coexist → eliminate**:

```text
 State 0 (as-is)          State 1 (facade)           State 2..n (coexist)          Final (to-be)
 ───────────────          ────────────────           ────────────────────          ─────────────
  clients                  clients                    clients                       clients
     │                        │                          │                             │
     ▼                        ▼                          ▼                             ▼
 ┌────────┐              ┌──────────┐               ┌──────────┐                  ┌──────────┐
 │ legacy │              │  facade  │               │  facade  │                  │  facade  │ (or remove)
 └────────┘              └────┬─────┘               └──┬────┬──┘                  └────┬─────┘
                              │ 100%                  A,B│    │rest                    │ 100%
                              ▼                          ▼    ▼                        ▼
                         ┌────────┐                ┌──────┐ ┌────────┐              ┌──────┐
                         │ legacy │                │ new  │ │ legacy │              │ new  │
                         └────────┘                └──────┘ └────────┘              └──────┘
```

**The steps, concretely:**

1. **Insert the facade** (the interception layer, Concept 11) routing 100% to the legacy. No behaviour change. Prove it in production — latency overhead, availability, logging.
2. **Pick a slice** (Concept 9) and build it in the new system, with its anti-corruption layer (Concept 15) and data strategy (Part E).
3. **Divert progressively**: internal users → a pilot cohort → percentage rollout → 100% (Concept 12, 17). Watch the baselines (Concept 8). Roll back by configuration if anything regresses.
4. **Retire the legacy slice**: once traffic is zero for a soak period, remove the legacy code path (or at least make it unreachable), its data responsibilities and its tests. Retirement is part of the slice's definition of done (Concept 32).
5. **Repeat**, re-planning after each slice with what you learned.
6. **Eliminate**: switch off the legacy, then decide whether the facade stays (as a gateway with ongoing value) or is removed.

**Why it works:** it turns one irreversible cutover into many small reversible ones; it lets new features ship in the new system from day one; and it makes progress *visible* — "37% of requests and 4 of 11 capabilities are on the new platform" is a sentence an executive can track.

**Strangler fig is a family, not one technique.** The interception point can be:

| Interception at | Example | Pattern names |
|---|---|---|
| **HTTP / UI** | Reverse proxy routes `/catalog/**` to the new service | Classic strangler fig, gateway routing |
| **Messages / events** | A router consumes the legacy queue and forwards some messages to the new consumer | **Event Interception** (PoLD) |
| **In-process call** | An interface with old and new implementations, chosen by a flag | **Branch by Abstraction** (Concept 16) |
| **Data** | The new system integrates with the original source of the data instead of the legacy copy | **Revert to Source** (Concept 20) |
| **Business flow** | New cross-organization activities go to the new system first | **Divert the Flow** (Concept 20) |

**When the strangler fig is a poor fit:**

- The legacy has **no usable seam** and inserting one is as expensive as replacing it (some mainframe or vendor systems — though Thoughtworks' *uncovering mainframe seams* article shows more is possible than people assume).
- The system is **small enough** to replace in weeks.
- **Requests can't be split** because every operation touches the same core state in one transaction — in that case the first work is decoupling *inside* the monolith (modularize first, Module 21), then strangle.

**The interview-grade sentence:** *"The strangler fig — Fowler's metaphor from 2004, renamed and rewritten in 2024 — means building the new system around the old: insert a facade routing everything to the legacy, then divert one slice at a time to new implementations while the rest continues to the legacy, progressively by cohort with rollback by configuration, and retire each legacy slice once nothing reaches it, until the legacy can be switched off. It's a family of techniques defined by where you intercept — HTTP routes, messages with event interception, in-process calls with branch by abstraction, or data with revert-to-source — and it works because it turns one irreversible cutover into many reversible ones and makes progress visible."*

---

## Concept 11 — The interception layer

The facade is the single most important piece of transitional architecture: whoever controls it controls the migration. It is also a new production component on the critical path, so design it as such.

**Options on .NET / Azure:**

| Interceptor | Strengths | Watch out for | Use when |
|---|---|---|---|
| **YARP in an ASP.NET Core app** | Code-level control; route by path, header, query, method; custom middleware for cohort decisions; same stack as the team; can become the new app's front door (Concept 27) | You own its availability, scaling and patching | Default for .NET web migrations; the System.Web adapters pattern |
| **Azure API Management** | Policies (routing, transformation, auth, rate limits), developer portal, versioning; no code | Cost per tier; latency; policy XML can grow into logic | API-centric migrations; external partners; many consumers |
| **Azure Front Door / Application Gateway** | Global or regional L7 routing by path/host; WAF; already in front of many apps | Coarse routing only (paths, headers); no business logic | Path-level splits for web apps already behind them |
| **Azure Container Apps / Kubernetes ingress** | Traffic splitting between revisions; header-based routing | Splits by revision/service, not by business cohort without help | When legacy and new both run on the same platform |
| **A message router** (a .NET worker consuming the legacy queue) | Event interception for async integrations; content-based routing | Ordering, duplicates, poison messages, back-pressure | Messaging-based seams (MSMQ, Service Bus, IBM MQ) |
| **The legacy app itself** (an `IHttpModule` or controller that forwards) | No new infrastructure | Puts the router inside the thing you're replacing | Rarely — when you can't change DNS or the edge |

**Design rules for the interception layer:**

1. **Keep it dumb.** Route, authenticate, add headers, log. No business logic, no data transformation beyond protocol translation. A facade that accumulates rules becomes a new legacy monolith (Concept 14).
2. **Make routing configuration-driven and hot-reloadable** — YARP routes from configuration or App Configuration; flags from Feature Management. Rolling back a slice must not need a deployment.
3. **Make it highly available** — it's now on every request's path: multiple instances, zone redundancy, health probes, a tested deployment pipeline. Measure its added latency (typically low single-digit milliseconds in-region for a well-configured proxy) and budget for it.
4. **Make it observable** — propagate W3C trace context, tag every request with the chosen destination, emit per-route metrics. It is the best vantage point you will ever have for comparing old and new.
5. **Preserve the contract** — same URLs, cookies, headers, CORS, caching headers, status codes, redirects. Clients must not notice which side served them.
6. **Handle cross-cutting concerns once** — TLS termination, authentication (validate once, pass identity downstream), correlation IDs, rate limiting.
7. **Plan its end-state** — does it become the long-term API gateway, or is it removed? Write that in an ADR at insertion time.

**YARP configuration for a route-based strangler** — new catalogue service, everything else to the legacy:

```json
{
  "ReverseProxy": {
    "Routes": {
      "catalog-new": {
        "ClusterId": "catalog",
        "Order": 10,
        "Match": { "Path": "/catalog/{**rest}" }
      },
      "orders-pilot": {
        "ClusterId": "orders",
        "Order": 20,
        "Match": {
          "Path": "/orders/{**rest}",
          "Headers": [ { "Name": "X-Migration-Cohort", "Values": [ "pilot" ], "Mode": "ExactHeader" } ]
        }
      },
      "legacy-fallback": {
        "ClusterId": "legacy",
        "Order": 1000,
        "Match": { "Path": "{**catch-all}" }
      }
    },
    "Clusters": {
      "catalog": { "Destinations": { "primary": { "Address": "https://catalog.internal.contoso.com/" } } },
      "orders":  { "Destinations": { "primary": { "Address": "https://orders.internal.contoso.com/" } } },
      "legacy":  { "Destinations": { "primary": { "Address": "https://legacy.internal.contoso.com/" } } }
    }
  }
}
```

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));   // reloads on config change

var app = builder.Build();
app.MapReverseProxy();   // lower Order = higher priority; the catch-all goes last
app.Run();
```

Header-based routing is fine for internal pilots; for **per-user or per-tenant cohorts and percentage rollouts** you want the decision in code, driven by feature flags (Concept 17), using YARP's direct forwarder:

```csharp
using System.Diagnostics;
using System.Net;
using Microsoft.FeatureManagement;
using Yarp.ReverseProxy.Forwarder;

builder.Services.AddHttpForwarder();
builder.Services.AddFeatureManagement().WithTargeting();   // targeting context from the signed-in user

var invoker = new HttpMessageInvoker(new SocketsHttpHandler
{
    UseProxy = false,
    AllowAutoRedirect = false,
    AutomaticDecompression = DecompressionMethods.None,
    UseCookies = false,
    EnableMultipleHttp2Connections = true,
    ActivityHeadersPropagator = new ReverseProxyPropagator(DistributedContextPropagator.Current),
    ConnectTimeout = TimeSpan.FromSeconds(15),
});

var legacy = builder.Configuration["Migration:LegacyUrl"]!;
var orders = builder.Configuration["Migration:OrdersUrl"]!;

app.Map("/orders/{**rest}", async (HttpContext ctx, IHttpForwarder forwarder, IVariantFeatureManager features) =>
{
    bool useNew = await features.IsEnabledAsync("Migration.Orders", ctx.RequestAborted);
    string destination = useNew ? orders : legacy;

    Activity.Current?.SetTag("migration.target", useNew ? "new" : "legacy");   // split every dashboard

    var error = await forwarder.SendAsync(ctx, destination, invoker,
        ForwarderRequestConfig.Empty, HttpTransformer.Default);

    if (error != ForwarderError.None)
    {
        var failure = ctx.GetForwarderErrorFeature()?.Exception;
        app.Logger.LogWarning(failure, "Forwarding to {Target} failed: {Error}", destination, error);
    }
});
```

*(Sketch against YARP 2.3 and Microsoft.FeatureManagement 4.x; check the targeting-context setup for your authentication scheme.)*

**Sticky routing.** If a user is routed to the new system, keep them there for the session — flipping between implementations mid-flow creates inconsistent experiences and, worse, writes to two systems for one business operation. Targeting by user or tenant ID (rather than per-request randomness) gives stickiness for free.

**The interview-grade sentence:** *"The interception layer is the most important piece of transitional architecture, because whoever controls it controls the migration — on .NET that's usually YARP in an ASP.NET Core app, or API Management for API-centric work, Front Door for coarse path splits, or a message router for queue-based seams. I keep it dumb — routing, auth, headers and logging only — with configuration-driven, hot-reloadable routes and flags so rollback never needs a deployment; I make it highly available and measure its latency because it's now on every request; I tag every trace with which side served it; I preserve the client contract exactly; I route cohorts stickily by user or tenant; and I record in an ADR whether it becomes the long-term gateway or gets removed."*

---

## Concept 12 — Slicing and routing strategies

There are two separate decisions: **what** to move (the slice) and **who** gets the new version first (the cohort). Combining them gives you fine-grained, low-risk control.

### What to move — slice dimensions

| Slice by | How it looks | Good for | Watch out for |
|---|---|---|---|
| **Route / endpoint** | `/catalog/**` → new | Web apps and APIs with clean URL structure | Endpoints that share session or in-memory state |
| **Business capability / bounded context** | All of "Notifications" → new | Long-term architecture (Module 22) | Capabilities spread across many endpoints and tables |
| **Read vs write** | Reads from a new read model, writes still to legacy | Very low-risk first slices; builds sync pipeline | Read-your-writes lag (Module 7) |
| **Product line / value stream** | New product type handled only by new system (PoLD's *Extract Product Lines*, *segment by product*) | Businesses with distinct products or channels | Shared reference data across product lines |
| **Lifecycle stage** | New orders in new system; in-flight orders finish in legacy | Workflows with long-running instances | Two systems for "the same thing" until old instances drain |
| **Event type** | `OrderPlaced` routed to new consumer; others to legacy (Event Interception) | Messaging-based integrations | Ordering across event types |
| **Geography / legal entity** | EU tenants first | Data-residency or regulatory drivers | Cross-region reporting |

### Who goes first — cohort strategies

| Cohort | Sequence | Notes |
|---|---|---|
| **Internal users / dogfooding** | Staff accounts only | Cheapest feedback; tolerant users |
| **Pilot customers** | A named list of friendly tenants | Choose representative ones, not only the simple ones |
| **Percentage rollout (canary)** | 1% → 5% → 25% → 50% → 100% of users/tenants | Sticky by user/tenant ID; compare cohorts statistically |
| **New customers only** | Accounts created after date X | No data migration for them; legacy cohort shrinks naturally — but may never reach zero |
| **By segment** | Small tenants first; enterprise last | Large tenants often have the most customization and the most data |

**Combine them.** "Order history (read slice) for internal users, then 5% of tenants, then everyone; then order placement (write slice) for new tenants only; then migrate existing tenants in waves" is a realistic plan — each step reversible, each one teaching something.

**Canary analysis.** Google's SRE Workbook chapter on canarying applies directly: compare the canary cohort against a control cohort on the same metrics over the same window (error rate, latency percentiles, business conversion), with pre-agreed thresholds for "proceed", "hold" and "roll back". Decide the thresholds *before* the rollout, not while staring at a graph.

**The interview-grade sentence:** *"I separate what moves from who gets it first. Slices can be routes, bounded contexts, reads before writes, product lines, lifecycle stages such as new orders versus in-flight ones, event types or geographies; cohorts go from internal users to pilot tenants to sticky percentage rollouts, or new customers only, or small tenants before enterprise ones. Combining them — order history reads for staff, then five percent of tenants, then everyone, then writes for new tenants, then existing tenants in waves — keeps every step reversible, and I compare canary and control cohorts against thresholds agreed before the rollout starts."*

---

## Concept 13 — Transitional architecture

**Transitional architecture** (a Patterns of Legacy Displacement pattern): *software elements installed to ease the displacement of a legacy system, which we intend to remove when the displacement is complete.* The facade, adapters, event routers, sync jobs, reconciliation reports and legacy mimics are all transitional.

People balk at building code they plan to throw away. Fowler's response in the 2024 strangler fig entry: the reduced risk and earlier value of the gradual approach outweigh the cost of the transitional pieces. The senior skill is to **design transitional architecture deliberately** rather than letting it accrete.

**Typical transitional components:**

| Component | Purpose | Removed when |
|---|---|---|
| Routing facade | Divert traffic slice by slice | Legacy is gone (or it graduates to a permanent gateway) |
| Anti-corruption layer adapters | Translate between legacy and new models | The legacy counterpart is gone |
| Legacy → new data sync (CDC) | Keep new read models current while legacy owns writes | Writes move to the new system |
| New → legacy reverse sync | Keep legacy consumers (reports, other systems) working after ownership moves | Those consumers are migrated |
| Legacy mimic | Make the new system look like the legacy to systems that can't change (Concept 20) | Those systems are migrated or retired |
| Reconciliation jobs and dashboards | Prove the two sides agree | Legacy data is retired |
| Feature flags for migration | Choose old vs new per cohort | Rollout reaches 100% and soaks |
| Dual-run / comparison harness | Parallel run (Concept 18) | Confidence threshold reached |

**Treat it as real architecture:**

1. **Design and review it** — it often carries more production risk than the target components, because it sits between two systems.
2. **Give every transitional component an owner, monitoring and an on-call runbook.**
3. **Record it in ADRs**, each with an explicit **removal trigger** ("remove the reverse sync when Finance reporting reads from the new store; target Q2").
4. **Tag it** — in the C4 model (a `Transitional` tag rendered with a dashed border) and in Azure resources (`lifecycle=transitional`, `remove-after=2027-06`) — so it's visible on every diagram and cost report.
5. **Keep it simple** — it's sacrificial; optimize for correctness and observability, not elegance or reuse.
6. **Budget for its removal** — removal tasks go on the roadmap at the same time as the component.

**Draw every state.** Module 31's advice applies: the migration plan is a sequence of C4 container views — State 0 (as-is), State 1 (facade inserted), State 2 (catalogue reads on new), … Final — each showing the transitional components that exist *in that state*. Reviewers can then ask the right question of each state: *is this state safe to stay in for six months if the programme pauses?*

**The interview-grade sentence:** *"Transitional architecture — the facade, adapters, CDC sync, reverse sync, legacy mimics, reconciliation jobs and migration flags — is code we deliberately plan to remove, and I treat it as real architecture: it's designed and reviewed because it often carries more risk than the target, each piece has an owner, monitoring and a runbook, each has an ADR with an explicit removal trigger, it's tagged as transitional on the C4 model and on Azure resources so it shows up on diagrams and cost reports, and its removal is on the roadmap from the day it's built. I draw every intermediate state and ask of each: is it safe to stay here for six months if the programme pauses?"*

---

## Concept 14 — How strangler figs fail

The strangler fig fails in recognizable ways. Knowing them — ideally from experience — is a strong senior signal.

| Failure mode | Symptom | Cause | Prevention |
|---|---|---|---|
| **The 80% stall** | Easy slices migrated; the hard core (shared data, the money path) stays in legacy for years; both systems run forever | Sequencing by ease; no decommissioning goal; the remaining 20% has 80% of the coupling | Tackle the core's data ownership early (Part E); make decommissioning dates explicit outcomes; track *legacy remaining*, not *new built* |
| **The god facade** | Business rules, transformations and orchestration accumulate in the proxy/gateway | It's the one place both worlds meet, so logic lands there | "Dumb pipe" rule in an ADR; code review on facade changes; move logic into services or ACLs |
| **Double maintenance** | Every change has to be made in both systems | New features built in legacy "because that capability hasn't moved yet" | **Stop adding to the legacy** rule (Concept 31): new features go in the new world, integrated back if needed |
| **The distributed monolith** | New services all call back into the legacy synchronously; latency and failure coupling grow | Slices extracted without their data | Move data ownership with the slice; asynchronous integration via events (Module 11) |
| **Legacy shaped the new model** | The new system mirrors legacy tables, names and quirks | No anti-corruption layer; "temporary" direct table access | ACL (Concept 15); design the new model from the domain, not from the old schema |
| **Transitional architecture becomes permanent** | Sync jobs and mimics still running three years later | No owners, no removal triggers | Concept 13's rules; quarterly review of transitional inventory |
| **No decommissioning** | Traffic moved but servers, licences, databases and jobs remain | Nobody owns turning things off; fear of the unknown consumer | Decommissioning as a deliverable with proof (Concept 32) |
| **Corporate antibodies** | New ways of working rejected by unchanged governance, change control, procurement | Organization didn't change (PoLD: technology is at most half the problem) | Protected pilot, sponsorship, changing processes alongside systems (Concept 33) |
| **Big bang in disguise** | "Incremental" plan where nothing reaches production until all slices are done | Coexistence considered too hard, so slices are released together | Each slice reaches real users; if coexistence is hard, that *is* the design problem to solve |
| **Lost in the long tail** | The last 5% of endpoints — admin pages, rare reports, exports — never get migrated | Low value per item | Usage data: retire unused ones; batch the rest; accept a "legacy appliance" for genuinely dead-end features with a sunset date |

**The 80% stall deserves special attention.** It's the most common outcome of strangler projects. The usual root cause is that teams migrate stateless, low-coupling edges first (correctly) but keep postponing the shared core *data*. By the time they reach it, the programme has spent its sponsorship. The mitigation is to start the *data ownership* work for the core early, in parallel with the edge slices, even if the core's *code* moves late.

**The interview-grade sentence:** *"Strangler figs fail in recognizable ways: the 80% stall, where the easy edges move and the coupled core with the shared data stays in legacy forever; the god facade that accumulates business logic; double maintenance because new features keep landing in the legacy; a distributed monolith of new services calling back into the legacy synchronously; a new model shaped by the old schema; transitional components that become permanent; nothing ever actually switched off; and organizational antibodies. My counter-measures are a stop-adding-to-legacy rule, starting the core's data-ownership work early, a dumb-facade ADR, anti-corruption layers, removal triggers on transitional pieces, and tracking legacy remaining rather than new built."*

---
# Part D — Companion patterns

## Concept 15 — Anti-corruption layer

The **anti-corruption layer (ACL)** comes from Eric Evans' *Domain-Driven Design* (2003) and is one of the context-map relationships from Module 22: when your model must integrate with a system whose model is different — or worse — you put a **translation layer** at the boundary so the other model does not leak into yours.

In a migration the ACL is what keeps the new system *new*. Without it, the first slice reads legacy tables directly, adopts their column names and their quirks ("status 7 means cancelled unless `Flag2` is set"), and the new system becomes a re-implementation of the old one.

**What an ACL contains:**

| Part | Role |
|---|---|
| **Facade / client** | Talks the legacy protocol: SOAP, WCF, stored procedures, flat files, a shared table |
| **Translator** | Maps legacy representations to new domain concepts and back — codes to enums, denormalized rows to aggregates, magic values to explicit states |
| **Adapter** | Exposes the translated capability through an interface owned by the new model (`ICustomerDirectory`) |

**Where it lives:**

- **Inside the new service** (most common): the domain depends on a port such as `ICustomerDirectory`; an infrastructure adapter implements it against the legacy.
- **As a separate component** when several new services need the same translation, or when the translation is heavy (Azure Architecture Center's ACL pattern shows it as its own service) — but beware it becoming a shared bottleneck.
- **On the legacy side**, rarely, when you can add a clean API to the legacy that new services call.

```csharp
// New domain port — expressed in the new model's language.
public interface ICustomerDirectory
{
    Task<Customer?> FindAsync(CustomerId id, CancellationToken ct);
}

public sealed record Customer(CustomerId Id, string DisplayName, CustomerStatus Status, Country Country);
public enum CustomerStatus { Active, Suspended, Closed }

// ACL adapter: talks to the legacy (here, a stored procedure) and translates.
public sealed class LegacyCustomerDirectory(LegacyDbConnectionFactory db) : ICustomerDirectory
{
    public async Task<Customer?> FindAsync(CustomerId id, CancellationToken ct)
    {
        await using var conn = await db.OpenAsync(ct);
        var row = await conn.QuerySingleOrDefaultAsync<LegacyCustomerRow>(          // Dapper
            "dbo.usp_GetCust", new { CustNo = id.ToLegacyCustNo() },
            commandType: CommandType.StoredProcedure);

        return row is null ? null : Translate(row);
    }

    // All legacy knowledge is concentrated — and tested — here.
    internal static Customer Translate(LegacyCustomerRow r) => new(
        CustomerId.FromLegacy(r.CUST_NO),
        DisplayName: string.IsNullOrWhiteSpace(r.TRADE_NAME) ? r.CUST_NAME.Trim() : r.TRADE_NAME.Trim(),
        Status: (r.STAT_CD, r.FLAG2) switch
        {
            ("A", _)     => CustomerStatus.Active,
            ("S", _)     => CustomerStatus.Suspended,
            ("7", "Y")   => CustomerStatus.Active,     // legacy quirk documented in ADR-0041 (retroactive)
            ("7", _)     => CustomerStatus.Closed,
            _            => throw new LegacyDataException($"Unknown STAT_CD '{r.STAT_CD}' for {r.CUST_NO}")
        },
        Country: Country.FromIso2(r.CTRY ?? "RS"));
}
```

**Rules:** the domain never sees `LegacyCustomerRow`; the translator is pure and has its own characterization tests over real legacy rows; unknown legacy values fail loudly (or go to a quarantine) rather than being guessed; and when the legacy source is replaced, only the adapter changes.

**The interview-grade sentence:** *"An anti-corruption layer, from Evans' DDD, is the translation layer that stops the legacy model leaking into the new one: a facade that speaks the legacy protocol — stored procedures, SOAP, files, shared tables — a translator that turns codes, magic values and denormalized rows into explicit domain concepts, and an adapter behind a port the new model owns. In a migration it's what keeps the new system from becoming a re-implementation of the old schema; I usually put it inside the consuming service, keep the translator pure and characterization-tested on real legacy rows, fail loudly on unknown values, and when the legacy goes away only the adapter changes."*

---

## Concept 16 — Branch by abstraction

**Branch by abstraction** (Paul Hammant, 2007; Fowler's bliki, 2014) is the **in-process strangler fig**: a way to make a large change to a component *on trunk*, in small steps, while the system keeps working and releasing — without a long-lived source-control branch.

**The steps:**

1. **Introduce an abstraction** over the part you want to replace (an interface in front of the legacy component). Move callers to the abstraction one at a time. *Release.*
2. **Build the new implementation** behind the same abstraction, incrementally, with its own tests. *Release* — it isn't used yet.
3. **Switch** callers or traffic to the new implementation — all at once, per call site, or per cohort via a flag. *Release.*
4. **Remove the old implementation** — and, if it no longer earns its keep, the abstraction. *Release.*

```csharp
// Step 1 — the abstraction, initially implemented by the legacy code.
public interface IPricingEngine
{
    Task<Quote> QuoteAsync(Cart cart, CancellationToken ct);
}

public sealed class LegacyPricingEngine : IPricingEngine            // wraps the 2,000-line PriceCalc.cs
{
    public Task<Quote> QuoteAsync(Cart cart, CancellationToken ct) =>
        Task.FromResult(PriceCalc.Calculate(cart.ToLegacyBasket()).ToQuote());
}

// Step 2 — the new implementation, built and tested in place, not yet used.
public sealed class RulesPricingEngine(IPriceRuleRepository rules, TimeProvider clock) : IPricingEngine
{
    public async Task<Quote> QuoteAsync(Cart cart, CancellationToken ct) { /* … */ }
}

// Step 3 — the switch: a flag-driven router behind the same abstraction.
public sealed class PricingEngineRouter(
    LegacyPricingEngine legacy, RulesPricingEngine modern, IVariantFeatureManager flags) : IPricingEngine
{
    public async Task<Quote> QuoteAsync(Cart cart, CancellationToken ct) =>
        await flags.IsEnabledAsync("Migration.RulesPricing", ct)
            ? await modern.QuoteAsync(cart, ct)
            : await legacy.QuoteAsync(cart, ct);
}

// Registration
builder.Services.AddScoped<LegacyPricingEngine>();
builder.Services.AddScoped<RulesPricingEngine>();
builder.Services.AddScoped<IPricingEngine, PricingEngineRouter>();

// Step 4 — once at 100% and soaked: delete LegacyPricingEngine, PriceCalc.cs and the router;
// register RulesPricingEngine as IPricingEngine directly.
```

**Branch by abstraction vs strangler fig:** same idea, different seam. Strangler fig intercepts at a *process boundary* (HTTP, messaging); branch by abstraction intercepts at a *code boundary* (an interface). They compose: you often use branch by abstraction inside the legacy to create a seam, then point the new implementation at an external service — at which point it *is* a strangler fig. PoLD's integration-middleware example notes the team used branch by abstraction at subsystem scale, with the queues and JDBC calls as the abstraction.

**Why not a long-lived branch?** A six-month feature branch is a big-bang rewrite in miniature: it diverges, merges painfully, and delivers nothing until merged. Branch by abstraction keeps everyone on trunk, keeps CI meaningful, and lets you release at every step — the trunk-based development practice from DORA's research.

**The interview-grade sentence:** *"Branch by abstraction is the in-process strangler fig: introduce an interface over the component to replace and move callers onto it, build the new implementation behind the same interface while it's unused, switch callers or cohorts to it — often with a feature-flag router — and finally delete the old implementation and maybe the abstraction, releasing from trunk at every step instead of living on a long branch. It composes with the process-level strangler: branch by abstraction creates the seam inside the legacy, and when the new implementation is a client of an external service, it has become a strangler fig."*

---

## Concept 17 — Feature flags for migration

Feature flags decouple **deployment** (code is in production) from **release** (users get the behaviour). In a migration they are the switch at every enabling point. Pete Hodgson's taxonomy (*Feature Toggles*, martinfowler.com) is the standard vocabulary:

| Toggle type | Lifetime | Dynamism | Migration use |
|---|---|---|---|
| **Release toggle** | Days to weeks | Per release / per cohort | "Serve order history from the new read model" — removed when at 100% |
| **Experiment toggle** | Days to weeks | Per user, random but sticky | A/B or canary comparisons of old vs new |
| **Ops toggle (kill switch)** | Can be long-lived | Instant, at runtime | "Fall back to legacy pricing" — flipped during an incident |
| **Permission toggle** | Long-lived | Per user/tenant | Pilot tenants; enterprise tenants last |

**On .NET/Azure:** **Microsoft.FeatureManagement** (4.8.0) with **Azure App Configuration** as the store gives you targeting (users, groups, default rollout percentage — sticky by user), time windows, variants and telemetry of evaluations to Application Insights. Configuration changes propagate without redeployment (App Configuration's refresh or push model).

```json
{
  "feature_management": {
    "feature_flags": [
      {
        "id": "Migration.Orders",
        "enabled": true,
        "conditions": {
          "client_filters": [
            {
              "name": "Microsoft.Targeting",
              "parameters": {
                "Audience": {
                  "Users": [ "qa.lead@contoso.com" ],
                  "Groups": [ { "Name": "pilot-tenants", "RolloutPercentage": 100 } ],
                  "DefaultRolloutPercentage": 5,
                  "Exclusion": { "Groups": [ "enterprise-tenants" ] }
                }
              }
            }
          ]
        }
      }
    ]
  }
}
```

**Migration-specific rules:**

1. **Target by a stable key** — user or tenant — so the cohort is sticky and a business operation never straddles both systems.
2. **One flag per slice, with an owner and an expiry date.** Toggle debt is real: stale flags multiply code paths and test combinations. Add a CI check or a report of flags older than their expiry.
3. **Keep the kill switch separate from the rollout flag** — rollout goes 5% → 100%; the kill switch forces 0% instantly regardless of rollout state.
4. **Flag evaluation is on the hot path** — cache it, and decide the behaviour when the flag store is unreachable (fail to legacy for migration flags).
5. **Test both paths** in CI while the flag exists.
6. **Remove the flag and the old path together** as part of decommissioning (Concept 32).
7. **Data flags are dangerous.** A flag that changes *where data is written* is not trivially reversible — flipping it back doesn't move the data written while it was on. Pair such flags with a sync or reverse-sync strategy (Part E), or don't make writes flag-switchable at all.

**The interview-grade sentence:** *"Feature flags decouple deployment from release, so in a migration they're the switch at every enabling point. Using Hodgson's taxonomy I use short-lived release and experiment toggles for rollout, a separate long-lived ops toggle as a kill switch that forces traffic back to the legacy instantly, and permission toggles for pilot tenants — on .NET via Microsoft.FeatureManagement with Azure App Configuration and targeting by a stable user or tenant key so cohorts are sticky. Every flag has an owner and expiry date, both paths are tested while it exists, it fails safe to the legacy if the flag store is unreachable, and I'm careful with flags that change where data is written, because flipping them back doesn't move the data."*

---

## Concept 18 — Parallel run, dark launching and shadow traffic

A characterization test shows the new code matches the old on the inputs you thought of. A **parallel run** shows it matches on the inputs *production* sends. The principle: **run both implementations on real traffic, return the legacy result, compare, and only switch once the differences are understood.**

**Variants:**

| Technique | What happens | Side effects | Best for |
|---|---|---|---|
| **In-process experiment** (GitHub's Scientist; Scientist.NET) | Call both implementations in the same request; return the *control* (legacy) result; publish mismatches | Candidate must be side-effect free (or run against a sandbox) | Pure computations: pricing, permissions, calculations, query results |
| **Dark launching** (PoLD, Fowler) | Call the new back end without using its result, to assess performance and load | Same caution | Capacity and latency validation before switching |
| **Shadow traffic / traffic mirroring** | The proxy copies requests to the new system and discards its responses | Writes must go to an isolated store, or be suppressed | Whole-service comparisons; load testing with real shape |
| **Record and replay** | Capture production requests (anonymized), replay against both offline, diff responses | None in production | Batch-style validation; regulated environments |
| **Parallel run of business processes** | Both systems process the real workload (e.g., month-end), outputs reconciled before the new one becomes authoritative | Two sets of outputs — only one may be sent onward | Finance, payroll, billing, regulatory reports |

**Scientist.NET in practice:**

```csharp
using GitHub;

public sealed class PricingExperiment(LegacyPricingEngine legacy, RulesPricingEngine candidate) : IPricingEngine
{
    public Task<Quote> QuoteAsync(Cart cart, CancellationToken ct) =>
        Scientist.ScienceAsync<Quote>("pricing-engine", experiment =>
        {
            experiment.Use(() => legacy.QuoteAsync(cart, ct));          // control: its result is returned
            experiment.Try(() => candidate.QuoteAsync(cart, ct));       // candidate: compared, never returned
            experiment.Compare((control, cand) =>
                control.Total == cand.Total && control.Lines.SequenceEqual(cand.Lines));
            experiment.AddContext("cartId", cart.Id);
            experiment.AddContext("tenant", cart.TenantId);
            experiment.RunIf(() => cart.Lines.Count <= 500);           // skip pathological carts
            experiment.Thrown((operation, ex) => { /* log candidate exceptions; never rethrow */ });
        });
}

// Publish results to telemetry — mismatches become a dashboard and a backlog.
public sealed class TelemetryResultPublisher(ILogger<TelemetryResultPublisher> log) : IResultPublisher
{
    public Task Publish<T, TClean>(Result<T, TClean> result)
    {
        if (result.Mismatched)
            log.LogWarning("Experiment {Name} mismatch. Context: {@Context}", result.ExperimentName, result.Contexts);
        return Task.CompletedTask;
    }
}
// Scientist.ResultPublisher = new TelemetryResultPublisher(logger);
```

**Operating a parallel run:**

1. **Define equivalence explicitly** — exact match, tolerance (rounding to 2 dp), or normalized comparison (ignore ordering, timestamps, IDs). Write the comparer like production code.
2. **Triage every mismatch category**: new bug (fix new), legacy bug (decide: preserve or fix — a business decision, recorded), equivalence too strict (fix comparer), non-determinism (inject clock/seed).
3. **Set a graduation criterion** before you start: e.g., *"< 0.01% mismatches over 14 days covering a month-end, all categories explained."*
4. **Mind the cost** — doubled compute and latency. Run the candidate asynchronously (`ScienceAsync` runs control and candidates concurrently) and sample (`RunIf`, a percentage) on hot paths.
5. **Never let the candidate cause side effects** — no emails, payments, messages or writes to shared stores. Wrap side-effecting dependencies in the candidate with no-op or sandbox implementations, or compare *intended* side effects (the command that would be sent) rather than performing them.
6. **Turn it off** when graduated — experiments are transitional architecture too.

**The interview-grade sentence:** *"Characterization tests cover the inputs I thought of; a parallel run covers the ones production sends. I run both implementations on real traffic, return the legacy result and compare — in process with Scientist.NET for pure computations like pricing or permissions, as shadow traffic mirrored by the proxy into an isolated store for whole services, as offline record-and-replay in regulated settings, or as a full parallel business run for things like month-end billing. The essentials are an explicit equivalence definition, triage of every mismatch category — including deciding with the business whether to preserve a legacy bug — a graduation criterion agreed up front, sampling to control cost, and an absolute rule that the candidate causes no side effects."*

---

## Concept 19 — Expand and contract (parallel change)

**Parallel change** (Danilo Sato on Fowler's bliki, 2014), also called **expand and contract**, changes an interface that has consumers you can't update atomically — an API, a message schema, a database column — in three steps, each independently deployable and reversible:

1. **Expand** — add the new form alongside the old. Producers write both; consumers can read either.
2. **Migrate** — move every consumer to the new form, one at a time, at their own pace. Backfill historical data.
3. **Contract** — once nothing uses the old form (prove it with telemetry), remove it.

**Database example — replacing `Orders.CustNo` (a legacy varchar) with `Orders.CustomerId` (a GUID foreign key):**

```sql
-- 1. EXPAND (migration 1): add the new column, nullable; no reader changes yet.
ALTER TABLE dbo.Orders ADD CustomerId uniqueidentifier NULL;

-- Application release A: writes both CustNo and CustomerId on every insert/update.
-- (Or a trigger, if the writer can't be changed yet — itself transitional.)

-- Backfill in batches to avoid long locks and log growth.
WHILE 1 = 1
BEGIN
    UPDATE TOP (5000) o
       SET o.CustomerId = c.CustomerId
      FROM dbo.Orders o
      JOIN dbo.CustomerKeyMap c ON c.CustNo = o.CustNo
     WHERE o.CustomerId IS NULL;
    IF @@ROWCOUNT = 0 BREAK;
END;

-- 2. MIGRATE: application releases B..n move readers (API, reports, other services) to CustomerId.
--    Verify: no queries reference CustNo (Query Store, code search, consumer sign-off).

-- 3. CONTRACT (migration 2, weeks later): enforce and remove.
ALTER TABLE dbo.Orders ALTER COLUMN CustomerId uniqueidentifier NOT NULL;
ALTER TABLE dbo.Orders ADD CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerId) REFERENCES dbo.Customers(CustomerId);
-- Drop indexes/constraints on CustNo, then:
ALTER TABLE dbo.Orders DROP COLUMN CustNo;
```

With **EF Core** (Module 19), each step is a separate migration in a separate release; never put expand and contract in the same deployment, and never let EF's model-diff produce a rename (`RenameColumn`) for a column other systems read.

**API example:** add `customerId` to the response while keeping `custNo` (expand); consumers switch (migrate; watch per-field usage if you can, or the `custNo`-using client IDs); remove `custNo` in the next major version or after a deprecation window (contract). Mark deprecation with OpenAPI's `deprecated: true` and the `Deprecation`/`Sunset` HTTP headers.

**Message example:** publish `OrderPlacedV2` alongside `OrderPlacedV1` (or add optional fields — tolerant readers), migrate subscribers, stop publishing V1. Module 11's schema-evolution rules apply.

**Why it matters for brownfield:** almost every migration step changes some contract that other teams depend on. Expand/contract converts "coordinate a release with six teams" into "six teams each change when ready." It is the standard answer to "how do you change a shared schema without downtime?"

**The interview-grade sentence:** *"Parallel change — expand, migrate, contract — is how I change a contract whose consumers I can't update atomically: first add the new form alongside the old, with producers writing both and a batched backfill; then move consumers one at a time at their own pace; and only when telemetry proves nothing uses the old form, enforce the new one and remove the old. Each step is a separate, reversible release — with EF Core that means separate migrations in separate deployments and never letting the model diff generate a rename — and the same pattern applies to API fields with Deprecation and Sunset headers and to message versions."*

---

## Concept 20 — The legacy-displacement catalogue

**Patterns of Legacy Displacement** (Cartwright, Horn and Lewis on martinfowler.com, 2021–2024) organizes modernization into four activities — *understand the outcomes; break the problem into smaller parts; successfully deliver the parts; change the organization so this happens continuously* — and names patterns for each. The written-up delivery patterns are worth knowing by name, because they describe moves you'll need and interviewers increasingly use the vocabulary.

| Pattern | What it is | .NET / Azure example |
|---|---|---|
| **Transitional Architecture** | Elements built to ease displacement, removed afterwards (Concept 13) | The YARP facade, CDC sync, reconciliation jobs |
| **Event Interception** | Intercept updates to system state (messages, events, calls) and route some to a new component | A worker reads the legacy MSMQ/Service Bus queue and forwards selected message types to the new service's queue — initially forwarding everything unchanged |
| **Legacy Mimic** | The new system interacts with the legacy (or its consumers) in a way that makes the legacy unaware anything changed | The new Billing service writes sale records into the legacy master database tables so legacy reports keep working; a new service exposes the old SOAP contract to a partner who can't change |
| **Divert the Flow** | Move cross-organization activities away from the legacy first — e.g., new business processes start in the new system, which then feeds the legacy as needed | New customer onboarding happens in the new portal; it creates the customer in the legacy via an ACL until the legacy customer module is retired |
| **Revert to Source** | Identify the *originating* source of data and integrate with that, instead of the legacy's copy | The new pricing service reads prices from the product information system that the legacy nightly-imported, rather than from the legacy's tables |
| **Critical Aggregator** | Data from many parts of the business is combined to support critical decisions (usually reporting); it's a hard dependency that blocks displacement | The finance dashboard that joins legacy and new data — build an aggregation that reads both, so either side can change underneath |
| **Extract Product Lines** | Identify and separate the system by product line, migrating one product line at a time | Move "subscriptions" to the new platform while "one-off orders" stay in legacy |
| **Feature Parity** | Replicate the legacy's functionality on a new stack — **usually an anti-pattern** for large systems (Concept 2) | — |
| **Stop the World cutover** | Suspend business activity while switching — acceptable only when coexistence is impossible and downtime is tolerable | A weekend cutover for a small internal system with a bounded data migration |
| **Dark Launching**, **Canary Release** | Call new back ends without using results; roll out to a subset of users (Concepts 12, 18) | Shadow traffic through YARP; targeting rollout via Feature Management |

**The worked example from the series, as a pattern sequence** (it's a good template to retell): a team replaced costly, out-of-support integration middleware between a legacy back end and a storefront. They (1) refactored an **Event Router** into the message path that forwarded everything unchanged (**Event Interception** inserted as **Transitional Architecture**, a pure refactoring); (2) built the new manager with a **Legacy Mimic** writing sale data back into the legacy master database and a translator so legacy message formats didn't leak into the new world; (3) diverted traffic by **product** — first individual product IDs, then progressively more product types; (4) discovered through archaeology that the old middleware's database fed business-critical KPI reports via the data warehouse, and added further mimics (a wire tap writing into warehouse tables) so the reports kept working; (5) decommissioned the middleware, removing licence, support and hosting costs. The authors' closing point: be explicit about the value of risk mitigation versus the cost of transitional architecture.

**The interview-grade sentence:** *"Patterns of Legacy Displacement gives modernization a shared vocabulary around four activities — understand outcomes, break the problem up, deliver the parts, change the organization — and its delivery patterns are moves I use constantly: event interception to put a router into a message flow, legacy mimics so the legacy or its reports don't notice the change, divert-the-flow to start new business processes in the new system, revert-to-source to integrate with the original data source instead of the legacy copy, and treating the critical aggregator — usually reporting — as the dependency that blocks displacement. It's also explicit that feature parity is usually an anti-pattern and that stop-the-world cutover is only for cases where coexistence is impossible."*

---
# Part E — The data problem

## Concept 21 — Why data is the hardest part

Code can be duplicated, routed, flagged and rolled back in minutes. Data can't. Most strangler fig failures and most migration incidents are about data. Five reasons:

1. **The shared database is an integration contract.** In most legacy estates the database is how systems talk: other applications, reports, ETL jobs, the data warehouse and partners read (and sometimes write) its tables directly. Every table is a public API with unknown consumers — Hyrum's law at its strongest. Sam Newman calls the shared database the most common and most troublesome form of coupling in monolith decomposition.
2. **There must be one source of truth per fact.** During coexistence, two systems may both hold a customer's address. At any moment exactly one of them must be authoritative for each piece of data, or you get lost updates and divergence.
3. **Transactions don't cross the boundary.** The legacy may update orders, stock and invoices in one ACID transaction. Split them across systems and you inherit the problems from Module 12: sagas, eventual consistency, compensations, the dual-write problem.
4. **Data has gravity.** Large data sets are slow and expensive to move; reports, ML pipelines and integrations accrete around them; latency between a new service in Azure and a database on-premises can make "just call the old DB" untenable.
5. **Moving data ownership is the true one-way door.** Once the new system has accepted writes that the old system doesn't have, rolling back means reconciling or reverse-syncing — not flipping a flag.

**Consequently, a migration plan is mostly a data plan.** For each slice, answer:

| Question | Why |
|---|---|
| Which data does this slice **read**? Which does it **write**? | Determines whether it can move without moving data ownership |
| **Who owns** each table/entity now, and who will after this slice? | One writer per fact, at every state |
| **Who else reads** these tables (reports, ETL, other apps, partners)? | They become legacy-mimic or reverse-sync requirements |
| What **consistency** do readers need? Is lag of seconds/minutes acceptable? | Chooses between synchronous access, CDC, or batch |
| What's the **rollback** story after new writes exist? | Determines whether the step is reversible |
| How will we **prove** both sides agree? | Reconciliation (Concept 25) |

**The interview-grade sentence:** *"Data is the hardest part of any migration because the shared database is usually the real integration contract, with unknown readers like reports, ETL and partners; because every fact needs exactly one source of truth at every intermediate state; because transactions don't cross the new boundary, which brings in sagas and the dual-write problem; because data has gravity; and because moving write ownership is the true one-way door — after the new system accepts writes, rollback means reconciliation, not a flag flip. So for every slice I ask what it reads and writes, who owns each fact before and after, who else reads it, what consistency readers need, what rollback looks like, and how we'll prove both sides agree."*

---

## Concept 22 — Transferring data ownership

The safest general strategy moves **reads before writes**, and moves **write ownership** for a given set of data **once, explicitly**. Think of it as a ladder; each rung is reversible except the one marked.

```text
Rung 0  New code reads the LEGACY database directly (through an ACL)           ← reversible
Rung 1  New code has its OWN store, kept in sync FROM legacy (CDC / events);    ← reversible
        new code reads its own store; legacy still owns all writes
Rung 2  New code accepts writes for a COHORT; writes go to new store,           ← reversible only with
        and are synced BACK to legacy so legacy readers keep working               reverse sync
Rung 3  New system is the SOURCE OF TRUTH for this data for everyone;           ← POINT OF NO RETURN
        legacy receives a reverse-sync copy (legacy mimic) for remaining readers   (practically)
Rung 4  Legacy readers migrated; reverse sync removed; legacy tables archived   ← decommissioning
        and dropped
```

**Rung 0 — read the legacy store in place.** Fastest start, no sync. Acceptable for a first slice or a short period. Risks: coupling to legacy schema (mitigated by the ACL), latency if the database is far away, load on the legacy database. Prefer **read-only views** created for the new service over direct table access — a view is a seam the legacy team can keep stable while they refactor tables.

**Rung 1 — own store, synchronized from legacy.** The new service builds its *own* model (not a table copy) from legacy changes via CDC or events (Concept 23). Reads are fast and decoupled; the legacy is still authoritative. This is where the read slice from Concept 9 lives, and where most sync bugs are found while the cost of a bug is still low.

**Rung 2 — cohort writes with reverse sync.** For a cohort (new tenants, a product line), the new service owns writes; changes are propagated back to the legacy store so legacy features, reports and integrations keep working (a Legacy Mimic). Two sync directions now exist — **never for the same entity at the same time**: ownership is partitioned by cohort, so for any given record exactly one side writes.

**Rung 3 — flip ownership for everyone.** The new store is authoritative for all records of this kind. Existing records are migrated (bulk copy plus incremental catch-up), writes stop on the legacy side (enforce it — revoke permissions, or put triggers that raise errors on direct writes), reverse sync keeps legacy readers fed. Rolling back now requires replaying new writes into the legacy — possible only if reverse sync is complete and tested, which is exactly why you prepare it at Rung 2.

**Rung 4 — retire.** Migrate the remaining legacy readers (reports to the new store or a warehouse feed — Revert to Source), switch off reverse sync, archive the legacy tables (read-only snapshot for audit/legal retention), then drop them.

**The Stripe sequence.** Stripe's well-known *"Online migrations at scale"* post describes the same discipline for moving a data model inside a running system in four phases: **dual-write** to old and new (inside the same database transaction where possible), **change all read paths** to the new model, **change all write paths** to write only the new model, and **remove the old data** — with verification between phases. Within a single database, writing both forms in one transaction is safe; across *two* databases it becomes the dual-write problem (Concept 23).

**The interview-grade sentence:** *"I move data ownership up a ladder: first the new code reads the legacy store through an anti-corruption layer, ideally via views; then it gets its own model, synchronized from the legacy by CDC while the legacy still owns writes; then it owns writes for a cohort, with a reverse sync feeding legacy readers and ownership partitioned so each record has exactly one writer; then it becomes the source of truth for everyone, which is the practical point of no return, so reverse sync must already be proven; and finally legacy readers move, sync stops and the old tables are archived and dropped. Reads before writes, one writer per record at every state."*

---

## Concept 23 — Synchronization during coexistence

During coexistence data must flow between old and new. How you move it determines whether the two sides stay correct.

**Never naive dual writes across stores.** "The new service writes to its database and then to the legacy database" (or the reverse) fails exactly like Module 11's dual-write problem: a crash, timeout or deadlock between the two writes leaves them inconsistent, with no record of what was missed. Use one of these instead:

| Mechanism | How it works | Latency | Strengths | Weaknesses |
|---|---|---|---|---|
| **Transactional outbox** (Modules 11, 12) | The writer records an event in an outbox table in the *same* transaction; a relay publishes it | Seconds | Exactly the business events you want; works when you can change the writer | Requires changing the writer — often impossible in the legacy |
| **SQL Server CDC** | Reads the transaction log into change tables; consumers poll `cdc.fn_cdc_get_all_changes_*` | Seconds–minutes | No change to legacy code; captures every change with before/after values and LSN ordering | Row-level, not business-level, events; operational overhead (capture/cleanup jobs, retention); schema changes need care |
| **SQL Server change tracking** | Records *that* a row changed (not the values); consumers query by version | Seconds–minutes | Lightweight; good for "what changed since version N" sync | No intermediate values; consumer must read current row |
| **Debezium** (Kafka Connect, or Debezium Server → Event Hubs/Service Bus) | Streams CDC tables (SQL Server) or logs (PostgreSQL, MySQL) as events | Sub-second–seconds | Standard, multi-database; snapshot + streaming; schema history | Another platform to run; still row-level events |
| **SQL Server 2025 / Azure SQL change event streaming** | The engine pushes row changes as CloudEvents to Event Hubs / Fabric Eventstream | Near real-time | No polling infrastructure; native | **Preview** as of October 2026; plan accordingly |
| **Cosmos DB change feed** | Ordered per-partition feed of changes | Seconds | Native, easy with the change feed processor | Deletes need soft-delete or TTL patterns |
| **Triggers writing to a staging/outbox table** | Legacy table triggers record changes | Synchronous | Works on ancient databases | Adds latency and failure modes to legacy writes; triggers are hidden logic |
| **Batch ETL / snapshots** | Periodic bulk copy (ADF, SSIS, bcp) | Hours | Simple for reporting and initial loads | Too stale for operational sync |

**Turning row changes into domain events.** CDC gives you *"row 4711 in `ORD_HDR` changed STAT_CD from 'P' to 'S'"*. The new service wants *"OrderShipped"*. Put the translation in an ACL consumer (Concept 15): it reads raw changes, groups them by transaction (CDC exposes the LSN), interprets them, and emits or applies domain-level changes. Keep raw-change consumers away from your domain.

**Rules for a correct sync consumer:**

1. **Idempotent** — redelivery is guaranteed somewhere; applying the same change twice must be harmless.
2. **Ordered per key** — process changes for the same entity in source order; use the LSN or a source version, and ignore stale changes.
3. **Exactly one writer per record** (Concept 22) — sync must never fight a local writer for the same record.
4. **Loop prevention** — with sync in both directions (for different cohorts), mark records by origin and never echo a change back to where it came from.
5. **Schema-change aware** — legacy schema changes break consumers; enrol the legacy team in the contract (expand/contract on their side too).
6. **Observable** — lag (oldest unprocessed change age), throughput, error/poison count, per-table counts.
7. **Replayable** — you will need to rebuild the new store from scratch at least once; design for a full re-snapshot plus catch-up.

```csharp
// Applying legacy CDC changes to the new Orders store: idempotent, ordered, one-writer aware.
public sealed class LegacyOrderChangeApplier(OrdersDbContext db, ILogger<LegacyOrderChangeApplier> log)
{
    public async Task ApplyAsync(LegacyOrderChange change, CancellationToken ct)
    {
        var order = await db.Orders.SingleOrDefaultAsync(o => o.LegacyOrderNo == change.OrderNo, ct);

        if (order is { Ownership: DataOwner.New })
        {
            // This record's writes moved to the new system (Rung 2/3) — legacy changes are echoes or bugs.
            log.LogWarning("Ignoring legacy change for new-owned order {OrderNo} at LSN {Lsn}", change.OrderNo, change.Lsn);
            return;
        }

        if (order is not null && order.SourceLsn.CompareTo(change.Lsn) >= 0)
            return;                                                     // stale or duplicate: idempotent no-op

        order ??= db.Orders.Add(Order.FromLegacy(change.OrderNo, DataOwner.Legacy)).Entity;
        LegacyOrderTranslator.Apply(change, order);                     // the ACL: codes → domain states
        order.SourceLsn = change.Lsn;

        await db.SaveChangesAsync(ct);                                  // optimistic concurrency token on Order
    }
}
```

**The interview-grade sentence:** *"During coexistence I never let code write to two stores in sequence — that's the dual-write problem. If I can change the writer I use a transactional outbox; if I can't, I capture changes from the legacy database with SQL Server CDC or change tracking, Debezium, or — once it's out of preview — SQL Server 2025's change event streaming to Event Hubs, and translate row changes into domain events in an anti-corruption consumer. The consumer must be idempotent, ordered per key by LSN or version, never fight the one writer that owns a record, prevent echo loops when sync runs both ways for different cohorts, survive legacy schema changes, expose lag as a metric and be able to rebuild from a fresh snapshot."*

---

## Concept 24 — Splitting a shared database

When a capability moves out of a monolith, its tables usually live in a database that other code also reads and writes. Sam Newman's *Monolith to Microservices* catalogues the moves; the most useful ones, translated to SQL Server/.NET:

| Pattern | Move | When |
|---|---|---|
| **Database view** | Other consumers read a view that projects the old shape; the owner can refactor underlying tables | First step to hide internals from readers |
| **Database wrapping service** | Put a service in front of a set of tables; consumers move from SQL to the API; the tables are then private | Many consumers you can't migrate at once; logic in stored procedures |
| **Schema per owner** | Move tables into a schema owned by one module (`billing.*`), grant others access only to views | Modular monolith step; precursor to physical split (Module 21) |
| **Split table** | A table holding data for two contexts (e.g., `Customer` with billing *and* marketing columns) becomes two tables owned separately | Mixed ownership in one table |
| **Move foreign-key relationship to code** | Cross-context FK constraints are dropped; integrity is maintained by the owning services and verified by reconciliation | Before physically separating databases |
| **Replace cross-schema joins** | Reports and queries joining both sides move to an API composition or a read model / warehouse | Before separation |
| **Synonyms during moves** | A SQL Server `SYNONYM` keeps old object names resolving while objects move to a new schema or database | Short-lived bridge for legacy code you can't change yet |
| **Database-per-service** | Physical separation (separate Azure SQL databases, different engine if justified) | After ownership, views and joins are sorted |

**A typical sequence for extracting Billing from a shared `Commerce` database:**

1. **Inventory** — every object touching billing tables: stored procedures, views, jobs, SSIS packages, reports, linked servers, external readers (SQL Audit / Extended Events for a couple of weeks catches the ones nobody remembers).
2. **Schema move** — `dbo.Invoice*` → `billing.*`; `SYNONYM`s keep old names working for legacy code during the transition.
3. **Expose views** for non-owner readers; revoke their direct table access. Unknown readers now fail loudly in test (and you find them).
4. **Remove cross-context FKs and joins** — replace with IDs, events and read models; add reconciliation.
5. **Make the owner the only writer** — a database role per module; write permissions only for the owning service's identity (with Entra ID / managed identities on Azure SQL, Module 29).
6. **Physically split** — new Azure SQL database for Billing; migrate data (bulk + CDC catch-up, Concept 25); point the Billing service at it; the views in the old database become a legacy mimic fed by reverse sync until readers move.
7. **Drop synonyms, views and reverse sync** when readers have moved.

**Stored procedures with business logic** are the brownfield .NET classic. Options: leave them as the implementation behind a wrapping service (fine for a while); characterization-test them and port logic into C# behind branch by abstraction; or, for set-based logic that's genuinely best in SQL, keep it — but owned by one service. Don't port 3,000 lines of T-SQL as a side effect of a hosting migration.

**EF Core against legacy schemas** (Module 19): scaffold with `dotnet ef dbcontext scaffold` to start, then *own* the model (don't re-scaffold over hand edits); map to views and keyless entity types for read models; use `HasDefaultSchema("billing")` per module context; and remember that EF can't express what the legacy assumed implicitly (triggers, defaults set by procedures) — characterization tests catch these.

**The interview-grade sentence:** *"To split a shared database I follow Newman's moves: inventory every object and reader, including the ones only SQL Audit reveals; move the capability's tables into a schema it owns, with synonyms as a short-lived bridge; give other readers views and revoke their direct table access so unknown consumers surface; replace cross-context foreign keys and joins with IDs, events, read models and reconciliation; make the owning service's managed identity the only writer; then physically split with bulk copy plus CDC catch-up, keeping the old views alive as a legacy mimic until readers move. Business logic in stored procedures I wrap first, and port only deliberately with characterization tests."*

---

## Concept 25 — Cutover, rollback and the point of no return

Each slice ends with a **cutover**: the moment the new path becomes authoritative for some traffic or data. Code cutovers are cheap to reverse; data cutovers need preparation.

**Data migration for a cutover — the standard shape:**

1. **Bulk load** — copy the historical data (snapshot) into the new store, transforming through the ACL. For large tables, partition the work and make it restartable.
2. **Incremental catch-up** — CDC from a recorded LSN/version (taken at snapshot time) applies everything that changed during the bulk load.
3. **Verify** — reconciliation (below) shows the stores agree within tolerance; lag is near zero.
4. **Flip** — stop legacy writes for the scope (permissions or a write-block), drain the last changes, switch the flag/route so writes go to the new system, start reverse sync if legacy readers remain.
5. **Watch** — heightened monitoring for the soak period; reconciliation runs frequently.

Most cutovers can be done with **minutes or seconds of write freeze** for the scope — sometimes zero, if ownership is partitioned by cohort (Rung 2).

**Rollback planning — before the cutover, not during:**

| Situation | Rollback |
|---|---|
| Code/routing only, no ownership change | Flip the flag/route back. Practice it |
| New system owns writes; reverse sync is complete | Stop new writes; let reverse sync drain; flip routing back; legacy is current |
| New system owns writes; reverse sync incomplete or broken | Data reconciliation and replay of new writes into the legacy — slow, risky, possibly manual. **Avoid being here** |
| Destructive step done (legacy tables dropped) | Restore from archive and replay — effectively no rollback |

**Name the point of no return.** Every migration has at least one step after which rollback is impractical — usually the moment ownership flips without reverse sync, or the legacy is switched off. Put it in the plan explicitly, with **go/no-go criteria** agreed in advance (reconciliation clean for N days, error rates within baseline, support volume normal, business sign-off), and cross it deliberately. This is the Module 30 "one-way door" made concrete.

**Reconciliation** is how you *prove* the two sides agree — before cutover, during coexistence, and before decommissioning:

| Level | Check | Example |
|---|---|---|
| **Counts** | Row/entity counts per scope and period | Orders per day per tenant on both sides |
| **Aggregates** | Sums, min/max, checksums per partition | Sum of invoice totals per month; `CHECKSUM_AGG(BINARY_CHECKSUM(...))` per key range |
| **Record-level diff** | Field-by-field comparison through the translator | Sample 1% daily; 100% for money before cutover |
| **Business invariants** | Rules that must hold across systems | Every shipped order has an invoice |
| **Flow balance** | In = out over a period | Payments received = payments applied + unapplied |

Run reconciliation as a scheduled job, publish results to a dashboard, alert on breaks, and keep a triaged **break log** (cause: sync lag, translation bug, legacy bug, real divergence). Reconciliation is transitional architecture — and the evidence for decommissioning.

```csharp
// A daily reconciliation slice: counts and totals per tenant per day, both sides, through the same rules.
public sealed record ReconRow(string TenantId, DateOnly Day, int Orders, decimal Total);

public sealed class OrderReconciliation(ILegacyOrderTotals legacy, INewOrderTotals modern, IReconSink sink)
{
    public async Task RunAsync(DateOnly day, CancellationToken ct)
    {
        var a = (await legacy.TotalsAsync(day, ct)).ToDictionary(r => r.TenantId);
        var b = (await modern.TotalsAsync(day, ct)).ToDictionary(r => r.TenantId);

        foreach (var tenant in a.Keys.Union(b.Keys))
        {
            a.TryGetValue(tenant, out var l);
            b.TryGetValue(tenant, out var n);
            bool ok = l is not null && n is not null && l.Orders == n.Orders && Math.Abs(l.Total - n.Total) <= 0.01m;
            await sink.RecordAsync(day, tenant, l, n, ok ? ReconStatus.Matched : ReconStatus.Break, ct);
        }
    }
}
```

**The interview-grade sentence:** *"A data cutover is bulk load from a snapshot, CDC catch-up from the LSN taken at snapshot time, reconciliation showing the stores agree, a brief write freeze for the scope, the flip, and a soak period — often with no freeze at all if ownership is partitioned by cohort. I plan rollback before cutover: routing changes flip back; once the new side owns writes, rollback is only cheap if reverse sync is complete and tested. Every migration has a point of no return, so I name it in the plan with go/no-go criteria agreed in advance, and I prove agreement continuously with reconciliation at several levels — counts, aggregates, sampled record diffs, business invariants and flow balances — with a triaged break log that later becomes the evidence for decommissioning."*

---
# Part F — .NET and Azure specifics

## Concept 26 — .NET Framework → modern .NET: the landscape

Many brownfield .NET estates are some mix of **.NET Framework 4.x** (ASP.NET MVC 5/Web API 2/Web Forms, WCF, Windows services) and **early .NET Core / .NET 5–8**. Two clocks drive the urgency:

| Runtime | Status (October 7, 2026) |
|---|---|
| .NET 10 (LTS) | Current; supported to November 2028 |
| .NET 9 (STS, now 24 months) | **Ends November 10, 2026** |
| .NET 8 (LTS) | **Ends November 10, 2026** |
| .NET 5, 6, 7, .NET Core 3.1 and earlier | Out of support |
| .NET Framework 4.7 – 4.8.1 | Supported as a Windows component; no announced end date; no new features |
| .NET Framework 4.6.2 | **Ends January 12, 2027** |
| .NET Framework 3.5 SP1 | Ends January 9, 2029 |

**Two different problems hide under ".NET upgrade":**

1. **Modern .NET → modern .NET** (8/9 → 10): usually a retarget plus breaking-change review — days to weeks per solution. Unsupported runtimes in production after November 10 is a security finding; this is the *urgent* problem for many teams right now, and should be routine (Module 33: treat runtime upgrades as recurring maintenance, not projects).
2. **.NET Framework → modern .NET**: a genuine migration, because some technologies **did not come across**:

| Not in modern .NET | Path |
|---|---|
| **ASP.NET Web Forms** | Rewrite UI (Blazor is the closest model; Copilot upgrade has a Web Forms → Blazor scenario); strangle page by page behind the YARP facade |
| **ASP.NET MVC 5 / Web API 2** (System.Web) | ASP.NET Core MVC/Minimal APIs — mostly mechanical for controllers; pipeline, DI, config, auth, session differ (Concept 27) |
| **WCF server** | **CoreWCF** (community, Microsoft-supported; 1.9 current) to move the service as-is; or gRPC / HTTP APIs to redesign the contract (Concept 30). The WCF *client* is available on modern .NET |
| **Windows Workflow Foundation (WF)** | Community port CoreWF, or redesign (durable functions, Logic Apps, a workflow engine, or plain code) |
| **.NET Remoting** | Redesign: gRPC, HTTP, messaging |
| **AppDomains** (beyond the default) | `AssemblyLoadContext` for plugin isolation; processes/containers for security isolation |
| **Code Access Security, partial trust** | OS/container boundaries |
| **System.EnterpriseServices (COM+)** | No direct replacement — redesign transactions (outbox, sagas) and component hosting |
| **`System.Web` everywhere in class libraries** | System.Web adapters (Concept 27), then refactor to `HttpContext`-free logic |
| **Windows-only APIs** (registry, WMI, EventLog, `System.Drawing` on non-Windows) | Windows Compatibility Pack keeps them on Windows; abstractions to go cross-platform |
| **`app.config`/`web.config` and `ConfigurationManager`** | `Microsoft.Extensions.Configuration` (a compat package exists for `ConfigurationManager`) |

**The dependency graph decides the order.** Applications depend on libraries, libraries on packages. The general sequence (also what the upgrade agent calls a **bottom-up** strategy):

1. **Inventory**: projects, target frameworks, NuGet packages (and whether each has a modern-.NET version or a replacement), P/Invoke and COM usage, Windows-only APIs.
2. **Convert projects to SDK-style** while still targeting .NET Framework — mechanical, verifiable, no runtime change.
3. **Move shared libraries to `netstandard2.0`** where possible, or **multi-target** (`<TargetFrameworks>net48;net10.0</TargetFrameworks>`), so both the old app and the new app can consume them during coexistence. Use `#if NET` sparingly and isolate differences behind interfaces.
4. **Upgrade leaf libraries first**, then middle layers, then applications — or strangle the application (Concept 27) while libraries are dual-targeted.
5. **Replace or isolate unsupported dependencies** (an old PDF library, a COM component) behind seams, so they don't block the rest.

**Platform target vs architecture target.** Porting to .NET 10 is a *platform* migration; it can and usually should be done *without* changing the architecture at the same time. Combining "move to .NET 10" with "split into microservices" multiplies risk. Get onto a supported runtime first (or concurrently via strangler), then evolve the architecture with the better tools that platform gives you.

**The interview-grade sentence:** *"In .NET brownfield work two clocks matter right now: .NET 8 and 9 both leave support on November 10, 2026, so moving to .NET 10 LTS is urgent but should be routine, and .NET Framework 4.6.2 ends in January 2027 while 4.7–4.8.1 live on as Windows components with no new features. Framework-to-modern-.NET is a real migration because Web Forms, WCF server, WF, Remoting, extra AppDomains and System.Web didn't come across, so I inventory dependencies, convert to SDK-style projects first, move shared libraries to netstandard2.0 or multi-target them so old and new apps can share them during coexistence, upgrade bottom-up or strangle the app top-down, isolate blockers behind seams — and I never combine the platform migration with an architecture change in the same step."*

---

## Concept 27 — Incremental ASP.NET → ASP.NET Core with YARP and System.Web adapters

This is Microsoft's productized strangler fig for web apps, and the canonical .NET brownfield answer.

**The shape:**

```text
                      ┌──────────────────────────────────────────┐
   browser ──HTTPS──▶ │ ASP.NET Core app (.NET 10)                │
                      │  • migrated routes handled locally         │
                      │  • System.Web adapters for shared libs     │
                      │  • remote session / remote auth clients    │
                      │  • fallback: MapForwarder / MapRemoteApp-  │
                      │    Fallback → YARP → Framework app         │
                      └───────────────┬──────────────────────────┘
                                      │ unmatched routes (YARP)
                                      │ + session/auth API calls (API key)
                                      ▼
                      ┌──────────────────────────────────────────┐
                      │ ASP.NET Framework app (net48)             │
                      │  • SystemWebAdapterModule                 │
                      │  • AddRemoteAppServer (session/auth APIs) │
                      │  • reachable only from the Core app        │
                      └──────────────────────────────────────────┘
```

**How it works:**

1. A **new ASP.NET Core app becomes the entry point.** It handles routes that have been migrated and **forwards everything else via YARP** to the Framework app.
2. **System.Web adapters** (`Microsoft.AspNetCore.SystemWebAdapters`) implement the *shape* of `System.Web.HttpContext` on top of ASP.NET Core's `HttpContext`. Shared class libraries that use `HttpContext.Current`, `HttpRequest`, `HttpResponse` can be compiled for `netstandard2.0`/multi-target and run in *both* apps unchanged — which removes the biggest blocker to moving controllers.
3. **Remote app integration**: the Framework app exposes endpoints (protected by a shared API key) so the Core app can use **remote session** (read/write the Framework app's session state during requests — sessions work very differently in the two stacks) and **remote authentication** (let the Framework app authenticate the user, e.g., Forms auth / OWIN cookies, until auth is migrated).
4. **Move routes one at a time** (controller by controller, page by page). When the Core app owns everything, remove the fallback, the remote session and auth, then the adapters themselves — the final cleanup is replacing adapter usage with native ASP.NET Core APIs.

**Configuration (current docs, SystemWebAdapters 2.3):**

```csharp
// Framework app — Global.asax.cs (package: Microsoft.AspNetCore.SystemWebAdapters.FrameworkServices)
protected void Application_Start()
{
    HttpApplicationHost.RegisterHost(builder =>
    {
        builder.Services.AddSystemWebAdapters()
            .AddRemoteAppServer(options =>
            {
                // A GUID-formatted key shared by both apps
                options.ApiKey = System.Configuration.ConfigurationManager.AppSettings["RemoteAppApiKey"];
            });
            // .AddSessionServer() / .AddAuthenticationServer() to enable remote session / auth
    });
    // ...existing startup code...
}
```

```csharp
// Core app — Program.cs (packages: Microsoft.AspNetCore.SystemWebAdapters.CoreServices, Yarp.ReverseProxy)
using Microsoft.AspNetCore.SystemWebAdapters;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSystemWebAdapters()
    .AddRemoteAppClient(options =>
    {
        options.RemoteAppUrl = new(builder.Configuration["RemoteAppUri"]!);
        options.ApiKey = builder.Configuration["RemoteAppApiKey"]!;
    })
    .AddSessionClient();                 // remote session; add .AddAuthenticationClient(true) for remote auth

builder.Services.AddReverseProxy();
builder.Services.AddControllersWithViews();

var app = builder.Build();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseSystemWebAdapters();              // after routing

app.MapDefaultControllerRoute();         // migrated controllers handled here

// Everything unmatched goes to the Framework app — lowest priority, skip remaining middleware.
app.MapForwarder("/{**catch-all}",
        app.Services.GetRequiredService<IOptions<RemoteAppClientOptions>>().Value.RemoteAppUrl.OriginalString)
    .WithOrder(int.MaxValue)
    .ShortCircuit();

app.Run();
```

**With Aspire (preview integration):** the AppHost runs the Framework app under IIS Express and wires the fallback, session and auth configuration automatically:

```csharp
// AppHost (package: Aspire.Hosting.IncrementalMigration — prerelease)
var builder = DistributedApplication.CreateBuilder(args);

var frameworkApp = builder.AddIISExpressProject<Projects.LegacyWeb>("framework");

builder.AddProject<Projects.ModernWeb>("core")
    .WithIncrementalMigrationFallback(frameworkApp, options =>
    {
        options.RemoteSession = RemoteSession.Enabled;
        options.RemoteAuthentication = RemoteAuthentication.DefaultScheme;
    });

builder.Build().Run();
// In the Core app: builder.AddSystemWebAdapters(); … app.MapRemoteAppFallback().ShortCircuit();
```

A multi-targeted ServiceDefaults project (`net481;net10.0`) gives *both* apps the same OpenTelemetry setup — one trace across the Core app, the forwarder and the Framework app, which is exactly the observability Concept 8 asks for.

**Practical constraints and gotchas:**

| Issue | Note |
|---|---|
| **Identical virtual directory layout** | Framework and Core apps must use the same virtual directory structure (route generation, auth) |
| **Lock down the Framework app** | It should be reachable only from the Core app (network rules / private endpoints), over TLS |
| **Session is the hardest coupling** | Remote session adds a round trip and serialization of session values (register the types); plan to eliminate session dependence slice by slice |
| **Auth migration** | Start with remote auth; migrate to ASP.NET Core auth (cookies with a shared Data Protection key ring, or OIDC with Entra ID, Module 29) as an explicit slice |
| **Static content and bundling** | Decide where static files are served from at each stage; caching headers must stay consistent |
| **Hosting** | Core app on App Service/Container Apps; Framework app on App Service (Windows), **Managed Instance** if it needs COM/registry/MSI, or a VM — Concept 29 |
| **Adapters are transitional** | The end state is native ASP.NET Core APIs; track adapter usage as debt |

Jimmy Bogard's series *Tales from the .NET Migration Trenches* is the best field report on running this pattern on a decade-old codebase — including cataloguing, shared libraries, session and authentication.

**The interview-grade sentence:** *"For ASP.NET Framework apps I'd use Microsoft's productized strangler fig: a new ASP.NET Core app on .NET 10 becomes the front door, handles migrated routes and forwards everything else through YARP with a lowest-priority, short-circuited fallback route; System.Web adapters let shared libraries that still use HttpContext.Current run in both apps; and remote session and remote authentication — the Framework app exposing them over an API-key-protected channel — bridge the two stacks until those concerns are migrated as their own slices. Aspire's incremental-migration integration, still in preview, wires this up locally with one trace across both apps. The Framework app is locked down behind the Core app, and the adapters themselves are transitional debt to remove at the end."*

---

## Concept 28 — Porting libraries and the tooling

**Bridging techniques for coexistence:**

| Technique | Use |
|---|---|
| **`netstandard2.0` libraries** | One binary consumable by .NET Framework 4.6.1+ (4.7.2+ recommended) and modern .NET — ideal for domain logic and DTOs shared during coexistence |
| **Multi-targeting** (`<TargetFrameworks>net48;net10.0</TargetFrameworks>`) | When a library needs different implementations per runtime; keep `#if NET` blocks small and behind interfaces |
| **SDK-style projects** | Prerequisite for both; enables Central Package Management, modern tooling, analyzers |
| **Windows Compatibility Pack** | Keeps Windows-only APIs (registry, EventLog, `System.Drawing` on Windows) available on modern .NET to unblock porting; isolate later |
| **`Microsoft.Data.SqlClient`** | Replace `System.Data.SqlClient` (deprecated) — a common, mechanical step with subtle defaults (`Encrypt=true` by default since 4.0) |
| **Analyzers / build warnings** | Platform compatibility analyzer (CA1416) flags Windows-only API use; obsolete-API warnings (SYSLIB) show what to replace |

**Tooling in October 2026:**

| Tool | What it does | Notes |
|---|---|---|
| **GitHub Copilot upgrade** (upgrade agent) | Assess → plan → execute upgrades: .NET Framework or older .NET → .NET 8+ (or Framework → 4.8.1), SDK-style conversion, Newtonsoft → System.Text.Json, SqlClient, Functions isolated worker, **Web Forms → Blazor**, add/upgrade Aspire; chooses bottom-up / top-down / all-at-once strategies | VS (`@Modernize` / "Modernize"), VS Code and Copilot CLI (`@upgrade`), GitHub.com; state in `.github/upgrades/{scenarioId}/` (`assessment.md`, `plan.md`, `tasks.md`); a commit per step; requires a GitHub Copilot subscription |
| **GitHub Copilot modernization** | Azure-migration scenarios: managed identity / Entra ID, storage, messaging, database, deployment | Formerly "app modernization"; the upgrade and Azure-migration agents are now documented separately |
| **.NET Upgrade Assistant** | The previous wizard-style tool | **Deprecated**; you'll still meet it in older guides and blog posts |
| **AWS Transform for .NET** | Agentic porting of .NET Framework to Linux-ready cross-platform .NET; builds, runs tests, produces Linux-readiness report | GA since May 2025; also available in Visual Studio |
| **CoreWCF tooling / `dotnet-svcutil`** | Move WCF services; generate clients | Concept 30 |
| **EF Core Power Tools / `dotnet ef dbcontext scaffold`** | Reverse-engineer legacy schemas into EF Core models | Own the model after the first scaffold |
| **Azure Migrate** | Discovery, assessment and migration planning for servers, databases and web apps | Portfolio-level inventory and rehost/replatform |

**Using the AI agents well (and safely):**

1. **Branch, commit per step, review every diff** — the agents already work this way; keep it.
2. **Put the safety net first** — characterization tests and CI must exist *before* the agent runs; "it builds" is not "it behaves the same".
3. **Use the plan as a design artifact** — edit `plan.md` to impose your order (e.g., shared libraries to `netstandard2.0` first) and review it like a design doc.
4. **Watch the semantic traps** that compile cleanly: serializer differences (Newtonsoft vs System.Text.Json: casing, number handling, polymorphism, reference loops), culture and globalization (ICU on Linux), `SqlClient` encryption defaults, `DateTime` kinds, default `HttpClient` behaviours, configuration precedence.
5. **Keep platform and architecture changes separate** — let the agent port; make architecture decisions yourself, in ADRs.

**The interview-grade sentence:** *"During coexistence I bridge with netstandard2.0 libraries and multi-targeting so the old and new apps can share code, after converting projects to SDK-style, with the Windows Compatibility Pack and the platform-compatibility analyzer to unblock and then track Windows-only APIs. For the porting itself the .NET Upgrade Assistant is deprecated; I'd use the GitHub Copilot upgrade agent — assessment, an editable plan, task-by-task commits on a branch, with a Web Forms-to-Blazor scenario — or AWS Transform for Linux targets. But I put characterization tests and CI in place first, review every diff, and watch for things that compile but behave differently: serializer defaults, culture on Linux, SqlClient encryption defaults and DateTime kinds."*

---

## Concept 29 — Rehost, replatform, refactor: the Rs on Azure

Cloud migration frameworks (Gartner's original 5 Rs, AWS's 7 Rs, Azure's Cloud Adoption Framework) classify what to do with each workload. For a .NET architect, the useful version:

| Strategy | Meaning | .NET/Azure example | Effort | When |
|---|---|---|---|---|
| **Retire** | Switch it off | Unused admin app found by the usage heatmap | Lowest | Always check first — typically a meaningful fraction of a portfolio |
| **Retain** | Leave as is (for now) | A stable internal tool on-premises with no driver to move | None | No outcome justifies the cost |
| **Rehost** ("lift and shift") | Same app, new infrastructure | IIS VM → Azure VM; or App Service via the migration assistant with no code change | Low | Data-centre exit deadline; buy time |
| **Replatform** ("lift and improve") | Small changes to use managed services | ASP.NET 4.8 app → App Service (Windows) or **Managed Instance on App Service** (COM/registry/MSI); SQL Server → Azure SQL Managed Instance; secrets → Key Vault; managed identity | Low–medium | Most legacy web apps as a first step |
| **Refactor** | Change code to fit the platform without changing the architecture fundamentally | Port to .NET 10 and containerize on Container Apps; replace local file storage with Blob Storage | Medium | Supported runtime, better scaling/cost |
| **Rearchitect** | Change the architecture | Strangle the monolith into a modular monolith plus a few services with Service Bus | High | Cost-of-change or scale outcomes demand it |
| **Rebuild / Replace** | New code, or a SaaS product | Replace a homegrown CRM with Dynamics 365; replace BizTalk flows with Logic Apps | Highest / varies | The capability isn't differentiating (Module 33), or the platform is at end of life |

**Combine them over time.** A sound default for a legacy .NET web app: **replatform first** (get onto managed infrastructure, managed identity, Key Vault, Application Insights — cheap, low-risk, immediate operational wins), then **refactor/rearchitect incrementally** with the strangler fig, driven by outcomes. From 2023 to 2025 Microsoft packaged exactly this two-step as the **Reliable Web App** pattern (replatform with minimal code changes plus reliability, security and observability improvements) and the **Modern Web App** pattern (refactor by decoupling high-demand parts into services with the strangler fig, Container Apps and Service Bus), around a fictional *Relecloud* app. That guidance has since been **retired** — the pages redirect to the App Service baseline architecture and the reference repositories are archived — but the two-step reasoning is still sound, and interviewers who used those patterns will recognize it. Today's equivalents are the App Service **baseline architecture** for the replatform target and the **Strangler Fig** pattern page for the incremental refactor.

**Managed Instance on Azure App Service** (GA August 2026) changes the rehost/replatform calculus for awkward .NET Framework apps: apps that depend on COM components, registry keys, MSI-installed software, custom fonts or drive mappings can run on App Service — with managed patching, scaling, diagnostics and managed identity — via plan-level install scripts, registry adapters and storage mounts. Constraints worth knowing: Windows web apps only (no WebJobs, no TCP/named pipes), Premium v4 plans, select regions, and Entra ID / managed identity rather than domain join, NTLM or Kerberos — so apps relying on Windows integrated authentication against AD still need work.

**Choose per workload, not per portfolio.** A portfolio of 60 apps will typically end up with a mix: a handful retired, many replatformed, a few rearchitected — the ones where the business outcome is.

**The interview-grade sentence:** *"I classify each workload with the Rs — retire, retain, rehost, replatform, refactor, rearchitect, rebuild or replace — always checking retire first, because usage data usually finds apps nobody needs. For a legacy .NET web app my default is to replatform first — App Service on the baseline architecture or, since August 2026, Managed Instance on App Service for apps with COM, registry or MSI dependencies, plus Azure SQL, Key Vault, managed identity and Application Insights — and then refactor incrementally with the strangler fig, only where an outcome like cost of change or scale justifies it. That's the two-step Microsoft used to package as the Reliable and Modern Web App patterns; the guidance has been retired, but the reasoning holds."*

---

## Concept 30 — The hard Windows-era technologies

These are the components that stall .NET migrations. Know the options for each.

| Technology | Options | Considerations |
|---|---|---|
| **WCF services** | (a) **CoreWCF** — port the service with minimal change (BasicHttp, NetTcp, WS-*, many bindings; 1.9 current; 1.8 support ends October 24, 2026); (b) **gRPC** — redesign with strongly typed contracts (good for internal service-to-service); (c) **HTTP APIs** (Minimal APIs + OpenAPI) — redesign for broad consumers | Who are the clients? External partners who can't change → CoreWCF or a **legacy mimic** that speaks SOAP; internal → redesign. Expand/contract: run new endpoint beside old, migrate clients, retire. WS-Security/transactions/duplex need special attention |
| **ASP.NET Web Forms** | Page-by-page rewrite to Blazor (or Razor Pages/MVC) behind the YARP facade; Copilot upgrade's Web Forms → Blazor scenario for a first pass | ViewState, postbacks, server controls and code-behind logic don't translate mechanically; extract business logic from code-behind into testable services *first* (sprout classes), then rewrite UI |
| **Windows Workflow Foundation** | CoreWF (community), or redesign: Durable Functions, Logic Apps, a saga/orchestration library, or plain code | Long-running persisted instances must drain or be migrated |
| **.NET Remoting** | gRPC, HTTP or messaging | Usually also a chance to remove chatty, object-oriented remote calls |
| **MSMQ** | Azure Service Bus (queues, sessions, DLQ, transactions within the namespace); NServiceBus/MassTransit/Rebus transports (licensing, Module 11); Event Interception router to bridge | MSMQ transactional semantics with local DTC transactions don't carry over; move to outbox + idempotent consumers |
| **Distributed transactions (MSDTC, `TransactionScope` across resources)** | Outbox + sagas (Modules 11–12); Azure SQL elastic transactions only within narrow scope | Redesign; don't try to replicate 2PC in the cloud |
| **BizTalk Server** | **Azure Logic Apps** (the named successor) + Service Bus + API Management + Functions; Microsoft's migration guidance and tooling for maps/schemas | BizTalk 2020 is the final version; mainstream support ends April 11, 2028, end of support April 10, 2030; 2016 is already out of mainstream. Migrate flow by flow — each orchestration is a natural slice |
| **Windows services** (`ServiceBase`) | .NET Worker Service (`BackgroundService`) on Windows Service hosting, or in containers on Container Apps; scheduled ones → Container Apps jobs or Functions timer triggers | Watch for local file system, machine-level config, service accounts → managed identity |
| **Scheduled tasks / SQL Agent jobs** | Container Apps jobs, Functions timers, Elastic Jobs for Azure SQL, Logic Apps | Inventory them all — they're the classic "forgotten consumer" |
| **SSIS packages** | Azure Data Factory (with SSIS integration runtime as a lift-and-shift step), Fabric Data Factory, or code | Often contain business rules — characterize before replacing |
| **COM / ActiveX / native DLLs** | Managed Instance on App Service or VMs (rehost); wrap behind a service with an ACL; replace | Usually the last thing to go; isolate early so it blocks nothing else |
| **Crystal Reports / SSRS** | Power BI paginated reports, SSRS on a VM, a reporting service | Reports are critical aggregators (Concept 20) — migrate their *data sources* deliberately |
| **Windows authentication (NTLM/Kerberos, AD groups)** | Microsoft Entra ID with OIDC; group claims; Entra Application Proxy as a bridge for on-prem apps | Identity is its own slice; can't be done implicitly during a hosting move (Module 29) |

**The pattern behind the table:** isolate each hard technology behind a seam early (so it doesn't block everything else), then choose per item between *port as-is* (CoreWCF, Managed Instance), *bridge* (event router, legacy mimic, remote session) and *redesign* (gRPC, Service Bus, Logic Apps) based on who depends on it and what outcome you're pursuing.

**The interview-grade sentence:** *"The technologies that stall .NET migrations are WCF, Web Forms, WF, Remoting, MSMQ and DTC transactions, BizTalk, Windows services and scheduled jobs, SSIS, COM components, and Windows authentication. My approach is to isolate each behind a seam early so it blocks nothing else, then decide per item between porting as-is — CoreWCF for WCF services with external clients, Managed Instance on App Service for COM — bridging with event routers, legacy mimics or remote session, and redesigning: gRPC or HTTP APIs, Service Bus with outbox and idempotent consumers instead of MSMQ and DTC, and Logic Apps for BizTalk, whose 2020 release is the last, with end of support in April 2030."*

---
# Part G — Leading brownfield change

## Concept 31 — Planning and measuring a migration

A migration plan is a design document (Module 30) whose "design" is a **sequence of states**. It is never a Gantt chart of components to rebuild.

**The skeleton of a migration plan:**

1. **Outcomes and measures** (Concept 5) — the why, with numbers.
2. **As-is** — C4 context and container views, data map, dependency map, usage heatmap, risks (Concept 6).
3. **To-be** — the target, at the level of detail you're confident in. Expect it to change; say so.
4. **Transition states** — State 0 … State N, each a container view with its transitional components, data ownership table and rollback path (Concept 13).
5. **Slices and sequence** — what moves in each step, for which cohort, and why that order (Concept 9, 12).
6. **Data plan** — per slice: reads, writes, owner before/after, sync mechanism, reconciliation, the point of no return (Part E).
7. **Risks and mitigations**, including *"what if we stop after State 3?"*
8. **Decommissioning plan** — what gets switched off when, with evidence required (Concept 32).
9. **Organization** — owners, team changes, knowledge transfer, governance changes (Concept 33).
10. **ADRs** — one per significant transition decision.

**Sequencing heuristics:**

- **Stabilize before you move** if the legacy is fragile: automate its deployment, add observability, write the runbook. A system you can't deploy safely is a system you can't migrate safely.
- **Walking skeleton first** (Concept 9), then the first real slice, then speed up.
- **Reads before writes; edges before core code — but the core's *data* work starts early** (Concept 14's 80% stall).
- **New features in the new world from the first slice on.**
- **Interleave value slices and enabling slices** — every few weeks the business should see something.
- **Re-plan after every slice.** The plan beyond the next two or three slices is a hypothesis.

**The "stop adding to the legacy" rule.** Without it the target keeps moving. Make it an explicit, published policy — ideally an ADR — with an exception process: *"From <date>, new features go into the new platform. Changes to the legacy are limited to defects, security, regulatory and changes required for migration. Exceptions need architecture sign-off and an entry in the migration backlog."* PoLD's term for the opposite problem is BAU continuing against the old systems while a separate programme builds the new one — the gap between them widens until the programme is abandoned.

**Measuring progress — track the legacy shrinking, not the new system growing:**

| Indicator | Why it's honest |
|---|---|
| **% of requests served by the new system** (from the facade's `migration.target` tag) | Can't be gamed by building unused services |
| **Capabilities fully migrated *and* retired in legacy** | "Migrated" without "retired" doesn't count |
| **Legacy surface remaining**: endpoints, pages, jobs, tables, lines of code, integrations | Should decrease monotonically |
| **Data ownership**: entities whose source of truth is the new system | The real progress in the hard part |
| **Transitional components alive** (and past their removal date) | Catches permanent transitional architecture |
| **Legacy run cost** (hosting, licences, support contracts) | The money the business was promised |
| **Outcome metrics** (lead time, change failure rate, process KPIs) | The reason you started |

**The interview-grade sentence:** *"A migration plan is a design doc whose design is a sequence of states: outcomes with measures, the as-is, a deliberately provisional to-be, each transition state as a container view with its transitional components, data-ownership table and rollback, the slice sequence and data plan with the point of no return, risks including what happens if we stop halfway, a decommissioning plan, organizational changes and ADRs. I stabilize a fragile legacy before moving it, publish a stop-adding-to-the-legacy rule with an exception process, re-plan after every slice, and measure progress by the legacy shrinking — share of traffic on the new system, capabilities migrated and retired, legacy surface and run cost remaining — rather than by how much new code exists."*

---

## Concept 32 — Decommissioning as a deliverable

The benefit of most migrations — cost savings, risk reduction, simpler operations — is realized only when the old thing is **actually switched off**. PoLD notes how often retirement is a key outcome and yet doesn't happen. Make it a deliverable with a definition of done.

**Decommissioning checklist for a legacy component:**

- [ ] **Traffic is zero** for a soak period covering business cycles (month-end, quarter-end, year-end if relevant), proven by facade telemetry *and* the component's own access logs.
- [ ] **No unknown consumers**: database audit shows no reads/writes; firewall/network logs show no inbound connections; scheduled jobs disabled. Use a **scream test** with care: disable (don't delete) for a period, with fast re-enable, after announcing it.
- [ ] **Data dispositioned**: migrated, archived (read-only, with retention period and access path for audit/legal), or deleted per policy — GDPR and financial retention rules apply.
- [ ] **Reconciliation closed**: final reconciliation report signed off by data owners.
- [ ] **Transitional architecture removed**: routes, flags, sync jobs, mimics, adapters, reconciliation jobs.
- [ ] **Code deleted** from the legacy codebase (or the repo archived), including tests and deployment pipelines.
- [ ] **Infrastructure deleted**: servers, databases, storage, DNS records, certificates, service accounts, firewall rules, secrets in Key Vault.
- [ ] **Contracts and licences terminated**, support contracts ended — with dates aligned to notice periods (often the real deadline driver).
- [ ] **Documentation updated**: C4 model (the legacy box disappears), ADRs superseded/deprecated, runbooks archived, CMDB/service catalogue updated.
- [ ] **Savings reported** to the sponsors.

**Make it visible and celebrated.** A "switch-off" is a milestone worth announcing; it reinforces the stop-adding-to-legacy rule and sustains sponsorship for the next slice.

**The interview-grade sentence:** *"I treat decommissioning as a deliverable with a definition of done, because the promised savings and risk reduction only arrive when the old thing is off: zero traffic over a soak period that covers month-end and other business cycles, proven by both facade telemetry and the legacy's own logs; no unknown consumers, checked with database audit and network logs and a careful, reversible scream test; data migrated, archived with a retention policy, or deleted; final reconciliation signed off; transitional pieces, code, pipelines, infrastructure, DNS, secrets and service accounts removed; licences terminated in line with notice periods; the C4 model and ADRs updated — and the savings reported back to the sponsors."*

---

## Concept 33 — Organization, people and Conway

Patterns of Legacy Displacement's blunt observation: **technology is at most half the legacy problem** — ways of working, organization structure and leadership are just as important. Legacy systems are produced by organizations; replace the system without changing the organization and you will be replacing it again in a few years.

**Ownership models — and their failure modes:**

| Model | Failure mode | Better |
|---|---|---|
| **"Legacy team" keeps the old system; "new team" builds the new** | Two-tier status; knowledge stays with the legacy team; new team rebuilds without understanding; legacy team resists | **Capability-aligned teams own both** the legacy and new implementations of their capability during migration (Team Topologies' stream-aligned teams) |
| **Separate modernization programme alongside BAU** | Requirements diverge; programme overtaken by business change (PoLD's treadmill) | Migration work flows through the same teams and backlog as business work |
| **Outsourced rewrite** | Knowledge leaves; contract locks scope; no organizational learning | Partners embedded in teams; knowledge transfer as an explicit deliverable |
| **Platform team builds the new platform, product teams "migrate later"** | Platform built without real workloads; adoption stalls | Thinnest viable platform shaped by the first slice's real needs |

**Conway's law and the inverse Conway manoeuvre** (Module 21): the target architecture's boundaries should match the team boundaries you intend to have. If you want Billing as an independently deployable service, form a team that owns Billing — legacy and new — before or as you extract it.

**People who hold the knowledge.** The engineers who have kept the legacy alive for ten years are the most important people in the migration. Treat them as experts, not as obstacles: involve them in discovery (they *are* the archaeology), pair them with newer engineers on slices, give them credit for retirements, and make sure the migration offers them a future on the new platform. Losing them mid-migration is a top-five risk.

**Sponsorship and communication.** Executives fund outcomes and visible progress. Report in the language of Concept 31's measures — share of traffic migrated, legacy cost removed, lead time — not in the number of services built. Be candid about the coexistence tax and the point of no return; surprise is what kills sponsorship.

**Corporate antibodies.** PoLD's example: a telecom wanted fast mobile releases but left change-control processes unchanged, so teams spent more time on change forms and meetings than on software. Migrations need governance that matches incremental delivery: small, frequent, low-risk changes approved by automation (pipelines, tests, flags), not by monthly boards. PoLD's organizational patterns — **Protected Pilot**, **Build as you mean to continue**, **Incremental Displacement** — are about protecting the new way of working long enough to prove it.

**Change management for users.** Users have muscle memory for the old system's workflows (including its workarounds). A slice that changes a workflow needs training, a feedback channel and support readiness — and sometimes a period where both UIs are available.

**The interview-grade sentence:** *"Technology is only half of a legacy problem, so I'd organize the migration so capability-aligned teams own both the legacy and new implementations of their capability, with migration work flowing through the same backlog as business work rather than a separate programme that BAU outruns — and use the inverse Conway manoeuvre so team boundaries match the target architecture. The engineers who've kept the legacy alive are the experts in its archaeology and the people I most need to keep, so they pair on slices and get credit for retirements. I report to sponsors in outcome terms, I'm candid about the coexistence cost, and I push for governance that approves small changes by automation, because unchanged change-control is how organizations reject incremental delivery."*

---

## Concept 34 — AI in brownfield work, 2026

AI changes the economics of brownfield work more than of greenfield work, because so much brownfield effort is **comprehension** and **mechanical transformation** — exactly what models are good at. But it does not remove the central risk: **behavioural equivalence**.

**Where AI helps a lot:**

| Task | How |
|---|---|
| **Code comprehension and archaeology** | Explaining unfamiliar modules, summarizing what a stored procedure does, tracing a business rule across layers, answering "where is X handled?" over a whole repository — Thoughtworks' *Legacy Modernization meets GenAI* describes building such comprehension tools over code graphs |
| **Documentation recovery** | Drafting as-is C4 models and retroactive ADRs from code, config and infrastructure (Module 31: drafts to verify, never facts) |
| **Mechanical porting** | The Copilot upgrade agent and AWS Transform for .NET: project conversion, API replacements, build-error resolution, test runs, a commit per step |
| **Characterization test generation** | Generating input grids and Verify-style golden-master tests over legacy code — a strong use, because the tests' job is to pin *current* behaviour, which the model can observe by running the code |
| **Translation drafts** | First passes of Web Forms → Blazor, WCF contracts → gRPC protos, SSIS logic → C# |
| **Diff triage** | Classifying parallel-run mismatches and reconciliation breaks by likely cause |

**Where it doesn't remove the risk:**

1. **"Compiles and tests pass" is not equivalence.** Generated ports can silently change semantics — serializer defaults, culture, rounding, null handling, exception paths, transaction boundaries. Only characterization tests, parallel runs and reconciliation prove equivalence.
2. **Plausible but wrong explanations.** A confident summary of what legacy code "does" can be wrong in exactly the edge cases that matter. Verify against execution (tests, traces), not against the summary.
3. **Volume amplifies review cost.** An agent can produce a 4,000-file diff in an afternoon; humans can't review it in an afternoon. Keep changes small and staged — the same incremental discipline as the rest of this module.
4. **Architecture decisions remain human.** Where the seams go, who owns which data, when to cross the point of no return — these are trade-offs involving the business and the organization.

**The practical stance:** use AI to make comprehension and mechanical work cheap, and reinvest the saved time in the safety net — more characterization tests, longer parallel runs, better reconciliation. Point agents at the migration's ADRs and plan (an `AGENTS.md` entry such as *"do not add features to the legacy solution; follow docs/migration/plan.md; don't change data ownership without an ADR"*), and require that every agent-generated change passes the same gates as human ones.

**The interview-grade sentence:** *"AI changes brownfield economics because so much of the work is comprehension and mechanical transformation: explaining unfamiliar code and stored procedures, drafting as-is C4 models and retroactive ADRs, porting projects with the Copilot upgrade agent or AWS Transform, generating characterization tests and triaging parallel-run diffs. But it doesn't remove the core risk — behavioural equivalence — because generated ports can compile, pass tests and still change serializer, culture, rounding or transaction semantics, and confident explanations can be wrong in exactly the edge cases that matter. So I reinvest the time AI saves into a stronger safety net, keep agent changes small and staged, point agents at the migration plan and ADRs, and keep seams, data ownership and the point of no return as human decisions."*

---

## Concept 35 — Brownfield in the interview

### The brownfield exercise

Architect loops often include a prompt like: *"Our order-management system is a 12-year-old ASP.NET MVC 5 monolith on .NET Framework 4.8 with one big SQL Server database shared with reporting and two partner integrations. Deployments are monthly and risky. The business wants to launch a marketplace for third-party sellers within a year. What would you do?"*

A strong answer follows a recognizable arc — and **says the arc out loud** so the interviewer can follow:

| Step | What to say/do | Time (45 min) |
|---|---|---|
| **1. Clarify outcomes and constraints** | What's the priority — marketplace launch (imminent disruption), cost of change, retirement? Team size and skills? Uptime requirements? Regulatory constraints? Budget? Who reads the database? | 5 min |
| **2. Discovery plan** | How you'd learn the as-is in 2 weeks: five lenses, as-is C4, data map, usage heatmap, surprises. State assumptions you'll make for the exercise | 3–5 min |
| **3. Stabilize** | Deployment automation, observability baseline, characterization tests on the money paths — so change becomes safe | 3–5 min |
| **4. Strategy choice** | Reject the rewrite explicitly with reasons; choose strangler fig; mention replatform vs refactor for hosting | 3 min |
| **5. Seams and first slices** | Facade (YARP + System.Web adapters) as walking skeleton; first slice = the *new* marketplace capability built in the new world (Divert the Flow) with an ACL to the legacy order system; then a read slice (order history from a CDC-fed read model) | 8–10 min |
| **6. Data plan** | Ownership ladder for orders; CDC from legacy; seller data owned by new service from day one; reporting as critical aggregator — keep it fed via legacy mimic/views; reconciliation | 8–10 min |
| **7. Risk, rollback, point of no return** | Flags and cohorts; parallel run for pricing/fees; named one-way door for order ownership with go/no-go criteria | 5 min |
| **8. Organization and measures** | Stop-adding-to-legacy rule; team ownership; progress metrics (traffic share, legacy surface, lead time); decommissioning milestones | 3–5 min |
| **9. Wrap-up** | What you'd do with more time; what would make you change course | 2 min |

**Signals interviewers listen for:**

- You ask about **outcomes before technology**, and about **who else depends on the database**.
- You **reject the big-bang rewrite with reasons**, not slogans — and acknowledge when a replacement could be right.
- You talk about **data ownership explicitly** and name the **point of no return**.
- You describe **intermediate states** and whether each is safe to stay in.
- You mention **decommissioning** and how you'd measure progress honestly.
- You bring in **people and organization** without being asked.
- You use concrete **.NET mechanisms** (YARP, adapters, CDC, Feature Management, Scientist, expand/contract with EF Core) at the right moments — not as name-dropping.

### "Tell me about a migration you led" — behavioral

Structure (Module 34's STAR, calibrated for architects): **situation and outcome sought** → **the strategy you chose and the alternatives you rejected** → **the seam and the first slice** → **the hardest part (almost always data or organization) and how you handled it** → **what you'd do differently** (e.g., "start the core data ownership work earlier; we stalled at 70% for two quarters") → **the measurable result** (legacy switched off, cost removed, lead time from weeks to days).

### Common traps

| Trap | Better |
|---|---|
| Drawing the beautiful target architecture for 30 minutes | Spend most of the time on transitions, data and risk |
| "We'll migrate to microservices" as step one | Modular monolith or strangler slices first; split where the outcome demands it (Module 21) |
| Ignoring the shared database's other readers | Ask in clarification; plan views/mimics/reverse sync |
| "We'll keep both in sync with dual writes" | Outbox/CDC, one writer per record |
| No rollback story | Flags, reverse sync, named point of no return |
| No end state for the facade and transitional parts | Removal triggers in ADRs |
| Promising feature parity | Usage-driven classification of behaviour |

**The interview-grade sentence:** *"In a brownfield exercise I narrate an explicit arc: clarify the outcome and constraints, including who else depends on the database; describe how I'd discover the as-is; stabilize with deployment automation, observability and characterization tests; reject the big-bang rewrite with reasons and choose a strangler fig; insert the facade as a walking skeleton; make the first slice the new capability the business wants, built in the new world behind an anti-corruption layer; plan data ownership rung by rung with CDC and reconciliation and name the point of no return; roll out by cohort with flags and parallel runs; and finish with the stop-adding-to-legacy rule, team ownership, honest progress measures and decommissioning milestones — spending most of the time on transitions and data rather than on the target picture."*

---
# Worked examples

## Worked example 1 — Strangling an ASP.NET MVC 5 order-management monolith

**Situation.** *Contoso Supply*: an ASP.NET MVC 5 + Web API 2 app on .NET Framework 4.8, IIS on two on-prem VMs, one SQL Server database (`Commerce`, ~400 tables) also read by SSRS reports and a nightly partner export. Twelve engineers in two teams. Monthly releases, frequent regressions. Outcome agreed with the business: **reduce cost of change for the ordering and catalogue areas** (lead time from ~6 weeks to < 1 week) and **leave the data centre within 18 months** (lease ends).

**Strategy.** Replatform first for the data-centre deadline, strangle second for cost of change — in parallel tracks.

| State | What changes | Transitional pieces | Rollback |
|---|---|---|---|
| **S0 as-is** | — | — | — |
| **S1 replatform** | App → Azure App Service (Windows) — **Managed Instance** for the plan because the PDF export uses a COM component; `Commerce` → Azure SQL Managed Instance (link feature / DMS for minimal downtime); secrets → Key Vault; Application Insights | Site-to-site VPN for partner SFTP | DNS back to on-prem (kept warm 4 weeks) |
| **S2 walking skeleton** | ASP.NET Core 10 front door with YARP fallback + System.Web adapters (remote session, remote auth); ServiceDefaults multi-targeted for one trace | Facade, adapters | Point Front Door back at the Framework app |
| **S3 catalogue reads** | Catalogue pages and `/api/catalog/*` served by a new Catalog service with its own read model fed by CDC from `Commerce` (ACL translator) | CDC consumer, reconciliation of product counts/prices | Flag `Migration.Catalog` off → fallback route |
| **S4 catalogue writes** | Product editing in the new back-office (Blazor); Catalog owns product data; **reverse sync** writes `dbo.Product*` for legacy order code and SSRS (Legacy Mimic) | Reverse sync, reconciliation | Cohort = product categories; flip back while reverse sync is complete |
| **S5 order history (read)** | Order history from an Orders read model (CDC) | CDC consumer | Flag |
| **S6 order placement** | New Orders service owns placement for **new customers first** (cohort), then existing customers in waves; legacy order tables fed by reverse sync for SSRS and partner export | Reverse sync, pricing parallel run (Scientist), reconciliation | Per cohort while reverse sync is complete — **point of no return** at the final wave, with go/no-go: 14 days clean reconciliation incl. month-end, error rates at baseline |
| **S7 reporting & export** | SSRS reports re-pointed to a reporting store (Revert to Source); partner export generated from Orders events | — | Keep old export in parallel for one cycle |
| **S8 decommission** | Remove fallback routes for migrated areas, reverse sync, legacy tables (archived), legacy controllers; facade remains as the app's gateway | — | — |

**Key decisions (ADRs):** ADR-0050 replatform before refactor; ADR-0051 YARP facade stays as the long-term gateway; ADR-0052 stop adding features to the Framework app (exceptions via architecture review); ADR-0053 CDC (not triggers) for legacy → new sync; ADR-0054 order ownership moves by customer cohort; ADR-0055 point of no return criteria for orders.

**What went wrong (realistically) and the fixes:** remote session added ~15 ms per request and broke when a session type wasn't registered for serialization → characterization tests on session-using pages, then removing session dependence was made its own slice. The CDC consumer missed a column rename by the legacy team → schema-change contract with expand/contract on the legacy side. The partner export depended on the *order of rows* in a table scan → Hyrum's law; the new export sorts explicitly to match.

## Worked example 2 — Moving Billing data ownership out of a shared database

**Situation.** Billing logic lives partly in stored procedures and partly in the monolith; `dbo.Invoice`, `dbo.InvoiceLine`, `dbo.Payment` are read by Orders code, the finance warehouse ETL and three SSRS reports, and written by Billing code *and* a nightly SQL Agent job.

**Plan, step by step:**

1. **Inventory** with SQL Audit for 3 weeks → finds a fourth reader nobody listed (a credit-control Excel workbook with a direct ODBC connection).
2. **Schema move** to `billing.*` with synonyms in `dbo` for legacy code. Views `reporting.vInvoice` for the warehouse and reports; revoke direct `SELECT` on base tables from everyone but Billing's identity. The Excel workbook breaks in UAT — good: it's migrated to the view.
3. **Write consolidation**: the SQL Agent job's logic (late-fee calculation) moved into the Billing module behind branch by abstraction, with a characterization suite over 5,000 historical invoices; job disabled.
4. **Cross-context FKs dropped** (`Orders.InvoiceId` FK becomes a plain column); a reconciliation job verifies every invoiced order has an invoice and vice versa.
5. **New Billing service** (ASP.NET Core 10 + Azure SQL database `billing-db`) built; **Rung 1**: CDC from `billing.*` populates it; reads move behind a flag; Scientist parallel run on invoice totals for 3 weeks (mismatches: rounding mode in one currency — a legacy bug the finance team chose to *keep* until year-end; recorded in an ADR).
6. **Rung 3 cutover** during a quiet window: bulk copy + CDC catch-up, reconciliation (counts, sums per month, 100% record diff on open invoices), write-block on legacy (`DENY INSERT, UPDATE, DELETE`), flag flip, reverse sync to `billing.*` for the remaining legacy readers (views unchanged → warehouse and reports unaffected).
7. **Rung 4**: warehouse ETL re-pointed to Billing's change events (Revert to Source); reports moved; reverse sync off after 2 month-ends; tables archived to a read-only database with 10-year retention; dropped.

## Worked example 3 — Replacing a pricing engine with branch by abstraction and a parallel run

A 2,000-line `PriceCalc.cs` (static, with 14 discount rules and 3 undocumented tenant special cases) is replaced by a rules-based engine.

1. **Seam:** `IPricingEngine` extracted; all 9 call sites moved; `LegacyPricingEngine` wraps the static class. Release.
2. **Net:** Verify golden-master tests over 20,000 anonymized production carts (captured from logs over a month, including month-end promotions) — pins current totals, line discounts and rounding.
3. **Build:** `RulesPricingEngine` built incrementally; the golden-master suite runs against both; mismatches triaged (two were legacy bugs → business decision: fix one, preserve one for contractual reasons).
4. **Parallel run:** `PricingExperiment` (Scientist) in production at 25% sampling for 3 weeks; graduation criterion: < 0.01% mismatches with all categories explained, p95 latency within 5 ms of legacy.
5. **Switch:** `PricingEngineRouter` with `Migration.RulesPricing` flag: internal users → 10% tenants → 100%, kill switch separate.
6. **Delete:** after 30 days at 100%: legacy class, router, experiment, flag, and the golden-master tests are replaced by specification tests for each rule.

## Worked example 4 — Displacing a WCF + MSMQ integration with Event Interception

**Situation.** A legacy warehouse system sends `ShipmentConfirmed` messages over MSMQ to a WCF-hosted "integration service" (.NET Framework) that updates orders and calls a partner SOAP API. The integration service is unsupported, undocumented and blocks leaving Windows Server 2012-era hosts.

1. **Event Interception (refactoring):** a .NET 10 worker (`MsmqBridge`, Windows) reads the MSMQ queue and re-publishes **unchanged** messages to a new MSMQ queue the old service now reads (config change), *and* mirrors them to an Azure Service Bus topic. No behaviour change; observability added.
2. **New consumer** (Container Apps worker) subscribes to the topic, translates messages through an ACL into `ShipmentConfirmed` domain events, updates orders through the Orders API, and calls the partner via a **Legacy Mimic**-style adapter that speaks the partner's existing SOAP contract.
3. **Divert by warehouse:** the bridge's routing table sends warehouse A's messages *only* to the new path; others still go to the old service. Reconciliation compares partner acknowledgements per shipment.
4. Warehouse by warehouse until all are diverted; the WCF service gets zero messages for a month-end; decommissioned.
5. Later, when the warehouse system can publish to Service Bus directly, the MSMQ bridge (transitional) is removed.

## Worked example 5 — Reviewing a big-bang migration plan

**The plan you're handed:** *"Phase 1 (12 months): rebuild all 140 screens of the claims system as microservices on AKS with a new database, achieving full feature parity. Phase 2 (1 weekend): migrate all data and switch over. Phase 3: decommission the old system."*

**Review findings (prioritized):**

1. **Single cutover concentrates all risk; rollback after new writes is effectively impossible** — blocking. Ask for a cohort-based plan with a named point of no return.
2. **Feature parity of 140 screens without usage data** — likely rebuilding unused features; ask for a usage heatmap and behaviour classification.
3. **No value for 12 months; the business can't freeze claims rules for a year** (regulatory changes are frequent) — the target will move; ask how BAU changes are handled.
4. **Architecture and platform change combined** (microservices + AKS + new DB + new team) — multiple simultaneous one-way doors; ask for justification against the outcome; consider modular monolith, replatform first.
5. **No data plan** — who reads the claims database today? Reports, actuarial extracts, regulators?
6. **No transitional architecture, no reconciliation, no characterization tests** — so no way to prove equivalence.
7. **Decommissioning is a phase, not a deliverable** with evidence.

**Recommended rewrite:** outcomes with measures; discovery; facade + walking skeleton; first slice = new claim-intake channel (Divert the Flow); claims read model via CDC; claim types migrated as product lines with parallel runs of adjudication rules; ownership by claim type with reverse sync to keep actuarial extracts working; quarterly decommission milestones.

---
# Common interview questions with model answers

**Q1. "What is the strangler fig pattern?"**
> "Fowler's term for replacing a system gradually: put a facade in front of the legacy that initially routes everything to it, build new implementations slice by slice, divert traffic for each slice progressively — by cohort, with rollback by configuration — and retire each legacy slice once nothing reaches it, until the legacy can be switched off. The interception point can be HTTP routes, message flows with event interception, in-process calls with branch by abstraction, or data with revert-to-source. It works because it turns one irreversible cutover into many reversible ones and delivers value from the first slice."

**Q2. "Why not just rewrite it?"**
> "Because large rewrites fail structurally: the business can't freeze, so the target moves; real requirements are undocumented and surface late; feature parity means rebuilding unused behaviour; nothing is delivered until the end; and all risk concentrates in one cutover that can't be rolled back once new data exists. A rewrite can be right for a small, well-understood system or when the process itself is being redesigned — but even then I'd deliver it as many small cutovers."

**Q3. "How do you choose what to migrate first?"**
> "After a walking skeleton — the facade in production routing everything to the legacy — I pick a slice that's valuable, loosely coupled with a clear seam, maximally educational because it exercises proxy, auth, deployment, observability and data sync, reversible, and in a hotspot so extraction lowers lead time. Often that's a new feature built in the new world, or a high-traffic read path from a CDC-fed read model. Vertical slices, never layers."

**Q4. "How do you keep data consistent while both systems run?"**
> "One writer per record at every state — ownership partitioned by cohort if both systems accept writes. Data flows from the owner to the other side through a transactional outbox if I can change the writer, or CDC — SQL Server CDC, change tracking, Debezium — if I can't; never sequential dual writes. Consumers are idempotent, ordered per key by LSN, loop-safe and replayable, and reconciliation at count, aggregate, record and invariant level proves the sides agree."

**Q5. "What's an anti-corruption layer and why does it matter in a migration?"**
> "Evans' translation layer between models: a facade that speaks the legacy's protocol, a translator from legacy codes and denormalized rows to explicit domain concepts, and an adapter behind a port the new model owns. Without it, the new system adopts the legacy schema and quirks and becomes a re-implementation of the old one; with it, legacy knowledge is concentrated, tested and removable."

**Q6. "How do you split a shared database?"**
> "Inventory every reader, including through SQL Audit; move the capability's tables into a schema it owns with synonyms as a temporary bridge; give other readers views and revoke table access; replace cross-context FKs and joins with IDs, events and read models plus reconciliation; make the owner's identity the only writer; then physically split with bulk copy plus CDC catch-up, keeping views fed by reverse sync until readers move."

**Q7. "How would you migrate an ASP.NET MVC 5 app to ASP.NET Core without a big bang?"**
> "Microsoft's incremental pattern: a new ASP.NET Core app on .NET 10 becomes the entry point and forwards unmigrated routes to the Framework app through YARP; System.Web adapters let shared libraries using HttpContext.Current run in both; remote session and remote auth bridge state and identity until migrated. I move controllers one at a time with characterization tests, then migrate session and auth as their own slices, and finally remove the fallback and the adapters. Aspire has a preview integration that wires this up with one trace across both apps."

**Q8. "How do you know the new implementation behaves like the old one?"**
> "Characterization tests first — golden-master tests with Verify over production-shaped inputs, pinning current behaviour including bugs. Then a parallel run on real traffic: Scientist.NET for pure computations, shadow traffic for services, a full parallel business cycle for things like billing — returning the legacy result, comparing with an explicit equivalence definition, triaging every mismatch and graduating against criteria agreed up front. And reconciliation of data after cutover."

**Q9. "What's the riskiest moment in a migration, and how do you manage it?"**
> "The point of no return — usually when write ownership flips without a working reverse sync, or the legacy is switched off. I make it explicit in the plan, prepare reverse sync before it so earlier steps stay reversible, agree go/no-go criteria in advance — clean reconciliation through a month-end, error rates and business KPIs at baseline, sign-off — and cross it deliberately with heightened monitoring."

**Q10. "How do strangler migrations fail?"**
> "Most often the 80% stall — the edges move, the coupled core with shared data stays and both systems run forever. Also the god facade, double maintenance because features keep landing in the legacy, a distributed monolith calling back into the legacy, transitional pieces that become permanent, and nothing ever switched off. I counter with a stop-adding-to-legacy rule, starting the core's data work early, a dumb-facade ADR, removal triggers, and tracking the legacy shrinking rather than the new system growing."

**Q11. "How do you measure migration progress?"**
> "By the legacy shrinking: share of requests served by the new system from facade telemetry, capabilities migrated *and* retired, legacy endpoints, jobs, tables and integrations remaining, entities whose source of truth has moved, transitional components alive past their removal date, legacy run cost — plus the outcome metrics we started for, like lead time and change failure rate."

**Q12. "Rehost, replatform or refactor?"**
> "Per workload, retire first if usage says nobody needs it. For a legacy .NET web app my default is replatform first — App Service or, since August 2026, Managed Instance on App Service for COM or registry dependencies, Azure SQL, Key Vault, managed identity, Application Insights — then refactor incrementally with the strangler fig where an outcome justifies it. I avoid combining platform and architecture changes in one step."

**Q13. "What do you do with a WCF service?"**
> "Depends on clients. External partners who can't change: port the service to CoreWCF, or front it with a legacy mimic that speaks the same SOAP contract. Internal clients: redesign as gRPC or HTTP APIs and migrate clients with expand/contract — new endpoint beside old, clients move, old retired. Either way I isolate it behind a seam so it doesn't block the rest."

**Q14. "How would you use feature flags in a migration?"**
> "A release flag per slice, targeted by user or tenant so cohorts are sticky, rolled out internal → pilot → percentages, with a separate kill switch that forces the legacy path instantly; Microsoft.FeatureManagement with App Configuration on .NET. Flags have owners and expiry dates, fail to the legacy if the store is unreachable, both paths are tested — and I'm careful with flags that change where data is written, because flipping them back doesn't move the data."

**Q15. "How does AI change legacy modernization?"**
> "Comprehension and mechanical porting become cheap — explaining code, drafting C4 models and retroactive ADRs, porting with the Copilot upgrade agent or AWS Transform, generating characterization tests. Behavioural equivalence doesn't become cheap: generated code can compile and pass tests while changing semantics, and explanations can be confidently wrong. I reinvest the saved time into a stronger safety net and keep changes small and staged."

**Q16. "Tell me about a migration that went wrong."**
> *Structure:* outcome sought → strategy → where it went wrong (be specific: stalled at the shared core, dual-write divergence, an unknown report consumer, the god facade) → what you did to recover → the mechanism-level lesson ("now I start the core's data ownership work in month one", "now SQL Audit runs for three weeks before any schema change", "now every transitional component gets a removal trigger in its ADR"). *Key signal:* ownership of the mistake and a changed practice.

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Framing | "The old system is a mess; let's rewrite it" | Respect for the legacy as an unwritten spec; outcomes first |
| Strategy | Big-bang rewrite with feature parity | Strangler fig with many reversible cutovers; rewrite only when small or process-changing |
| Outcomes | "Modernize to microservices/cloud" | Measurable outcomes: cost of change, retirement date, process KPIs, run cost |
| Discovery | Stop at the legacy boundary; trust the wiki | Five lenses; usage heatmap; SQL Audit; find shadow IT; surprises list |
| Safety | Change first, test later | Characterization tests, golden masters, baselines before touching anything |
| First slice | Trivial page, or the payment core | Walking skeleton, then valuable, decoupled, educational, reversible slice |
| Slicing | Horizontal layers | Vertical slices by capability/route/product line; cohorts for rollout |
| Facade | Business logic in the gateway | Dumb, observable, HA, configuration-driven, tags `migration.target` |
| Transitional architecture | Hidden, unowned, permanent | Designed, owned, ADR'd with removal triggers, tagged, budgeted for removal |
| Models | New service reads legacy tables directly | Anti-corruption layer; new model designed from the domain |
| In-code change | Six-month feature branch | Branch by abstraction on trunk; release every step |
| Flags | Random per-request routing; flags forever | Sticky targeting; kill switch; expiry; careful with write-path flags |
| Verification | "Tests pass" | Parallel run with explicit equivalence; mismatch triage; graduation criteria |
| Contracts | Breaking changes coordinated across six teams | Expand → migrate → contract |
| Data | Dual writes; two writers per record | Outbox/CDC; one writer per record per state; idempotent ordered consumers |
| Shared DB | Ignore other readers | Inventory, views, revoke access, reverse sync, legacy mimic |
| Cutover | Big weekend; no rollback | Bulk + CDC catch-up; reconciliation; named point of no return with go/no-go |
| .NET | Port and re-architect at once | Platform migration separate from architecture change; SDK-style, netstandard2.0, multi-target |
| ASP.NET | Rewrite the MVC app wholesale | YARP + System.Web adapters, remote session/auth, route by route |
| Tooling | Upgrade Assistant (deprecated) or blind trust in agents | Copilot upgrade / AWS Transform with branches, per-step commits, safety net |
| Hosting | Everything to AKS | Rs per workload; replatform first, strangle where it pays; Managed Instance for COM |
| Hard tech | WCF/MSMQ/BizTalk ignored until last | Isolated early; port, bridge or redesign per dependency; BizTalk → Logic Apps before 2030 |
| Progress | "We built 14 services" | Legacy shrinking: traffic share, retired capabilities, remaining surface, cost |
| Finishing | Traffic moved, servers still on | Decommissioning definition of done; licences ended; savings reported |
| Organization | Legacy team vs new team; separate programme | Capability teams own both; same backlog; keep the knowledge holders; governance by automation |
| AI | Generate the whole port and ship | AI for comprehension and mechanics; equivalence proven by tests, parallel runs, reconciliation |
| Interview | 30 minutes on the target picture | Most time on transitions, data, risk, rollback and decommissioning |

---
# Practice exercises

1. **Outcome interview.** For a system you know, write the outcomes three different stakeholders (a product owner, the head of operations, the CFO) would give for "modernizing" it. Where do they conflict? Write the one-paragraph agreed outcome with measures.
2. **Discovery in a day.** Pick a legacy (or just older) .NET solution. Produce: as-is C4 container view, a dependency list with protocols, a data map (which project writes which tables), and a list of three surprises. Use `git log` hotspot analysis to find the top 10 files by change frequency.
3. **Create seams.** In a legacy class with static dependencies (`DateTime.Now`, a static data helper, `ConfigurationManager`), introduce object seams without changing behaviour. Prove it with characterization tests written *before* the refactoring.
4. **Golden master.** Write a Verify-based golden-master suite over a non-trivial legacy calculation with at least 200 input cases, including boundary and malformed inputs. Make it deterministic (`FakeTimeProvider`, invariant culture, scrubbing).
5. **YARP facade.** Build an ASP.NET Core 10 app that forwards everything to an existing app (any sample MVC 5 app, or another ASP.NET Core app standing in for it), then move one route to a local handler. Add a `migration.target` tag to traces and a Feature Management flag that routes a targeted user to the new handler.
6. **System.Web adapters.** Clone the `dotnet/systemweb-adapters` samples and run the remote session and remote authentication samples. Write down what happens to a request at each hop.
7. **Branch by abstraction + Scientist.** Replace a calculation via an interface, a new implementation, a Scientist.NET experiment with a custom comparer and result publisher, and a flag-driven router. Inject a deliberate rounding difference and observe the mismatch report.
8. **Expand/contract.** With EF Core migrations, replace a column with a new one across three releases (expand + dual-write + batched backfill; reader migration; contract). Write the rollback for each release.
9. **CDC sync consumer.** Enable SQL Server CDC on a table in a local SQL Server container, build a consumer that applies changes idempotently and in LSN order into a new model through a translator, then kill it mid-batch and prove it recovers without duplicates.
10. **Reconciliation.** For the store pair from exercise 9, write a reconciliation job at three levels (counts, sums per day, sampled record diff) and a break log. Introduce a translation bug and see which level catches it first.
11. **Migration plan.** Write a migration plan (Appendix A template) for Worked example 5's claims system as it *should* be done: states S0–S6 as container views, data ownership table per state, point of no return, decommissioning milestones.
12. **Copilot upgrade dry run.** On a throwaway branch of a .NET Framework sample (e.g., an old MVC 5 or WinForms sample), run the GitHub Copilot upgrade agent in guided mode. Review `assessment.md` and `plan.md` as if they were a junior engineer's design doc: what would you reorder, and which semantic risks did it miss?
13. **Interview drill.** Set a 45-minute timer and answer Concept 35's brownfield prompt aloud, following the arc. Record yourself and check: did you ask about other database readers? Name the point of no return? Mention decommissioning and organization? How many minutes did you spend on the target picture?

---
# Free resources and learning material

All free to read online unless marked *(book)*. Grouped by purpose; start with the ★ items. Tool and lifecycle pages were checked on October 7, 2026.

### Strangler fig and incremental modernization — read these first
- ★ [Strangler Fig — Martin Fowler (rewritten August 2024)](https://martinfowler.com/bliki/StranglerFigApplication.html) — the metaphor, why gradual beats replacement, the four activities, transitional architecture.
- [Original Strangler Fig Application — Martin Fowler (2004)](https://martinfowler.com/bliki/OriginalStranglerFigApplication.html) — the first post, for history.
- ★ [Patterns of Legacy Displacement — Cartwright, Horn, Lewis](https://martinfowler.com/articles/patterns-legacy-displacement/) — the hub: outcomes, breaking up the problem, delivery patterns, organizational change, and the integration-middleware worked example.
- ★ [Transitional Architecture](https://martinfowler.com/articles/patterns-legacy-displacement/transitional-architecture.html) — why temporary components are worth building.
- [Feature Parity](https://martinfowler.com/articles/patterns-legacy-displacement/feature-parity.html) — why "do what the old one does" usually fails.
- [Legacy Mimic](https://martinfowler.com/articles/patterns-legacy-displacement/legacy-mimic.html) — keeping the legacy and its consumers unaware.
- [Event Interception](https://martinfowler.com/articles/patterns-legacy-displacement/event-interception.html) — routing state changes to new components.
- [Divert the Flow](https://martinfowler.com/articles/patterns-legacy-displacement/divert-the-flow.html) — move cross-organization activities away from legacy first.
- [Revert to Source](https://martinfowler.com/articles/patterns-legacy-displacement/revert-to-source.html) — integrate with the original data source, not the legacy copy.
- [Critical Aggregator](https://martinfowler.com/articles/patterns-legacy-displacement/critical-aggregator.html) — the reporting dependency that blocks displacement.
- [Extract Product Lines](https://martinfowler.com/articles/patterns-legacy-displacement/extract-product-lines.html) — slicing by product line.
- [Thoughtworks Technology Podcast: Patterns of Legacy Displacement, part 1](https://www.thoughtworks.com/insights/podcasts/technology-podcasts/patterns-legacy-displacement-pt1) and [part 2](https://www.thoughtworks.com/insights/podcasts/technology-podcasts/patterns-of-legacy-displacement-pt2) — the authors talk through the patterns.
- [Embracing the Strangler Fig pattern for legacy modernization — Premanand Chandrasekaran (Thoughtworks)](https://www.thoughtworks.com/insights/articles/embracing-strangler-fig-pattern-legacy-modernization-part-one) — lessons from practice.
- [How to break a Monolith into Microservices — Zhamak Dehghani](https://martinfowler.com/articles/break-monolith-into-microservices.html) — decoupling order, data, and "decouple vertically".
- [Sacrificial Architecture — Martin Fowler](https://martinfowler.com/bliki/SacrificialArchitecture.html) — building things you expect to replace.
- [Things You Should Never Do, Part I — Joel Spolsky (2000)](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/) — the classic argument against rewrites.

### Seams, legacy code and safety nets
- ★ [Legacy Seam — Martin Fowler (2024)](https://martinfowler.com/bliki/LegacySeam.html) — seams from code to whole systems.
- [Uncovering mainframe seams — Ferri & Coggrave](https://martinfowler.com/articles/uncovering-mainframe-seams.html) — finding seams where people think there are none.
- [Characterization test — Wikipedia](https://en.wikipedia.org/wiki/Characterization_test) — Feathers' concept, briefly.
- ★ [Verify (VerifyTests) for .NET](https://github.com/VerifyTests/Verify) — snapshot/approval testing; the standard golden-master tool.
- [ApprovalTests.Net](https://github.com/approvals/ApprovalTests.Net) — the older approval-testing library you'll meet in legacy suites.
- [code-maat — Adam Tornhill](https://github.com/adamtornhill/code-maat) — mine git history for hotspots and temporal coupling.
- [Hyrum's Law](https://www.hyrumslaw.com/) — every observable behaviour will be depended on.
- [Refactoring catalog — Martin Fowler](https://refactoring.com/catalog/) — the named refactorings you'll use to create seams.
- [TimeProvider and FakeTimeProvider — .NET docs](https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview) — the injectable clock for deterministic tests.
- *(book)* Michael Feathers, *Working Effectively with Legacy Code* — seams, characterization tests, dependency-breaking techniques.

### Companion patterns
- ★ [Branch By Abstraction — Martin Fowler](https://martinfowler.com/bliki/BranchByAbstraction.html) — the in-process strangler.
- ★ [Parallel Change (expand and contract) — Danilo Sato](https://martinfowler.com/bliki/ParallelChange.html) — changing interfaces in three reversible steps.
- ★ [Feature Toggles (aka Feature Flags) — Pete Hodgson](https://martinfowler.com/articles/feature-toggles.html) — the toggle taxonomy and toggle debt.
- [Dark Launching — Martin Fowler](https://martinfowler.com/bliki/DarkLaunching.html) and [Canary Release — Danilo Sato](https://martinfowler.com/bliki/CanaryRelease.html).
- ★ [Anti-corruption Layer pattern — Azure Architecture Center](https://learn.microsoft.com/azure/architecture/patterns/anti-corruption-layer).
- ★ [Strangler Fig pattern — Azure Architecture Center](https://learn.microsoft.com/azure/architecture/patterns/strangler-fig).
- [Gateway Routing pattern — Azure Architecture Center](https://learn.microsoft.com/azure/architecture/patterns/gateway-routing) — the facade's job description.
- [Strangler fig pattern — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/strangler-fig.html) and [Anti-corruption layer pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/acl.html).
- ★ [Scientist.NET](https://github.com/scientistproject/Scientist.net) — parallel experiments for safe refactoring of critical paths; and the original [GitHub Scientist (Ruby)](https://github.com/github/scientist) README explaining the idea.
- [Canarying releases — Google SRE Workbook](https://sre.google/workbook/canarying-releases/) — canary vs control analysis.
- [Reliable product launches at scale — Google SRE Book](https://sre.google/sre-book/reliable-product-launches/) — launch checklists and gradual rollouts.

### Data migration and shared databases
- ★ [Online migrations at scale — Stripe Engineering](https://stripe.com/blog/online-migrations) — dual-write, change reads, change writes, remove old data.
- ★ [Migrating critical traffic at scale with no downtime, part 1 — Netflix Tech Blog](https://netflixtechblog.com/migrating-critical-traffic-at-scale-with-no-downtime-part-1-ba1c7a1c7835) — replay testing and sticky canaries for a migration.
- [Evolutionary Database Design — Pramod Sadalage & Martin Fowler](https://martinfowler.com/articles/evodb.html) — migrations, transition phases, database refactoring.
- [Database Refactoring catalogue — Scott Ambler](https://databaserefactoring.com/) — named refactorings with transition periods.
- ★ [About change data capture (SQL Server)](https://learn.microsoft.com/sql/relational-databases/track-changes/about-change-data-capture-sql-server) and [About change tracking](https://learn.microsoft.com/sql/relational-databases/track-changes/about-change-tracking-sql-server).
- [Change event streaming (SQL Server 2025 / Azure SQL) — overview (preview)](https://learn.microsoft.com/sql/relational-databases/track-changes/change-event-streaming/overview).
- [Debezium connector for SQL Server](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html) — snapshot + streaming CDC.
- [Azure Cosmos DB change feed](https://learn.microsoft.com/azure/cosmos-db/change-feed).
- [Azure Database Migration Service](https://learn.microsoft.com/azure/dms/dms-overview) — online migrations of SQL Server to Azure SQL.
- [EF Core reverse engineering (scaffolding)](https://learn.microsoft.com/ef/core/managing-schemas/scaffolding/) — starting a model from a legacy schema.
- [Sam Newman — *Monolith to Microservices*](https://samnewman.io/books/monolith-to-microservices/) *(book page)* — the database decomposition patterns (views, wrapping services, split table, moving FKs to code).

### .NET migration: platform, ASP.NET and tooling
- ★ [Incremental ASP.NET to ASP.NET Core migration — overview](https://learn.microsoft.com/aspnet/core/migration/inc/overview) and ★ [Remote app setup (YARP fallback, remote session/auth, Aspire)](https://learn.microsoft.com/aspnet/core/migration/fx-to-core/inc/remote-app-setup).
- ★ [dotnet/systemweb-adapters](https://github.com/dotnet/systemweb-adapters) — source and samples (remote session, remote auth, MachineKey, modules/handlers).
- [Incremental ASP.NET to ASP.NET Core Migration — .NET Blog (Mike Rousos)](https://devblogs.microsoft.com/dotnet/incremental-asp-net-to-asp-net-core-migration/) — the announcement and walkthrough.
- ★ [Tales from the .NET Migration Trenches — Jimmy Bogard](https://www.jimmybogard.com/tales-from-the-net-migration-trenches/) — field report on a decade-old codebase.
- [YARP overview](https://learn.microsoft.com/aspnet/core/fundamentals/servers/yarp/yarp-overview) and [dotnet/yarp](https://github.com/dotnet/yarp) — configuration, direct forwarding, transforms.
- ★ [GitHub Copilot upgrade — overview](https://learn.microsoft.com/dotnet/core/porting/github-copilot-upgrade/overview) — scenarios, strategies, `.github/upgrades` state.
- [GitHub Copilot modernization for Azure migration](https://learn.microsoft.com/dotnet/azure/migration/appmod/overview).
- [.NET Upgrade Assistant (deprecated)](https://learn.microsoft.com/dotnet/core/porting/upgrade-assistant-overview) — for reading older guides.
- [Porting from .NET Framework to .NET — overview](https://learn.microsoft.com/dotnet/core/porting/framework-overview).
- [.NET Framework technologies unavailable on .NET](https://learn.microsoft.com/dotnet/core/porting/net-framework-tech-unavailable) — Web Forms, WCF server, WF, Remoting, AppDomains, CAS.
- [Windows Compatibility Pack](https://learn.microsoft.com/dotnet/core/porting/windows-compat-pack).
- [.NET Standard](https://learn.microsoft.com/dotnet/standard/net-standard) and [Target frameworks / multi-targeting](https://learn.microsoft.com/dotnet/standard/frameworks).
- ★ [.NET and .NET Core support policy](https://dotnet.microsoft.com/platform/support/policy/dotnet-core) and [.NET 8 and .NET 9 end of support — .NET Blog](https://devblogs.microsoft.com/dotnet/dotnet-8-9-end-of-support/).
- [.NET Framework support policy](https://dotnet.microsoft.com/platform/support/policy/dotnet-framework) — 4.6.2 ends January 2027; 3.5 SP1 in 2029.
- [CoreWCF](https://github.com/CoreWCF/CoreWCF) and [CoreWCF support policy](https://dotnet.microsoft.com/platform/support/policy/corewcf).
- [Feature Management for .NET — reference](https://learn.microsoft.com/azure/azure-app-configuration/feature-management-dotnet-reference) and [microsoft/FeatureManagement-Dotnet](https://github.com/microsoft/FeatureManagement-Dotnet).
- [AWS Transform for .NET](https://aws.amazon.com/transform/net/) — agentic porting of .NET Framework to Linux-ready .NET.

### Azure migration and hosting
- ★ [App Service baseline architecture (zone-redundant, network-secured)](https://learn.microsoft.com/azure/architecture/web-apps/app-service/architectures/baseline-zone-redundant) — today's replatform target; the old Reliable/Modern Web App URLs redirect here.
- [Modern Web App pattern for .NET — reference implementation (archived)](https://github.com/Azure/modern-web-app-pattern-dotnet) and [Reliable Web App (archived)](https://github.com/Azure/reliable-web-app-pattern-dotnet) — read-only, but still a worked example of replatform-then-strangle with Relecloud.
- ★ [Managed Instance on Azure App Service — overview](https://learn.microsoft.com/azure/app-service/overview-managed-instance) — COM, registry, MSI, install scripts (GA August 2026).
- [Azure Migrate overview](https://learn.microsoft.com/azure/migrate/migrate-services-overview) — discovery and assessment for portfolios.
- [Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/) — migration and rationalization (the Rs).
- [Why migrate from BizTalk Server to Azure Logic Apps](https://learn.microsoft.com/azure/logic-apps/biztalk-server-migration-overview) and [BizTalk Server product lifecycle update (December 2025)](https://techcommunity.microsoft.com/blog/integrationsonazureblog/microsoft-biztalk-server-product-lifecycle-update/4478559).
- [Application Map in Application Insights](https://learn.microsoft.com/azure/azure-monitor/app/app-map) — runtime dependency discovery.
- [Azure Resource Graph overview](https://learn.microsoft.com/azure/governance/resource-graph/overview) — what's actually deployed.
- [Aspire documentation](https://aspire.dev/) — the AppHost model used by the incremental-migration integration.

### Discovery, domains and organization
- [EventStorming](https://www.eventstorming.com/) — Alberto Brandolini's site; the discovery workshop PoLD recommends.
- [DDD Crew — context mapping](https://github.com/ddd-crew/context-mapping) — relationships including anti-corruption layer.
- [Learn Wardley Mapping](https://learnwardleymapping.com/) — strategic mapping for what to build, buy or retire.
- [Team Topologies — key concepts](https://teamtopologies.com/key-concepts) — stream-aligned teams and cognitive load.
- [Conway's Law — Martin Fowler](https://martinfowler.com/bliki/ConwaysLaw.html) — including the inverse Conway manoeuvre.
- [DORA](https://dora.dev/) — delivery metrics to measure "cost of change" outcomes.
- [Monolith First — Martin Fowler](https://martinfowler.com/bliki/MonolithFirst.html) — context for not jumping straight to microservices.

### Case studies
- [Deconstructing the Monolith — Shopify Engineering](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) — modularizing in place rather than rewriting.
- [Under Deconstruction: The State of Shopify's Monolith](https://shopify.engineering/shopify-monolith) — what happened next, with honest lessons.
- [Patterns of Legacy Displacement — integration middleware example](https://martinfowler.com/articles/patterns-legacy-displacement/#AnExampleIntegrationMiddlewareRemoval) — event interception, legacy mimics, segment by product.

### AI-assisted modernization
- [Legacy Modernization meets GenAI — Thoughtworks on martinfowler.com](https://martinfowler.com/articles/legacy-modernization-gen-ai.html) — code comprehension tooling over legacy codebases.
- [AGENTS.md](https://agents.md/) — where to point coding agents at the migration plan and ADRs.

### Books (not free, but the canon)
- *(book)* Sam Newman — *Monolith to Microservices* (O'Reilly, 2019).
- *(book)* Michael Feathers — *Working Effectively with Legacy Code* (2004).
- *(book)* Marianne Bellotti — *Kill It with Fire: Manage Aging Computer Systems (and Future Proof Modern Ones)* (No Starch, 2021).
- *(book)* Nick Tune with Jean-Georges Perrin — *Architecture Modernization* (Manning, 2024).
- *(book)* Adam Tornhill — *Your Code as a Crime Scene*, 2nd ed. (Pragmatic, 2024).
- *(book)* Ola Ellnestam & Daniel Brolund — *The Mikado Method* (Manning, 2014) — sequencing large refactorings safely.

---
# Quick-recall sheet

**One sentence.** Change a running system through small, reversible, value-delivering transitions where old and new coexist behind a seam you control; never one big cutover.

**Legacy.** Code without tests (Feathers) · valuable code you're afraid to change · platform/support disappearing. Respect the original design.

**Why rewrites fail.** Moving target · hidden requirements · Feature Parity trap · value deferred · one giant cutover with no rollback · second-system effect · the treadmill. Rewrite OK if small or process is changing — still deliver in small cutovers.

**Incremental economics.** Early value, bounded risk, learning, option value, continuous delivery — paid for with a coexistence tax (transitional architecture, two systems, sync, cognitive load). Make the trade explicit. Many two-way doors, few small one-way doors.

**Legacy knowledge.** Hyrum's law · Chesterton's fence · edge cases, data fixes, batch jobs, integration side effects, shadow IT, workarounds. Classify behaviour: preserve exactly / preserve outcome / drop / fix.

**Outcomes.** Cost of change · business process · retire system · imminent disruption · operational risk · cost · scale/new markets. "Newer tech" isn't one. Measure: DORA, process KPIs, % on legacy, run cost.

**Discovery.** Structure · runtime · data · business · people. Outputs: as-is C4, dependency map, data map, capability overlay, usage heatmap, EOL calendar, surprises. Cross the boundary; find the spreadsheets.

**Seams.** Change behaviour without editing in place; enabling point. Object (DI), assembly, HTTP route, queue, view/synonym, file, UI, business segment. Good seams: routable, observable, cheap to switch. Hotspots = change frequency × complexity.

**Safety net.** Characterization tests (pin bugs too) · golden master with Verify · deterministic (FakeTimeProvider, invariant culture, scrub) · black-box snapshots · production baselines · `migration.target` trace tag.

**First slice.** Walking skeleton first (facade at 100% legacy). Then valuable, decoupled, educational, reversible, hotspot. New feature in new world / CDC-fed read path / bounded capability. Vertical, not layers.

**Strangler fig.** Insert facade → divert slice by slice by cohort → retire → repeat → eliminate. Interception at HTTP, messages (Event Interception), code (Branch by Abstraction), data (Revert to Source), business flow (Divert the Flow).

**Facade rules.** Dumb · config-driven hot reload · HA, latency budgeted · observable · contract-preserving · cross-cutting once · sticky cohorts · end state in an ADR. YARP / APIM / Front Door / message router.

**Slices × cohorts.** Route, capability, read vs write, product line, lifecycle, event type, geography × internal, pilot, % (sticky), new customers, segments. Canary vs control with pre-agreed thresholds.

**Transitional architecture.** Facade, ACLs, CDC sync, reverse sync, mimics, reconciliation, flags, experiments. Owner, monitoring, ADR with removal trigger, tagged, removal budgeted. Every state: safe to stay 6 months?

**Failure modes.** 80% stall · god facade · double maintenance · distributed monolith · legacy-shaped model · permanent transitional pieces · nothing switched off · corporate antibodies · big bang in disguise · long tail.

**ACL.** Facade (legacy protocol) + translator (codes → domain) + adapter (port the new model owns). Pure, tested, fails loudly on unknowns.

**Branch by abstraction.** Abstraction → new impl unused → switch (flag) → delete old. Trunk, release every step.

**Flags.** Release / experiment / ops (kill switch) / permission. Sticky by user/tenant. Owner + expiry. Fail to legacy. Write-path flags aren't freely reversible. Microsoft.FeatureManagement + App Configuration.

**Parallel run.** Scientist.NET (Use/Try/Compare, return control) · dark launch · shadow traffic · record & replay · parallel business cycle. Explicit equivalence, mismatch triage, graduation criteria, sampling, no side effects.

**Expand/contract.** Add new beside old (write both, batched backfill) → migrate consumers → prove unused → remove. Separate releases; no EF `RenameColumn` on shared columns; Deprecation/Sunset headers.

**PoLD catalogue.** Transitional Architecture · Event Interception · Legacy Mimic · Divert the Flow · Revert to Source · Critical Aggregator · Extract Product Lines · Feature Parity (anti) · Stop the World · Dark Launching · Canary Release.

**Data truths.** Shared DB = integration contract · one source of truth per fact · no cross-boundary transactions · data gravity · ownership flip = real one-way door.

**Ownership ladder.** R0 read legacy via ACL/views → R1 own model synced from legacy → R2 cohort writes + reverse sync → R3 source of truth for all (point of no return) → R4 readers moved, archive, drop. Reads before writes; one writer per record.

**Sync.** Outbox if you can change the writer; CDC / change tracking / Debezium / (preview) change event streaming if not. Never sequential dual writes. Consumers: idempotent, ordered by LSN, one-writer aware, loop-safe, schema-change aware, lag metric, replayable.

**Split shared DB.** Inventory (SQL Audit) → schema per owner + synonyms → views for readers, revoke tables → drop cross-context FKs/joins → owner-only writes (managed identity) → physical split (bulk + CDC) → views as mimic → remove.

**Cutover.** Bulk load → CDC catch-up from snapshot LSN → reconcile → write freeze → flip → soak. Rollback planned in advance; reverse sync makes it cheap. Name the point of no return; go/no-go criteria. Reconcile: counts, aggregates, record diffs, invariants, flow balance; break log.

**.NET clock (Oct 2026).** .NET 8 & 9 end Nov 10, 2026 · .NET 10 LTS to Nov 2028 · Framework 4.6.2 ends Jan 12, 2027 · 3.5 SP1 Jan 9, 2029 · 4.7–4.8.1 follow Windows.

**Framework → .NET.** Not ported: Web Forms, WCF server (CoreWCF), WF (CoreWF), Remoting, AppDomains, CAS, System.Web. Inventory → SDK-style → netstandard2.0 / multi-target → bottom-up or strangle top-down → isolate blockers. Platform change ≠ architecture change.

**ASP.NET incremental.** ASP.NET Core front door + YARP fallback (`MapForwarder` lowest order, `ShortCircuit`) · System.Web adapters · remote session/auth (API key) · Aspire `WithIncrementalMigrationFallback` (preview) · same virtual dirs · lock down Framework app · remove adapters at the end.

**Tooling.** Upgrade Assistant deprecated → GitHub Copilot upgrade (assess/plan/execute, `.github/upgrades`, Web Forms → Blazor) + Copilot modernization (Azure) · AWS Transform for .NET · Windows Compatibility Pack · CA1416. Safety net first; watch serializer, culture, SqlClient Encrypt, DateTime kinds.

**Rs.** Retire · retain · rehost · replatform · refactor · rearchitect · rebuild/replace. Default: replatform (App Service baseline; Managed Instance on App Service for COM/registry/MSI) then strangle incrementally. (Reliable/Modern Web App guidance: retired, repos archived.)

**Hard tech.** WCF → CoreWCF / gRPC / HTTP (+ mimic for partners) · Web Forms → Blazor page by page · MSMQ/DTC → Service Bus + outbox + idempotency · BizTalk → Logic Apps (EOS Apr 2030) · Windows services → Worker Service / Container Apps jobs · SSIS → ADF · COM → isolate / Managed Instance · Windows auth → Entra ID.

**Plan.** Outcomes · as-is · provisional to-be · states with transitional parts, ownership, rollback · slices · data plan + point of no return · risks (stop halfway?) · decommissioning · organization · ADRs. Stabilize first. Stop adding to legacy. Re-plan every slice.

**Measure.** Legacy shrinking: traffic share, capabilities migrated *and* retired, remaining surface, data ownership moved, transitional parts alive, legacy cost, outcome metrics.

**Decommission.** Zero traffic through business cycles · no unknown consumers (audit, network, scream test) · data migrated/archived/deleted · final reconciliation · transitional parts, code, infra, DNS, secrets, accounts removed · licences ended · docs updated · savings reported.

**Organization.** Tech ≤ 50% of the problem · capability teams own legacy + new · same backlog · inverse Conway · keep knowledge holders · outcome-language reporting · governance by automation · user change management.

**AI.** Cheap comprehension and mechanics; equivalence still proven by tests, parallel runs, reconciliation. Small staged changes; agents pointed at plan and ADRs.

**Interview arc.** Outcomes & constraints (who else reads the DB?) → discovery → stabilize → reject rewrite with reasons → facade/walking skeleton → first slice (new capability, ACL) → data ladder + CDC + reconciliation + point of no return → cohorts, flags, parallel run → stop-adding rule, teams, measures, decommissioning.

---
# Appendix A — Migration plan template

```markdown
# Migration plan: <system / capability>
Status: Draft | In review | Approved · Owner: <name> · Last updated: <date>
Related: RFC-NNNN · ADRs: NNNN–NNNN · C4 workspace: docs/workspace.dsl (views: AsIs, S1…Sn, ToBe)

## 1. Outcomes and measures
| Outcome | Measure | Baseline | Target | By |
|---|---|---|---|---|

## 2. Constraints
Uptime / maintenance windows · regulatory · contracts and licence dates · team capacity · budget

## 3. As-is (verified <date>)
Context + container views · dependency map · data map (writer/readers per table) · usage heatmap ·
end-of-support calendar · surprises

## 4. To-be (provisional)
Container view · what we're confident about · what we expect to learn

## 5. Transition states
| State | What changes | Transitional components (owner, removal trigger) | Data ownership changes | Rollback | Safe to pause here? |
|---|---|---|---|---|---|
| S0 as-is | | | | | |
| S1 facade (walking skeleton) | | | | | |
| … | | | | | |

## 6. Slices and cohorts
| Slice | Why this order | Cohort sequence | Flag | Verification (char. tests / parallel run / reconciliation) |

## 7. Data plan
Per entity: reads/writes per state · sync mechanism · reconciliation levels · **point of no return** and go/no-go criteria

## 8. Risks and mitigations
Including: what if we stop after S<n>? · key-person risk · unknown consumers · performance of the facade

## 9. Decommissioning plan
| Legacy component | Criteria | Evidence | Target date | Cost removed |

## 10. Organization
Team ownership · knowledge transfer · governance changes · stop-adding-to-legacy rule (link ADR) · communication cadence

## 11. Progress reporting
Traffic share · capabilities migrated and retired · legacy surface remaining · transitional parts alive · legacy cost · outcome metrics
```

---
# Appendix B — Transition-state review checklist

For each state in the plan (and before each slice ships):

- [ ] The state is drawn as a C4 container view, with transitional components visibly tagged.
- [ ] Every fact (table/entity) has exactly one writer in this state; the ownership table is explicit.
- [ ] All other readers of moved data are listed, and each is served (view, mimic, reverse sync, migrated).
- [ ] Sync mechanisms are outbox/CDC-based — no sequential dual writes; consumers are idempotent, ordered and observable (lag alert).
- [ ] Reconciliation exists for the moved data, with a break log and an owner.
- [ ] Routing for the slice is configuration/flag-driven, sticky per user/tenant, with a separate kill switch.
- [ ] Rollback from this state is written down and has been rehearsed.
- [ ] If this state includes the point of no return, go/no-go criteria are agreed and signed off.
- [ ] Characterization tests and/or parallel-run results exist for the behaviour being moved; equivalence is defined.
- [ ] The facade gained no business logic.
- [ ] Each transitional component has an owner, monitoring, runbook and an ADR with a removal trigger.
- [ ] Baselines and dashboards are split by `migration.target`.
- [ ] The state is safe to remain in for six months if the programme pauses (cost, risk, support load acceptable).
- [ ] Decommissioning tasks created by this state are on the roadmap.

---
# Appendix C — Cutover runbook skeleton

```markdown
# Cutover: <slice / entity> — <date/time window>
Owner: <name> · Decision-maker for go/no-go: <name> · Comms: <channel>

## Preconditions (T-7 days → T-1 day)
- [ ] Bulk load complete; CDC catch-up lag < <N> s for 72 h
- [ ] Reconciliation clean for <N> days incl. <business cycle>; break log empty or explained
- [ ] Reverse sync tested end-to-end (if rollback depends on it)
- [ ] Rollback rehearsed in staging on <date>; duration <X> min
- [ ] Support and stakeholders informed; freeze on related changes

## Go / no-go (T-0)
Criteria: error rate ≤ baseline + <x>%, p95 ≤ <y> ms, reconciliation clean, on-call staffed → GO / NO-GO (recorded)

## Steps
1. Enable write block on legacy for scope (DENY / feature flag) — verify
2. Drain: wait for CDC lag = 0; record final LSN
3. Final reconciliation (counts + aggregates + 100% diff of open items)
4. Flip flag/route `<flag>` to new for scope — verify with synthetic transaction
5. Start reverse sync (if applicable) — verify first propagated change
6. Announce completion

## Watch (T+0 → T+<soak>)
Dashboards: <links> · Reconciliation every <N> min for first 24 h · Abort triggers: <list>

## Rollback
Before point of no return: <steps> · After: <steps or "not supported — forward fix only">

## Close-out
Post-cutover review · update C4 state · ADR status · decommissioning tasks created
```

---
# Appendix D — Example transition ADRs (Y-statement form)

```markdown
# 0051. Keep the YARP facade as the long-term gateway
Status: Accepted · Confidence: Medium
In the context of the strangler migration of the ordering monolith,
facing the need to route between legacy and new implementations for at least 18 months,
we decided for an ASP.NET Core 10 front door with YARP that remains as the application gateway afterwards,
and neglected Azure API Management as the interceptor (cost, policy-XML logic creep) and Front Door rules (path-only routing),
to achieve cohort-level routing with code-level control and one trace across both stacks,
accepting that we own its availability, scaling and patching.
Removal trigger: n/a (becomes permanent) — the fallback route and System.Web adapters are removed when the Framework app is retired.

# 0052. Stop adding features to the .NET Framework application
Status: Accepted
In the context of the ordering migration, facing BAU features continuing to land in the legacy,
we decided that from 2026-11-01 new features are built only in the new platform; legacy changes are limited to defects,
security, regulatory and migration-enabling changes; exceptions need architecture review and a migration-backlog entry,
and neglected allowing feature work in both,
to achieve a target that stops moving and avoids double maintenance,
accepting that some features will be delivered later or need an integration back into the legacy.

# 0055. Point of no return for order ownership
Status: Accepted · Confidence: High
In the context of moving order write ownership to the Orders service,
facing that rollback after the final cohort requires replaying new orders into the legacy,
we decided that the final wave proceeds only when: reconciliation has been clean for 14 days including a month-end,
error rates and order conversion are within baseline, reverse sync lag p99 < 30 s, and Finance has signed off,
and neglected a time-boxed cutover date independent of evidence,
to achieve a deliberate, evidence-based crossing of the one-way door,
accepting that the programme date may move.
```

---

*Next: **Module 33 — Cost, build-vs-buy, and technical debt**: the language executives respond to — how to price the coexistence tax, the cost of delay and the cost of doing nothing, and how to argue for (or against) a migration in terms the business will fund.*

