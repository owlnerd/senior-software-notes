# Module 33 — Cost, Build-vs-Buy, and Technical Debt: The Language Executives Respond To
*Phase 7: The Architect-Specific Track · Senior/Architect Interview Prep for .NET & C#*

> **State of practice verified on October 7, 2026.** The economics in this module are old and stable — Cunningham's debt metaphor (1992), net present value, Reinertsen's cost of delay (2009), core-vs-context, Wardley maps. What moved in the last year are the **prices, policies and licences** that executives will quote back at you:
>
> - **FinOps.** The FinOps Foundation's **Framework 2026** (presented March 2026) adds a new capability, **Executive Strategy Alignment**, deepens **Scopes** (introduced in 2025 to cover SaaS, licensing, data centre and AI spend, not just public cloud), and renames *FinOps Tools & Services* to *Automation, Tools & Services*. The **State of FinOps 2026** survey reports that **98% of FinOps teams now manage AI spend** (31% in 2024). FinOps is no longer "the cloud bill team"; it is how technology value is governed.
> - **FOCUS** (FinOps Open Cost and Usage Specification), the vendor-neutral billing-data schema: **v1.3** was ratified in December 2025 (contract-commitment dataset, split-cost allocation for shared resources, recency/completeness metadata); **v1.4 is now the latest**, and v1.5 is in scope planning. **Azure Cost Management exports** can produce FOCUS-format datasets with a selectable schema version, and the open-source **Microsoft FinOps toolkit** (FinOps hubs, Power BI reports) ingests them. Check which FOCUS version your export emits — providers lag the spec.
> - **Azure commitment discounts are changing.** Announced July 2026: **from February 1, 2027, newly purchased reservations for services covered by an Azure savings plan (compute and databases — VMs, App Service, Azure SQL and others) can no longer be exchanged.** Existing eligible reservations get **one final exchange**; trading a reservation in for a **savings plan** remains possible; the cancellation policy is unchanged. The practical effect: reservations become a rigid bet on a specific SKU and region; savings plans become the default flexible commitment.
> - **AI is a first-class cost line.** **Azure AI Foundry was renamed Microsoft Foundry** at Ignite (November 2025). Azure OpenAI models are sold in several billing modes for the same model — **Standard** (Global / Data Zone / Regional), **Priority**, **Provisioned** (PTUs, billed per hour whether used or not, discounted by monthly/annual reservations) and **Batch** (asynchronous, ~24-hour target, **about 50% below Global Standard**). Prompt caching discounts repeated prefixes. Token prices change every few months — verify before you put a number in a business case.
> - **Exit costs are falling by regulation.** The **EU Data Act** has applied since **September 12, 2025**; cloud **switching charges, including egress for a switch, must be abolished entirely by January 12, 2027** (until then they may only be cost-based). Azure, AWS and Google have offered **free data transfer out for customers leaving** since early 2024. Ordinary operational egress — serving your users, running multi-cloud in parallel — is **not** abolished.
> - **The .NET open-source relicensing wave changed build-vs-buy for libraries.** **FluentAssertions 8** (January 2025) moved to an Xceed licence that requires a paid licence for commercial use (about $130 per developer per year at launch). **MediatR 13+ and AutoMapper 15+** (July 2, 2025) moved to **Lucky Penny Software** under a dual **RPL-1.5 / commercial** licence with a free community edition below a revenue threshold (MediatR 12.5 is the last Apache-2.0 release; AutoMapper is now on v16). **MassTransit v9** is commercial (from **Massient**), with v8 security patches promised through 2026. **QuestPDF's Community Licence v3** (July 6, 2026) is free only for organizations under **USD 1M annual revenue**, and never for publicly traded companies or public-sector bodies. Outside .NET: **Redis** went source-available in 2024, spawning the BSD-licensed **Valkey** fork, then added **AGPLv3** with Redis 8 (May 2025).
> - **Accounting and tax for software spend moved — for US entities.** **FASB ASU 2025-06** (September 18, 2025) removes the waterfall "project stages" from internal-use software capitalization (ASC 350-40): capitalize once management has authorized and committed funding and completion is probable; effective for annual periods beginning after **December 15, 2027**, early adoption permitted. For US tax, **Section 174A** (One Big Beautiful Bill Act, July 4, 2025) restores **immediate deduction of domestic R&E, which explicitly includes software development**, for tax years beginning after December 31, 2024; **foreign** R&E is still amortized over 15 years. Most of the world reports under **IFRS (IAS 38)**, with different rules. *This is context for conversations with finance, not accounting or tax advice — I'm not an accountant; your CFO's team decides.*
> - **AI and technical debt.** The **2025 DORA report** (*State of AI-assisted Software Development*) found AI use near-universal (90%) and now associated with **higher throughput** but still with **lower delivery stability** — "AI amplifies what's already there." **GitClear's 2025** analysis of 211M changed lines saw duplicated code blocks rise about **8×** during 2024 while "moved" (refactored) code fell below 10% of changes. Early 2026's "SaaSpocalypse" repricing of software stocks made *"can't we just build it with agents?"* a board-level question; the honest answer is in Concept 27.
> - **Runtime support is a recurring cost.** As Module 32 noted, **.NET 8 and .NET 9 leave support on November 10, 2026**; **.NET 10 LTS** runs to November 2028.
> - **Tooling referenced:** Azure Well-Architected Framework **Cost Optimization** pillar (five principles), **Infracost** (Terraform `azurerm`, and in 2026 ARM templates and compiled Bicep JSON), Azure Retail Prices API, NetArchTest/ArchUnitNET, Mapperly, CodeScene/code-maat, SonarQube, NDepend.
>
> Prices, policies and licences are "verified on this date"; the principles are stable.

## Orientation

Here is the sentence to carry through the whole module: **every architectural decision is an investment decision — you spend money, time and options now to buy future cash flows, lower risk and more options later — and the architect's job is to make that trade visible in the units the business uses (money, time, risk) and to choose it deliberately, rather than let it be made by default.**

The curriculum entry reads: *cost, build-vs-buy, and technical debt — the language executives respond to.* Module 32 ended with the promise to show *how to price the coexistence tax, the cost of delay and the cost of doing nothing, and how to argue for (or against) a migration in terms the business will fund.* Those three topics are one topic seen from three angles:

- **Cost** is what a system takes from the business every month — cloud bills, licences and, mostly, people.
- **Build-vs-buy** is the decision about *which* costs you take on: capital and ownership (build), or rent and dependency (buy).
- **Technical debt** is the cost you have deferred — and the interest you pay on it every time you change the system.

How this connects to earlier modules:

- **Module 5** (back-of-envelope estimation) gave you QPS, storage and bandwidth; this module turns those numbers into money.
- **Modules 6, 10, 26, 27** (scalability, caching, compute, data platform) contain most of the architectural cost levers.
- **Module 13 and 28** (reliability, SLOs, error budgets) — reliability has a price per nine, and this module shows how to compare it with the cost of downtime.
- **Module 21** (modular monolith vs microservices) — "infra multiplies sub-linearly; people multiply super-linearly" is a cost model; here it gets numbers.
- **Module 22** (DDD) — core, supporting and generic subdomains *are* the build-vs-buy map.
- **Module 23 and 11** discussed the MediatR and MassTransit licence changes; here they become a procurement decision.
- **Module 30 and 31** (design docs, ADRs) — a business case is a design doc for money; a debt register is an ADR log for shortcuts.
- **Module 32** (brownfield) — the coexistence tax, the cost of doing nothing and decommissioning savings are priced here.

Why it matters in interviews:

1. **Architect loops test business judgment explicitly.** Module 1's rubric includes *business alignment* and *trade-off reasoning*. Questions like *"How would you convince the CFO to fund this?"*, *"Build or buy for identity?"*, *"How do you manage technical debt?"* and *"The cloud bill doubled — what do you do?"* are standard.
2. **Cost is the trade-off candidates forget.** In design rounds most candidates discuss latency, consistency and availability; few say what the design costs per month or per request. Saying it is an immediate senior signal.
3. **Executives don't fund "refactoring".** They fund faster delivery, lower risk, lower run-cost and new revenue. Candidates who can translate are trusted with larger scopes.
4. **Build-vs-buy and debt are where engineers' biases show.** Interviewers listen for whether you default to building, dismiss debt as "the business's fault", or can say *"we should buy this, even though I'd enjoy building it."*

This module has six jobs:

1. **Build a first-principles economic toolkit** — TCO, time value of money, cost of delay, opportunity cost, risk in money, options thinking, CapEx/OpEx.
2. **Make you fluent in cloud cost engineering** — how bills work, architectural cost drivers, unit economics, rate and usage optimization, FinOps, Azure tooling, AI cost.
3. **Give you a rigorous build-vs-buy method** — core vs context, full TCO on both sides, lock-in and exit cost, open-source risk, vendor evaluation, the 2026 AI question.
4. **Treat technical debt as an economic concept** — what it is and isn't, types, how interest is paid, how to measure, when to borrow, how to repay.
5. **Teach you to make the case** — executive communication, business cases for migrations and paydowns, portfolio thinking, saying no.
6. **Ground everything in .NET and Azure**, and prepare you to narrate it under interview conditions.

Seven framings to carry through:

1. **People cost dominates.** In most software organizations engineering salaries dwarf the cloud bill; an architecture that saves $3k a month in compute but costs one extra engineer is a loss.
2. **Money has a time dimension.** A dollar of value next month is worth more than a dollar next year — and *delay* is usually the biggest hidden cost in any plan.
3. **Cost is a quality attribute.** It trades off against latency, availability and security like any other — so design for it, measure it, and give it a fitness function.
4. **Unit economics beat totals.** "The bill went up 40%" means nothing; "cost per order went from 3.1¢ to 2.4¢ while orders doubled" means everything.
5. **Build what differentiates; buy, rent or adopt what doesn't** — and price the whole lifetime of both, including exit.
6. **Technical debt is only expensive where you change code.** Interest is paid on change, so find the hotspots before you find the villains.
7. **Translate, don't lecture.** Executives decide in money, time and risk; the architect's job is to present the options in those units, with honest uncertainty, and a recommendation.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Why architects must speak money | Architecture is capital allocation; unfunded architecture doesn't exist |
| 2 | Cost as a quality attribute | It trades off with every other NFR — design, measure and test for it |
| 3 | Total cost of ownership | Build + run + change + people + risk + exit, over the system's whole life |
| 4 | Money over time | NPV, payback, IRR, discount rate, sensitivity — and why ROI alone misleads |
| 5 | Cost of delay | What waiting costs per week; CD3/WSJF to sequence work |
| 6 | Opportunity cost and the cost of doing nothing | The status quo is an option with a price, usually rising |
| 7 | Risk in money | Expected loss, cost of downtime, the price of each extra nine |
| 8 | Options thinking | Reversibility and deferral have value; buy options cheaply, exercise late |
| 9 | CapEx, OpEx, capitalization and tax | Same spend, different P&L and cash story — know enough to talk to finance |
| 10 | How cloud bills work | Meters × rates; four levers: usage, rate, architecture, waste |
| 11 | Architectural cost drivers | Compute, storage, transfer, minimum SKUs, observability, tokens — and people |
| 12 | Unit economics | Cost per transaction/tenant/feature; cost elasticity; the Frugal Architect |
| 13 | Rate optimization on Azure | PAYG, savings plans, reservations (Feb 2027 change), AHB, dev/test, spot |
| 14 | Usage optimization | Right-size, scale to zero, fewer/cheaper environments, tiering, log volume |
| 15 | FinOps | Inform → optimize → operate; personas; allocation; showback/chargeback; FOCUS |
| 16 | Azure cost tooling | Cost Management, budgets, anomalies, exports, Advisor, FinOps toolkit, Policy, Infracost |
| 17 | Cost in design | Price the design doc, cost in PRs, cost as a fitness function |
| 18 | AI and LLM cost | Tokens, caching, batch, routing, PTU break-even, the agent multiplier |
| 19 | Cloud cost anti-patterns | The recurring ways architectures burn money |
| 20 | Build vs buy, framed properly | Build, buy, rent, adopt, assemble, partner — a spectrum, not a binary |
| 21 | Core vs context | Core domain and Wardley evolution decide what deserves custom code |
| 22 | TCO of building vs buying | Building's sticker price is small; ownership and opportunity cost are not |
| 23 | Lock-in and exit cost | Lock-in is a cost to price, not a sin to avoid; switching cost × probability |
| 24 | Open source is not free | Supply-chain, sustainability and licence risk — the .NET relicensing wave |
| 25 | Evaluating vendors | Weighted criteria, PoC, due diligence, contract terms that protect the exit |
| 26 | .NET build-vs-buy calls | Identity, messaging, mediator/mapping, PDF, search, flags, observability |
| 27 | AI and build-vs-buy, 2026 | Agents cut the cost to write, not the cost to own |
| 28 | Technical debt, properly | Cunningham's metaphor: principal, interest, and what isn't debt |
| 29 | Types of technical debt | Quadrant; code, design, architecture, test, platform, data, knowledge, AI |
| 30 | How interest is paid | Lead time, defects, incidents, onboarding, unpredictability — on change |
| 31 | Measuring and finding debt | Hotspots, code health, DORA metrics, dependency age, surveys |
| 32 | Deliberate debt | When borrowing is right, and how to record the loan |
| 33 | Paying it down | Register, prioritize by interest, capacity allocation, fix-as-you-touch, no big-bang |
| 34 | Architecture and platform debt in .NET | Fitness functions, the runtime clock, dependency hygiene |
| 35 | Debt in the AI era | Faster code generation, faster debt accumulation; review and refactor budgets |
| 36 | Speaking to executives | Bottom line up front, three numbers, options with ranges, a recommendation |
| 37 | The business case for a migration or paydown | Status quo vs options; coexistence tax; staged funding with kill criteria |
| 38 | Portfolio thinking and saying no | Run/grow/transform, capacity allocation, trade-offs made explicit |
| 39 | Cost, buy and debt in the interview | Price the design, frame the decision, narrate the trade |

---
# Part A — The economic toolkit

## Concept 1 — Why architects must speak money

An architecture is a set of decisions that are expensive to change. "Expensive" is literal: every significant decision commits money (cloud spend, licences), time (engineering months), and options (what you can easily do later). That makes architecture a form of **capital allocation** — and capital allocation is what executives do all day.

**What executives actually optimize.** Strip away the jargon and most business decisions are about four things:

| Executive concern | Typical question | What it means for architecture |
|---|---|---|
| **Revenue / growth** | "Will this help us sell more or enter a new market?" | Time-to-market, new capabilities, scale, tenancy |
| **Cost / margin** | "What will this cost to build and run?" | Run cost per unit, people cost, licences |
| **Risk** | "What could go wrong, and how bad?" | Outages, breaches, regulatory exposure, key-person risk, vendor risk |
| **Speed / optionality** | "How fast can we respond to change?" | Lead time, cost of change, reversibility |

Engineers usually argue in a fifth currency — *technical quality* — which executives can't price. Technical quality matters *because* it moves the other four. The architect's skill is the translation.

**Three layers of cost the architect controls:**

1. **Build cost** — engineering time to create it. Visible, estimated, usually *under*estimated.
2. **Run cost** — infrastructure, licences, operations, on-call, support. Recurring, often for a decade.
3. **Change cost** — the cost of every future modification. Invisible at decision time, usually the largest over a system's life, and the one that technical debt inflates.

**Why "the architect speaks money" is not a soft skill.** Without a cost model you cannot answer the questions that matter: *Is the multi-region design worth it? Should we build our own workflow engine? Is it time to pay down this debt? Is the migration worth funding?* Every one of those is a comparison of costs and benefits over time under uncertainty. Without the toolkit, decisions default to the loudest voice, the latest fashion, or "it depends."

**The interview-grade sentence:** *"I treat architecture as capital allocation: each significant decision commits money, engineering time and future options, so I make the trade explicit in the units executives use — revenue, cost, risk and speed. I separate build cost, run cost and change cost, because the last one is invisible at decision time but usually the largest over a system's life, and it's the one technical debt inflates. Technical quality matters to the business because it moves those numbers, so my job is the translation."*

---

## Concept 2 — Cost as a quality attribute

Treat cost like latency or availability: a **quality attribute** with requirements, trade-offs, measurements and tests. The Azure Well-Architected Framework makes it a pillar for exactly this reason, and its first sentence is worth remembering: *a cost-optimized workload isn't necessarily a low-cost workload*. The goal is the best **value** within constraints, not the lowest bill.

**Cost trades off with everything:**

| Trade-off | Example |
|---|---|
| Cost ↔ availability | Zone redundancy roughly doubles some compute and storage; multi-region active-active can more than double total cost (Concept 7 prices it) |
| Cost ↔ latency | Caching (Module 10) costs memory to buy latency; premium tiers, provisioned throughput, larger SKUs |
| Cost ↔ consistency | Strong consistency in Cosmos DB costs more RUs per read than session or eventual (Module 7) |
| Cost ↔ security | Private endpoints, firewalls, DDoS protection, Defender plans, longer log retention |
| Cost ↔ observability | Log ingestion is often a top-five line item (Concept 19) |
| Cost ↔ speed of delivery | Managed services cost more per unit than self-hosted but much less in people time |
| Cost ↔ flexibility | Commitments (reservations, savings plans, enterprise agreements) buy discounts with lock-in |

**Make cost requirements explicit.** In the requirements phase (Module 4), ask for them like any other NFR:

- *"What's the budget envelope — monthly run cost at launch and at 10× scale?"*
- *"What should a unit cost — a tenant, an order, a document processed?"*
- *"Which matters more if we have to choose: the cost or the 99.95%?"*
- *"Are there committed spend agreements (a MACC — Microsoft Azure Consumption Commitment) we should consume?"*

**Cost requirements are often missing — supply them.** If nobody gives you a number, propose one: *"I'll assume the platform should run under $15k/month at launch and under 2¢ per order at scale; tell me if that's wrong."* An assumed number invites correction; no number invites a surprise.

**The WAF's five cost principles** are a useful checklist (current headings): **develop cost-management discipline** (a cost model, accountability, realistic budgets), **design with a cost-efficiency mindset** (baseline and guardrails; treat environments differently; don't design beyond planned growth), **design for usage optimization** (use what you pay for; scale dynamically; prefer active-active over idle standby when you've paid anyway), **design for rate optimization** (commitments for predictable usage, licence benefits, consumption pricing where utilization is low, cheaper regions where allowed, higher density), and **monitor and optimize over time** (capture and classify expense, alerts, continuous review, decommission).

**The interview-grade sentence:** *"I treat cost as a quality attribute: it has requirements, trade-offs, measurements and tests like latency or availability. It trades against availability, latency, consistency, security, observability and delivery speed, so I ask for a budget envelope and a target unit cost during requirements — and if nobody gives me one, I propose one so it can be corrected. As the Well-Architected Framework says, cost-optimized isn't the same as cheap; it's the best value within the constraints."*

---

## Concept 3 — Total cost of ownership

**TCO** is the full cost of a system or decision over its life — not the purchase price, not the first month's bill. Most bad cost decisions compare a visible number on one side with an invisible one on the other: *"the SaaS costs $90k a year, we can build it for $60k"* — where $60k is three months of two engineers and the next seven years of ownership are missing.

**A TCO model, by category:**

| Category | Build / one-off | Recurring |
|---|---|---|
| **People** | Design, development, testing, migration, training | Maintenance, upgrades, on-call, support, security patching, knowledge retention |
| **Infrastructure** | Environments set-up, data migration | Compute, storage, network, backup, DR, observability, non-production environments |
| **Licences / services** | Purchase, implementation services | Subscriptions, support contracts, per-seat/per-call fees, price escalations |
| **Integration** | Building integrations, data mapping | Keeping integrations working as both sides change |
| **Risk** | — | Expected cost of outages, breaches, compliance findings, vendor failure (Concept 7) |
| **Opportunity** | What else the team could have built (Concept 6) | Ongoing capacity tied up in ownership |
| **Exit** | — | Cost to migrate away or decommission at the end (Concept 23) |

**People cost, done properly.** Engineers cost much more than their salary. A **fully loaded cost** adds employer taxes, benefits, equipment, office/remote stipends, management overhead, recruiting and tooling — commonly **1.25–1.5× salary**, sometimes more. Use your finance team's figure if one exists; otherwise state the assumption.

| Illustration (assumptions stated) | US senior engineer | Serbian/CEE senior engineer |
|---|---|---|
| Salary | $180,000 | $70,000 |
| Loaded at 1.35× | $243,000 / year | $94,500 / year |
| Productive weeks per year (~46) | ≈ $5,283 / week | ≈ $2,054 / week |

So "two engineers for three months" in the US scenario is about **$137k** — before ownership. The relative numbers matter more than the absolute ones: an architecture decision that adds or removes half an engineer of ongoing ownership often outweighs any infrastructure saving.

**Lifetime matters.** Enterprise systems typically live 7–15 years. Build cost is paid once; ownership is paid every year. When you can't estimate maintenance precisely, use a **range** and a **sensitivity** (Concept 4) rather than leaving it out.

**Hidden TCO items people forget:**

- **Non-production environments** (dev, test, staging, performance, per-PR) — often 30–50% of total cloud spend in organizations that never review them.
- **Observability** — ingestion and retention.
- **Security and compliance work** — pen tests, audits, SOC 2 evidence, vulnerability remediation of everything you own.
- **Upgrades** — runtime (.NET LTS every two years), framework, OS, database version, SDKs, certificates.
- **Knowledge risk** — the cost of the one person who understands the custom thing leaving.
- **Decommissioning** — data archival, migration, contract notice periods.

**The interview-grade sentence:** *"I compare options on total cost of ownership over the system's expected life — build, infrastructure, licences, integration, risk, opportunity and exit — with people cost fully loaded, typically 1.25 to 1.5 times salary. The trap is comparing a visible number on one side, like a subscription fee, with an invisible one on the other, like three months of build with no ownership. I make sure non-production environments, observability, security and compliance work, upgrades, key-person risk and decommissioning are in the model, and where I can't estimate precisely I use ranges and sensitivity rather than leaving the line out."*

---

## Concept 4 — Money over time: NPV, payback, IRR and sensitivity

A cost now and a benefit later aren't directly comparable. Money has a **time value**: a dollar today can be invested, and a dollar promised in three years is uncertain. Finance teams handle this with **discounting**, and architects who can speak it get taken seriously.

**The core tools:**

| Measure | Definition | What it tells you | Watch out for |
|---|---|---|---|
| **ROI** | (benefit − cost) / cost | Simple ratio | Ignores timing and risk; a 100% ROI over 10 years is poor |
| **Payback period** | Time until cumulative cash flow turns positive | How long the money is at risk; executives like it | Ignores everything after payback |
| **NPV** (net present value) | Σ cash flowₜ / (1 + r)ᵗ | Value created in today's money at discount rate *r* | Sensitive to *r* and to far-future guesses |
| **IRR** (internal rate of return) | The *r* at which NPV = 0 | Comparable to the company's hurdle rate | Can mislead for odd cash-flow shapes; use with NPV |

**The discount rate** is set by finance — usually the company's **cost of capital** or a **hurdle rate** (commonly 8–15% for established companies; much higher for start-ups, where cash is scarce and risk is high). Ask for it; don't invent one if you can avoid it, and show results at two rates if you must.

**A worked example** (the Module 32 ordering migration, simplified, in $k, incremental vs the status quo):

| Year | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| Cash flow | −400 (build + coexistence) | −150 (more coexistence, early savings) | +300 | +350 | +350 |

- Sum of cash flows: **+450k**; simple ROI ≈ 82% on the 550k invested.
- **NPV at 10% ≈ +214k**; **NPV at 15% ≈ +127k**.
- **Payback ≈ 2.7 years**; **IRR ≈ 24.5%** — above a 10–15% hurdle.

Now the important part — **sensitivity**:

- If benefits come in **30% lower** (years 1–4), NPV at 10% ≈ **−25k**.
- If the investment overruns by **50%** (−600 in year 0, −250 in year 1), NPV at 10% ≈ **−77k**.

That's the real message for an executive: *"This is worth doing if we deliver close to plan; it destroys value if the benefits slip by 30% or the build overruns by half. So I propose funding it in stages, with a checkpoint after the first two slices measuring actual savings and velocity."* Sensitivity turns a single fragile number into a decision with a risk shape — and leads naturally to staged funding (Concept 37).

**A small C# helper** for the exercises — the logic is trivial; the discipline is in the inputs:

```csharp
public static class Finance
{
    // cashFlows[0] is "now" (year 0), positive = inflow, negative = outflow
    public static decimal Npv(decimal rate, IReadOnlyList<decimal> cashFlows)
    {
        decimal npv = 0m, factor = 1m;
        for (var t = 0; t < cashFlows.Count; t++)
        {
            npv += cashFlows[t] / factor;
            factor *= 1 + rate;
        }
        return npv;
    }

    public static double? PaybackYears(IReadOnlyList<decimal> cashFlows)
    {
        decimal cumulative = 0m;
        for (var t = 0; t < cashFlows.Count; t++)
        {
            var previous = cumulative;
            cumulative += cashFlows[t];
            if (previous < 0 && cumulative >= 0)
                return t - 1 + (double)(-previous / cashFlows[t]);   // linear interpolation within year t
        }
        return null; // never pays back within the horizon
    }

    // Scenario helper: scale benefits (positive flows after year 0) by a factor
    public static decimal[] ScaleBenefits(IReadOnlyList<decimal> flows, decimal factor) =>
        flows.Select((cf, t) => t > 0 && cf > 0 ? cf * factor : cf).ToArray();
}

// var plan = new decimal[] { -400, -150, 300, 350, 350 };
// Finance.Npv(0.10m, plan)        // ≈ 213.6
// Finance.PaybackYears(plan)      // ≈ 2.71
```

**Rules of thumb for architects:**

1. **Show cash flows by year**, not just totals — timing is the story.
2. **Lead with payback and NPV**, mention IRR if finance uses it, avoid bare ROI.
3. **Always show a downside scenario** (benefits −30%, costs +50%, delay +6 months) — it signals honesty and invites the right conversation.
4. **Front-load value** (Module 32's incremental slices): moving a benefit from year 3 to year 1 improves NPV and payback far more than squeezing costs.
5. **Distinguish cash from capacity.** "Saves 2 engineers" frees capacity for other work — valuable — but it saves cash only if headcount actually changes. Say which.

**The interview-grade sentence:** *"For investment decisions I lay out incremental cash flows by year and use payback and NPV at the company's discount rate rather than bare ROI, because timing and risk matter — a migration that costs 550k and returns 1M over four years has an NPV around 210k at 10% and pays back in under three years. Then I show sensitivity: if benefits come in 30% lower, or the build overruns by half, the NPV goes negative — which is exactly why I'd propose staged funding with a checkpoint on real savings. I'm also explicit about whether a benefit is cash or freed capacity."*

---

## Concept 5 — Cost of delay

Most plans count what things cost to *do* and ignore what they cost to *wait for*. **Cost of delay** (Don Reinertsen, *The Principles of Product Development Flow*) is the value lost per unit of time that a piece of work is not done — expressed as **money per week**. Reinertsen's provocation: if you quantify only one thing, quantify cost of delay.

**Where cost of delay comes from:**

| Source | Example | Shape |
|---|---|---|
| Revenue not earned | A feature that would add $40k/week in sales | Linear: every week costs the same |
| Cost not saved | Licence renewal at $25k/month that a migration would retire | Linear until the renewal date, then a **step** |
| Deadline | Regulatory requirement with fines from a date | Zero, then a cliff |
| Market window | Competitor launching; seasonal peak | Decays: value is lost permanently if late |
| Risk exposure | Unsupported runtime (.NET 8 after November 10, 2026) | Grows: probability of an incident accumulates |
| Learning | An experiment whose result unblocks a decision | Indirect: delays other decisions |

**CD3 — cost of delay divided by duration.** When work competes for the same team, schedule by **CoD ÷ duration**, highest first. It minimizes the total cost of delay across the portfolio (the same idea as weighted shortest job first).

Three items, cost of delay in $k per week, duration in weeks:

| Item | CoD ($k/week) | Duration (weeks) | CD3 |
|---|---|---|---|
| A — checkout improvement | 40 | 4 | 10 |
| B — retire licensed reporting tool | 15 | 1 | **15** |
| C — new partner integration | 60 | 10 | 6 |

Total delay cost by sequence (each item accrues its CoD until it finishes): **B → A → C = $1,115k**; A → B → C = $1,135k; … worst, **C → A → B = $1,385k**. The intuitive "do the biggest value first" (C) is the *worst* order here — a $270k difference created purely by sequencing.

**WSJF** in SAFe is the same idea with relative scoring: (business value + time criticality + risk reduction/opportunity enablement) ÷ job size. Useful when you can't get dollar figures; dollar figures are better when you can.

**Why architects care:**

1. **Big-bang projects maximize cost of delay** — nothing is delivered until everything is (Module 32). Incremental delivery is a cost-of-delay strategy.
2. **Technical debt raises the duration of every future item** — so it multiplies cost of delay across the whole roadmap (Concept 30).
3. **"Let's do it properly next quarter"** has a price; compute it.
4. **Platform deadlines are cost-of-delay cliffs** — runtime end of support, licence renewals, certificate expiry, regulatory dates. Put them on the roadmap as dated steps.

**The interview-grade sentence:** *"I quantify cost of delay — the value lost per week while something isn't done, from revenue not earned, costs not retired, deadlines, closing market windows or accumulating risk — because it's usually the largest hidden cost in a plan. When items compete for one team I sequence by cost of delay divided by duration; in a simple three-item example that alone changes total delay cost by over a quarter of a million. It's also why I prefer incremental delivery and why technical debt is so expensive: it stretches the duration of everything on the roadmap."*

---

## Concept 6 — Opportunity cost and the cost of doing nothing

**Opportunity cost** is the value of the best alternative you give up. A team that spends a quarter building an in-house feature-flag system has not spent that quarter on the product. The flag system's real cost is its build and ownership *plus* the product work that didn't happen.

**The cost of doing nothing.** Every proposal is implicitly compared with the status quo, and the status quo is usually presented as free. It almost never is:

| Status-quo cost | How to price it |
|---|---|
| **Rising run cost** | Licence escalations, extended-support fees, hardware refresh, growth on an inefficient architecture |
| **Rising change cost** | Lead time trend in the affected area; % of capacity spent on unplanned work |
| **Risk accumulation** | Unsupported components × probability of incident × impact (Concept 7) |
| **Lost revenue** | Deals lost because a capability is missing (ask sales for the list), churn attributed to quality |
| **Talent cost** | Attrition and hiring difficulty for obsolete stacks; premium contractor rates |
| **Deadlines** | Regulatory or contractual dates you'll miss |

**Present the status quo as an option with a price.** A business case with three columns — *do nothing*, *option A*, *option B* — where "do nothing" has a rising cost line is far more persuasive than one where it's blank. It also keeps you honest: sometimes "do nothing" is the right answer, and a good architect says so.

**Sunk cost is irrelevant.** Money already spent can't be recovered by continuing. *"We've invested two years in this platform"* is not an argument for spending another; the only question is whether the *next* dollar is better spent here or elsewhere. Executives know this in principle and forget it under pressure; architects who state it calmly help.

**The interview-grade sentence:** *"Every option has an opportunity cost — the best alternative use of the same people and money — and every business case has an implicit status-quo column, which is almost never free: run costs rise, change gets slower, unsupported components accumulate risk, deals are lost and people leave. So I present doing nothing as an option with its own rising cost line, I'm willing to recommend it when it's genuinely cheapest, and I keep sunk cost out of the argument — the only question is where the next dollar does the most good."*

---

## Concept 7 — Risk in money: expected loss and the cost of nines

Executives budget for risk constantly — insurance, reserves, hedging. Translate technical risk into the same form: **expected loss = probability × impact**, per year.

**Cost of downtime.** For a revenue-generating system, start with revenue per hour, then adjust:

- **Peak factor** — outages tend to happen under load, at peak; revenue per peak hour can be 2–5× the average.
- **Recovery** — some lost transactions retry later (reducing loss); some customers churn (increasing it).
- **Penalties** — SLA credits, contractual penalties.
- **Internal cost** — incident response, overtime, support load.
- **Reputation** — hard to price; name it, don't fake precision.

**The price of a nine.** Module 13 and 28 established that each extra nine costs more. Compare that cost with what it saves:

| Availability | Downtime per year | Expected direct revenue loss ($50M/yr revenue, flat) |
|---|---|---|
| 99.9% | 8.76 h | ≈ $50k |
| 99.95% | 4.38 h | ≈ $25k |
| 99.99% | 0.88 h | ≈ $5k |

Revenue per average hour is $50M ÷ 8,760 ≈ $5.7k. Going from 99.9% to 99.95% saves about **$25k/year** at average rates — **≈ $75k** with a 3× peak factor. If the multi-zone/multi-region design to get there costs **$180k/year** in infrastructure and operational effort, the investment doesn't pay on direct revenue alone. It may still be right — contractual SLAs, regulatory requirements, brand damage, or a business where an hour's outage at the wrong moment is catastrophic (payments on Black Friday). The point is to make that argument explicitly rather than assume "more nines = better."

**Other risks to put in money:**

| Risk | Expected-loss framing |
|---|---|
| Security breach | Probability (informed by exposure: unpatched, internet-facing, sensitive data) × (response + notification + fines + churn). Industry breach-cost reports give ranges |
| Unsupported runtime/dependency | Probability of an unpatched vulnerability being exploited or an audit finding × its cost; plus extended-support fees if available |
| Key-person dependency | Probability they leave this year × (time for someone else to become effective × their loaded cost + delay costs) |
| Vendor failure / relicensing | Probability × migration cost (Concepts 23–24) |
| Data loss | Probability × (recovery cost + regulatory + customer impact); compare with the cost of backup/DR tiers |

**Precision is not the goal; comparability is.** A risk estimated as "$50k–$300k per year" is far more useful than "high". It lets an executive compare it with the $120k mitigation you're proposing — which is their job.

**The interview-grade sentence:** *"I translate technical risk into expected annual loss — probability times impact — because that's how executives already think about insurance and reserves. For availability I start from revenue per hour, adjust for peak, recovery, penalties and response cost, and compare the saving from each extra nine with what it costs: for a $50M business, 99.9 to 99.95 saves maybe $25–75k a year, so a $180k multi-region design needs another justification such as contractual SLAs. I give ranges rather than false precision, because the goal is to make the risk comparable with the cost of mitigating it."*

---

## Concept 8 — Options thinking: the value of reversibility and waiting

Uncertainty makes **flexibility valuable**. Finance has a name for this — **real options**: the right, but not the obligation, to take an action later. An option has value even if you never exercise it, because it lets you act when you know more.

**Architectural options you can buy:**

| Option | What it costs now | What it buys |
|---|---|---|
| A **seam** (interface, module boundary, anti-corruption layer) | Some indirection and design time | The option to replace an implementation cheaply later (Module 32) |
| **Modular monolith** before microservices | Discipline to keep modules separate | The option to extract a service when the need is real (Module 21) |
| **Expand/contract** schema changes | Extra releases | The option to roll back each step |
| **Feature flags** | Flag management, two code paths | The option to turn things off; to release per cohort |
| **Short commitments**, savings plans over reservations | A smaller discount | The option to change SKU, region or architecture |
| **Standard protocols and data formats** (OIDC, OpenTelemetry, SQL, S3-compatible APIs) | Occasionally less feature-rich | The option to switch vendor (Concept 23) |
| **Spikes and prototypes** | Days of work | The option to decide with evidence (Module 30) |

**Don't overbuy options.** Every option has a carrying cost. "Database-agnostic repositories in case we switch databases" is usually an option nobody exercises (Module 20), bought at the price of every feature being harder. The test: *how likely is the change, how expensive would it be without the option, and how much does the option cost to carry?*

**The last responsible moment.** Defer irreversible decisions until the cost of not deciding exceeds the value of waiting for more information — not later (that's procrastination with interest) and not earlier (that's guessing). Amazon's **one-way vs two-way doors** (Module 30) is the same idea: decide two-way doors fast, one-way doors carefully.

**Build options cheaply, exercise them late.** The cheapest options are design choices — boundaries, contracts, standard interfaces. The most expensive are infrastructure hedges like running two clouds "just in case."

**The interview-grade sentence:** *"Under uncertainty flexibility has real value, so I think of seams, modular boundaries, expand-contract changes, feature flags, short commitments, standard protocols and spikes as options: they cost a little now and let us act cheaply when we know more. I buy options where change is plausible and expensive without them, and avoid carrying ones nobody will exercise — database-agnostic repositories and running two clouds just in case are the classic overpriced ones. I defer one-way-door decisions to the last responsible moment and make two-way doors quickly."*

---

## Concept 9 — CapEx, OpEx, capitalization and tax: enough to talk to finance

You don't need to be an accountant, but you need to understand why finance asks *"is this capex or opex?"* — because the same spend can look very different on the company's financial statements, and that changes what's easy to fund.

**The basic distinction:**

| | **CapEx** (capital expenditure) | **OpEx** (operating expenditure) |
|---|---|---|
| What | Buying or building an asset that provides value over multiple years | Running costs consumed in the period |
| Software examples | Capitalized development of internal-use software; perpetual licences; servers | Cloud consumption, SaaS subscriptions, maintenance, research, most bug fixing |
| P&L effect | Spread over the asset's useful life as **amortization/depreciation** | Expensed immediately |
| Why executives care | Improves near-term operating profit (EBITDA), but commits budget and creates an asset that can be impaired | Flexible, but hits operating profit now |

**The cloud shift.** Moving from owned servers to cloud turns hardware CapEx into consumption OpEx. Some companies prefer that (flexibility, no up-front cash); some resist it (it worsens operating margin). Commitments like reservations sit in between. Knowing which your CFO prefers explains a lot of "irrational" infrastructure decisions — and 37signals' widely discussed cloud exit (Concept 23) is partly a CapEx-vs-OpEx story.

**Capitalizing software development.** Under US GAAP, development of **internal-use software** can be capitalized once criteria are met, and then amortized over its useful life. **ASU 2025-06** replaces the old waterfall stages with a simpler threshold — management has authorized and committed funding, and it's *probable* the software will be completed and used as intended — which fits agile delivery far better (effective for annual periods beginning after December 15, 2027). Under **IFRS (IAS 38)**, which most companies outside the US use, development costs are capitalized only when specific criteria are demonstrated (technical feasibility, intention and ability to complete and use, probable future benefits, resources, reliable measurement).

**What this means for you as an architect:**

1. **Time tracking may matter.** Companies that capitalize development often need engineers' time classified (new capability vs maintenance). Don't sneer at it; it's how the work gets funded.
2. **"Maintenance" is usually OpEx; "new capability" can be CapEx.** Framing a platform upgrade as part of delivering new capability can change which budget pays for it. Be honest — finance and auditors will check.
3. **Write-offs are real.** If a capitalized system is abandoned, the remaining asset is written off — a visible hit. This makes executives reluctant to kill failing projects (sunk cost again, Concept 6), and is worth understanding when you propose replacing something recently capitalized.
4. **Tax treatment differs from accounting treatment.** In the US, Section 174 required software development costs to be capitalized and amortized for tax from 2022; **Section 174A** (2025) restored immediate deduction for **domestic** R&E, explicitly including software development, while **foreign** R&E is still amortized over 15 years. This affected start-up cash flow and offshore-vs-onshore cost comparisons.

**Say the boundary out loud.** *"Whether this is capitalized is finance's call; my job is to give them a clear breakdown of the work."*

**The interview-grade sentence:** *"I understand enough of the CapEx/OpEx distinction to work with finance: capital spend creates an asset amortized over years and flatters near-term operating profit, operating spend hits the period. Cloud turns hardware CapEx into consumption OpEx; capitalizable development of internal-use software is usually new capability, while maintenance is expensed — and US GAAP's ASU 2025-06 replaced the old waterfall stages with a probable-to-complete test that suits agile teams. Tax can differ again, as Section 174A showed. I don't make those calls; I give finance a clean breakdown of the work, and I remember that abandoning a capitalized system means a visible write-off."*

---
# Part B — Cloud cost engineering

## Concept 10 — How cloud bills work

A cloud bill is a sum of **meters × rates**. Every service emits usage on one or more meters (vCPU-hours, GB-months, GB transferred, requests, RUs, tokens); each meter has a price that depends on region, tier and any discounts you've bought. Understanding that structure is what lets you predict and change the bill.

**Pricing models you'll meet on Azure:**

| Model | Examples | Behaviour | Architectural implication |
|---|---|---|---|
| **Provisioned capacity** | VMs, App Service plans, AKS nodes, SQL vCores, Cosmos provisioned RU/s, Premium Service Bus | Pay for capacity whether used or not | Utilization is everything; idle = waste |
| **Autoscaled capacity** | VM scale sets, App Service autoscale, Cosmos autoscale (scales between 10% and 100% of max, billed on the highest RU/s per hour) | Pay for capacity that follows load, with floors | Floors and scale-in behaviour set the baseline |
| **Pure consumption** | Functions Flex Consumption, Container Apps consumption, Cosmos serverless, storage transactions, Log Analytics ingestion, tokens | Pay per unit of work | Cost scales linearly with traffic — good for spiky, dangerous for runaway loops |
| **Tiered / step** | API Management tiers, Service Bus tiers, Front Door tiers | Price jumps at tier boundaries | Minimums matter for small workloads; one feature can force a tier |
| **Commitment** | Reservations, savings plans, provisioned AI throughput reservations, enterprise agreements (MACC) | Discount for committing to spend or capacity | Rate lever, not a usage lever |

**The four levers of cloud cost** — every optimization falls into one:

1. **Usage** — consume less: right-size, scale down, turn off, delete, tier, sample (Concept 14).
2. **Rate** — pay less per unit: commitments, licence benefits, cheaper regions, negotiated discounts (Concept 13).
3. **Architecture** — need less: caching, batching, async, better data models, fewer hops, choosing the right compute model (Concepts 11, 17).
4. **Waste** — stop paying for nothing: orphaned disks, idle environments, forgotten test resources, unused licences (Concept 19).

**Usage × rate.** FinOps practitioners often say engineering owns usage and finance/procurement owns rate. Both are necessary; neither is sufficient. A 30% savings-plan discount on a wasteful architecture is still wasteful; a perfect architecture at list price leaves money on the table.

**List, contracted, effective.** FOCUS (Concept 15) distinguishes **list cost** (public price), **contracted cost** (after negotiated discounts) and **effective cost** (after amortizing commitments). When someone says "this costs $X", ask which.

**The interview-grade sentence:** *"A cloud bill is meters times rates: provisioned capacity you pay for whether used or not, autoscaled capacity with floors, pure consumption that scales with traffic, tiered services with minimums, and commitments that buy a lower rate. Every optimization pulls one of four levers — usage, rate, architecture or waste — and engineering mostly owns usage and architecture while finance owns rate, so both have to work together. And I always ask whether a number is list, contracted or effective cost."*

---

## Concept 11 — Architectural cost drivers

Most cloud spend is decided at design time. Know where the money goes so you can say, in a design review, *"this choice is the expensive one."*

| Driver | What drives it | Typical design levers |
|---|---|---|
| **Compute** | Peak capacity, floors, idle time, over-sized SKUs, per-instance minimums, sidecars and agents | Right compute model (Module 26), autoscale incl. to zero, density, async to flatten peaks, ARM-based SKUs where supported, Native AOT/trimming for smaller footprints (Module 17) |
| **Databases** | Provisioned throughput, vCores, storage, IOPS, replicas, backups | Partitioning and indexing (Modules 12, 27), caching reads (Module 10), read replicas only where needed, serverless/autoscale for spiky load, archiving cold data |
| **Cosmos DB specifically** | RU per operation × operations; cross-partition queries; indexing every property; strong consistency | Good partition key, point reads (≈1 RU for 1 KB), selective indexing policy, session consistency, TTL, change-feed instead of polling |
| **Storage** | GB-months by tier, transactions, redundancy (LRS/ZRS/GRS), snapshots | Lifecycle policies to cool/cold/archive, right redundancy per data class, delete what you don't need |
| **Data transfer** | Internet egress, cross-region and cross-zone traffic, private endpoint processing, NAT/firewall throughput | Keep chatty components in one region, CDN/Front Door caching, compress, avoid cross-region replication you don't need, beware "free" cross-zone assumptions |
| **Networking appliances** | Azure Firewall, Application Gateway/WAF, NAT Gateway, VPN/ExpressRoute — hourly plus per-GB | Share hub components across workloads; size to need |
| **Observability** | Log ingestion per GB, retention, high-cardinality metrics, trace volume | Sampling, log levels, basic/auxiliary table plans for verbose logs, retention by data class, metrics instead of logs for counts (Module 28) |
| **Messaging** | Tier minimums (Premium messaging units), operations, throughput units | Batch sends, right tier, avoid polling |
| **AI / tokens** | Tokens × price per model, retries, agent loops, context size | Concept 18 |
| **Licences on cloud** | Windows/SQL Server licence included in VM or SQL price | Azure Hybrid Benefit, Linux, PaaS that includes licensing efficiently |
| **People** | Number of things to operate, on-call rotations, bespoke components | Fewer moving parts, managed services, standardization (Module 21's people-cost curve) |

**Back-of-envelope cost estimation** extends Module 5. For a design, estimate the main meters at expected and 10× load:

```text
Orders API at 200 req/s average, 600 req/s peak, 1 KB documents, 30-day retention of logs

Compute:   peak 600 req/s ÷ ~400 req/s per 2-vCPU replica (measured) ≈ 2 replicas, keep 3 for HA
           → 3 × 2 vCPU always on; scale to 6 at peak
Cosmos:    reads 160/s as point reads ≈ 1 RU each = 160 RU/s; writes 40/s × ~6 RU (1 KB, lean index) = 240 RU/s
           → ~400 RU/s average, ~1,200 RU/s peak → autoscale max 1,500 RU/s
Logs:      200 req/s × 86,400 s × 2 KB per request of logs ≈ 34.6 GB/day ≈ 1 TB/month  ← probably the surprise
Egress:    200 req/s × 3 KB × 86,400 × 30 ≈ 1.55 TB/month
```

Then price the meters with the Azure Pricing Calculator or the Retail Prices API (Concept 16). The point of this exercise in an interview is not the dollar figure; it's spotting that **logging at 2 KB per request costs more than the database**, and saying what you'd do about it.

**People are the biggest architectural cost driver.** Module 21's point bears repeating with numbers: infrastructure for one more microservice may be $300–1,000 a month; the ongoing engineering ownership of it (pipelines, upgrades, on-call, contracts, debugging across a boundary) is a fraction of an engineer — easily ten times the infrastructure.

**The interview-grade sentence:** *"Most cloud spend is fixed at design time, so in a review I look at the drivers: compute floors and idle capacity, database throughput — in Cosmos that's RUs per operation, so partition keys, point reads, indexing policy and consistency level — storage tiers and redundancy, data transfer especially cross-region and egress, network appliances, observability volume, tokens, licences, and above all the number of things people have to operate. I do a back-of-envelope cost estimate like a capacity estimate, and it often reveals that something like logging at 2 KB a request costs more than the database."*

---

## Concept 12 — Unit economics

Totals mislead. A cloud bill that grew 40% is good news if the business grew 100%, and bad news if it grew 5%. **Unit economics** divides cost by a business driver so that cost can be compared over time and against revenue.

**Choosing the unit.** It should be something the business recognizes and that drives cost:

| Business | Good units |
|---|---|
| E-commerce | Cost per order, per active customer, per 1,000 page views |
| B2B SaaS | Cost per tenant (by plan tier), per active user, per API call |
| Payments | Cost per transaction, per $1,000 processed |
| Media | Cost per streaming hour, per GB delivered |
| AI product | Cost per conversation, per document processed, per successful task |

**Gross margin.** In SaaS, hosting and support costs (cost of goods sold, COGS) directly reduce gross margin, which investors watch closely. Knowing that each enterprise tenant costs $420/month to serve while paying $1,500 tells the business whether to change pricing, architecture, or tiers. Knowing that free-tier users cost $0.80/month each tells marketing what the free tier really costs.

**Cost elasticity.** How does cost grow with the driver? Ideally **sub-linearly** (economies of scale — shared infrastructure amortized over more units). If it grows linearly with no floor, fine; if it grows **super-linearly** (cross-partition fan-outs, N² chatty calls, log volume growing with tenants × features), the architecture has a scaling problem that will become a margin problem.

**The Frugal Architect.** Werner Vogels' seven laws (re:Invent 2023) are a good architect's checklist: *make cost a non-functional requirement*; *systems that last align cost to business*; *architecting is a series of trade-offs*; *unobserved systems lead to unknown costs*; *cost-aware architectures implement cost controls*; *cost optimization is incremental*; *unchallenged success leads to assumptions.* The second is the unit-economics law: cost should rise and fall with the thing that brings revenue.

**Instrumenting unit cost in .NET.** You can't get cost per tenant from the bill alone when tenants share resources. Emit the *usage* per tenant from the application, then allocate the bill proportionally. Cosmos DB tells you the RU charge of every operation; LLM APIs return token counts:

```csharp
using System.Diagnostics.Metrics;
using Microsoft.Azure.Cosmos;

public sealed class TenantUsageMetrics
{
    private readonly Counter<double> _requestUnits;
    private readonly Counter<long> _tokens;

    public TenantUsageMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("Contoso.Orders.Usage");
        _requestUnits = meter.CreateCounter<double>("contoso.cosmos.request_units", unit: "{RU}",
            description: "Cosmos DB request units consumed, by tenant and operation");
        _tokens = meter.CreateCounter<long>("contoso.ai.tokens", unit: "{token}",
            description: "LLM tokens consumed, by tenant, model and direction");
    }

    public void RecordCosmos(string tenantPlan, string operation, double requestCharge) =>
        _requestUnits.Add(requestCharge,
            new("tenant.plan", tenantPlan), new("db.operation", operation));

    public void RecordTokens(string tenantPlan, string model, string direction, long count) =>
        _tokens.Add(count,
            new("tenant.plan", tenantPlan), new("gen_ai.request.model", model), new("gen_ai.token.type", direction));
}

// Usage in a repository
var response = await container.ReadItemAsync<Order>(id, new PartitionKey(tenantId), cancellationToken: ct);
usage.RecordCosmos(tenant.Plan, "read", response.RequestCharge);
```

Two cautions. **Cardinality** — tagging metrics with thousands of tenant IDs can explode metric storage cost (the observability bill strikes again); tag by plan or segment in metrics, and put per-tenant detail in sampled logs or a periodic aggregation job. **Allocation rules** — decide and document how shared costs (the AKS cluster, the APIM instance, the platform team) are split: by measured usage, by fixed proportion, or held centrally. FOCUS 1.3+ adds explicit columns for split cost allocation precisely because everyone struggles with this.

**The interview-grade sentence:** *"I track unit economics rather than totals — cost per order, per tenant by plan, per conversation — because a bill growing 40% is good news if the business doubled. For SaaS that feeds directly into gross margin and pricing. I watch cost elasticity: cost should grow sub-linearly with the business driver, and super-linear growth is a scaling bug that will become a margin problem. In .NET I instrument usage at the source — Cosmos request charges and token counts as OpenTelemetry metrics tagged by plan, with per-tenant detail kept out of high-cardinality metrics — and allocate shared costs with documented rules."*

---

## Concept 13 — Rate optimization on Azure

Rate optimization lowers the price per unit without changing the architecture. Know the instruments and — especially after the 2026 policy change — their trade-offs.

| Instrument | What it is | Typical saving vs pay-as-you-go | Flexibility | Use for |
|---|---|---|---|---|
| **Pay-as-you-go** | List or contracted price | — | Total | Spiky, experimental, short-lived, or shrinking usage |
| **Azure savings plan for compute / databases** | Commit to an hourly **spend** for 1 or 3 years; applies across eligible services, SKUs and regions | Significant, less than reservations | High — follows your usage across eligible services | Steady baseline spend whose shape will change |
| **Reservations** | Commit to a specific resource type, SKU family, region, quantity for 1 or 3 (some 5) years | Highest | **Low, and falling**: from **February 1, 2027**, new reservations for savings-plan-covered services can't be exchanged; existing ones get one final exchange; trade-in to a savings plan remains | Truly stable workloads: known SKU, region and size for the term |
| **Azure Hybrid Benefit** | Use existing Windows Server / SQL Server licences with Software Assurance (or subscriptions) on Azure | Removes the licence component of VM/SQL prices | Depends on licence agreements | Windows/SQL estates with existing licences |
| **Dev/Test pricing** | Discounted rates for non-production in Dev/Test subscriptions/offers | Removes licence charges on many services | Non-production only | All non-production environments |
| **Spot VMs** | Spare capacity, evictable | Large | Can be evicted any time | Batch, CI agents, stateless workers that tolerate interruption |
| **Region choice** | Prices differ by region | Varies | Latency, data residency constraints | Non-production; latency-tolerant batch |
| **Negotiated agreements** (EA, MCA-E, MACC) | Volume discounts for committed consumption | Varies | Committed spend must be consumed | Organization level — architects should know whether a MACC exists |

**Commitment strategy in one paragraph.** Cover the **stable baseline** with commitments and leave the **variable top** on pay-as-you-go. Measure utilization of your commitments — an unused reservation is pure waste. Prefer **savings plans** where the workload's shape (SKU, region, compute service) may change — such as during a migration from VMs to Container Apps — and **reservations** only where you're confident for the term. The 2027 exchange change makes that distinction sharper: previously you could buy a reservation and fix mistakes by exchanging; soon you can't.

**Architects affect rate too.** Choosing **Linux** over Windows, **PaaS with licensing included efficiently**, a compute service covered by your organization's savings plan, or a region where your organization has capacity agreements are architecture decisions with rate consequences.

**AI capacity has the same shape.** Provisioned throughput for models (PTUs) is a reservation for tokens-per-minute capacity; monthly and annual reservations discount it further. Same rule: baseline on committed capacity, burst on pay-as-you-go (Concept 18).

**The interview-grade sentence:** *"Rate optimization lowers the price per unit: I cover the stable baseline with commitments and leave the variable top on pay-as-you-go. On Azure that means savings plans for spend that will change shape — especially during a migration — and reservations only for SKUs and regions I'm confident about for the term, which matters more now that new reservations for savings-plan-covered services lose exchangeability from February 2027. Then Azure Hybrid Benefit for existing Windows and SQL licences, Dev/Test pricing for non-production, Spot for interruptible work, and knowing whether the organization has a consumption commitment to draw down. Architecture choices like Linux over Windows affect rate too."*

---

## Concept 14 — Usage optimization

Usage optimization reduces what you consume. It is where engineering has the most leverage — and where the savings are permanent rather than a discount.

**The main moves, ranked roughly by effort-to-savings ratio:**

| Move | What to do | Watch out for |
|---|---|---|
| **Delete waste** | Orphaned disks, unattached public IPs, old snapshots, abandoned resource groups, idle App Service plans, stopped-but-not-deallocated VMs | Ownership is often unclear — tagging (Concept 15) fixes that |
| **Turn off non-production** | Schedule dev/test off nights and weekends (≈ 65% fewer hours); ephemeral per-PR environments torn down automatically | Make it the default, with an opt-out, not an opt-in |
| **Right-size** | Reduce SKUs to measured utilization (CPU, memory, DTU/vCore, RU/s) with headroom | Size to p95 load + headroom, not average; check memory, not just CPU |
| **Autoscale, including to zero** | Container Apps and Functions scale to zero; App Service and AKS autoscale on real signals | Cold starts (Module 26), minimum replicas for HA in production |
| **Fewer environments** | Does every team need dev, test, QA, staging, perf, pre-prod? Can non-production share a cluster? | Isolation needs (security, data) — be deliberate, not dogmatic |
| **Cheaper non-production SKUs** | Basic/Standard tiers, LRS storage, single instance, short log retention | Performance tests need production-like sizing — run them on demand |
| **Data lifecycle** | Blob lifecycle policies to cool/cold/archive; TTL in Cosmos; partition and archive old SQL data | Retrieval costs and latency from archive tiers; compliance retention |
| **Observability volume** | Sampling, log levels per category, drop noisy health-check logs, basic/auxiliary log plans for verbose tables, retention by table | Don't sample away the data you need for incidents; keep errors unsampled |
| **Efficient code** | Fewer allocations and CPU per request (Module 17), fewer DB round trips (Module 19's N+1), caching | Measure first — micro-optimization rarely moves the bill unless at scale |

**Find the big rocks first.** Sort the bill by service and by resource; the top 10 lines are usually 70–80% of the spend. Optimizing a $40/month function is noise; a 20% reduction on a $30k/month Cosmos account is $72k a year.

**Measure log volume per table.** Observability is so often the surprise that it deserves a standard query. In a Log Analytics workspace:

```kql
// Billable ingestion by table over the last 30 days (Quantity is in MB)
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize IngestedGB = round(sum(Quantity) / 1000, 1) by DataType
| sort by IngestedGB desc
```

Typical findings: verbose `AppTraces` from Information-level logging in a hot path, dependency telemetry for every Redis call, health-probe requests logged as requests, duplicated logs (stdout plus SDK). Fixes go in code and configuration: `appsettings.json` log-level filters per category, OpenTelemetry sampling, and Data Collection Rules that filter or route verbose tables to cheaper plans.

**Usage optimization is continuous.** Workloads drift; new features add meters; teams spin up resources. Treat it as part of operating the system — a monthly review of the top lines per workload, with an owner — not a once-a-year cost-cutting exercise.

**The interview-grade sentence:** *"Usage optimization is where engineering has the most leverage and the savings are permanent: delete waste, turn non-production off outside working hours or make it ephemeral, right-size to p95 plus headroom, autoscale including to zero where cold starts are acceptable, reduce and downsize environments, apply data lifecycle policies, and control observability volume with sampling, per-category log levels and cheaper log plans for verbose tables. I start with the top ten lines of the bill, which are usually most of the spend, and I make the review continuous with an owner per workload."*

---

## Concept 15 — FinOps

**FinOps** is the operating model for managing technology spend collaboratively across engineering, finance and the business. The FinOps Foundation defines it around maximizing the business **value** of technology — not minimizing spend. As of the 2026 Framework it explicitly covers public cloud, SaaS, licensing, data centre and AI (*Scopes*) and includes **Executive Strategy Alignment** as a capability.

**The core model:**

| Element | Content |
|---|---|
| **Principles** | Teams collaborate; business value drives technology decisions; everyone takes ownership of their usage; data should be accessible, timely and accurate; FinOps is enabled centrally; take advantage of the variable cost model (paraphrased) |
| **Phases** (iterative) | **Inform** (visibility, allocation, benchmarking) → **Optimize** (usage and rate) → **Operate** (continuous improvement, governance, automation) |
| **Domains** | Understand usage and cost; quantify business value; optimize usage and cost; manage the FinOps practice |
| **Personas** | FinOps practitioner, engineering, finance, procurement, product, leadership — each with different needs |
| **Maturity** | Crawl → walk → run, per capability; don't try to run everything at once |

**Allocation is the foundation.** You can't make anyone accountable for costs they can't see. Allocation means every cost is attributed to a **workload, team, product and environment**. On Azure the tools are:

- **Hierarchy** — management groups, subscriptions per workload/environment, resource groups per component. Subscription boundaries are the cleanest allocation boundary.
- **Tags** — `workload`, `owner`, `costCenter`, `environment`, and for transitional pieces `lifecycle=transitional` and `remove-after` (Module 32). Enforce with Azure Policy; inherit from resource groups where sensible.
- **Shared-cost rules** — how platform components (hub networking, shared AKS, APIM, observability workspace) are split.

```bicep
// Tag every resource in the workload consistently from one place
param workload string = 'orders'
param environment string = 'prod'
param costCenter string = 'CC-4410'
param owner string = 'team-orders@contoso.com'

var tags = {
  workload: workload
  environment: environment
  costCenter: costCenter
  owner: owner
}

resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-${workload}-${environment}'
  location: resourceGroup().location
  sku: { name: 'P1v3' }
  kind: 'linux'
  properties: { reserved: true }
  tags: tags
}
```

Enforcement uses Azure Policy — the built-in **"Require a tag on resources"** (deny) or **"Inherit a tag from the resource group if missing"** (modify) definitions, assigned at management-group or subscription scope. Deny is strict and can break deployments of resource types that don't support tags well; modify/inherit is gentler. Pick per tag.

**Showback and chargeback.** **Showback** reports each team's costs to them (and their leadership); **chargeback** actually bills them against their budget. Showback is usually enough to change behaviour and avoids internal accounting wars; chargeback makes sense when business units have real P&L ownership.

**FOCUS** gives billing data a common schema across Azure, AWS, Google, Oracle and a growing list of SaaS and data platforms — consistent column names like `BilledCost`, `EffectiveCost`, `ListCost`, `ServiceCategory`, `ChargeCategory`, `CommitmentDiscountId`. Its value for an architect: multi-cloud and SaaS spend can be reported in one model, and allocation and unit-economics queries are portable. Azure Cost Management can export in FOCUS format; the FinOps toolkit's FinOps hubs ingest and normalize it.

**The interview-grade sentence:** *"FinOps is the operating model that makes engineering, finance and the business jointly accountable for the value of technology spend — inform, optimize, operate, iteratively — and as of the 2026 framework it covers SaaS, licences and AI as well as cloud, with executive strategy alignment as an explicit capability. Its foundation is allocation: subscriptions per workload and environment, mandatory tags enforced with Azure Policy, and documented rules for shared platform costs. I prefer showback before chargeback, and FOCUS-format exports so cost data has one schema across providers."*

---

## Concept 16 — Azure cost tooling

Know which tool does what, so you can design cost governance into a platform rather than bolt it on.

| Tool | Purpose | Notes |
|---|---|---|
| **Azure Pricing Calculator** | Pre-deployment estimates | Save and share estimates; record the date you priced (Module 30) |
| **Azure Retail Prices API** | Programmatic, unauthenticated list prices for all meters | Good for estimating tools, PR checks, design-doc tables |
| **Cost Management — cost analysis** | Actual and amortized cost by any dimension (service, resource group, tag, meter) | Use *amortized* view to see commitments spread over time |
| **Budgets** | Alerts at actual or forecast thresholds, by scope and filter | Can trigger action groups (email, webhook, automation) |
| **Anomaly detection and alerts** | ML-based detection of unusual daily spend at subscription scope | Catches runaway loops and misconfigurations within a day |
| **Exports** | Scheduled cost/usage data to storage, including **FOCUS** format and price sheets, with backfill | Feed your own reporting, FinOps hubs, or a data platform |
| **Azure Advisor — cost** | Right-sizing, idle resources, commitment purchase recommendations | Recommendations are inputs, not orders — check against SLOs |
| **Microsoft FinOps toolkit** | Open-source: **FinOps hubs** (data pipeline on Azure Data Explorer/Fabric), Power BI reports, workbooks, PowerShell, optimization engine | Monthly releases; aligned to the FinOps Framework |
| **Azure Policy** | Guardrails: allowed SKUs/regions, required tags, deny expensive resource types in sandboxes | Prevents waste instead of reporting it |
| **AKS cost analysis add-on** | Cost by namespace and workload inside clusters (OpenCost-based) | Shared clusters need this for allocation |
| **Infracost** (third-party) | Cost diff of infrastructure-as-code changes in pull requests | Terraform `azurerm`; in 2026, ARM templates and compiled Bicep JSON |

**A budget as code** — every workload gets one at creation, with a forecast alert that fires before the money is spent:

```bicep
targetScope = 'resourceGroup'

param monthlyBudget int = 15000
param contactEmails array = [ 'team-orders@contoso.com' ]
param startDate string = '2026-11-01'

resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: 'budget-orders-prod'
  properties: {
    category: 'Cost'
    amount: monthlyBudget
    timeGrain: 'Monthly'
    timePeriod: { startDate: startDate }
    notifications: {
      actualOver80: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        thresholdType: 'Actual'
        contactEmails: contactEmails
      }
      forecastOver100: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Forecasted'
        contactEmails: contactEmails
      }
    }
  }
}
```

*(Check the latest API version for `Microsoft.Consumption/budgets` in your Bicep tooling.)*

**Querying list prices from .NET** — useful for an estimating tool or a design-doc generator:

```csharp
using System.Net.Http.Json;
using System.Text.Json.Serialization;

public sealed record RetailPrice(
    [property: JsonPropertyName("retailPrice")] decimal RetailPriceValue,
    [property: JsonPropertyName("unitOfMeasure")] string UnitOfMeasure,
    [property: JsonPropertyName("productName")] string ProductName,
    [property: JsonPropertyName("skuName")] string SkuName,
    [property: JsonPropertyName("meterName")] string MeterName,
    [property: JsonPropertyName("armRegionName")] string Region);

public sealed record RetailPricePage(
    [property: JsonPropertyName("Items")] List<RetailPrice> Items,
    [property: JsonPropertyName("NextPageLink")] string? NextPageLink);

public static async IAsyncEnumerable<RetailPrice> GetPricesAsync(HttpClient http, string filter)
{
    string? url = "https://prices.azure.com/api/retail/prices?$filter=" + Uri.EscapeDataString(filter);
    while (url is not null)
    {
        var page = await http.GetFromJsonAsync<RetailPricePage>(url)
                   ?? throw new InvalidOperationException("Empty response");
        foreach (var item in page.Items) yield return item;
        url = page.NextPageLink;
    }
}

// await foreach (var p in GetPricesAsync(http,
//     "serviceName eq 'Virtual Machines' and armRegionName eq 'westeurope' " +
//     "and armSkuName eq 'Standard_D4s_v5' and priceType eq 'Consumption'"))
//     Console.WriteLine($"{p.ProductName} {p.SkuName} {p.MeterName}: {p.RetailPriceValue} per {p.UnitOfMeasure}");
```

List prices are a starting point; your effective prices depend on agreements and commitments. For the real numbers, use your Cost Management price sheet export.

**Governance design pattern:** budgets and anomaly alerts per workload (detect) → Policy guardrails on SKUs, regions and tags (prevent) → monthly cost review of top lines with owners (correct) → FOCUS exports into a reporting model with unit economics (understand).

**The interview-grade sentence:** *"On Azure I design cost governance in four layers: prevent with Azure Policy guardrails on SKUs, regions and required tags; detect with a budget and forecast alert per workload, deployed as code, plus anomaly alerts; understand through FOCUS-format exports into the FinOps toolkit or our data platform, with amortized views and unit economics; and correct through a monthly review of the top lines with named owners and Advisor recommendations treated as inputs. Before deployment I estimate with the pricing calculator or the Retail Prices API, and I like Infracost-style cost diffs in pull requests."*

---

## Concept 17 — Cost in design: estimates, pull requests and fitness functions

Cost should appear at three points in the delivery lifecycle: in the **design doc**, in the **pull request**, and in **production feedback**.

**1. In the design doc** (Module 30). Add a *Cost* section with:

- A **cost model**: main meters, quantities at launch and at 10×, prices with the date they were checked.
- **Unit cost** at both volumes and the trend between them (elasticity).
- **Cost of alternatives** — the cheaper and the more expensive option you rejected, and why.
- **People cost** of operating the design (on-call, upgrades, components to own).
- **Assumptions and sensitivity** — what would make it 2× more expensive.

Reviewers should ask cost questions as routinely as latency questions: *"What's the most expensive line? What does it cost per tenant? What happens to cost if a tenant sends 10× the traffic?"*

**2. In the pull request.** Infrastructure changes are spend changes. A cost diff on IaC PRs (Infracost for Terraform, ARM or compiled Bicep; or an internal tool on the Retail Prices API) makes *"this adds $1,900/month"* visible before merge. Set thresholds that require an extra approver above some amount — a **cost guardrail in the workflow**, not a monthly surprise.

**3. In production — cost as a fitness function.** An **architectural fitness function** (Ford, Parsons, Kua — *Building Evolutionary Architectures*) is an automated check that a quality attribute stays within bounds. Cost fitness functions:

| Fitness function | How |
|---|---|
| Cost per unit stays below target | Daily job: allocated cost ÷ business volume; alert on threshold or trend |
| No untagged resources | Azure Policy compliance; fail the pipeline on non-compliant deployments |
| Non-production off outside hours | Scheduled check of running resources in dev subscriptions |
| Log ingestion per request bounded | Query ingestion GB ÷ request count; alert on regression after a release |
| RU per operation bounded | Track Cosmos request charge per operation type in tests and production; fail a perf test if a query's RU jumps |
| Token cost per conversation bounded | Track tokens per successful task; alert on regressions after prompt or model changes |

A cheap and effective one for .NET teams using Cosmos DB: assert request charges in integration tests against the emulator or a test account, so a missing index or an accidental cross-partition query fails CI:

```csharp
[Fact]
public async Task Order_lookup_by_id_is_a_point_read()
{
    var response = await _container.ReadItemAsync<Order>(_orderId, new PartitionKey(_tenantId));
    Assert.True(response.RequestCharge <= 2.0,
        $"Expected a point read (~1 RU for 1 KB), got {response.RequestCharge} RU");
}

[Fact]
public async Task Recent_orders_query_stays_in_one_partition_and_under_budget()
{
    var query = _container.GetItemQueryIterator<Order>(
        new QueryDefinition("SELECT * FROM o WHERE o.tenantId = @t AND o.createdAt > @since ORDER BY o.createdAt DESC")
            .WithParameter("@t", _tenantId).WithParameter("@since", DateTime.UtcNow.AddDays(-7)),
        requestOptions: new QueryRequestOptions { PartitionKey = new PartitionKey(_tenantId), MaxItemCount = 50 });

    var page = await query.ReadNextAsync();
    Assert.True(page.RequestCharge <= 20, $"Recent-orders query cost {page.RequestCharge} RU");
}
```

**The interview-grade sentence:** *"I put cost in three places: the design doc has a cost model with meters at launch and 10×, unit cost and elasticity, the cost of rejected alternatives, people cost and sensitivity; pull requests that change infrastructure show a cost diff, with an extra approver above a threshold; and in production I use cost fitness functions — cost per unit against target, no untagged resources, non-production off at night, log bytes per request, and in .NET with Cosmos even request-charge assertions in integration tests so an accidental cross-partition query fails CI."*

---

## Concept 18 — AI and LLM cost

AI features have a cost profile unlike most of your system: cost per request can be **cents rather than micro-cents**, varies with input and output size, and multiplies with agent loops. The FinOps community's statistic that 98% of FinOps teams now manage AI spend is a measure of how quickly this became material.

**The cost equation for one LLM call:**

```text
cost ≈ (uncached input tokens × input price) + (cached input tokens × cached price)
     + (output tokens × output price)          [+ tool calls, retrieval, embeddings, retries]
```

**Worked example (illustrative prices — check current ones):** a support assistant with a 2,000-token system prompt and instructions, 1,000 tokens of user context, 500 output tokens; input $2 per million tokens, output $8 per million.

| Scenario | Cost per request | At 1M requests/month |
|---|---|---|
| Baseline | 3,000 × $2/M + 500 × $8/M = **$0.0100** | **$10,000** |
| Prompt caching on the 2,000-token prefix (cached input billed at ~10% in this illustration) | **$0.0064** | **$6,400** (−36%) |
| Plus routing 70% of requests to a small model priced at ~1/10 | **≈ $0.0024** | **≈ $2,370** |

Three architectural levers did more than any negotiation could: **cache stable prefixes** (structure prompts so the static part comes first), **route by difficulty** to the cheapest model that meets the quality bar (measure with evaluations), and **trim context** (retrieval that sends five relevant chunks rather than fifty).

**Other levers:**

| Lever | Use when |
|---|---|
| **Batch API** (about 50% off Global Standard on Azure, ~24h target) | Offline enrichment, classification, evaluation runs, backfills |
| **Provisioned throughput (PTUs)** | Sustained, predictable load where latency consistency matters; break-even depends on utilization — compute it from *measured* throughput, keep burst on Standard |
| **Data Zone vs Global vs Regional** | Data residency may require Data Zone/Regional at a higher price — a compliance cost to put in the business case |
| **Output limits and structured outputs** | Output tokens are the expensive ones; ask for concise, structured answers |
| **Semantic caching** of whole responses | Repeated questions (FAQ-like traffic) — with care about staleness and personalization |
| **Deterministic code first** | Don't call a model for what a regex, a lookup table or a rules engine can do |

**The agent multiplier.** An agent that plans, calls tools and re-reads growing context may make 10–50 model calls per task, each with a larger context than the last. Cost per *successful task* — not per call — is the unit to watch, along with caps: maximum steps, maximum tokens per task, budget per tenant per day. Retries and loops are the AI equivalent of a runaway autoscaler; anomaly alerts on token meters are worth having from day one.

**Unit economics matter more here than anywhere.** If a feature costs 4¢ per use and the customer pays $20/month, heavy users can turn the feature margin-negative. Model cost per user segment *before* pricing the feature.

**The interview-grade sentence:** *"AI cost is per token and can be cents per request, so I model it explicitly: input, cached input and output tokens times price, plus retrieval and retries. The big levers are architectural — put stable instructions first so prompt caching applies, route requests to the cheapest model that passes our evaluations, trim retrieved context, cap output — and then commercial: Batch at roughly half price for offline work, provisioned throughput only when measured utilization beats pay-as-you-go, with burst on Standard. For agents I track cost per successful task with step and token caps and anomaly alerts, and I check that heavy users don't make the feature margin-negative."*

---

## Concept 19 — Cloud cost anti-patterns

Recognizing these in a design review is a strong signal — and many have caused real bill shocks.

| Anti-pattern | Why it's expensive | Fix |
|---|---|---|
| **The log firehose** | Information-level logging of every request/dependency in hot paths; ingestion often rivals compute | Log levels per category, sampling, metrics for counts, cheaper table plans, retention by class |
| **Always-on non-production** | Dev/test running 168 h/week, often at production SKUs | Schedules, ephemeral environments, smaller SKUs |
| **Chatty microservices across zones/regions** | Data transfer charges, network appliances, latency → bigger instances (Module 21) | Co-locate, coarser APIs, async, modular monolith |
| **Cross-region everything** | Geo-replication of data nobody needs in two regions; transfer charges | Replicate by data class and RPO; don't default to GRS/multi-region writes |
| **Cosmos DB as a relational database** | Cross-partition queries and full indexing burn RUs; hot partitions force over-provisioning | Model for access patterns, good partition key, selective indexing, change feed (Module 27) |
| **Polling** | Timers hitting databases or queues every second, 24/7 | Events, change feed, long-polling consumers, backoff |
| **Premium-by-default** | Premium SKUs for every environment "to be safe" | Size from measurements; premium only where a feature or SLO requires it |
| **Idle provisioned capacity** | Provisioned RU/s, vCores and nodes sized for annual peak | Autoscale, serverless, scale-to-zero, scheduled scaling |
| **NAT/firewall hairpinning** | All egress through centralized appliances billed per GB | Private endpoints, service endpoints, route only what policy requires |
| **Unbounded retention** | Logs, backups, blobs, events kept forever by default | Lifecycle and retention policies, archived with an access path |
| **Runaway consumption** | Recursive triggers, retry storms, agent loops on consumption-priced services | Budgets, anomaly alerts, concurrency caps, idempotency (Modules 11, 13) |
| **Orphans and zombies** | Disks, IPs, snapshots, gateways left behind | Tag ownership, policy, periodic cleanup automation |
| **Commitments bought blind** | Reservations for workloads about to migrate; unused savings | Commit after the architecture stabilizes; track utilization |
| **Optimizing pennies** | Weeks of engineering to save $200/month | Compute engineering cost vs savings; start with the top lines |

The last one matters: **engineering time is the most expensive resource you have.** Spending two weeks of a senior engineer (~$10k loaded) to save $150/month pays back in over five years — usually a bad trade, unless it's also simpler.

**The interview-grade sentence:** *"The recurring cloud cost anti-patterns I look for are the log firehose, always-on non-production, chatty services across zones and regions, cross-region replication by default, Cosmos used like a relational database, polling, premium-by-default and idle provisioned capacity, centralized egress appliances billed per GB, unbounded retention, runaway consumption from retry storms or agent loops, orphaned resources and commitments bought before the architecture stabilized. And I avoid the opposite mistake — spending weeks of engineering time, the most expensive resource we have, to save pennies."*

---
# Part C — Build vs buy

## Concept 20 — Build vs buy, framed properly

"Build or buy?" is really a choice among **sourcing strategies** along a spectrum of control versus ownership burden:

| Strategy | What you get | What you own | Examples |
|---|---|---|---|
| **Build** | Exactly what you need, full control | Everything: code, ops, security, upgrades, knowledge | Pricing engine, domain workflows |
| **Assemble** | Custom solution from components and libraries | Your glue and the risk of each component | ASP.NET Core + EF Core + Service Bus + OSS libraries |
| **Adopt open source (self-hosted)** | Mature capability, no licence fee (usually) | Operations, upgrades, security patching, licence compliance, supply-chain risk | Keycloak, PostgreSQL, Grafana stack, Valkey |
| **Rent managed / PaaS** | The capability operated by the cloud provider | Configuration, integration, cost, the provider dependency | Azure SQL, Service Bus, Entra ID, Azure AI Search |
| **Buy (licensed product)** | A vendor's product, often self-hosted or embedded | Integration, customization, upgrades, licence compliance | Commercial libraries (PDF, reporting), on-prem products |
| **Subscribe (SaaS)** | A complete business capability | Configuration, integration, data export, vendor management | Salesforce, Zendesk, Stripe, Auth0 |
| **Outsource / partner** | Someone else builds or runs it | Contract management, knowledge transfer, quality oversight | Agency builds a mobile app |

**The decision has more than two outcomes and it's not permanent.** Many good architectures are deliberate mixes: SaaS for the CRM, managed services for infrastructure, open-source libraries assembled into custom code for the core domain — and a planned path to change sourcing as the business evolves (buy now to learn, build later when it becomes differentiating; or build now because nothing fits, replace with a product when the market matures).

**The questions that decide it** — in roughly this order:

1. **Is it core?** Does doing this better than competitors create advantage? (Concept 21)
2. **Does a good-enough product exist?** Mature market, standard practices, fits 80%+ of needs without heavy customization?
3. **What's the full TCO of each option over the expected life?** (Concept 22)
4. **What's the time-to-value and cost of delay?** Buying usually wins on time (Concept 5).
5. **What's the risk profile?** Vendor viability, lock-in and exit cost (Concept 23), security and compliance, key-person risk of building.
6. **Do we have — or want — the capability?** Skills, team capacity, desire to own it long term.
7. **How do data and integration work?** Who owns the data; how hard is it to get out; how many integration points.

**The engineer's bias.** Engineers tend to over-build: building is fun, products have annoying limitations, and the cost of ownership is invisible at decision time. Product and finance people sometimes over-buy: a product demo looks complete, and integration and customization costs are invisible. The architect's job is to correct both biases with the same TCO model.

**The interview-grade sentence:** *"Build versus buy is really a spectrum of sourcing options — build, assemble, adopt open source, rent a managed service, license a product, subscribe to SaaS or partner — trading control against ownership burden, and good architectures mix them deliberately and revisit the choice as the business changes. I decide by asking whether the capability is core, whether a good-enough product exists, the full lifetime TCO, time-to-value, the risk and exit profile, and whether we have and want the capability — and I consciously correct for engineers' tendency to over-build and for the demo effect that makes buying look complete."*

---

## Concept 21 — Core vs context: what deserves custom code

The most important build-vs-buy question is strategic, not financial: **does this capability differentiate the business?**

**Three lenses that agree more often than not:**

**1. Geoffrey Moore's core vs context.** *Core* activities create sustainable differentiation; *context* is everything else needed to run the business. Invest in core; minimize, standardize and outsource context. A great payroll system doesn't win customers for a logistics company.

**2. DDD subdomains** (Module 22). The **core domain** is where the business competes — build it, with your best people, with rich models. **Supporting subdomains** are specific to your business but not differentiating — build simply, or buy/customize if possible. **Generic subdomains** are solved problems (identity, billing, email, document generation, search) — buy, rent or adopt. The DDD Crew's **core domain chart** plots business differentiation against model complexity to make this visible.

| Subdomain | Example in an e-commerce marketplace | Default sourcing |
|---|---|---|
| Core | Seller ranking and pricing algorithms, fraud-adjusted checkout flow | Build |
| Supporting | Seller onboarding workflow, returns policy rules | Build simply, or configure a product |
| Generic | Identity, payments processing, email delivery, tax calculation, search engine, observability | Rent/buy/adopt |

**3. Wardley mapping.** Map the value chain from user need down to components, and place each component on an **evolution axis**: *genesis* (novel, uncertain) → *custom-built* → *product (+rental)* → *commodity (+utility)*. Each stage calls for different methods: genesis and custom for build and experimentation; product and commodity for buy and outsource. Components evolve over time — something you built in 2015 (a message broker, a container orchestrator, a feature-flag service) may be a commodity now. **Wardley's key insight for build-vs-buy:** building something that has become a commodity is waste; buying something still in genesis is impossible or premature.

**Differentiation can be in the composition, not the parts.** A company's advantage might come from how it combines commodity parts — then the *integration and orchestration* is core, even though each component is bought.

**Watch for "core by vanity".** Teams sometimes declare their infrastructure core because it's interesting: a homegrown service mesh, a custom ORM, an in-house job scheduler. The test: *would a customer notice or pay for it if it were twice as good?* If not, it's context.

**The interview-grade sentence:** *"The first build-versus-buy question is strategic: does the capability differentiate us? I use three lenses that usually agree — Moore's core versus context, DDD's core, supporting and generic subdomains, and Wardley mapping's evolution axis from genesis through custom and product to commodity. Build the core domain with the best people; build supporting subdomains simply or configure a product; rent, buy or adopt generic ones like identity, payments, email and search. Components evolve, so something worth building ten years ago may be a commodity now — and I test claims of 'core' by asking whether a customer would notice or pay if it were twice as good."*

---

## Concept 22 — The TCO of building vs the TCO of buying

Both sides have hidden costs. A fair comparison puts them in the same table over the same horizon.

**Building — what the sticker price hides:**

| Cost | Why it's underestimated |
|---|---|
| **Initial build** | Estimates cover the happy path; edge cases, admin UI, audit, i18n, accessibility, migrations and security hardening arrive later |
| **Ongoing maintenance** | Bug fixes, dependency and runtime upgrades, security patches, performance work — every year, forever; over a decade it commonly exceeds the build cost |
| **Feature parity drift** | The market's products improve every quarter; your build only improves when you invest |
| **Operations** | Hosting, monitoring, on-call, incident response, backups, DR |
| **Compliance** | Pen tests, audit evidence, data-protection requests, accessibility — for something you now own |
| **Knowledge risk** | The builders leave; documentation lags |
| **Opportunity cost** | The team's capacity not spent on the core (Concept 6) — often the decisive item |

**Buying — what the price list hides:**

| Cost | Why it's underestimated |
|---|---|
| **Subscription/licence growth** | Per-seat/per-transaction pricing scales with you; renewal increases; tier jumps when you need one feature |
| **Implementation and integration** | Connecting to identity, data, workflows, other systems; data migration; consultants |
| **Customization** | Configuration that becomes code; customization fights the product's model; upgrades break it |
| **Workarounds** | Processes bent to fit the product; shadow spreadsheets to cover gaps |
| **Vendor management** | Procurement, security reviews, contract renewals, relationship management |
| **Lock-in and exit** | Data export, re-implementation, retraining at end of life (Concept 23) |
| **Vendor risk** | Acquisition, price changes, relicensing, product deprecation, outages you can't fix |

**A fair comparison** — five-year horizon, illustrative numbers for a generic capability (say, a customer-notification service with email/SMS/push, templates, preferences and delivery tracking), US loaded cost ~$5.3k/engineer-week:

| Five-year cost ($k) | Build | Buy (SaaS) |
|---|---|---|
| Initial build / implementation | 3 engineers × 12 weeks ≈ 190 | 1 engineer × 4 weeks integration ≈ 21 |
| Licence / subscription | — | 60/year × 5 = 300 |
| Infrastructure and providers (email/SMS gateways) | 25/year × 5 = 125 | Included in usage tiers; provider pass-through ≈ 60 |
| Maintenance & enhancement | 0.75 engineer ≈ 180/year × 5 = 900 | 0.1 engineer ≈ 24/year × 5 = 120 |
| Operations / on-call share | 20/year × 5 = 100 | 5/year × 5 = 25 |
| Exit cost (end of horizon) | — | ≈ 60 (re-integration, template migration) |
| **Total** | **≈ 1,315** | **≈ 586** |

The subscription — the only number on the buy side most people look at — is about half of buy's TCO; maintenance is two-thirds of build's. Swap in your own numbers, but keep the same rows. The comparison flips only when (a) the capability is core, (b) volume makes per-unit SaaS pricing punitive, (c) requirements genuinely don't fit any product, or (d) regulatory or data-residency constraints rule products out.

**The buy-then-build path.** For an uncertain capability, buying first is often the cheapest way to *learn what you actually need*. If it later becomes core, or volume makes the product expensive, build then — with real requirements. Design the integration behind a port (Module 20) so the switch is contained.

**The interview-grade sentence:** *"I compare build and buy on the same five-to-ten-year table. Building hides edge cases, maintenance and upgrades every year — which over a decade usually exceed the build — operations, compliance, knowledge risk and above all opportunity cost. Buying hides subscription growth and tier jumps, implementation and integration, customization that breaks on upgrade, workarounds, vendor management and exit cost. In a typical comparison for a generic capability the subscription is only half of buy's TCO and maintenance is two-thirds of build's. It flips toward build when the capability is core, volume makes per-unit pricing punitive, nothing fits, or regulation rules products out — and for uncertain capabilities, buying first behind a port is often the cheapest way to learn."*

---

## Concept 23 — Lock-in and exit cost

Lock-in is not a moral failing; it's a **cost to be priced**. Every choice locks you into something — a cloud, a database, a framework, a language, your own code. The question is whether the benefits outweigh the expected cost of leaving.

**Gregor Hohpe's framing** (*Don't get locked up into avoiding lock-in*, martinfowler.com): **expected switching cost = cost of switching × probability of needing to switch.** Spending heavily to avoid a switch that probably won't happen — abstracting every cloud service "to stay portable" — often costs more than the switch would, and denies you the benefits of the platform in the meantime.

**Types of lock-in** — they have different prices:

| Type | Example | Mitigation if worth it |
|---|---|---|
| **Vendor/product** | Proprietary SaaS, proprietary database features | Data export rights, standard interfaces, an anti-corruption layer |
| **Platform** | Cosmos DB APIs, Durable Functions, Service Bus sessions | Ports and adapters around the parts most likely to change |
| **Data** | Data in proprietary formats or locations; egress costs | Open formats (Parquet, JSON), regular exports, contractual data return |
| **API/protocol** | Proprietary SDKs vs OIDC, OpenTelemetry, AMQP, SQL, S3-compatible | Prefer standards where cost is similar |
| **Skills** | Team expertise in a stack | Usually fine — skills are an asset as well as a lock |
| **Contractual** | Multi-year commitments, MACC, reservations | Match commitment term to architectural confidence |
| **Architectural** | Your own design choices — shared database, framework everywhere | Seams (Module 32) |
| **Ecosystem** | Marketplaces, integrations, partner networks | Weighed against their benefits |

**Exit costs are falling — partly.** The EU Data Act abolishes cloud **switching** charges, including egress for the switch, from January 12, 2027 (cost-based only until then), and Azure, AWS and Google already waive egress for customers leaving. But the costs that dominate exits were never egress: **re-implementation, data transformation, retesting, retraining and the coexistence period** (Module 32). Regulation removes the toll on the road, not the length of the journey.

**Cloud repatriation as a case study.** 37signals publicly moved Basecamp and HEY off AWS: compute in 2023 onto purchased Dell hardware, then roughly 18 PB of storage from S3 to Pure Storage arrays in 2025, with AWS waiving egress. They report savings of about $2M per year on compute and project well over $10M over five years overall. It's a real data point — and a specific one: a profitable company with **steady, predictable load**, an existing operations team and data-centre space, and a **CapEx-friendly** owner. Critics note hardware refresh, operations staffing and facilities costs must be counted too. The general lesson is not "cloud is a scam" but **"steady load at scale changes the rent-versus-own maths; spiky or uncertain load favours renting."**

**Practical posture:**

1. **Embrace the platform where it gives real leverage** (managed services, serverless, platform identity) — that's what you're paying for.
2. **Keep options cheap at the seams that matter** — your domain model independent of SDK types; standard protocols for identity and telemetry; data exportable in open formats.
3. **Negotiate the exit up front** — data return format and timeline, termination assistance, price caps on renewal (Concept 25).
4. **Revisit when the probability changes** — a pricing change, an acquisition, a relicensing event, a regulatory requirement.

**The interview-grade sentence:** *"I treat lock-in as a cost to price rather than a sin: expected switching cost is the cost of switching times the probability of needing to, so heavy abstraction to avoid an unlikely switch often costs more than the switch — and denies us the platform's benefits meanwhile. Lock-in comes in kinds — vendor, platform, data, protocol, skills, contractual, architectural — with different prices. The EU Data Act removes switching charges from January 2027, but egress was never the main exit cost; re-implementation, data transformation and the coexistence period are. So I embrace the platform where it gives leverage, keep options cheap at the seams that matter — standard protocols, domain code free of SDK types, exportable data — and negotiate the exit in the contract."*

---

## Concept 24 — Open source is not free

Open source is the default building material of .NET systems — and its costs are real, just not on an invoice. The 2024–2026 relicensing wave made this concrete for every .NET team.

**The costs of adopting open source:**

| Cost | Detail |
|---|---|
| **Integration and ownership** | You run, upgrade and patch it; you debug it at 3 a.m. |
| **Supply-chain security** | Vulnerabilities in direct and transitive dependencies; compromised packages; abandoned maintainers |
| **Licence compliance** | Copyleft obligations (GPL, AGPL, RPL), attribution, source-available licences that forbid some uses |
| **Sustainability risk** | One or two maintainers; burnout; projects archived; or — increasingly — **relicensed** |
| **Upgrade churn** | Breaking changes in major versions; forced upgrades for security fixes |

**The .NET relicensing wave, as a build-vs-buy lesson:**

| Library | Change | Your options |
|---|---|---|
| **IdentityServer** (2020) | Became Duende IdentityServer, commercial with a community edition for small companies | Pay; move to OpenIddict, Keycloak, or a managed identity provider (Entra ID / External ID, Auth0) |
| **FluentAssertions 8** (Jan 2025) | Xceed licence; commercial use requires a paid licence (~$130/developer/year at launch) | Pin to 7.x; switch to a community fork (e.g., AwesomeAssertions) or plain xUnit/NUnit/Shouldly assertions; or pay |
| **MediatR 13+, AutoMapper 15+** (July 2025) | Lucky Penny Software, dual RPL-1.5 / commercial, free community edition below a revenue threshold | Pin to MediatR 12.x / AutoMapper 14.x; replace with plain services / source-generated alternatives (Mapperly, the Mediator source generator); or pay (Module 23) |
| **MassTransit v9** (2026) | Commercial (Massient); v8 receives security patches through 2026 | Stay on v8 short-term; pay; or use the Azure Service Bus SDK directly, Wolverine, Rebus, NServiceBus (commercial) — Module 11 |
| **QuestPDF** | Community Licence v3 (July 2026): free under USD 1M revenue; never for public companies or public sector | Pay (unlimited developers per licence); alternatives |
| **Redis** (2024 → 2025) | Source-available (RSAL/SSPL), then AGPLv3 added in Redis 8 | Valkey (BSD); Azure Managed Redis; AGPL review |

**How to respond to a relicensing — the decision:**

1. **Comply** — pay. Often the cheapest option: a $1–5k/year licence is a few days of an engineer. Paying also funds the thing you depend on.
2. **Pin** — stay on the last permissive version. Cheap now, but you accumulate **dependency debt**: no new fixes, eventual incompatibility with new .NET versions. Put a review date on it.
3. **Replace** — with an alternative library or your own simple code. Worth it when usage is shallow (most MediatR usage is a few dozen handlers) or the library was always marginal (mapping libraries).
4. **Fork** — maintain your own copy. Rarely worth it unless a community fork exists with real maintainers.

Run the TCO for each, including migration effort and future maintenance. Don't let indignation make the decision — *"we won't pay on principle"* can cost ten times the licence.

**Hygiene that makes this manageable:**

```xml
<!-- Directory.Packages.props — Central Package Management with explicit ranges for pinned dependencies -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>
  <ItemGroup>
    <!-- Pinned below the relicensed major; ADR-0061, review by 2027-03-31 -->
    <PackageVersion Include="MediatR" Version="[12.0.0,13.0.0)" />
    <PackageVersion Include="FluentAssertions" Version="[7.0.0,8.0.0)" />
    <PackageVersion Include="MassTransit" Version="[8.0.0,9.0.0)" />
  </ItemGroup>
</Project>
```

- **Inventory** direct and transitive dependencies with licences: `dotnet list package --include-transitive` (also available as `dotnet package list` in recent SDKs), an SBOM generator (Microsoft's `sbom-tool`, CycloneDX), and a licence scanner in CI.
- **Vulnerability scanning**: `dotnet list package --vulnerable --include-transitive`, NuGet audit warnings at restore, Dependabot or Renovate.
- **Health signals** before adopting: OpenSSF Scorecard, deps.dev, maintainer count, release cadence, funding model.
- **Shallow integration** for non-core libraries: keep their types out of your domain (ports, Module 20), so replacing them is days, not months.
- **An ADR per significant dependency decision**, including the response to a relicensing.

**The interview-grade sentence:** *"Open source has no invoice but real costs: ownership, supply-chain security, licence compliance, sustainability and upgrade churn. The .NET relicensing wave — IdentityServer, FluentAssertions, MediatR and AutoMapper, MassTransit, QuestPDF, and Redis outside .NET — made that concrete. When it happens I compare four options on TCO: pay, which is often a few days of an engineer per year; pin to the last permissive version with a review date, accepting dependency debt; replace, which is cheap when usage is shallow; or fork, which rarely pays. I make it manageable with Central Package Management, licence and vulnerability scanning and SBOMs in CI, health checks like OpenSSF Scorecard before adoption, keeping library types out of the domain, and an ADR for each decision."*

---

## Concept 25 — Evaluating vendors and products

When the answer is "buy" (or "rent"), the next risk is buying the *wrong* thing. A disciplined evaluation is short, evidence-based and ends in a recorded decision.

**A practical process:**

1. **Requirements, prioritized** — must-have, should-have, nice-to-have; include NFRs (SLA, data residency, identity integration, audit, API limits, export) — not just features.
2. **Long list → short list** — analyst reports, peer recommendations, Thoughtworks Tech Radar, existing enterprise agreements; cut to 2–4 candidates on must-haves.
3. **Weighted scoring** — agreed **before** demos to avoid post-hoc rationalization (template in Appendix B).
4. **Proof of concept** — on *your* hardest scenarios, with *your* data shape and identity, time-boxed (2–4 weeks). Demos show the happy path; PoCs find the walls.
5. **Due diligence** — security questionnaire and certifications (SOC 2 Type II, ISO 27001), data processing agreement, sub-processors, data residency, vendor financial health, roadmap, support quality (reference calls with customers your size).
6. **Commercials** — full TCO over the term (Concept 22), pricing model and how it scales with your growth, renewal caps.
7. **Decision record** — an ADR with options considered, scores, PoC findings, risks and the exit plan.

**Criteria that are routinely under-weighted:**

| Criterion | Question to ask |
|---|---|
| **Pricing model scaling** | What do we pay at 3× and 10× our volume? Which tier jumps are coming? |
| **API and integration quality** | Rate limits, webhooks, idempotency, versioning policy, SDK quality for .NET |
| **Identity integration** | OIDC/SAML SSO with Entra ID, SCIM provisioning, role mapping |
| **Data ownership and export** | Full export in an open format, on demand, at no extra cost? Deletion guarantees? |
| **Extensibility** | Configuration vs code; where does customization live; does it survive upgrades? |
| **Operational transparency** | Status page history, incident post-mortems, SLA credits that matter |
| **Vendor viability** | Profitable? Recently acquired? Dependent on one big customer? |
| **Exit plan** | What would leaving look like in three years? |

**Contract terms that protect you** — work with procurement and legal; architects should know what to ask for:

- **Data return**: format, timeline and assistance on termination.
- **Price protection**: caps on renewal increases; pricing for growth tiers fixed in advance.
- **SLA** with meaningful credits and termination rights for chronic failure.
- **Change of control** clauses (acquisition), **deprecation notice** periods for features you rely on.
- **Security and audit rights**, breach notification timelines.
- **Escrow** for critical on-premises software from small vendors (source code released if the vendor fails).

**The interview-grade sentence:** *"For a buy decision I run a short, evidence-based evaluation: prioritized requirements including NFRs like SLA, residency, SSO and export; a short list of two to four; weighted scoring agreed before any demo; a time-boxed proof of concept on our hardest scenarios; due diligence on security certifications, data processing, residency, vendor health and reference customers our size; full TCO including how pricing scales at 3× and 10×; and an ADR with the exit plan. I make sure the contract covers data return, renewal price caps, SLA credits with termination rights, change of control and deprecation notice."*

---

## Concept 26 — .NET build-vs-buy calls

The general method applied to decisions .NET architects make constantly. These are defaults, not laws — the reasoning matters more than the verdict.

| Capability | Default | Reasoning |
|---|---|---|
| **Workforce identity** | **Microsoft Entra ID** | Generic subdomain; security-critical; organizations already have it |
| **Customer identity (CIAM)** | **Entra External ID**, Auth0/Okta CIC, or Keycloak; Duende IdentityServer or OpenIddict when you need a self-hosted OIDC server with deep customization | Never build your own OAuth/OIDC server from scratch (Module 29). Duende is commercial; OpenIddict is Apache-2.0 but you own more |
| **Local accounts inside an app** | ASP.NET Core Identity | Built in; but it's an account store, not an identity provider |
| **Messaging transport** | Azure Service Bus / Event Hubs (managed) | Commodity infrastructure (Module 27) |
| **Messaging framework** | Direct SDK for simple cases; Wolverine/Rebus (OSS) or NServiceBus/MassTransit v9 (commercial) when sagas, outbox and retries justify it | Value is in the reliability patterns; compare licence cost with building outbox/retry yourself (Module 11) |
| **Mediator / CQRS dispatch** | Plain injected handlers; source-generated mediator if you want pipeline behaviours | Shallow capability; the MediatR licence change made "replace" cheap (Module 23) |
| **Object mapping** | Hand-written mappings or **Mapperly** (source generator, compile-time) | Reflection mappers hide bugs; source generators are fast, debuggable and free |
| **Background jobs** | `BackgroundService` + queue; Hangfire/Quartz.NET; Azure Functions/Container Apps jobs | Scheduling with persistence and retries is generic — adopt or rent |
| **Workflow / orchestration** | Durable Functions / Durable Task, Logic Apps, Temporal, or a saga library | Building a durable workflow engine is a multi-year project |
| **Feature flags** | Microsoft.FeatureManagement + Azure App Configuration; LaunchDarkly etc. for experimentation | Building flags is easy; building targeting, audit and experimentation is not |
| **Search** | Azure AI Search, Elasticsearch/OpenSearch | Relevance tuning and indexing pipelines are generic |
| **PDF/reporting** | Licensed library (QuestPDF, commercial libraries) or a rendering service | Check revenue thresholds; generic capability |
| **Observability** | OpenTelemetry + Azure Monitor/Application Insights, or Grafana/Datadog | Standard instrumentation; backend is a buy decision (Module 28) |
| **Resilience** | Polly / Microsoft.Extensions.Resilience (Module 25) | Free and standard |
| **Payments, tax, email/SMS delivery** | SaaS (Stripe, Adyen, tax engines, SendGrid/Azure Communication Services) | Regulated, generic, high-risk to build |
| **The core domain** | Build — with clean architecture, DDD, tests | This is what the business pays you for |

**Replacing a relicensed mapper with Mapperly** — an example of "replace is cheap when usage is shallow":

```csharp
using Riok.Mapperly.Abstractions;

[Mapper]
public static partial class OrderMappings
{
    // Generated at compile time: no reflection, visible in the IDE, and a build warning on unmapped members
    public static partial OrderDto ToDto(this Order order);
    public static partial IQueryable<OrderSummaryDto> ProjectToSummary(this IQueryable<Order> orders);
}

// var dto = order.ToDto();
// var page = await db.Orders.Where(o => o.TenantId == tenantId).ProjectToSummary().ToListAsync(ct);
```

**And replacing a mediator with plain DI** when pipeline behaviours aren't load-bearing:

```csharp
public sealed record PlaceOrder(Guid CustomerId, IReadOnlyList<OrderLine> Lines);

public sealed class PlaceOrderHandler(OrdersDbContext db, TimeProvider clock)
{
    public async Task<Guid> HandleAsync(PlaceOrder command, CancellationToken ct)
    {
        var order = Order.Place(command.CustomerId, command.Lines, clock.GetUtcNow());
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}

// builder.Services.AddScoped<PlaceOrderHandler>();
// app.MapPost("/orders", (PlaceOrder cmd, PlaceOrderHandler h, CancellationToken ct) => h.HandleAsync(cmd, ct));
// Cross-cutting concerns: endpoint filters, middleware, decorators (Scrutor) — where you actually need them.
```

**The interview-grade sentence:** *"In .NET my defaults follow the core-versus-generic split: Entra ID for workforce identity and a managed CIAM, Duende or OpenIddict only when I need a self-hosted OIDC server — never a homegrown one; managed brokers with a messaging framework only when sagas and outbox justify its licence; plain handlers or a source-generated mediator, and Mapperly or hand-written mappings instead of reflection mappers; adopt or rent jobs, workflow engines, flags, search, PDF generation and observability backends; SaaS for payments, tax and messaging delivery; and build the core domain properly. The licence changes made replacing shallow dependencies like mediators and mappers cheap — that's often a better trade than paying or pinning forever."*

---

## Concept 27 — AI and build-vs-buy, 2026

Coding agents have changed one input to the build-vs-buy equation dramatically: **the cost to write code**. They have changed the others much less. Early 2026 saw software stocks reprice on the thesis that companies would build instead of buy; enterprise surveys report many organizations building more internal tools with agentic coding. The architect's job is to separate the real shift from the hype.

**What genuinely changed:**

| Effect | Implication |
|---|---|
| **Thin internal tools are cheap to build** | Admin screens, internal dashboards, simple single-workflow apps, glue integrations — build cost fell from weeks to days |
| **Prototypes are nearly free** | "Build to learn, then decide" (Concept 22) is cheaper than ever |
| **Replacing shallow libraries is cheap** | Swapping a relicensed mapper or mediator is an afternoon with good tests |
| **Customizing around products is cheaper** | Integration code and adapters to make a bought product fit |

**What didn't change:**

| Cost | Why AI doesn't remove it |
|---|---|
| **Ownership** | Security patches, upgrades, on-call, incident response, compliance evidence, accessibility — every year |
| **Correctness in hard domains** | Payments, tax, identity, regulated workflows: the value of a product is years of edge cases and certifications, not lines of code |
| **Data and network effects** | A CRM's value includes its ecosystem, integrations and the data model hundreds of companies refined |
| **Operational excellence at scale** | SaaS vendors amortize SRE, security teams and certifications across thousands of customers |
| **Governance** | AI-built internal tools proliferate as shadow IT — unreviewed, unowned, holding sensitive data |
| **Review cost** | Generated code still needs humans who understand it (Concept 35) |

**The 2026 heuristic:** **AI lowers the cost to write, not the cost to own.** So:

- **Build more** where the capability is **thin, internal, low-risk, and specific to how you work** — the classic "we pay $40k a year for a tool we use 10% of" case.
- **Keep buying** where value comes from **depth, compliance, scale or ecosystem**.
- **Put AI-built internal tools under the same governance** as other software: an owner, identity integration, data classification, a place in the service catalogue, and a sunset rule.
- **Recompute TCO** with lower build cost but unchanged ownership rows — the table in Concept 22 shifts, but usually not as far as the hype suggests.

**AI products themselves are a build-vs-buy decision.** Model (buy/rent — almost no one should train foundation models), platform (Microsoft Foundry, Bedrock, Vertex — rent), orchestration and evaluation (adopt frameworks or build thin), and the **domain-specific part** — your data, workflows, evaluations and guardrails — which is where differentiation lives and is worth building.

**The interview-grade sentence:** *"Agents have cut the cost to write code, not the cost to own it. So I'd now happily build thin internal tools, prototypes and replacements for shallow libraries that we used to buy — but keep buying where value comes from depth, compliance, scale or ecosystem, like payments, identity or a CRM, because ownership costs — patching, upgrades, on-call, audit — haven't changed. I'd recompute the TCO with lower build cost but the same ownership rows, and put AI-built tools under normal governance — owner, SSO, data classification, catalogue entry — so they don't become a new generation of shadow IT. For AI features themselves, rent the models and platform and build the domain-specific data, workflow and evaluation layer."*

---
# Part D — Technical debt

## Concept 28 — Technical debt, properly

**The original metaphor.** Ward Cunningham introduced it in his 1992 OOPSLA experience report on the WyCash portfolio system. His point was specific: shipping code that reflects your *current, incomplete* understanding of the problem is like taking on debt — it lets you move faster now, and that's fine **as long as you pay it back by refactoring as your understanding improves**. The danger is not the borrowing; it's never repaying, because *every minute spent on not-quite-right code counts as interest*. In a later video he regretted that people took the metaphor as licence to write poor code; his debt was about learning, not sloppiness.

**The financial structure, made precise:**

| Financial term | Software meaning |
|---|---|
| **Principal** | The effort to fix the shortcut — refactor the design, add the tests, upgrade the platform |
| **Interest** | The extra effort every change costs because the shortcut exists — slower changes, more bugs, harder onboarding |
| **Interest rate** | How much extra effort per change — high in tangled hotspots, near zero in code nobody touches |
| **Default** | The point where the system can't economically be changed any more and must be replaced (Module 32) |
| **Refinancing** | Containing debt behind a seam or anti-corruption layer so its interest stops spreading |

**The consequence most people miss:** **interest is only paid when you change the code.** Messy code that nobody touches costs almost nothing (until a security patch or a platform upgrade forces a change). This is why "rewrite all the ugly code" is economically wrong and "fix the hotspots" is right (Concept 31).

**What isn't technical debt** — keeping the term precise preserves its persuasive power:

| Not debt | Why |
|---|---|
| **Bugs** | Defects are defects; some debt causes bugs, but a bug is a quality problem, not a deferred design investment |
| **Missing features** | Scope you chose not to build is a product decision |
| **Code you dislike** | Style preferences and "I'd have done it differently" aren't debt unless they slow change |
| **Old technology that works and isn't changing** | Age isn't debt; risk from unsupported platforms is (Concept 34) |
| **Inevitable evolution** | A design that was right and became wrong as requirements changed is closer to "design drift" — still worth fixing if it slows change, but nobody "took a loan" |

Fowler distinguishes **cruft** (deficiencies in internal quality that make change harder) from the debt metaphor itself; the metaphor is useful *because* it frames cruft economically: pay down principal when the interest saved exceeds the cost.

**The interview-grade sentence:** *"I use Cunningham's original sense: technical debt is shipping code that reflects our current understanding to move faster, which is fine if we repay it by refactoring as we learn. Principal is the cost to fix; interest is the extra effort every change costs while it exists — and interest is only paid where we change code, so untouched mess is cheap and hotspots are expensive. I keep the term precise: bugs, missing features, code I'd write differently and old-but-stable technology aren't debt by themselves, and that precision is what makes the metaphor persuasive with executives."*

---

## Concept 29 — Types of technical debt

**Fowler's technical debt quadrant** classifies debt by intent and awareness:

| | **Reckless** | **Prudent** |
|---|---|---|
| **Deliberate** | "We don't have time for design" | "We must ship now and deal with the consequences" — a conscious loan |
| **Inadvertent** | "What's layering?" — debt from lack of skill | "Now we know how we should have done it" — debt discovered by learning |

The quadrant matters because the remedies differ: **prudent-deliberate** debt needs a record and a repayment plan (Concept 32); **reckless** debt needs skills, standards and review; **prudent-inadvertent** debt is normal and is handled by continuous refactoring.

**Kinds of debt by where it lives:**

| Type | Examples in .NET systems | Typical interest |
|---|---|---|
| **Code debt** | Long methods, duplication, missing abstractions, static helpers, God classes | Slower changes, bugs in duplicated logic |
| **Design / architecture debt** | Wrong boundaries, shared database, cyclic dependencies between projects, business logic in controllers or stored procedures | Cross-team coordination, cascading changes, can't scale teams (Modules 20–22) |
| **Test debt** | Low or brittle coverage, slow suites, no characterization tests, flaky integration tests | Fear of change, long feedback loops, regressions |
| **Infrastructure / delivery debt** | Manual deployments, snowflake servers, no IaC, slow pipelines | Low deployment frequency, risky releases, long recovery |
| **Platform / dependency debt** | .NET versions near or past end of support, pinned relicensed packages, old SDKs, `System.Data.SqlClient` | Security exposure, forced emergency upgrades, blocked features |
| **Data debt** | Inconsistent schemas, duplicated sources of truth, missing constraints, unclear ownership | Reconciliation work, reporting errors, migration difficulty (Module 32) |
| **Documentation / knowledge debt** | Missing ADRs, out-of-date diagrams, knowledge in one head | Onboarding time, key-person risk, repeated mistakes (Module 31) |
| **Operational / observability debt** | No traces, unclear alerts, missing runbooks | Long incidents, high on-call load |
| **Security debt** | Known vulnerabilities deferred, secrets in config, over-broad permissions | Breach probability (Concept 7) |
| **AI-generated debt** | Duplicated generated code, inconsistent patterns, unreviewed logic nobody fully understands | Larger, less coherent codebases (Concept 35) |
| **ML/data-pipeline debt** | Sculley et al.'s *Hidden Technical Debt in ML Systems*: glue code, pipeline jungles, entanglement, undeclared consumers | Fragile models, costly change |

**Social debt.** Debt also accumulates in the organization: team boundaries misaligned with architecture (Conway, Module 21), unclear ownership, hero culture. It's often the root cause of the technical kinds.

**The interview-grade sentence:** *"I classify debt two ways. Fowler's quadrant — deliberate or inadvertent, reckless or prudent — tells me the remedy: a prudent deliberate loan needs a record and a repayment trigger, reckless debt needs skills and standards, and prudent inadvertent debt is normal learning handled by continuous refactoring. Then by where it lives — code, design and architecture, tests, delivery, platform and dependencies, data, knowledge, operations, security, and now AI-generated and ML pipeline debt — because each has a different kind of interest and a different owner. Social debt, like team boundaries that don't match the architecture, is often the root cause."*

---

## Concept 30 — How interest is paid

To fund paydown you have to show the interest. It appears in five places — most of them measurable.

| Interest | Symptom | Measure |
|---|---|---|
| **Slower change** | Simple features take weeks; estimates balloon in certain areas | Lead time for changes per area; cycle time per ticket by component; time-in-development |
| **More defects** | Bugs cluster in the same modules; fixes create new bugs | Defects per module; change failure rate (DORA); escaped defects |
| **Unpredictability** | Estimates in some areas are unreliable; long tail of cycle times | Variance of cycle time per area |
| **Operational load** | Incidents, toil, on-call pages, manual steps | Incident count and MTTR by component; toil hours |
| **People cost** | Long onboarding, attrition, reluctance to work on an area | Onboarding time to first meaningful change; survey sentiment; exit-interview themes |

**Evidence you can cite.** Tornhill and Borg's *Code Red* study (39 proprietary codebases, 30,737 files; IEEE/ACM TechDebt 2022) found that low-quality ("red") code had **15× more defects** than healthy code, issue resolution took **124% longer** on average, and maximum cycle times were **9× longer** — the unpredictability point. McKinsey's CIO surveys estimated technical debt at **20–40% of the value of the technology estate** before depreciation and reported **10–20% of new-product technology budgets diverted** to dealing with it. Industry estimates of developer time lost to debt and bad code range around **23–42%**. Treat these as orders of magnitude, not as your numbers — but they make the point that interest is large and measurable.

**Converting interest to money.** A simple, defensible model:

```text
annual interest in an area ≈ (engineering capacity spent in that area per year)
                           × (fraction of that effort caused by the debt)
                           + (incident and defect costs attributable to it)
                           + (cost of delay on roadmap items that slip because of it)
```

The middle term is the hard one; estimate it from cycle-time comparisons (similar changes in healthy vs unhealthy areas), engineers' estimates (survey "how much faster would this be if…"), or published ratios used conservatively. Present it as a range.

**Interest compounds.** Debt makes the next shortcut more tempting (it's already messy), slows the refactoring that would fix it, and drives away the people who know how. That's why "we'll fix it later" gets more expensive over time — a compounding cost of delay (Concept 5).

**The interview-grade sentence:** *"Debt interest shows up as slower change, more defects, unpredictable estimates, operational load and people costs — and most of that is measurable per area: lead time and cycle-time variance, defect density and change failure rate, incidents and MTTR, onboarding time. The Code Red study's 15 times more defects and roughly twice the time to resolve issues in low-quality code is a useful order of magnitude. To put it in money I estimate engineering capacity spent in the area times the fraction caused by the debt, plus attributable incidents and roadmap delay, presented as a range — and I point out that it compounds."*

---

## Concept 31 — Measuring and finding debt

You can't manage what you can't see. Combine **code signals**, **delivery signals** and **people signals** — no single tool tells the whole story.

**1. Hotspots: complexity × change frequency.** Adam Tornhill's core insight (Module 32's Concept 7): prioritize code that is *both* complex *and* frequently changed — that's where interest is paid. Tools: **CodeScene** (commercial, "code health" scores per file, hotspots, temporal coupling, knowledge/bus-factor maps), the open-source **code-maat**, or a quick `git log` analysis combined with a complexity metric. A typical result: 5% of files account for most of the change effort and defects.

**2. Static analysis.** **SonarQube/SonarCloud** reports code smells, duplication, coverage and a *technical debt* estimate (remediation time, SQALE-style) and a debt ratio. **NDepend** estimates debt and annual interest per issue for .NET, with dependency graphs and trend tracking. **Roslyn analyzers** (.NET SDK analyzers, StyleCop, Roslynator) enforce standards at build time. Use their absolute "days of debt" numbers sceptically — they count smells, not interest — but **trends** and **hotspot overlap** are valuable.

**3. Architecture conformance.** Dependency rules checked by tests (NetArchTest, ArchUnitNET — Concept 34); cyclic dependencies between projects; coupling metrics.

**4. Dependency and platform age.** Runtime versions vs support dates; outdated and vulnerable packages:

```bash
dotnet list package --outdated --include-transitive
dotnet list package --vulnerable --include-transitive
# Recent SDKs also accept the noun-first form: dotnet package list --outdated
```

**libyear** ("how many years behind the latest release, summed over dependencies") is a simple, trendable metric.

**5. Delivery metrics by area.** DORA's four keys (deployment frequency, lead time for changes, change failure rate, time to restore) — plus the newer rework-rate measure — sliced by service or module. An area with 3× the lead time and twice the change failure rate is a debt candidate.

**6. People signals.** Developer surveys ("Which area slows you down most?", "How confident are you changing X?"), onboarding time, the areas people avoid. Google's engineering-productivity researchers (Jaspan and Green, *IEEE Software* 2023) report that the code-level metrics they evaluated were poor predictors of where engineers felt hindered by debt, and that engineers' own survey responses were the practical signal. Ask your engineers; they know.

**7. Debt register** — the list of known debt items with owner, interest estimate and repayment trigger (Concept 33). It's where findings from all the above become decisions.

**Measure trends, not absolutes.** "Our debt ratio is 7%" means little. "Code health in the Orders hotspots went from 4.1 to 6.8 and lead time in Orders dropped from 9 to 4 days" means a lot.

**The interview-grade sentence:** *"I find debt by combining signals: hotspots — files that are both complex and frequently changed — from CodeScene or git history, because that's where interest is paid; static analysis like SonarQube or NDepend for trends rather than their absolute 'days of debt'; architecture conformance tests; dependency and runtime age against support dates; DORA metrics sliced per area; and developer surveys — Google's researchers found code metrics predicted poorly where engineers felt hindered, so engineers' own reports are the practical signal. Findings go into a debt register with owners and interest estimates, and I track trends, like code health in the hotspots against lead time in that area."*

---

## Concept 32 — Deliberate debt: when borrowing is right

Taking on debt is sometimes the correct business decision — just as companies borrow money to grow. Prudent, deliberate debt is a tool.

**Good reasons to borrow:**

| Reason | Example |
|---|---|
| **Market timing** | A launch window, a trade show, a competitor move — cost of delay exceeds the interest (Concept 5) |
| **Learning** | You don't know if the feature will be used; build the simple version, measure, then invest if it sticks |
| **Survival** | A start-up with six months of runway optimizes for finding product-market fit, not for elegance |
| **Sacrificial architecture** | Fowler's term: build something you know you'll replace when it succeeds |
| **Regulatory deadline** | Comply on the date; clean up afterwards |

**Bad reasons:** "we never have time for quality" (that's a permanent, unplanned loan); "tests slow us down" (they speed you up after the first few weeks); "it's just a prototype" — when the prototype is going to production.

**The terms of a responsible loan** — record them, like a loan agreement:

1. **What shortcut** we're taking and **why** (the business reason and the cost of delay avoided).
2. **The expected interest** — which areas it affects and how.
3. **The repayment trigger** — a date, a condition ("before onboarding the second enterprise tenant", "if the feature exceeds 1,000 weekly users"), or a metric threshold.
4. **The owner** who will raise it when the trigger hits.
5. **Containment** — how we keep the debt from spreading (behind an interface, in one module, feature-flagged).

That's an ADR (Module 31) with a repayment section, plus an entry in the debt register:

```yaml
# docs/debt/TD-0042.yaml
id: TD-0042
title: Tenant configuration stored as JSON blob in Tenants table
type: design
quadrant: prudent-deliberate
created: 2026-10-07
owner: team-platform
adr: ADR-0063
reason: >
  Launch tenant self-service by Nov 15 (sales commitment for 3 enterprise pilots).
  Proper relational model + migration estimated at 3 extra weeks.
interest: >
  Every new tenant setting requires a code change and redeploy; no querying across tenants;
  validation only in application code. Estimated 1-2 days per new setting.
containment: Accessed only through ITenantSettings in Platform.Application; no direct SQL elsewhere.
repayment_trigger: "Before the 10th tenant setting, or by 2027-03-31, whichever first"
principal_estimate: "2-3 engineer-weeks"
status: open
```

**Make the business a co-signer.** If the business chooses speed over quality, they should know they're taking a loan and agree to the repayment trigger. That turns *"engineering wants time to refactor"* six months later into *"we're repaying the loan we agreed in October."*

**The interview-grade sentence:** *"Deliberate debt is sometimes the right business decision — a launch window, learning whether a feature is used, start-up survival, sacrificial architecture or a regulatory date — when the cost of delay exceeds the interest. I treat it like a loan with written terms: what shortcut and why, the expected interest, a repayment trigger as a date, condition or metric, an owner and a containment strategy — recorded as an ADR and a debt-register entry, with the business explicitly co-signing, so that repaying it later is honouring an agreement rather than asking for refactoring time."*

---

## Concept 33 — Paying it down

Paying down debt is an investment decision like any other: **repay principal where the interest saved exceeds the cost**, soonest where interest is highest.

**Prioritize by interest, not by ugliness:**

```text
priority ≈ (interest per change × change frequency × expected life of the code) − principal
```

High-churn hotspots in long-lived code first; ugly but stable corners last (or never); code about to be replaced — don't refactor, contain (Module 32).

**Funding models** — pick one deliberately, and most teams need a combination:

| Model | How | Strength | Weakness |
|---|---|---|---|
| **Continuous ("boy scout", fix as you touch)** | Leave code better than you found it; refactor in the course of feature work | No negotiation; aligns effort with interest automatically (you touch hotspots most) | Doesn't handle large structural debt |
| **Capacity allocation** | A fixed share — commonly 15–25% — of each team's capacity for debt and maintenance | Predictable; protects platform upgrades | Can become "the refactoring tax" without priorities; needs a backlog of high-value items |
| **Scheduled paydown** | Dedicated iterations or "fix-it weeks" | Visible; good for cross-cutting work | Debt accumulates between them; tempting to cancel |
| **Investment case** | A funded initiative with a business case (Concept 37) | Right for big items: splitting a database, platform migration | Requires the translation work; slower to start |
| **Pay when forced** | Upgrade only when support ends or a vulnerability appears | Cheap until it isn't | Emergency work at the worst time; the most expensive model |

**Rules that make paydown work:**

1. **Tie paydown to value.** "Refactor the pricing module" is a cost; "cut lead time on pricing changes from 3 weeks to 4 days so the Q2 pricing roadmap fits" is an investment.
2. **Small, continuous, merged to trunk.** Large refactoring branches are big-bang rewrites in miniature. Use branch by abstraction, expand/contract and the **Mikado method** (try the change, note what breaks, revert, fix prerequisites first, repeat) to keep each step shippable.
3. **Measure before and after.** Lead time, cycle-time variance, defect rate in the area — so the next investment case has evidence.
4. **Stop the bleeding first.** Add an architecture test or analyzer so the debt can't grow while you pay it down (a **ratchet**, Concept 34).
5. **Prefer containment for code with a short future.** If a module will be replaced in a year, put it behind an anti-corruption layer and stop investing.
6. **Platform upgrades are maintenance, not projects.** .NET LTS every two years, dependency updates weekly (automated), database and OS versions on a calendar. Budget them inside capacity allocation so they never need a business case.
7. **Don't confuse paydown with rewrite.** Rewriting a module is sometimes right — when principal is lower than continued interest *and* the module is small enough to replace safely. Use Module 32's techniques either way.

**The interview-grade sentence:** *"I prioritize debt repayment by interest, not ugliness — interest per change times change frequency times the remaining life of the code, minus the principal — so long-lived hotspots come first, stable corners maybe never, and code about to be replaced gets contained rather than refactored. Funding is usually a combination: continuous fix-as-you-touch, a protected capacity share of around 15 to 25% for maintenance including platform upgrades, and investment cases for big structural items. I tie each paydown to a value metric like lead time, keep changes small on trunk with branch by abstraction and the Mikado method, add a ratchet so the debt can't grow meanwhile, and measure before and after."*

---

## Concept 34 — Architecture and platform debt in .NET: fitness functions and dependency hygiene

Two kinds of debt are especially well-suited to automation in .NET: **architecture erosion** and **platform/dependency age**.

**Architecture fitness functions with NetArchTest or ArchUnitNET.** Encode the dependency rules from Modules 20–22 as unit tests so erosion fails the build:

```csharp
using NetArchTest.Rules;
using Xunit;

public class ArchitectureRules
{
    private static readonly System.Reflection.Assembly Domain = typeof(Contoso.Orders.Domain.Order).Assembly;
    private static readonly System.Reflection.Assembly Application = typeof(Contoso.Orders.Application.PlaceOrderHandler).Assembly;

    [Fact]
    public void Domain_has_no_infrastructure_dependencies()
    {
        var result = Types.InAssembly(Domain)
            .ShouldNot().HaveDependencyOnAny(
                "Microsoft.EntityFrameworkCore",
                "Microsoft.Azure.Cosmos",
                "Azure.Messaging.ServiceBus",
                "Contoso.Orders.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful,
            "Domain types with forbidden dependencies: " + string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Modules_do_not_reach_into_each_others_internals()
    {
        var result = Types.InAssembly(Application)
            .That().ResideInNamespace("Contoso.Orders")
            .ShouldNot().HaveDependencyOn("Contoso.Billing.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // A ratchet: usage of a legacy API may shrink but never grow (TD-0031)
    private const int LegacyPriceCalcDependentsBaseline = 9;

    [Fact]
    public void Legacy_price_calculator_dependents_do_not_increase()
    {
        var dependents = Types.InAssembly(Application)
            .That().HaveDependencyOn("Contoso.Legacy.PriceCalc")
            .GetTypes()
            .Count();

        Assert.True(dependents <= LegacyPriceCalcDependentsBaseline,
            $"{dependents} types depend on PriceCalc (baseline {LegacyPriceCalcDependentsBaseline}). " +
            "Use IPricingEngine instead; lower the baseline when you remove a dependency.");
    }
}
```

The **ratchet** pattern is the most useful debt tool many teams never use: freeze the current count of a bad thing (usages of a legacy API, compiler warnings, nullable-disabled files, `#pragma warning disable`), fail the build if it increases, and lower the baseline whenever someone reduces it. Debt then only goes one way.

**Platform debt: the runtime clock.** .NET's release train makes platform debt predictable:

- **LTS releases** every two years (in November of even years), supported for three years; **STS** releases in odd years, now supported for two years — so .NET 8 and 9 both end on November 10, 2026, and .NET 10 LTS runs to November 2028.
- A team that upgrades **every LTS** does a modest upgrade every two years. A team that skips releases accumulates breaking changes and ends up in an emergency project when support ends.
- Put runtime end-of-support dates in the roadmap as **cost-of-delay cliffs** (Concept 5), and fund the upgrade from maintenance capacity, not from an investment case.

**Dependency hygiene:**

| Practice | Tooling |
|---|---|
| One version per package across the solution | Central Package Management (`Directory.Packages.props`), transitive pinning |
| Automated update PRs, grouped and scheduled | Dependabot or Renovate, with auto-merge for patch updates that pass CI |
| Vulnerability gates | NuGet audit at restore (warnings → errors for high severity), `dotnet list package --vulnerable` in CI |
| Licence gates | Licence scanner in CI; approved-licence list; ADR for exceptions (Concept 24) |
| Trend metric | libyear or "% of packages on latest major" per repository |
| SDK/analyzer consistency | `global.json` for SDK version; `Directory.Build.props` for analyzers, `Nullable`, `TreatWarningsAsErrors` with ratchets |

**The interview-grade sentence:** *"In .NET I automate the two debts that erode silently. Architecture rules from the layering and module boundaries become NetArchTest or ArchUnitNET tests, plus ratchets that freeze the count of bad things — legacy API usages, warnings, nullable-disabled files — so debt can only decrease. Platform debt follows a predictable clock — LTS every two years, STS now two years of support, so .NET 8 and 9 both end on November 10, 2026 — so I upgrade every LTS from maintenance capacity and put end-of-support dates on the roadmap as cliffs. Dependencies get Central Package Management, Dependabot or Renovate, vulnerability and licence gates in CI, and a trend metric like libyear."*

---

## Concept 35 — Technical debt in the AI era

AI coding tools change the **rate** at which code — and therefore debt — is produced. The 2025 DORA report's framing applies directly: **AI amplifies what's already there.** Teams with strong practices get faster; teams with weak ones accumulate debt faster.

**The evidence so far:**

- **DORA 2025**: AI adoption is now associated with higher delivery throughput, but still with lower delivery stability; DORA's AI Capabilities Model identifies conditions — strong version-control practices, small batches, quality internal platforms, user focus — under which AI helps.
- **GitClear 2025** (211M changed lines, 2020–2024): copy/pasted and duplicated blocks rose sharply (about 8× more duplicated five-plus-line blocks in 2024), while "moved" code — a signature of refactoring and reuse — fell below 10% of changes. Code is being *added* faster than it's being *consolidated*.

**New forms of debt:**

| Debt | Mechanism | Counter-measure |
|---|---|---|
| **Duplication** | Generating a new helper is easier than finding the existing one | Duplication detection in CI; prompts and agent instructions pointing to existing abstractions |
| **Inconsistency** | Each generated change follows a slightly different pattern | Conventions in `AGENTS.md`/instructions files, analyzers, templates; review for consistency |
| **Comprehension debt** | Code that works but that nobody on the team fully understands | Review standards: the author must be able to explain it; smaller PRs |
| **Test illusion** | Generated tests that assert implementation details or mirror the code's bugs | Review tests as carefully as code; property-based and characterization tests |
| **Volume** | Larger PRs and more code to maintain | PR size limits; delete as you go; measure code growth vs feature growth |
| **Prompt and agent configuration debt** | Prompts, tool definitions, evaluation sets as untested, unversioned artifacts | Treat them as code: versioned, reviewed, evaluated in CI |

**AI is also a powerful paydown tool.** The same tools that create debt make repayment cheaper: generating characterization tests, mechanical refactorings across a codebase, upgrading frameworks (the Copilot upgrade agent, Module 32), explaining legacy code. The economics of Concept 33 shift — **principal gets cheaper**, so more paydown items clear the bar. Reinvest some of the speed gain into refactoring budgets rather than only into features.

**The interview-grade sentence:** *"AI amplifies existing practice: DORA 2025 associates it with higher throughput but still lower stability, and GitClear's data shows duplicated code rising sharply while refactoring falls — code is added faster than it's consolidated. The new debts are duplication, inconsistency, comprehension debt — code that works but nobody understands — tests that mirror bugs, sheer volume, and unversioned prompts and agent configuration. My counter-measures are conventions in agent instruction files, duplication and size checks, a review standard that the author must be able to explain the change, and treating prompts and evaluations as code. And because AI makes principal cheaper — generating characterization tests, mechanical refactors, upgrades — I reinvest part of the speed gain into paydown."*

---
# Part E — Making the case

## Concept 36 — Speaking to executives

Executives are not less intelligent than engineers; they are **time-constrained, context-switching across dozens of decisions, and accountable in different units**. Good executive communication respects that.

**The translation table:**

| Engineers say | Executives hear | Say instead |
|---|---|---|
| "We need to refactor the pricing module" | "Engineers want to play" | "Pricing changes take three weeks; we can cut that to four days, which makes the Q2 pricing roadmap feasible" |
| "Our tech debt is huge" | Vague complaint | "About 30% of the Orders team's capacity — roughly $580k a year — goes to working around this design" |
| "We should move to microservices" | Fashion | "Two teams block each other on every release; splitting Billing out removes that and cuts release coordination from two days to none" |
| "The .NET version is out of support" | Technical housekeeping | "From November 10 we can't get security patches for this system, which processes card data — that's an audit finding and a breach risk" |
| "This vendor's API is terrible" | Preference | "Integrating this vendor will take 8 weeks instead of 2, and their rate limits cap us at 3× today's volume" |
| "It depends" | Evasion | "It depends on whether we expect more than 50 enterprise tenants next year; if yes, option B; if no, option A" |

**Structure: bottom line up front (BLUF).** Lead with the recommendation and the ask; then the reasons; then the detail for those who want it. Barbara Minto's *pyramid principle* is the classic treatment.

```text
Recommendation:  Fund phase 1 of the ordering migration: 2 engineers for 4 months (≈ $175k).
Why:             Pricing changes take 3 weeks today and the Q2 roadmap needs 11 of them;
                 phase 1 cuts that to under a week and retires a $90k/year licence.
Risk & options:  If phase 1 doesn't hit the targets by March, we stop; we keep the faster pricing path either way.
Ask:             Approval today so we can start before the November code freeze.
```

**The three numbers.** For any significant proposal, be ready with: **what it costs** (range), **what it returns or avoids** (range, with timing), and **what happens if we don't** (the cost of doing nothing). Everything else is supporting detail.

**Honest uncertainty.** Give ranges and say what they depend on. Executives distrust single precise numbers for uncertain things — they've seen too many projects. *"Between $150k and $250k, most likely $180k; the range depends on how much of the legacy reporting we need to rebuild, which we'll know after the first month"* is credible. Pair uncertainty with a **checkpoint** that resolves it.

**Options, not ultimatums.** Present two or three real options (including *do nothing*), with trade-offs and a recommendation. It respects the executive's role — they decide — and it shows you've thought about alternatives.

**Know your audience:**

| Audience | Cares most about | Bring |
|---|---|---|
| **CFO / finance** | Cash, margin, predictability, capitalization | Cash flows by quarter, NPV/payback, run-cost impact, CapEx/OpEx split |
| **CEO / GM** | Growth, competitive position, risk to the business | Time-to-market, revenue enabled, strategic options |
| **CTO / CIO** | Delivery capacity, reliability, security, talent | Lead-time and incident metrics, risk register, team impact |
| **Product leadership** | Roadmap, customer outcomes | Which roadmap items get faster or become possible |
| **Risk / compliance** | Exposure, audit findings | Control gaps, deadlines, evidence |

**Use their artefacts.** If the company uses OKRs, tie the proposal to an objective. If finance has a business-case template, use it. Speaking the organization's language is part of speaking money.

**The interview-grade sentence:** *"With executives I translate technical problems into their units — lead time on roadmap items, capacity and dollars lost to a design, audit and breach exposure, revenue enabled — and lead with the bottom line: recommendation, why, risk, ask. I always have three numbers ready — what it costs, what it returns and when, and what not doing it costs — given as honest ranges with a checkpoint that resolves the uncertainty. I present two or three real options including doing nothing, with a recommendation, and I tailor the emphasis: cash flows and payback for the CFO, growth and risk for the CEO, delivery and reliability for the CTO."*

---

## Concept 37 — The business case for a migration or a debt paydown

Module 32 described *how* to migrate; this is *how to get it funded*. The same structure works for a large debt paydown, a platform upgrade programme or a build-vs-buy switch.

**Business-case skeleton** (template in Appendix A):

1. **Decision requested** — what, how much, by when.
2. **Problem in business terms** — with measured evidence (lead time, incidents, cost, risk, lost deals).
3. **Options** — do nothing, two or three alternatives, each with cost, benefit, timing, risk.
4. **Financials** — cash flows by year/quarter for each option vs the status quo; payback, NPV; sensitivity.
5. **Non-financial benefits and risks** — strategic options, talent, compliance.
6. **Plan** — phases, each delivering value; checkpoints with **kill criteria**.
7. **Measures** — how we'll know it's working (leading and lagging indicators).
8. **Recommendation.**

**Pricing the status quo** (Concept 6) — for a legacy system, typically:

| Status-quo cost line | Source |
|---|---|
| Licences and support contracts, with renewal increases | Contracts |
| Extended support fees for out-of-support components | Vendor quotes |
| Infrastructure that can't move to cheaper platforms | Bill |
| Engineering capacity lost to interest | Concept 30 model |
| Incident cost | Incident history × cost per incident |
| Risk exposure | Concept 7 expected loss |
| Lost revenue / cost of delay of blocked roadmap items | Product and sales |

**Pricing the coexistence tax honestly.** Module 32 insisted you price the cost of running old and new together. Make it explicit lines: transitional components (facade, sync, reconciliation), double hosting and licences during overlap, double on-call knowledge, data synchronization operations — **with the date each line ends**. Executives accept a coexistence tax when it's visible and time-bound; they lose trust when it appears as overruns.

**Front-load value — it's the strongest financial lever.** Because of discounting and cost of delay, a plan whose first slice retires a licence or speeds up the most-changed area has a much better NPV and payback than one that delivers everything at the end — even at the same total cost. That's the economic case for the strangler fig.

**Staged funding with kill criteria.** Ask for the first phase, not the whole programme. Define in advance:

- **What phase 1 will deliver and measure** (e.g., catalogue reads migrated; pricing lead time under 5 days; licence notice served).
- **Go criteria for phase 2** (targets met within tolerance; actual cost within 20% of estimate).
- **Kill or re-plan criteria** (e.g., velocity in the new platform no better than legacy after 3 months; reconciliation breaks unresolved).
- **What we keep if we stop** — every phase should leave the system better even if the programme ends (Module 32's "safe to pause" states).

This converts a frightening one-way decision into a sequence of two-way doors (Concept 8) — and the sensitivity analysis from Concept 4 shows executives exactly why you're proposing it.

**Arguing *against* a migration.** Sometimes the honest business case says no: the system is stable, rarely changed, cheap to run, and the risk is containable (patching, isolation, extended support). Saying *"don't migrate this — contain it and spend the money on X"* is as senior as leading a migration. Retire, retain and rehost are legitimate Rs (Module 32, Concept 29).

**The interview-grade sentence:** *"To get a migration or a big paydown funded I write a business case with the decision requested, the problem in measured business terms, real options including doing nothing — priced with its rising licence, support, interest, incident and risk costs — cash flows, payback, NPV and sensitivity for each, and a phased plan. I price the coexistence tax as explicit, dated lines, and I front-load value because moving benefits earlier is the strongest financial lever. I ask for phase one only, with go and kill criteria agreed in advance and every phase leaving the system better if we stop — and I'm prepared to recommend containment instead of migration when the numbers say so."*

---

## Concept 38 — Portfolio thinking and saying no

Architects rarely decide one investment in isolation; they shape a **portfolio** competing for the same people.

**Run / grow / transform.** A common lens (Gartner's "run, grow, transform" or the McKinsey-style three horizons): **run** the business (keep the lights on — operations, maintenance, compliance, platform upgrades), **grow** it (enhance existing products), **transform** it (new capabilities and markets). Making the split explicit — e.g., 30/50/20 — exposes two common pathologies: *run* starved until it fails noisily, or *run* consuming 70% because debt was never paid.

**Capacity allocation per team.** The same idea at team level: a protected share for maintenance and debt (Concept 33), the rest for roadmap. Agree it with product leadership once, rather than negotiating every refactoring.

**Make trade-offs explicit.** When someone asks for something new, ask what it replaces: *"We can do the partner integration this quarter if we move the reporting rebuild to Q3; that delays retiring the reporting licence and costs about $22k. Which do you prefer?"* That's not obstruction — it's making the decision-maker's trade visible.

**Saying no well:**

1. **Acknowledge the goal** — the requester has a real need.
2. **Show the cost** in their units — time, money, risk, displaced work.
3. **Offer alternatives** — a smaller version, a later date, a bought solution, a manual workaround.
4. **Let the accountable person decide**, and record it.

"No" from an architect should almost always be *"not like that, and here's what it would cost — here's what I'd do instead."*

**The interview-grade sentence:** *"I think in portfolios rather than single projects: run, grow and transform, with an explicit split so maintenance isn't starved until it fails or allowed to swallow everything because debt was never paid — and a protected maintenance share per team agreed once with product. When new work arrives I make the trade explicit — what it displaces and what that costs — and when I say no it's 'not like that': acknowledge the goal, show the cost in the requester's units, offer alternatives, and let the accountable person decide and record it."*

---

## Concept 39 — Cost, build-vs-buy and debt in the interview

These topics appear in three forms. Be ready for each.

### 1. Embedded in a system design round

Most candidates never mention cost. Do it briefly at three moments:

| Moment | What to say |
|---|---|
| **Requirements** | "Is there a budget envelope or target cost per order? I'll assume roughly X unless you tell me otherwise." |
| **High-level design** | "This is a generic capability — I'd use Entra External ID / Azure AI Search / Service Bus rather than build it." |
| **Deep dive / trade-offs** | "Multi-region active-active roughly doubles run cost; at this revenue, an extra nine saves less than it costs unless there's a contractual SLA — so I'd start with zone-redundant in one region and a tested regional failover." "Logging at this volume would cost more than the database; I'd sample." |
| **Wrap-up** | "Main cost drivers are Cosmos RUs and log ingestion; I'd put budgets, anomaly alerts and a cost-per-order fitness function in place." |

A minute of cost reasoning, at the right moments, distinguishes an architect from a designer.

### 2. Direct questions

*"How would you decide whether to build or buy X?"*, *"How do you manage technical debt?"*, *"Our cloud bill doubled — what do you do?"*, *"How would you convince leadership to fund a rewrite?"* — model answers below.

### 3. Behavioral

*"Tell me about a time you made the business case for a technical investment"*, *"…a build-vs-buy decision you got wrong"*, *"…when you pushed back on a costly design."* Structure (Module 34's STAR, architect-calibrated): the business problem in numbers → the options you considered → how you priced them → how you communicated → the decision and outcome in numbers → what you'd do differently.

### The "cloud bill doubled" arc (a frequent prompt)

1. **Don't panic, normalize**: did the business double? Compute unit cost.
2. **Find the delta**: cost analysis by service, resource, meter and tag over time; anomaly history; which deployment preceded the jump.
3. **Classify**: growth (fine), architecture regression (a release), waste (orphans, non-prod), rate (expired commitments), price change.
4. **Quick wins**: waste, non-production schedules, log volume, obvious right-sizing.
5. **Structural fixes**: the design issue behind the regression.
6. **Prevent recurrence**: budgets and anomaly alerts per workload, cost diffs in PRs, unit-cost fitness functions, ownership through tags, a monthly review.

### Common traps

| Trap | Better |
|---|---|
| Never mentioning cost in a design | Requirements, high-level choice, trade-off, wrap-up |
| "We'd build our own…" for a generic capability | Core vs generic; buy/rent with an exit plan |
| "Avoid vendor lock-in at all costs" | Expected switching cost; embrace where leverage is high |
| "Tech debt is the business's fault" | Debt is a shared decision; record loans, show interest, fund repayment |
| "We need to rewrite it" | Hotspots, containment, incremental paydown, business case with stages |
| Only infrastructure costs | People cost, opportunity cost, cost of delay |
| A single precise number | Ranges, sensitivity, checkpoints |

**The interview-grade sentence:** *"In a design round I mention cost at four moments — a budget assumption in requirements, buy-or-rent choices for generic components, priced trade-offs like what an extra nine or multi-region costs against what it saves, and the main cost drivers and guardrails at the end. For direct questions I use the frameworks — core versus generic and lifetime TCO for build-versus-buy, interest and hotspots for debt, unit cost and delta analysis for a bill spike. And in behavioral answers I tell the story in numbers: the business problem, the options and how I priced them, how I communicated, the decision and the measured outcome."*

---
# Worked examples

## Worked example 1 — Making the business case for the ordering migration

**Context** (Module 32's *Contoso Supply*): ASP.NET MVC 5 monolith; data-centre lease ends in 18 months; pricing and catalogue changes take ~6 weeks; a $90k/year reporting licence; two teams of six. Leadership asks: *"Why should we spend half a million on this?"*

**Step 1 — price the status quo** (annual, $k, ranges):

| Line | Today | Trend | Evidence |
|---|---|---|---|
| Data-centre extension after lease | — | 220–300 from month 19 | Landlord quote |
| Reporting licence | 90 | +8%/yr | Contract |
| Interest in Orders/Pricing (capacity lost) | 440–640 | Rising | 12 engineers × $243k loaded × 15–22% (cycle-time comparison and survey) |
| Incidents attributed to the monolith | 60–120 | Flat | Incident log × cost per incident |
| Cost of delay: 11 pricing roadmap items blocked | 300+ | Seasonal | Product estimate |

**Step 2 — options:**

| Option | Summary | Cost (yr 0–1) | Main benefit |
|---|---|---|---|
| A — Do nothing | Extend data centre; continue | 0 up-front; status-quo costs continue | None; costs rise |
| B — Rehost only | Lift-and-shift to Azure VMs | ≈ 120 | Avoids data-centre extension; no change-cost benefit |
| C — Replatform + strangle (recommended) | Module 32's plan | ≈ 550 over two years | Avoids extension, retires licence, cuts pricing lead time, reduces incidents |

**Step 3 — financials for C vs A** (incremental, $k): years 0–4 = −400, −150, +300, +350, +350 → **NPV₁₀ ≈ +214**, **payback ≈ 2.7 years**, **IRR ≈ 24.5%**. Sensitivity: benefits −30% → NPV ≈ −25; costs +50% → NPV ≈ −77.

**Step 4 — staging and kill criteria:** Phase 1 (4 months, ≈ $175k): replatform + facade + catalogue reads. Go criteria: hosting move done before lease notice date; pricing lead time < 2 weeks for migrated areas; spend within 20%. Kill/re-plan: if new-platform lead time is no better than legacy after phase 1, stop after replatforming (option B outcome, already paid for) and re-plan.

**Step 5 — the one-paragraph executive summary:** *"Recommend funding phase 1 of the ordering modernization: about $175k over four months. It gets us out of the data centre before the lease ends — avoiding $220–300k a year of extension costs — and starts cutting pricing change time from six weeks toward one, which unblocks eleven roadmap items. The full two-year programme returns roughly $210k in today's money and pays back in under three years, but it's sensitive to delivering the benefits, so we'll check real results after phase 1 and stop if they're not there; even then, we'll have left the data centre."*

## Worked example 2 — Build vs buy: customer identity for a B2B SaaS

**Context:** a .NET B2B SaaS with 40 enterprise customers, growing to 200; needs SSO federation with customers' identity providers (Entra ID, Okta, SAML), MFA, user self-service, audit, SCIM provisioning, and an API for machine clients.

**Classification:** generic subdomain — security-critical, standards-heavy, no customer pays more for a nicer login screen. Default: don't build.

| Option | Pros | Cons | Indicative 5-year TCO shape |
|---|---|---|---|
| **Build own OIDC server** | Full control | Months to build; perpetual security risk; certifications; enterprise federation edge cases | Highest — and highest risk |
| **Duende IdentityServer** (self-hosted, commercial) | Deep customization, mature, .NET-native | Licence; you host, patch, scale and secure it; UI and federation glue are yours | Medium: licence + 0.5–1 engineer ownership |
| **OpenIddict** (Apache-2.0) | No licence fee, .NET-native, flexible | More assembly and ownership than Duende; same hosting burden | Medium: no licence, more engineering |
| **Keycloak** (Apache-2.0, Java) | Feature-rich (federation, SCIM via extensions), free | Different stack to operate; upgrades; Java skills | Medium |
| **Managed CIAM** (Entra External ID, Auth0/Okta CIC) | Federation, MFA, compliance, scaling handled; fastest time-to-value | Per-MAU/per-connection pricing grows; less control; lock-in at the identity layer | Lowest early; check pricing at 200 tenants and per-federation charges |

**Decision process:** must-haves (enterprise federation per tenant, SCIM, MFA, audit export, .NET SDK quality, EU data residency); PoC with three real customer IdPs; model price at 200 tenants and 3× users; check per-enterprise-connection charges (often the tier-jump trap).

**Typical outcome:** managed CIAM, accessed through standard OIDC from ASP.NET Core (so the application depends on the protocol, not the vendor SDK — a cheap option, Concept 8), with an ADR recording the exit plan (export users and federation configs; re-point OIDC). Revisit if per-connection pricing at 200 enterprise tenants exceeds the TCO of a self-hosted option by a large margin — at which point Duende or OpenIddict on Container Apps becomes the rational move, with real requirements in hand.

## Worked example 3 — Cutting an Azure bill by 35% without hurting SLOs

**Context:** a SaaS platform's monthly bill is **$120k**: AKS compute $42k, Cosmos DB $28k, Log Analytics $18k, Azure SQL $12k, bandwidth/egress $8k, non-production $12k. Revenue grew 15% this year; the bill grew 60%. Unit cost per active tenant rose from $310 to $430.

**Diagnosis** (cost analysis by meter and tag, plus telemetry):

1. **Log Analytics** jumped after a release that logged full request/response bodies at Information level for "debugging".
2. **Non-production** runs 24/7 on production-sized node pools.
3. **Cosmos DB**: a new query feature does cross-partition queries; indexing policy indexes everything including large blobs; provisioned RU/s sized for peak.
4. **AKS**: CPU requests set 3× actual; nodes at 25% utilization; no commitment coverage.
5. **Egress**: images served from Blob Storage directly to users worldwide.

**Actions and savings (monthly):**

| Action | Saving |
|---|---|
| Log level and category filters, drop bodies, sample successful requests, verbose tables to cheaper plan, 30-day retention for traces | −$11k |
| Non-production off nights/weekends, smaller node pools, ephemeral PR environments | −$7k |
| Cosmos: fix the query to stay in-partition, selective indexing policy, autoscale instead of fixed peak RU/s | −$9k |
| AKS: right-size requests from measured usage, cluster autoscaler, savings plan on the stable baseline | −$12k |
| Front Door caching for static images | −$3k |
| **Total** | **−$42k / month (35%)** |

**New unit cost:** $78k ÷ the same tenant count → about **$280 per tenant**, below last year's $310. **Prevention:** per-workload budgets and anomaly alerts, Infracost-style diffs in IaC PRs, a fitness function on log GB per 1,000 requests, the Cosmos RU assertions in CI from Concept 17, and a monthly review of the top ten lines. Engineering effort: about 3 engineer-weeks (≈ $16k) for ≈ $500k/year saved.

## Worked example 4 — Funding a hotspot paydown

**Context:** an 8-engineer team; hotspot analysis shows **40% of their changes** touch the pricing module (`PricingEngine`, `DiscountRules`, a 2,000-line `PriceCalc`). CodeScene code health in those files is "red"; cycle time for pricing tickets is about 2.2× that of similar tickets elsewhere.

**Interest estimate:** team loaded cost 8 × $243k = **$1.94M/yr**; 40% in the hotspot = **$778k/yr** of effort; assume refactoring reduces the effort of hotspot work by **40%** (deliberately conservative versus the Code Red 124% figure) → **≈ $311k/yr** of capacity freed.

**Principal:** 2 engineers × 10 weeks = 20 engineer-weeks ≈ **$106k**, using branch by abstraction and the golden-master tests from Module 32's Worked example 3.

**Case:** payback ≈ **4 months**; five-year net ≈ **$1.45M** of capacity (cash only if headcount changes — say so, Concept 4). Even if the effect is half as large, payback is under 9 months. Add the *cost of delay* angle: the Q2 pricing roadmap of 11 items doesn't fit at current speed.

**How it's presented:** *"Pricing work consumes 40% of the Orders team and takes more than twice as long as comparable work. A ten-week, two-person refactoring should free around $300k of capacity a year — paying back in about four months — and it's what makes the Q2 pricing roadmap fit. We'll measure cycle time on pricing tickets monthly; if we don't see at least a 25% improvement by week 6, we stop and keep what's done."*

## Worked example 5 — The price of a nine

**Context:** a checkout API at 99.9% availability; a proposal for multi-region active-active to reach 99.99%; business revenue $50M/year through the API.

- Downtime: 99.9% ≈ 8.76 h/yr; 99.99% ≈ 0.88 h/yr → ≈ 7.9 h avoided.
- Revenue at risk: ≈ $5.7k/h average; ×3 peak factor ≈ $17k/h; partial recovery (customers retry) 40% → net ≈ $10k/h → **≈ $80k/yr** saved.
- Cost of active-active: second region infrastructure ≈ $9k/month ($108k/yr), Cosmos multi-region writes and data transfer ≈ $30k/yr, engineering for conflict handling, testing and operations ≈ 0.5 engineer ($120k/yr) → **≈ $260k/yr**.

**Conclusion:** on direct revenue alone the extra nine costs about three times what it saves. **Unless** there's a contractual SLA with penalties, a regulatory requirement, or brand risk you can quantify, the better investment is the cheaper middle option — zone redundancy plus a tested, automated regional failover (warm standby), and reducing *mean time to restore*, which often buys most of the availability for a fraction of the cost (Module 13). The architect's contribution is not the verdict; it's turning "we need 99.99%" into a priced trade-off the business can decide.

## Worked example 6 — Responding to a relicensing

**Context:** a .NET platform of 14 services uses MediatR 12 in all of them (about 220 handlers, 6 pipeline behaviours: validation, logging, transactions, caching, authorization, metrics) and AutoMapper 13 in 9 services (~300 maps, some with custom resolvers). The company is above the community-edition revenue threshold.

| Option | MediatR | AutoMapper |
|---|---|---|
| **Pay** | Commercial licence per year (team-size tier) — compare with engineer-days | Same |
| **Pin** | Stay on 12.x with a review date; acceptable short-term (stable, small surface) | Stay on 14.x; same caveats |
| **Replace** | Plain handlers + decorators or a source-generated mediator; behaviours become endpoint filters/decorators; ≈ 2–4 engineer-weeks with tests | Mapperly or hand-written mappings; reflection-based custom resolvers need rewriting; ≈ 3–5 engineer-weeks; finds latent mapping bugs |
| **Fork** | Not worth it | Not worth it |

**A reasonable decision:** pin both immediately (ADR, review in 6 months); replace AutoMapper incrementally as services are touched (fix-as-you-touch, with a ratchet test counting `IMapper` usages), because compile-time mapping has independent value; for MediatR, compare the licence with the replacement cost — if the licence is a few engineer-days a year and the pipeline behaviours are load-bearing, **paying may be the cheapest option**; if behaviours are thin, replace. Record the reasoning, including that principle is not a cost line.

---
# Common interview questions with model answers

**Q1. "How do you decide whether to build or buy?"**
> "First, is it core — does doing it better differentiate us? I use DDD's core, supporting and generic subdomains and Wardley's evolution axis. If it's generic and a good-enough product exists, I lean to buy or rent. Then I compare lifetime TCO on the same table — build including maintenance, operations and opportunity cost; buy including subscription growth, integration, customization and exit — plus time-to-value, risk and exit cost. For uncertain needs, buying first behind a port is often the cheapest way to learn, and I record the decision with an exit plan."

**Q2. "What is technical debt, and how do you manage it?"**
> "Cunningham's sense: shortcuts or outdated understanding in the code that make every future change cost more — principal to fix, interest on each change. Interest is only paid where we change code, so I find hotspots — complex, frequently changed code — and combine that with delivery metrics per area and developer surveys. Deliberate debt gets written terms: reason, expected interest, repayment trigger, owner. Repayment is funded through fix-as-you-touch, a protected capacity share and investment cases for big items, prioritized by interest, with ratchets so it can't grow meanwhile."

**Q3. "How would you convince leadership to fund paying down debt?"**
> "Translate it: show capacity lost in the area — say 40% of a team's changes with twice the cycle time — as money and as roadmap delay; show incidents and risk; propose a small, time-boxed paydown with a measurable target like cycle time on pricing tickets, a payback estimate with a conservative assumption, and a checkpoint to stop if it's not working. Tie it to a roadmap outcome they want, not to code quality."

**Q4. "Our cloud bill doubled. What do you do?"**
> "Normalize first — did the business double? Then find the delta by service, meter, tag and time, correlate it with deployments and anomalies, and classify it as growth, architectural regression, waste, rate change or price change. Quick wins are usually waste, non-production and log volume; then fix the design regression; then prevent it — budgets and anomaly alerts per workload, cost diffs in PRs, a unit-cost fitness function and owners for the top lines."

**Q5. "How do you think about cost in a system design?"**
> "As a quality attribute with a budget and a unit-cost target. I estimate the main meters at launch and 10× — compute, database throughput, storage, transfer, logs, tokens — and price them, I choose buy or rent for generic components, I price trade-offs like extra nines or multi-region against what they save, and I design guardrails: budgets, alerts, tags and fitness functions. And I remember people cost usually dominates infrastructure."

**Q6. "Reservations or savings plans?"**
> "Commitments cover the stable baseline; pay-as-you-go covers the variable top. Savings plans commit to hourly spend and follow usage across eligible services, SKUs and regions — right when the shape will change, as in a migration. Reservations give the biggest discount for a specific SKU and region but, from February 2027, new ones for savings-plan-covered services can't be exchanged, so I'd use them only where I'm confident for the term. And I track commitment utilization — an unused reservation is pure waste."

**Q7. "How do you avoid vendor lock-in?"**
> "I don't avoid it at all costs — I price it. Expected switching cost is the cost of switching times the probability, and abstracting everything to stay portable often costs more than a switch would while denying us the platform's benefits. I embrace the platform where it gives leverage, keep cheap options at the important seams — standard protocols like OIDC and OpenTelemetry, domain code free of SDK types, exportable data — and negotiate data return and price caps in contracts. Regulation like the EU Data Act removes switching fees, but not re-implementation cost."

**Q8. "One of your key open-source dependencies just went commercial. What do you do?"**
> "Inventory where and how deeply we use it, then compare four options on TCO: pay — often a few engineer-days a year; pin to the last permissive version with a review date, accepting dependency debt; replace — cheap if usage is shallow, as with mediators and mappers; or fork — rarely worth it. I record the decision in an ADR. Longer term, I keep library types out of the domain so replacement is days not months, and I have licence scanning and SBOMs in CI so we see these changes."

**Q9. "When is it right to take on technical debt?"**
> "When the cost of delay exceeds the interest: a launch window, a learning experiment, start-up survival, a regulatory date or sacrificial architecture. It must be deliberate and prudent, contained behind a seam, and written down with the business co-signing — the reason, expected interest, a repayment trigger and an owner — so repaying it later is honouring an agreement."

**Q10. "How do you justify a higher-availability architecture?"**
> "Translate both sides into money: the downtime avoided per year times revenue per hour, adjusted for peak, recovery, penalties and response cost, versus the full cost of the design including people. For many businesses an extra nine costs more than it saves on direct revenue, so the justification has to come from contractual SLAs, regulation or quantified brand risk — or we choose a cheaper option such as zone redundancy with tested regional failover and faster recovery."

**Q11. "What's the TCO of building something in-house?"**
> "Much more than the build: edge cases and hardening, maintenance and upgrades every year — which over a system's life usually exceed the build — operations and on-call, security and compliance work, knowledge risk, and above all opportunity cost, because the team isn't working on the core. People cost should be fully loaded, typically 1.25–1.5× salary."

**Q12. "How do you measure technical debt?"**
> "By its interest rather than its volume: hotspots from version-control history combined with complexity or code health; cycle time, change failure rate and defect density per area; incidents; onboarding time; dependency and runtime age; and developer surveys, which Google's researchers found more useful than code metrics for locating debt that hinders engineers. Static-analysis 'days of debt' are useful for trends, not as a price."

**Q13. "Does AI change build-vs-buy?"**
> "It cuts the cost to write code, not to own it. I'd now build thin internal tools, prototypes and replacements for shallow libraries — but keep buying where value comes from depth, compliance, scale or ecosystem, because patching, upgrades, on-call and audit haven't got cheaper. And AI-built tools need the same governance as other software, or they become shadow IT."

**Q14. "How do you control the cost of an LLM feature?"**
> "Model cost per successful task, not per call: tokens in, cached and out, plus retrieval and retries. Architectural levers first — stable prefixes for prompt caching, routing to the cheapest model that passes evaluations, trimmed context, capped output; then Batch pricing for offline work and provisioned throughput only when measured utilization beats pay-as-you-go. For agents, step and token caps per task and per tenant, anomaly alerts on token meters, and a check that heavy users don't make the feature margin-negative."

**Q15. "Explain CapEx versus OpEx and why it matters to you."**
> "Capital spend creates an asset depreciated or amortized over years; operating spend is expensed in the period. Cloud turns hardware CapEx into OpEx; capitalizable development of internal-use software is usually new capability, while maintenance is expensed. It matters because it changes how a proposal looks on the P&L and which budget pays — so I give finance a clean breakdown of the work and let them make the call."

**Q16. "Tell me about a build-vs-buy decision that went wrong."**
> *Structure:* the decision and the reasoning at the time → what you underestimated (usually ownership on the build side, or customization and pricing growth on the buy side) → the signal that showed it → how you corrected (replace, renegotiate, contain behind a port) → the changed practice (e.g., "we now model pricing at 3× and 10× volume and require a PoC on our hardest scenario before signing", "we now include five years of maintenance in every build estimate"). *Key signal:* ownership and a mechanism-level lesson.

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Framing | Cost is finance's problem | Architecture as capital allocation; cost as a quality attribute |
| Requirements | No budget or unit-cost target | Asks for one or proposes one to be corrected |
| TCO | Compares subscription fee with build estimate | Same lifetime table: maintenance, ops, compliance, opportunity, exit |
| People cost | Ignores it or uses raw salary | Fully loaded; often the dominant line |
| Time | Bare ROI or totals | Cash flows by year, payback, NPV, sensitivity |
| Delay | Counts cost to do, not to wait | Cost of delay; CD3 sequencing; front-loaded value |
| Status quo | Treated as free | Priced, rising, sometimes the right answer |
| Risk | "High", "low" | Expected loss ranges compared with mitigation cost |
| Availability | More nines = better | Price of a nine vs downtime avoided |
| Options | Hedge everything, or nothing | Buy cheap options where change is plausible; defer one-way doors |
| Finance | Ignores CapEx/OpEx | Understands enough to give finance clean data |
| Cloud bill | Totals | Unit economics and elasticity |
| Rate | Buy reservations for everything | Baseline commitments, savings plans where shape changes, utilization tracked |
| Usage | Micro-optimizes code | Top lines first: waste, non-prod, logs, right-sizing, data lifecycle |
| Governance | Monthly bill surprise | Policy guardrails, budgets, anomaly alerts, PR cost diffs, fitness functions |
| Observability | Logs everything at Information | Sampling, levels, cheaper plans, retention by class |
| AI cost | Per-call thinking, no caps | Cost per successful task, caching, routing, batch, caps |
| Build vs buy | Builds generic capabilities | Core vs generic; Wardley evolution; build the core |
| Buying | Demo-driven | Weighted criteria before demos, PoC on hardest scenario, pricing at 10×, exit terms |
| Lock-in | Avoid at all costs | Expected switching cost; leverage where it pays |
| OSS | "Free" | Ownership, supply chain, licence; pay/pin/replace/fork on TCO |
| Debt definition | Everything ugly is debt | Principal and interest; interest only where code changes |
| Debt finding | Static-analysis totals | Hotspots, delivery metrics per area, surveys |
| Deliberate debt | Silent shortcuts | Written loan terms, repayment trigger, business co-signs |
| Paydown | Big rewrite project | Prioritized by interest; fix-as-you-touch, capacity share, staged investment cases |
| Erosion | Code review only | Architecture tests and ratchets |
| Platform | Upgrades when forced | Every LTS from maintenance capacity; EOS dates as cliffs |
| AI and debt | Ships generated code unreviewed | Conventions, explainability standard, prompts as code, reinvest in refactoring |
| Executives | Technical arguments, single numbers | BLUF, three numbers, ranges, options, recommendation |
| Migration case | Ask for the whole programme | Staged funding, kill criteria, safe-to-pause phases |
| Saying no | "That's a bad idea" | "Not like that — here's the cost and an alternative" |

---
# Practice exercises

1. **Loaded-cost card.** Find (or estimate) the fully loaded cost of an engineer in your market. Compute cost per week and per day. Use it to price the last three "small" technical initiatives you saw.
2. **NPV and sensitivity.** Using the C# `Finance` helper, model a migration you know: cash flows by year, NPV at 8% and 15%, payback, and three downside scenarios. Write the one-paragraph executive summary.
3. **Cost of delay.** Take five items from a real backlog, estimate CoD per week and duration, sequence by CD3, and compute total delay cost for your current order vs CD3 order.
4. **Price a nine.** For a system you know, compute revenue per hour, adjust for peak and recovery, and compare the value of going from its current availability to the next nine with a realistic cost of the architecture to get there.
5. **Back-of-envelope cost estimate.** For Module 37's URL shortener or notification system, estimate compute, database, storage, transfer and logs at launch and at 10×, price them with the Azure Pricing Calculator or the Retail Prices API code in Concept 16, and compute unit cost and elasticity.
6. **Log audit.** Run the KQL `Usage` query on a Log Analytics workspace. Identify the top three tables, trace each back to code or configuration, and propose changes with estimated savings.
7. **Budget as code.** Deploy the Bicep budget from Concept 16 to a sandbox subscription with actual and forecast alerts, plus an Azure Policy assignment requiring a `costCenter` tag.
8. **Cosmos cost fitness function.** Against the Cosmos DB emulator, write tests that assert request charges for your main queries; deliberately introduce a cross-partition query and an over-broad indexing policy and watch the tests fail.
9. **Build-vs-buy TCO.** Pick a generic capability your team built (feature flags, notifications, a job scheduler, an admin portal). Fill in the Concept 22 table honestly over five years for build vs the best product, and write the ADR you'd write today.
10. **Relicensing drill.** Inventory your solution's dependencies and licences (`dotnet list package --include-transitive` plus a licence scanner). For each package under a non-permissive or commercial licence, write the pay/pin/replace/fork decision with estimated costs.
11. **Hotspot interest.** Run a `git log` hotspot analysis (Module 32, Concept 7) and compare cycle times of tickets touching the top five hotspots vs the rest. Convert the difference into annual capacity cost and write a paydown proposal using Worked example 4's structure.
12. **Debt register.** Create entries (Concept 32 YAML) for the three biggest debt items in a codebase you know, with interest estimates, repayment triggers and owners. Add one ratchet test with NetArchTest.
13. **Executive rewrite.** Take a technical proposal you've written or seen. Rewrite it as a half-page BLUF with the three numbers, options including do-nothing, a recommendation and a checkpoint.
14. **Interview drill.** Out loud, in 10 minutes each: (a) "Our cloud bill doubled"; (b) "Build or buy customer identity for our SaaS?"; (c) "Convince our CFO to fund paying down debt in the billing module." Record and check: did you normalize, price, give ranges, offer options and recommend?

---
# Free resources and learning material

All free to read online unless marked *(book)*. Start with the ★ items. Pages for policies, prices and licences were checked on October 7, 2026.

### Economics for architects: value, delay, risk and options
- ★ [Cost of Delay — Black Swan Farming (Joshua Arnold)](https://blackswanfarming.com/cost-of-delay/) — the clearest free introduction, with urgency profiles.
- ★ [Cost of Delay Divided by Duration (CD3)](https://blackswanfarming.com/cost-of-delay-divided-by-duration/) — the scheduling method, with a worked comparison against FIFO.
- [WSJF — Scaled Agile Framework](https://framework.scaledagile.com/wsjf) — the relative-scoring variant used in SAFe.
- [Net present value — Investopedia](https://www.investopedia.com/terms/n/npv.asp) — NPV, discount rates and a worked example in plain language.
- [Real Options Underlie Agile Practices — InfoQ](https://www.infoq.com/articles/real-options-enhance-agility/) — options thinking applied to software decisions.
- ★ [Embracing Risk — Google SRE Book](https://sre.google/sre-book/embracing-risk/) — the cost of each extra nine and why 100% is the wrong target.
- [Sacrificial Architecture — Martin Fowler](https://martinfowler.com/bliki/SacrificialArchitecture.html) — when building to throw away is the economical choice.
- [Is High Quality Software Worth the Cost? — Martin Fowler](https://martinfowler.com/articles/is-quality-worth-cost.html) — why internal quality *lowers* cost; the best one-page argument for executives.
- *(book)* Donald Reinertsen — *The Principles of Product Development Flow* — cost of delay, batch size and queues.
- *(book)* Gregor Hohpe — *The Software Architect Elevator* — riding between the engine room and the penthouse; see also [architectelevator.com](https://architectelevator.com/).

### Cloud cost, unit economics and FinOps
- ★ [The Frugal Architect — Werner Vogels](https://thefrugalarchitect.com/) — seven laws for cost-aware architecture.
- ★ [FinOps Framework — FinOps Foundation](https://www.finops.org/framework/) — principles, phases, domains, capabilities, personas, scopes.
- ★ [FinOps Framework 2026 — what changed](https://www.finops.org/insights/2026-finops-framework/) — Executive Strategy Alignment, technology categories, converging disciplines.
- [State of FinOps 2026](https://data.finops.org/) — survey data on priorities, AI spend and FinOps reporting lines.
- ★ [FOCUS — FinOps Open Cost and Usage Specification](https://focus.finops.org/) — the billing-data schema, column library and use-case queries.
- [Introducing FOCUS 1.3](https://www.finops.org/insights/introducing-focus-1-3/) — contract commitments, split cost allocation, recency and completeness.
- [37signals cloud exit](https://basecamp.com/cloud-exit) and [DHH — "Our cloud-exit savings will now top ten million over five years"](https://world.hey.com/dhh/our-cloud-exit-savings-will-now-top-10-million-over-five-years-c7d9b5bd) — the repatriation case, in their own words.
- [Dropbox — Magic Pocket](https://dropbox.tech/infrastructure/magic-pocket-infrastructure) — the classic case of building storage infrastructure at a scale where owning wins.
- *(book)* J.R. Storment & Mike Fuller — *Cloud FinOps*, 2nd ed. (O'Reilly).

### Azure cost engineering
- ★ [Cost Optimization pillar — Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/cost-optimization/) — with the [design principles](https://learn.microsoft.com/azure/well-architected/cost-optimization/principles), [checklist](https://learn.microsoft.com/azure/well-architected/cost-optimization/checklist) and [trade-offs](https://learn.microsoft.com/azure/well-architected/cost-optimization/tradeoffs).
- ★ [Microsoft Cost Management documentation](https://learn.microsoft.com/azure/cost-management-billing/) — cost analysis, budgets, exports, allocation.
- [Create and manage budgets](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets) and [identify anomalies and unexpected changes in cost](https://learn.microsoft.com/azure/cost-management-billing/understand/analyze-unexpected-charges).
- [Exports, including FOCUS format](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-improved-exports).
- ★ [Microsoft FinOps documentation](https://learn.microsoft.com/cloud-computing/finops/) and the [FinOps toolkit](https://learn.microsoft.com/cloud-computing/finops/toolkit/finops-toolkit-overview) ([GitHub](https://github.com/microsoft/finops-toolkit)) — FinOps hubs, Power BI reports, workbooks.
- ★ [Self-service exchanges and refunds for Azure Reservations](https://learn.microsoft.com/azure/cost-management-billing/reservations/exchange-and-refund-azure-reservations) and the [July 2026 announcement on exchanges for savings-plan-covered services](https://techcommunity.microsoft.com/t5/finops-blog/reservation-exchanges-for-azure-services-covered-by-savings/ba-p/4542437).
- [Azure savings plan for compute — overview](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/savings-plan-compute-overview) and [trading in reservations for a savings plan](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/reservation-trade-in).
- [Azure Hybrid Benefit](https://azure.microsoft.com/pricing/offers/hybrid-benefit/) — using existing Windows Server and SQL Server licences.
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) and [Azure Retail Prices REST API](https://learn.microsoft.com/rest/api/cost-management/retail-prices/azure-retail-prices).
- [Azure Advisor cost recommendations](https://learn.microsoft.com/azure/advisor/advisor-cost-recommendations).
- [Azure Monitor Logs cost calculations](https://learn.microsoft.com/azure/azure-monitor/logs/cost-logs) and [table plans (Analytics, Basic, Auxiliary)](https://learn.microsoft.com/azure/azure-monitor/logs/logs-table-plans) — the levers behind the log firehose.
- [Optimize Azure Cosmos DB costs](https://learn.microsoft.com/azure/cosmos-db/plan-manage-costs) and [optimize reads and writes](https://learn.microsoft.com/azure/cosmos-db/optimize-cost-reads-writes).
- [AKS cost analysis](https://learn.microsoft.com/azure/aks/cost-analysis) — allocation inside shared clusters.
- [Assign policy definitions for tag compliance](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-policies) — enforcing allocation tags.
- [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/) and [free data transfer out when leaving Azure](https://azure.microsoft.com/updates/now-available-free-data-transfer-out-to-internet-when-leaving-azure/).
- [Azure OpenAI pricing (Standard, Provisioned, Batch)](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/) — check current per-token prices before modelling.
- [Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/) — cost governance and the migration Rs at portfolio level.
- [Infracost documentation](https://www.infracost.io/docs/) and [ARM and Bicep support](https://www.infracost.io/resources/blog/arm-bicep-support-in-cli) — cost diffs in pull requests.

### Build vs buy, strategy and lock-in
- ★ [Learn Wardley Mapping](https://learnwardleymapping.com/) and [Simon Wardley's book (free on Medium)](https://medium.com/wardleymaps) — evolution, value chains and what to build, buy or outsource.
- ★ [DDD Crew — Core Domain Charts](https://github.com/ddd-crew/core-domain-charts) — plotting differentiation against complexity to find the core.
- [Utility vs Strategic Dichotomy — Martin Fowler](https://martinfowler.com/bliki/UtilityVsStrategicDichotomy.html) — why not all software deserves the same investment.
- ★ [Don't get locked up into avoiding lock-in — Gregor Hohpe](https://martinfowler.com/articles/oss-lockin.html) — switching cost × probability; types of lock-in.
- [Choose Boring Technology — Dan McKinley](https://mcfunley.com/choose-boring-technology) ([slides](https://boringtechnology.club/)) — innovation tokens and the cost of novelty.
- [Thoughtworks Technology Radar](https://www.thoughtworks.com/radar) — an informed second opinion on tools, platforms and techniques.
- [EU Data Act — European Commission](https://digital-strategy.ec.europa.eu/en/policies/data-act) — applies since 12 September 2025; cloud switching rules.
- [EU Data Act: when cloud switching fees are abolished — cloudmagazin](https://www.cloudmagazin.com/en/2026/07/06/eu-data-act-when-cloud-switching-fees-are-abolished-what-cios-need-to-examine/) — what changes on 12 January 2027, and what doesn't.
- [Surviving the "SaaSpocalypse" — build, buy and AI (JLL research, July 2026, PDF)](https://www.jll.com/content/dam/jllcom/en/us/documents/reports/research-reports/26-guides-surviving-the-saaspocalypse.pdf) — a sober industry view of AI's effect on build-vs-buy.

### Open source economics and the .NET relicensing wave
- ★ [.NET OSS Projects: Better to Re-license or Die? — Aaron Stannard](https://aaronstannard.com/relicense-or-die/) — the consumer's options and the sustainability argument.
- [AutoMapper and MediatR commercial editions launch — Jimmy Bogard](https://www.jimmybogard.com/automapper-and-mediatr-commercial-editions-launch-today/) — the licence model in the author's words.
- [MassTransit documentation (Massient)](https://masstransit.massient.com/) — v9 licensing and configuration.
- [Fluent Assertions relicensing — DevClass](https://www.devclass.com/2025/01/16/another-open-source-project-shifts-to-restrictive-license-fluent-assertions-following-xceed-partnership) — what changed and how the community responded.
- [QuestPDF licence selection guide (v3, July 2026)](https://www.questpdf.com/license/guide.html) — revenue thresholds and eligibility, as an example of reading a licence carefully.
- [Redis is open source again — The New Stack](https://thenewstack.io/redis-is-open-source-again/) and [Valkey](https://valkey.io/) — the relicensing, the fork and the return to AGPL.
- [Riok.Mapperly](https://github.com/riok/mapperly) and [Mediator (source generator)](https://github.com/martinothamar/Mediator) — compile-time alternatives to reflection-based libraries.
- [OpenSSF Scorecard](https://scorecard.dev/) and [deps.dev](https://deps.dev/) — dependency health signals before you adopt.
- [Microsoft SBOM tool](https://github.com/microsoft/sbom-tool) — generating SBOMs for .NET builds.
- [Central Package Management](https://learn.microsoft.com/nuget/consume-packages/central-package-management) and [NuGet auditing for vulnerabilities](https://learn.microsoft.com/nuget/concepts/auditing-packages).
- [Dependabot](https://docs.github.com/code-security/dependabot) and [Renovate](https://docs.renovatebot.com/) — automated dependency updates.

### Technical debt
- ★ [Technical Debt — Martin Fowler](https://martinfowler.com/bliki/TechnicalDebt.html) — cruft, principal and interest; links to Cunningham's explanation.
- ★ [Technical Debt Quadrant — Martin Fowler](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html) — deliberate/inadvertent × reckless/prudent.
- [The WyCash Portfolio Management System — Ward Cunningham, OOPSLA 1992](http://c2.com/doc/oopsla92.html) — the original metaphor, in two paragraphs.
- ★ [Code Red: The Business Impact of Code Quality — Tornhill & Borg (arXiv)](https://arxiv.org/abs/2203.04374) — 15× defects, 124% longer time-in-development, 9× cycle-time uncertainty.
- [Increasing, not Diminishing: Investigating the Returns of Highly Maintainable Code (arXiv)](https://arxiv.org/abs/2401.13407) — follow-up regression analysis on the same data.
- [Defining, Measuring, and Managing Technical Debt — Jaspan & Green, Google (IEEE Software 2023)](https://research.google/pubs/defining-measuring-and-managing-technical-debt/).
- [Tech debt: Reclaiming tech equity — McKinsey](https://www.mckinsey.com/capabilities/mckinsey-digital/our-insights/tech-debt-reclaiming-tech-equity) and [Breaking technical debt's vicious cycle — McKinsey](https://www.mckinsey.com/capabilities/mckinsey-digital/our-insights/breaking-technical-debts-vicious-cycle-to-modernize-your-business) — the CIO-survey numbers executives quote.
- [Hidden Technical Debt in Machine Learning Systems — Sculley et al., NeurIPS 2015](https://papers.nips.cc/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html).
- [NDepend — technical debt and annual interest](https://www.ndepend.com/docs/technical-debt) — debt and interest per issue for .NET code.
- [code-maat — Adam Tornhill](https://github.com/adamtornhill/code-maat) — hotspots and temporal coupling from git history.
- [libyear](https://libyear.com/) — a simple dependency-freshness metric.
- [NetArchTest](https://github.com/BenMorris/NetArchTest) and [ArchUnitNET](https://github.com/TNG/ArchUnitNET) — architecture fitness functions for .NET.
- [The Mikado Method](https://mikadomethod.info/) — sequencing large refactorings safely.

### AI, productivity and debt
- ★ [Announcing the 2025 DORA Report: State of AI-assisted Software Development](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) — AI as amplifier; throughput up, stability still down; the AI Capabilities Model.
- [DORA research](https://dora.dev/) — the metrics and capabilities behind "cost of change".
- [AI Copilot Code Quality 2025 — GitClear](https://www.gitclear.com/ai_assistant_code_quality_2025_research) — duplication rising, refactoring falling.

### Accounting and tax context (not advice)
- [FASB revises internal-use software cost guidance (ASU 2025-06) — Crowe](https://www.crowe.com/insights/take-into-account/fasb-revises-internal-use-software-cost-guidance) — what replaced the project stages, and when.
- [IAS 38 Intangible Assets — IFRS Foundation](https://www.ifrs.org/issued-standards/list-of-standards/ias-38-intangible-assets/) — the international rules for capitalizing development.
- [New Section 174A restores domestic R&E deductibility — Morgan Lewis](https://www.morganlewis.com/pubs/2025/07/new-section-174a-restores-domestic-r-and-e-deductibility-but-other-changes-bring-mixed-results) — US tax treatment of software development from 2025.

### Communicating with executives
- [Writing an engineering strategy — Will Larson](https://lethain.com/eng-strategies/) — diagnosis, policies and actions; a model for decision documents.
- [StaffEng](https://staffeng.com/) — essays and stories on influence, strategy and working with leadership.
- *(book)* Barbara Minto — *The Pyramid Principle* — answer first, then the supporting logic.
- *(book)* Neal Ford, Rebecca Parsons, Patrick Kua & Pramod Sadalage — *Building Evolutionary Architectures*, 2nd ed. — fitness functions, including for cost.

---
# Quick-recall sheet

**One sentence.** Every architecture decision is an investment: make the trade visible in money, time and risk, and choose deliberately.

**Three costs.** Build (visible, underestimated) · run (recurring) · change (invisible, largest, inflated by debt).

**Cost is an NFR.** Trades with availability, latency, consistency, security, observability, speed. Ask for (or propose) a budget envelope and a unit-cost target. WAF: cost discipline · cost-efficiency mindset · usage optimization · rate optimization · monitor and optimize.

**TCO.** People (loaded 1.25–1.5×) + infra + licences + integration + risk + opportunity + exit, over 7–15 years. Hidden: non-prod, observability, compliance, upgrades, key-person risk, decommissioning.

**Money over time.** Cash flows by year · payback · NPV at finance's rate · IRR vs hurdle · sensitivity (benefits −30%, costs +50%) · cash vs freed capacity · front-load value.

**Cost of delay.** $/week of waiting; shapes: linear, step, cliff, decay, growing risk. CD3 = CoD ÷ duration, highest first. Big-bang maximizes CoD; debt stretches every duration.

**Status quo** is an option with a rising price. Sunk cost is irrelevant.

**Risk in money.** Probability × impact; ranges. Downtime: revenue/h × peak × (1 − recovery) + penalties + response. Price each nine against what it saves.

**Options.** Seams, modules, expand/contract, flags, short commitments, standards, spikes. Don't overbuy. Last responsible moment; one-way vs two-way doors.

**CapEx/OpEx.** Asset amortized vs period expense. ASU 2025-06: no project stages; probable-to-complete (effective FY after Dec 15, 2027). IAS 38 elsewhere. US §174A: domestic software R&E deductible from 2025; foreign 15-yr. Finance decides.

**Cloud bills.** Meters × rates. Provisioned · autoscaled · consumption · tiered · commitment. Levers: usage · rate · architecture · waste. List vs contracted vs effective.

**Drivers.** Compute floors/idle · DB throughput (Cosmos RU: partition key, point reads, indexing, consistency) · storage tiers · transfer/egress · appliances · logs · tokens · licences · people.

**Unit economics.** Cost per order/tenant/conversation; gross margin; elasticity sub-linear good, super-linear = bug. Instrument usage (RequestCharge, tokens) by plan, not high-cardinality tenant tags; documented shared-cost rules. Frugal Architect's seven laws.

**Rate.** Baseline committed, top PAYG. Savings plans when shape changes; reservations when sure — **no exchanges for new reservations on savings-plan-covered services from Feb 1, 2027** (one final exchange for existing; trade-in remains). AHB · Dev/Test · Spot · region · MACC.

**Usage.** Delete waste · non-prod off/ephemeral · right-size to p95 · autoscale to zero · fewer/cheaper environments · lifecycle · log volume (KQL `Usage` by DataType) · top ten lines first · continuous review.

**FinOps.** Value, not cheapness. Inform → optimize → operate. 2026: Executive Strategy Alignment; Scopes (cloud, SaaS, licensing, DC, AI). Allocation: subscriptions + enforced tags + shared-cost rules. Showback before chargeback. FOCUS 1.4 latest; Azure exports in FOCUS.

**Azure tools.** Prevent (Policy) · detect (budgets as code, anomaly alerts) · understand (exports, FinOps toolkit, amortized view) · correct (monthly review, Advisor) · estimate (calculator, Retail Prices API, Infracost incl. ARM/Bicep).

**Cost in design.** Doc: meters at 1× and 10×, unit cost, alternatives, people cost, sensitivity. PR: cost diff + threshold approver. Prod: cost fitness functions (unit cost, tags, non-prod off, log bytes/request, RU assertions in CI, tokens/task).

**AI cost.** (uncached × in) + (cached × cached) + (out × out) + retrieval + retries. Levers: cache prefixes · route to cheapest passing model · trim context · cap output · Batch ~50% off · PTU only above measured break-even · per-task step/token caps · anomaly alerts · margin by segment.

**Anti-patterns.** Log firehose · always-on non-prod · chatty cross-zone/region · replicate everything · Cosmos as RDBMS · polling · premium-by-default · idle provisioned · egress hairpins · unbounded retention · runaway loops · orphans · blind commitments · optimizing pennies.

**Build vs buy spectrum.** Build · assemble · adopt OSS · rent PaaS · license · SaaS · partner. Questions: core? product exists? lifetime TCO? time-to-value? risk/exit? capability? data/integration? Correct for engineers' over-building and demo-driven buying.

**Core vs context.** Moore · DDD core/supporting/generic · Wardley genesis → custom → product → commodity. Build the core; commoditized components are waste to build. "Would a customer pay if it were twice as good?"

**TCO both sides.** Build hides maintenance (often > build over a decade), ops, compliance, knowledge, opportunity. Buy hides subscription growth/tier jumps, integration, customization, workarounds, vendor mgmt, exit. Buy-first-to-learn behind a port.

**Lock-in.** Expected switching cost = cost × probability (Hohpe). Types: vendor, platform, data, protocol, skills, contractual, architectural, ecosystem. EU Data Act: switching charges gone Jan 12, 2027 — re-implementation isn't. Embrace leverage; cheap seams; negotiate exit.

**OSS.** Ownership, supply chain, licence, sustainability, churn. Relicensing wave: IdentityServer, FluentAssertions 8, MediatR 13/AutoMapper 15, MassTransit v9, QuestPDF v3 (<$1M), Redis → Valkey/AGPL. Pay · pin (review date) · replace (cheap if shallow) · fork (rarely). CPM, SBOM, licence/vuln gates, Scorecard, ports, ADRs.

**Vendor evaluation.** Prioritized reqs incl. NFRs · short list · weights before demos · PoC on hardest case · due diligence · pricing at 3×/10× · ADR with exit. Contract: data return, price caps, SLA + termination, change of control, deprecation notice, escrow.

**.NET defaults.** Entra ID / managed CIAM (Duende/OpenIddict when self-hosting justified; never homegrown OIDC) · managed brokers · plain handlers / source-gen mediator · Mapperly · adopt jobs/workflow/flags/search/PDF · OTel + chosen backend · SaaS for payments/tax/messaging delivery · build the core.

**AI & build-vs-buy.** Cuts cost to write, not to own. Build thin internal tools, prototypes, shallow-library replacements; keep buying depth, compliance, scale, ecosystem. Govern AI-built tools. Rent models/platform; build domain data, workflows, evals.

**Debt.** Cunningham: ship current understanding, repay by refactoring. Principal · interest · rate · default · refinancing (contain). Interest only where code changes. Not debt: bugs, missing features, taste, old-but-stable.

**Types.** Quadrant: deliberate/inadvertent × reckless/prudent. Code · design/architecture · test · delivery · platform/dependency · data · knowledge · ops · security · AI-generated · ML pipeline · social.

**Interest.** Slower change · defects · unpredictability · ops load · people cost. Code Red: 15× defects, +124% time, 9× max cycle time. McKinsey: 20–40% of estate value; 10–20% of new-product budget diverted. Money ≈ capacity in area × fraction caused + incidents + roadmap delay. Compounds.

**Find it.** Hotspots (complexity × churn) · static analysis for trends · architecture tests · dependency/runtime age · DORA per area · surveys (code metrics predict poorly) · register.

**Deliberate debt.** Launch window, learning, survival, sacrificial, regulatory date. Loan terms: shortcut + why, interest, repayment trigger, owner, containment. ADR + register; business co-signs.

**Paydown.** Priority ≈ interest/change × frequency × remaining life − principal. Fix-as-you-touch + 15–25% capacity + investment cases. Tie to value metric. Small on trunk (branch by abstraction, Mikado). Ratchet. Contain short-lived code. Upgrades are maintenance.

**.NET automation.** NetArchTest/ArchUnitNET rules + ratchets · LTS every two years (.NET 8 & 9 end Nov 10, 2026; .NET 10 to Nov 2028) · CPM · Dependabot/Renovate · vuln and licence gates · libyear.

**AI-era debt.** Amplifier (DORA 2025). Duplication up, refactoring down (GitClear). Duplication, inconsistency, comprehension, test illusion, volume, prompt debt. Conventions, explainability standard, prompts as code; AI makes principal cheaper — reinvest.

**Executives.** Translate to their units · BLUF (recommendation, why, risk, ask) · three numbers (cost, return + timing, cost of not doing) · ranges + checkpoint · options incl. do-nothing · tailor to CFO/CEO/CTO/product/risk · use their artefacts.

**Business case.** Decision · problem in numbers · options · cash flows/NPV/payback/sensitivity · non-financials · phased plan with go/kill criteria · measures · recommendation. Price status quo and coexistence tax (dated lines). Front-load value. Ask for phase 1. Be willing to say "contain, don't migrate".

**Portfolio.** Run/grow/transform split · protected maintenance share · make displacement explicit · "not like that — here's the cost and an alternative."

**Interview.** Cost at requirements, high-level (buy generic), trade-offs (price the nine), wrap-up (drivers + guardrails). Bill-doubled arc: normalize → delta → classify → quick wins → structural fix → prevent.

---
# Appendix A — One-page business case template

```markdown
# Business case: <initiative>
Owner: <name> · Sponsor: <exec> · Date: <date> · Status: Draft | Approved
Related: Design doc RFC-NNNN · ADRs NNNN · Debt register TD-NNNN

## 1. Decision requested
<Fund phase 1: N people for M months (≈ $X), starting <date>.>

## 2. Problem, in business terms (with evidence)
| Measure | Today | Source |
|---|---|---|
| Lead time for <area> changes | | DORA / Jira |
| Capacity lost to interest | | Hotspot + cycle-time analysis |
| Run cost / licences | | Cost Management / contracts |
| Incidents (12 months) | | Incident log |
| Risk exposure (expected annual loss, range) | | Risk register |
| Blocked roadmap items / cost of delay | | Product |

## 3. Options
| | A. Do nothing | B. <minimal> | C. <recommended> |
|---|---|---|---|
| One-off cost (range) | | | |
| Annual run-cost change | | | |
| Benefits (and when) | | | |
| Key risks | | | |
| What we can do afterwards (options created) | | | |

## 4. Financials (incremental vs A, $k)
| Year | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| Cash flow | | | | | |
NPV @ <finance rate> · Payback · (IRR) · Cash vs freed capacity: <explain>
Sensitivity: benefits −30% → NPV …; costs +50% → NPV …; delay +6 months → …
Coexistence tax (dated lines): <component — $/month — ends when>

## 5. Non-financial benefits and risks
Strategic options · compliance · talent · customer impact

## 6. Plan and checkpoints
| Phase | Delivers | Measure | Go criteria | Kill / re-plan criteria | What we keep if we stop |
|---|---|---|---|---|---|

## 7. Recommendation
<One paragraph, bottom line first.>
```

---
# Appendix B — Build-vs-buy scorecard

```markdown
# Build vs buy: <capability>
Classification: core | supporting | generic  ·  Wardley stage: genesis | custom | product | commodity

## Must-haves (pass/fail)
- [ ] <SSO with Entra ID / SAML, SCIM>   - [ ] <EU data residency>   - [ ] <full data export, open format>
- [ ] <SLA ≥ 99.9% with credits>          - [ ] <.NET SDK or clean REST API>   - [ ] <SOC 2 Type II / ISO 27001>

## Weighted criteria (agree weights BEFORE demos; score 1–5)
| Criterion | Weight | Build | Option 1 | Option 2 | Option 3 |
|---|---|---|---|---|---|
| Functional fit (hardest scenarios, from PoC) | 20 | | | | |
| 5-year TCO (from table below) | 20 | | | | |
| Time to value | 10 | | | | |
| Integration & API quality | 10 | | | | |
| Security & compliance | 10 | | | | |
| Pricing scalability at 3× / 10× | 10 | | | | |
| Vendor viability & roadmap | 5 | | | | |
| Exit cost & data portability | 10 | | | | |
| Team capability / desire to own | 5 | | | | |
| **Weighted total** | 100 | | | | |

## 5-year TCO ($k)
| Line | Build | Option 1 | Option 2 |
|---|---|---|---|
| Build / implementation | | | |
| Licences / subscription (with growth) | | | |
| Infrastructure | | | |
| Maintenance & enhancement (loaded) | | | |
| Operations / on-call | | | |
| Compliance & security work | | | |
| Exit cost | | | |
| **Total** | | | |

## PoC findings · Risks · Exit plan · Decision (link ADR)
```

---
# Appendix C — Technical debt register and review

```markdown
# Technical debt register — <system>
Review cadence: monthly (team) · quarterly (with product) · Owner: <tech lead>

| ID | Title | Type | Quadrant | Area (hotspot?) | Interest (est., range) | Principal (est.) | Trigger | Owner | Status |
|---|---|---|---|---|---|---|---|---|---|
| TD-0042 | Tenant config as JSON blob | design | prudent-deliberate | Platform (no) | 1–2 days per new setting | 2–3 eng-weeks | 10th setting or 2027-03-31 | team-platform | open |
| TD-0031 | Legacy PriceCalc usages | code | prudent-inadvertent | Pricing (YES) | ~30% of pricing ticket effort | 10 eng-weeks | Q2 roadmap | team-orders | in progress (ratchet: 9) |

## Quarterly review questions
1. Which items' interest went up (hotspot churn, incidents, cycle time)?
2. Which triggers fired? Who is repaying?
3. Which items are in code about to be replaced → contain, close?
4. Are ratchet baselines going down?
5. What new deliberate loans did we take, and did the business co-sign?
6. Is the protected maintenance capacity being used for the highest-interest items?
```

---
# Appendix D — Cost section for a design document

```markdown
## Cost
Prices checked: <date> · Source: Azure Pricing Calculator / Retail Prices API / price sheet export

### Cost model
| Component | Meter | Qty at launch | $/month at launch | Qty at 10× | $/month at 10× |
|---|---|---|---|---|---|
| Compute (Container Apps) | vCPU-s, GiB-s | | | | |
| Cosmos DB | RU/s (autoscale max) + storage | | | | |
| Log Analytics | GB ingested | | | | |
| Egress | GB | | | | |
| AI | tokens in/cached/out | | | | |
| **Total** | | | | | |

### Unit cost and elasticity
Cost per <order/tenant/conversation> at launch · at 10× · trend (sub-linear / linear / super-linear and why)

### People cost
Components to operate · on-call impact · upgrades/year · estimated FTE of ownership

### Alternatives considered (cost)
| Option | $/month at launch | at 10× | Why rejected |
|---|---|---|---|

### Sensitivity
What would double the cost? (traffic shape, payload size, retention, a tenant 10× larger)

### Guardrails
Budget + forecast alert · anomaly alerts · required tags · cost fitness functions · PR cost-diff threshold
```

---

*Next: **Module 34 — STAR, calibrated to seniority**: opening Phase 8, how senior answers describe what you delivered while staff and architect answers explain why it mattered and how it shaped the system — and how to tell the cost, build-vs-buy and debt stories from this module in that format.*

