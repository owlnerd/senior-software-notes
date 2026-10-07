# Module 30 — Writing and Defending a Design Document
*Phase 7: The Architect-Specific Track · Senior/Architect Interview Prep for .NET & C#*

> **State of practice verified on October 7, 2026.** Design-document practice moves more slowly than cloud platforms, but several facts below are recent, and a few are things interviewers now ask about directly:
>
> - **Public design-doc processes you can cite and read:** Google's design-doc culture (Malte Ubl's widely cited write-up), **Oxide's RFD 1** (RFDs stored in a Git repo, discussed in pull requests, with explicit states), **Kubernetes KEPs** (template with *Proposal*, *Risks and Mitigations*, *Alternatives*, test plan, graduation criteria and a production-readiness questionnaire), the **Rust RFC** template, **Python PEP 1**, the **Go proposal process**, and **HashiCorp's PRD + RFC pair** (product managers usually write the *Problem Requirements Document*, engineers follow with the RFC).
> - **.NET's own design process is public:** **dotnet/designs** holds design proposals for the runtime, framework and SDK (written from `meta/template.md`, reviewed as pull requests, merged on broad agreement); the **dotnet/runtime API review process** moves issues through `api-suggestion` → `api-ready-for-review` → `api-approved` or `api-needs-work`, reviewed by the **FXDC** (framework design core) board, with notes published in **dotnet/apireviews**; **dotnet/csharplang** publishes proposals and Language Design Meeting notes.
> - **Will Larson's *Crafting Engineering Strategy*** (O'Reilly) shipped in **October–November 2025** — the current reference for writing strategy documents that sit *above* individual design docs.
> - **arc42**: the English template is **version 9.0 (July 2025)**; the generated release bundle **2026.10.2** (October 2, 2026) ships it in 15 formats including Markdown, AsciiDoc and Word. Twelve sections, unchanged in shape.
> - **ISO/IEC/IEEE 42010:2022** (second edition, November 2022) is the current standard for architecture *description*: stakeholders, concerns, viewpoints and views. It renamed "system of interest" to **"entity of interest"** and "architecture framework" to **"architecture description framework"**. It deliberately says nothing about process, notation or format.
> - **Azure Well-Architected Framework** now includes **architect-role guidance**, including an *Architecture design specification*, *Architecture decision record* (append-only, recording the **confidence level** of each decision) and *Architecture design diagrams* pages, alongside the five pillars and the free **Azure Well-Architected Review** assessment. AWS's framework has six pillars.
> - **Microsoft's Code-With Engineering Playbook** (ISE) publishes design-review recipes and templates (game plan, milestone/epic, feature/story, task), a **trade study** template, a **decision log** with ADRs, and **async design reviews via pull requests**.
> - **AI changed the economics of writing, not of reviewing.** **GitHub Spec Kit** (MIT, launched September 2025; GitHub calls it an experiment) drives a *specify → plan → tasks → implement* loop for coding agents. **Cloudflare** described in 2026 an AI reviewer that checks specs and code against its internal RFC corpus (non-blocking findings from approved RFCs; blocking only for "enforced" MUST requirements). Practitioners now write about an **"RFC glut"**: producing a polished doc takes an hour, seriously reviewing one still takes half a day, so *review attention* — not authorship — is the binding constraint (Concept 27).
> - **Aspire** (formerly .NET Aspire) dropped ".NET" from its name with **Aspire 13** (November 2025) and requires the .NET 10 SDK — relevant here only because it is a cheap way to build the throwaway prototypes and spikes that produce evidence for a design doc.
>
> Practice and tooling facts are "verified on this date"; the principles in this module are stable.

## Orientation

Here is the sentence to carry through the whole module: **a design document is an argument, written before the expensive part, that a specific change is the best available answer to a well-stated problem under explicit constraints — and defending it means defending the reasoning, updating it when the evidence says so, and leaving behind a decision that other people can understand, trust and execute.**

Everything in the first 29 modules was *material* for that argument:

- **Module 3** gave you the 7-step framework; a design doc is that framework written down and made reviewable.
- **Modules 4 and 5** gave you requirements and estimation; the doc's credibility rests on them.
- **Modules 6–13** gave you the vocabulary of trade-offs — consistency, partitioning, caching, messaging, reliability — that fills the *alternatives* and *risks* sections.
- **Modules 14–25** gave you the .NET depth that lets you answer "how exactly would that work?" without hand-waving.
- **Modules 26–29** gave you compute, data, observability and security choices — each of which is a section a reviewer will probe.
- **Module 31** (next) gives you ADRs and C4 — the *durable* form of decisions and diagrams that a design doc produces.
- **Modules 32 and 33** give you brownfield migration and the language of cost — the two things that most often decide whether a design is approved.

Why it matters in an interview: the architect loop described in Module 2 almost always contains a **design-document round** in one of five shapes — write one as a take-home and defend it to a panel; present a design you shipped and say what you'd change; critique a document they hand you; produce a doc-structured design live; or role-play a stakeholder negotiation over a design. Senior IC loops test the same skill implicitly: a strong system-design answer *sounds like* a well-structured design doc read aloud. The questions sound like *"Walk me through a design doc you wrote that got rejected — what happened?"*, *"Here's our proposal; what are the three biggest problems?"*, *"Why didn't you choose Cosmos DB?"*, *"Your principal engineer thinks this is over-engineered — convince her or concede"*, *"How do you get a decision when two senior engineers disagree?"*, *"What would make you abandon this design?"* Weak answers describe boxes. Strong answers name **the problem, the constraints, the options, the decision, what was given up, how it would fail, how it would be rolled back, and what evidence would change the author's mind.**

This module has seven jobs:

1. **Build a first-principles model** of what a design document is for, when it's worth writing, who reads it, and why it is best understood as an argument.
2. **Teach the anatomy** — every section of a strong design doc, what goes in it, and what reviewers look for there.
3. **Teach writing that works** — process, clarity, diagrams, explicit trade-offs, quantification, length, tooling, and how AI changes all of it.
4. **Teach the review process** — models, preparation, meetings, giving feedback, structured evaluation methods and decision rights.
5. **Teach defense** — the mindset, anticipating objections, responding in the moment, changing your mind well, resolving disagreement and talking to each stakeholder.
6. **Make it .NET- and Azure-concrete** — the .NET team's own processes, Azure's architect guidance, and a full worked design doc for a .NET system on Azure.
7. **Prepare you for the interview round itself** — formats, presenting a past design, critiquing a doc live, and doing a take-home.

Seven framings to carry through:

1. **The document is a decision instrument, not documentation.** Its job is to produce a good decision cheaply; recording the decision is a by-product.
2. **Write to think.** Most design flaws are discovered by the author while writing the alternatives and failure-mode sections, before any reviewer reads a word.
3. **The alternatives section is the design.** A proposal with no serious alternatives is an announcement; reviewers can only trust a choice they can see being compared.
4. **Every claim should be checkable.** Numbers over adjectives, scenarios over "-ilities", evidence over confidence.
5. **Readers are busy and different.** Put the decision on page one, and let each audience — approver, implementer, operator, security, finance — find its section fast.
6. **Defend the reasoning, not the design.** Conceding a good objection precisely makes you more credible, not less; the goal is the best decision, not winning.
7. **A decision needs a decider and a deadline.** Review without decision rights becomes design by committee; review without a deadline becomes a document that never ships.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What a design document is | A pre-implementation proposal that makes a decision reviewable — distinct from PRDs, ADRs, specs and runbooks |
| 2 | Why write one | Thinking, cheap change, scaled review, consent, memory |
| 3 | When to write one — and when not to | Reversibility, blast radius, cross-team impact and novelty decide; tier the effort |
| 4 | Audiences, stakeholders and concerns | Each reader has concerns; the doc is organized so each finds theirs |
| 5 | The design doc as an argument | Claim, grounds, warrant, qualifier, rebuttal — the shape every strong doc shares |
| 6 | Header, metadata and status | Owner, approvers, status lifecycle, decision date — the doc's contract |
| 7 | The summary | Write it last; decision requested on page one |
| 8 | Context and problem statement | Why now, evidence, cost of doing nothing |
| 9 | Goals and non-goals | Measurable goals; non-goals are scope weapons |
| 10 | Requirements and quality attribute scenarios | Six-part scenarios with response measures; RFC 2119 words |
| 11 | Constraints and assumptions | Fixed constraints vs assumptions with a validation plan |
| 12 | Estimation and capacity | Show the arithmetic; let reviewers check it |
| 13 | The proposed design | Overview → components → contracts → data → flows → failure behavior |
| 14 | Alternatives considered | Real options including do-nothing and buy, compared on the same criteria |
| 15 | Cross-cutting concerns | Security, reliability, observability, performance, cost, operability, compliance, data |
| 16 | Rollout, migration and rollback | How it ships safely and how it un-ships |
| 17 | Risks and open questions | Owned, dated, with mitigations — the honest section |
| 18 | Plan, effort, cost and staffing | Milestones, dependencies, people and money |
| 19 | Appendices, glossary and FAQ | Depth without clutter; the FAQ pre-answers objections |
| 20 | The writing process | Outline, one-pager, socialize, expand, cut, summary last |
| 21 | Clarity | BLUF, pyramid principle, precise words, concrete numbers |
| 22 | Diagrams that earn their place | One purpose per diagram, legends, sequence diagrams, diagrams as code |
| 23 | Making trade-offs explicit | Trade-off tables, decision matrices and their traps, sensitivity points |
| 24 | Quantification and evidence | The evidence ladder from intuition to production data; confidence levels |
| 25 | Length, altitude and tiers | One-pager, standard doc, major proposal — and the six-page narrative |
| 26 | Docs-as-code and tooling | Markdown in Git with PR review vs collaborative editors; discoverability |
| 27 | AI and design documents in 2026 | Use AI to critique, not to substitute for judgment; the review-attention bottleneck |
| 28 | Review models | Async comments, meetings, boards, "Yes, if", advice process, PR-based RFCs, API boards |
| 29 | Preparing for review | Pre-wiring, choosing reviewers, asking for specific feedback, reading time |
| 30 | Running a design review | Roles, silent reading, risk-first agenda, timebox, recorded outcomes |
| 31 | Being a good reviewer | Prioritized, labeled, goal-anchored feedback; "Yes, if" |
| 32 | Structured evaluation methods | ATAM, ARID, risk storming, pre-mortems, WAF reviews, readiness reviews |
| 33 | Decision rights and closing | DACI/RAPID, consent vs consensus, disagree and commit, escalation |
| 34 | The defense mindset | Defend the reasoning; separate ego from design |
| 35 | Anticipating objections | Red-team yourself; the standard challenge list; the FAQ |
| 36 | Responding to challenges | Listen, restate, classify, answer with evidence, concede precisely |
| 37 | Changing your mind well | State what would change your mind; update visibly |
| 38 | Kinds of disagreement and how to resolve each | Facts, predictions, values, risk appetite — different tools for each |
| 39 | Defending to different stakeholders | Executives, implementers, operators, security, product, finance |
| 40 | The document's afterlife | Drift, as-built notes, supersession, retrospectives, ADR extraction |
| 41 | The design-document interview round | Formats and what each one scores |
| 42 | Presenting a past design | A ten-minute structure, anonymized, with what you'd change |
| 43 | Critiquing a design doc live | A systematic review pass and prioritized findings |
| 44 | The take-home design doc | Time budget, scope, structure and the traps graders see most |

---
# Part A — First principles: what a design document is for

## Concept 1 — What a design document is, and what it isn't

A **design document** is a written proposal, produced *before* the expensive part of a change, that describes a problem, the constraints on solving it, the options considered and the one recommended, in enough detail that knowledgeable people can evaluate the decision and others can implement it.

Three properties define it:

1. **It is pre-implementation.** It exists while change is still cheap. A document written after the code is an *as-built description* (valuable, but different).
2. **It is a proposal.** It asks for a decision — approval, rejection, or "yes, if" — from identified people.
3. **It is about the *why* and the *shape*, not every line.** It covers the choices that are expensive to reverse; it leaves code-level detail to code.

Organizations use many names for overlapping artifacts. Knowing the distinctions is itself a senior signal, because each answers a different question:

| Artifact | Question it answers | Typical author | Lifetime | Example |
|---|---|---|---|---|
| **PRD** (Product/Problem Requirements Document) | *What problem, for whom, and what must be true when it's solved?* | Product manager (with engineering) | Until the product changes | HashiCorp's PRD; Amazon's PR/FAQ ("working backwards") |
| **Design doc / RFC / RFD / proposal** | *How should we solve it, and why this way over the others?* | Engineer / architect | Living during design; frozen once decided | Google design docs, Oxide RFDs, Rust RFCs, Kubernetes KEPs, dotnet/designs |
| **ADR** (Architecture Decision Record) | *What did we decide, in what context, with what consequences?* | Whoever made the decision | Permanent, append-only | Michael Nygard's format (Module 31) |
| **Architecture description** (arc42, C4, 42010-style views) | *What is the system's structure today?* | Architect / team | Maintained with the system | arc42's twelve sections; C4 diagrams |
| **Technical spec / API spec** | *What exactly is the contract?* | Implementing team | Versioned with the interface | OpenAPI document, AsyncAPI, protobuf, .NET API proposal |
| **Strategy doc** | *What approach do we take across many decisions?* | Staff+/principal/leadership | Quarters to years | Will Larson's diagnosis → policy → operations format |
| **Implementation plan / task breakdown** | *What are the steps and who does them?* | Team lead | Weeks | Work items; Spec Kit's `tasks.md` |
| **Runbook / operational doc** | *How do I operate or recover it?* | Owning team / SRE | Maintained with the system | Incident playbooks |
| **Postmortem** | *What happened, why, and what changes?* | Incident owner | Permanent | Blameless incident review |

Relationships worth stating explicitly: a **PRD precedes** a design doc (problem before solution); a design doc **produces** one or more **ADRs** (the durable residue of its decisions) and **updates** the architecture description; several design docs over time **inform** a strategy (Larson's advice is to write several design docs, then synthesize the strategy from what they have in common).

Two things a design doc is *not*:

- **Not an implementation manual.** If the document is mostly code or step-by-step instructions, the code should have been written instead (the point Malte Ubl's Google write-up makes). Include code only where an interface or an algorithm *is* the decision.
- **Not a sales brochure.** A doc that lists only benefits, no costs, no alternatives and no risks is not reviewable — and experienced reviewers discount it immediately.

**The interview-grade sentence:** *"A design document is a pre-implementation proposal that makes a decision reviewable: it states the problem, the constraints, the options and the recommendation with its costs, so knowledgeable people can evaluate it while change is still cheap. It sits between a PRD, which defines the problem, and ADRs and architecture descriptions, which record the decisions and the resulting structure — and it is neither an implementation manual nor a pitch, because a document that lists only benefits can't be reviewed."*

---

## Concept 2 — Why write one: the five returns

Writing a design doc costs real time — typically days for a significant change. It pays back in five distinct ways, and naming them helps you decide how much to invest:

**1. It improves the author's thinking.** Writing forces sequence, precision and completeness that conversation lets you skip. Leslie Lamport puts it bluntly: *"If you're thinking without writing, you only think you're thinking."* The alternatives and failure-mode sections in particular surface flaws the author would otherwise discover in production. In practice, the author is the doc's first and most important reviewer.

**2. It moves change to the cheapest moment.** Changing a box on a diagram costs minutes; changing a deployed data model costs weeks; changing a public API costs a deprecation cycle measured in years. The exact multipliers in the classic cost-of-change curves are disputed, but the direction isn't. Microsoft's ISE playbook frames this as the first measure of design reviews: switching from App Service to AKS at design time is a document edit; after implementation it's a project.

**3. It scales review beyond the room.** A meeting reaches the people in it, at one time, at the speed of the slowest explanation. A document reaches anyone, asynchronously, across time zones, and lets each reviewer spend their attention on their area. Uber's early practice of sending planning documents to every engineer, then to domain mailing lists as it grew, was explicitly about keeping a consistent engineering bar across a fast-growing organization.

**4. It builds consent and accountability.** People support decisions they had a chance to shape. A written proposal with named approvers turns "I didn't know about that" into "I reviewed that." It also makes the author accountable for specific claims — "p99 under 200 ms at 5,000 requests per second" can be checked later; "it will be fast" cannot.

**5. It becomes organizational memory.** Two years later, someone asks "why on earth is this on Service Bus and not Event Hubs?" A design doc (and the ADR it produced) answers that, including the constraints that held *then* — so the new team can tell whether the decision still applies or the context has changed. Without it, teams either preserve bad decisions out of fear or reverse good ones out of ignorance (Chesterton's fence).

There's also a sixth, personal return: **design docs are the visible artifact of senior engineering.** At staff and principal levels, much of the impact is in decisions made and steered, not lines of code; documents are how that impact is seen, evaluated and remembered. This is why architect interviews ask for them.

What a design doc does *not* buy: certainty. A reviewed design can still be wrong; the doc's value is that it was wrong *for stated reasons*, which makes correction faster.

**The interview-grade sentence:** *"I write design docs for five returns: writing forces me to find flaws before anyone else has to; it moves change to the moment it costs a document edit rather than a migration; it scales review asynchronously across teams and time zones; it builds consent and makes my claims checkable; and it leaves memory of the constraints behind a decision so later teams can tell whether it still holds. I size the investment to which of those returns the change actually needs."*

---

## Concept 3 — When to write one, and when not to

Not every change deserves a design doc. Teams that require one for everything create ceremony that engineers route around; teams that never write them relearn the same lessons in production. The deciding factors:

| Factor | Write a doc when… | Skip or shrink it when… |
|---|---|---|
| **Reversibility** | The decision is a *one-way door*: data models, public APIs, storage engines, tenancy models, identity, protocols, vendor lock-in | It's a *two-way door*: easy to undo with a flag, a revert or a config change |
| **Blast radius** | Failure would affect many users, tenants, teams or revenue | Failure stays inside one component and is quick to fix |
| **Cross-team impact** | Other teams must change, depend on it or operate it | One team owns everything touched |
| **Novelty** | New technology, pattern or domain for the organization | Repeating a well-trodden pattern with an existing template |
| **Cost and duration** | More than roughly a few engineer-weeks, or significant cloud spend | A day or two of work |
| **Controversy** | Reasonable people already disagree | Everyone agrees and the risk is low |
| **Compliance** | Touches regulated data, security boundaries or audit requirements | No regulated surface |

The **one-way/two-way door** framing comes from Jeff Bezos's 2015 shareholder letter (Type 1 and Type 2 decisions): Type 1 decisions deserve slow, careful deliberation; Type 2 decisions should be made quickly by small groups, and the failure mode of large organizations is applying Type 1 process to Type 2 decisions. That is precisely the calibration a design-doc process needs.

**Tier the effort** rather than choosing between "full doc" and "nothing":

| Tier | Size | When | Review |
|---|---|---|---|
| **0 — No doc** | A PR description | Two-way doors, local changes | Code review |
| **1 — One-pager / mini design doc** | 1–2 pages | Moderate changes, one team, some risk | Async, one or two reviewers |
| **2 — Standard design doc** | ~4–10 pages plus appendices | Significant features, new services, cross-team | Async review plus a review meeting |
| **3 — Major proposal** | 10–20+ pages, possibly several linked docs | One-way doors with large blast radius: platform choices, data migrations, tenancy, identity | Formal review (architecture review board, ATAM-style session), explicit decider |

Google's practice, as Ubl describes it, is similar: roughly 10–20 pages for large projects and short "mini design docs" of 1–3 pages for incremental changes.

**When the doc isn't the right tool yet:**

- **When you don't know enough to choose.** Run a **spike** first — a timeboxed experiment to answer a specific question ("can Cosmos DB's change feed sustain our write rate within our RU budget?"). Then write the doc with evidence. The ISE playbook's *engineering feasibility spike* recipe exists for exactly this.
- **When the problem isn't agreed.** If stakeholders disagree about *what* to solve, write a PRD or a problem statement first; designing against a contested problem wastes everyone's review time.
- **When the decision is already made.** If leadership has decided, don't write a fake "proposal" with strawman alternatives. Write an honest *decision memo* or ADR that records the decision and its rationale, and design the *how*.

**The interview-grade sentence:** *"I decide whether a change needs a design doc by reversibility, blast radius, cross-team impact, novelty, cost and controversy — one-way doors like data models, public APIs and tenancy get a full doc, two-way doors get a PR description — and I tier the effort from a one-pager to a major proposal rather than applying heavyweight process to everything, which is the failure mode Bezos's Type 1 versus Type 2 distinction warns about. If we don't know enough to choose, I run a spike first; if we don't agree on the problem, I write the problem statement first."*

---

## Concept 4 — Audiences, stakeholders and concerns

A design doc has several readers who want different things from it. ISO/IEC/IEEE 42010 gives the precise vocabulary: **stakeholders** have **concerns** (matters of interest about the system); a description addresses those concerns through **views**, each governed by a **viewpoint** that says what it shows and for whom. You don't need the formalism in a design doc, but you do need its central insight: *the document succeeds when each stakeholder can find and evaluate their concerns.*

| Reader | Primary concerns | Sections they read first | What makes them approve |
|---|---|---|---|
| **Approver / decider** (eng director, principal, architecture board) | Is this the right problem? Right solution? Acceptable risk and cost? | Summary, goals/non-goals, alternatives, risks, cost | Clear decision requested; honest trade-offs; bounded risk |
| **Senior technical reviewers** | Correctness, scalability, consistency, failure modes, simplicity | Design, alternatives, cross-cutting, estimation | Evidence; failure behavior designed, not hoped for |
| **Implementing engineers** | Can we build it? Is it clear? Is the scope realistic? | Design details, contracts, plan, open questions | Unambiguous contracts; realistic milestones; no hidden work |
| **Operators / SRE / on-call** | Can we run it at 3 a.m.? What pages? How do we roll back? | Reliability, observability, rollout/rollback, runbooks | SLOs, alerts, dashboards, rollback plan, failure-mode table |
| **Security / privacy** | Trust boundaries, data handling, identity, compliance | Security section, threat model (Module 29), data flows | Threat model with mitigations; least privilege; data classification |
| **Product / business** | Does it solve the user problem? When? What's cut? | Summary, goals, non-goals, plan | Scope matches the PRD; trade-offs in user terms |
| **Finance / leadership** | What does it cost to build and to run? | Cost section, plan | Build and run cost estimate; cost drivers; scaling curve (Module 33) |
| **Dependent teams** | What changes for us? When? | Interfaces, migration, plan | Contract changes, timelines, support model |
| **Future maintainers** | Why is it like this? | Context, alternatives, decisions | Rejected options with reasons; stated assumptions |

Practical consequences for structure:

1. **Layered reading.** The summary must stand alone for the approver who reads only that; the design section must stand alone for the implementer; the reliability section must stand alone for SRE. Repeat key numbers where needed rather than forcing cross-references.
2. **Name the audience in the header.** "Reviewers: Payments team (contract), SRE (operability), Security (data classification)." It tells people what they're being asked to look at.
3. **Ask each reviewer for something specific.** "@security: please check the token flow in §6.3" gets better review than "please take a look."
4. **Write for the least-context reader you need.** If a dependent team must understand the change, don't assume your team's jargon — or include a short glossary.

A subtle point interviewers like: **the most important stakeholder may not be in the room** — the on-call engineer two years from now, or the customer whose data is being migrated. Senior authors write sections explicitly for them.

**The interview-grade sentence:** *"I write a design doc for its stakeholders' concerns, in the ISO 42010 sense: the approver needs the decision, alternatives, risk and cost; reviewers need evidence and failure behavior; implementers need unambiguous contracts and a realistic plan; operators need SLOs, alerts and rollback; security needs the threat model; finance needs build and run cost; and future maintainers need the rejected options and assumptions. So the summary stands alone, each section stands alone for its audience, and I ask each reviewer for a specific review rather than 'please take a look.'"*

---

## Concept 5 — The design document as an argument

The single most useful mental model for both writing *and* defending: **a design doc is an argument.** The philosopher Stephen Toulmin's model of practical argument (1958) maps onto it almost exactly, and once you see it, weak docs become easy to diagnose:

| Toulmin element | Meaning | In a design doc | Missing it looks like… |
|---|---|---|---|
| **Claim** | The conclusion you want accepted | "We should move notifications to an event-driven service on Service Bus." | A doc that describes a system but never says what decision it wants |
| **Grounds (data)** | The facts supporting the claim | Current failure rates, latency data, incident history, load estimates, spike results | Assertions without evidence: "the current approach doesn't scale" |
| **Warrant** | Why the grounds support the claim | "Because notification failures currently roll back order transactions, decoupling them removes the main cause of checkout errors." | Data and conclusion side by side with no causal link |
| **Backing** | Support for the warrant | Known patterns (outbox — Module 11), vendor docs, prior incidents, benchmarks | "Best practice says…" |
| **Qualifier** | How strongly the claim holds | "For our expected scale (≤ 2,000 msg/s) and our latency tolerance (seconds)" | Universal claims: "this always scales" |
| **Rebuttal** | Conditions under which the claim fails | "If we need sub-second delivery or strict cross-channel ordering, this design is wrong." | No alternatives, no risks, no "what would change my mind" |

Why this matters:

1. **It tells you what to write.** Each section of the anatomy (Part B) supplies one of these: context and requirements are *grounds*; the design is the *claim*; the rationale is the *warrant*; references and spikes are *backing*; scope and assumptions are *qualifiers*; alternatives and risks are *rebuttals*.
2. **It tells you how a review will attack.** Every challenge in a review targets one element: "Is that data right?" (grounds), "Why does that follow?" (warrant), "Does it still hold at 10×?" (qualifier), "What about option X?" (rebuttal). Classifying the challenge tells you how to answer (Concept 36).
3. **It tells you why alternatives matter so much.** An argument that never considers rebuttals isn't persuasive to a skeptical audience — and senior reviewers are a skeptical audience by job description.
4. **It reframes "defense."** You aren't defending a design; you're defending an argument. If a piece of grounds turns out wrong, you revise the argument; sometimes the claim survives, sometimes it doesn't. Either outcome is success if the decision improves.

A design doc also has **two kinds of claims**, and confusing them causes a lot of pointless arguments:

- **Descriptive claims** — what is true or will be true: "the database handles 3,000 writes/s," "this will take two sprints." These are settled by evidence.
- **Normative claims** — what we *should* value: "availability matters more than consistency for this feature," "we prefer managed services over self-hosting." These are settled by goals, principles or a decider.

Strong docs make the normative choices explicit ("we prioritize X over Y because of goal G") so reviewers argue about them openly instead of disguising value disagreements as technical ones (Concept 38).

**The interview-grade sentence:** *"I treat a design doc as an argument in Toulmin's sense: the recommendation is the claim; requirements, measurements and spikes are the grounds; the rationale explaining why those facts lead to this design is the warrant, backed by known patterns and prior evidence; scope and assumptions are the qualifiers; and the alternatives and risks are the rebuttals. That tells me what each section must supply and what each review challenge is attacking — and it lets me separate descriptive claims, which evidence settles, from value choices, which goals or a decider settle."*

---
# Part B — The anatomy of a design document

Templates differ by company, but strong ones converge on the same skeleton. Here's how the public templates line up — useful both for choosing your own and for recognizing a company's in an interview:

| Purpose | Google (Ubl) | Rust RFC | Kubernetes KEP | HashiCorp RFC | dotnet/designs | This module |
|---|---|---|---|---|---|---|
| What & why, briefly | Context and scope | Summary; Motivation | Summary; Motivation | Background / overview | Summary; Scenarios and user experience | §6–8 |
| Scope | Goals and non-goals | — | Goals; Non-Goals | Requirements (via PRD) | Requirements; Non-goals | §9–11 |
| The design | The actual design | Guide-level and reference-level explanation | Proposal; Design Details | Proposal; Implementation | Design | §12–13 |
| Options | Alternatives considered | Rationale and alternatives; Prior art | Alternatives | Abandoned ideas | Q&A | §14 |
| Concerns | Cross-cutting concerns | Drawbacks | Risks and Mitigations; PRR questionnaire | — | — | §15, §17 |
| Shipping | — | — | Test plan; Graduation criteria; Upgrade/downgrade; Version skew | — | — | §16, §18 |
| Unknowns | — | Unresolved questions; Future possibilities | — | — | Q&A | §17 |

The sections below follow the right-hand column. Not every doc needs every section; a Tier 1 one-pager (Concept 25) collapses them to a few paragraphs. A full template is in **Appendix A** of this module.

## Concept 6 — Header, metadata and the status lifecycle

The top of the document is a small contract: who owns it, who must approve it, where it stands and when a decision is due. It sounds administrative; it is what prevents the two most common process failures — *nobody knows who decides* and *the doc sits in review forever*.

```markdown
# RFC-0142: Event-driven notification service for Orders

| Field | Value |
|---|---|
| Status | In review (since 2026-10-07) — decision due 2026-10-21 |
| Author / owner | Ivan D. (Orders platform) |
| Approver (decider) | Principal engineer, Commerce |
| Required reviewers | Payments (contract), SRE (operability), Security (data), FinOps (run cost) |
| Optional reviewers | Customer comms team |
| Tier | 2 (cross-team, new service, reversible within one quarter) |
| Related | PRD-88 Notification reliability · ADR-031 Outbox for integration events · Incident 2026-07-14 |
| Last updated | 2026-10-07 — v0.4 (see changelog) |
```

**A status lifecycle** — every serious process has one. Representative examples:

| Process | States |
|---|---|
| **Oxide RFD** | prediscussion → ideation → discussion → published → committed (or abandoned) |
| **Kubernetes KEP** | provisional → implementable → implemented (or deferred, rejected, withdrawn, replaced) |
| **Python PEP** | Draft → Accepted → Final (or Rejected, Withdrawn, Deferred, Superseded) |
| **Rust RFC** | open PR → final comment period with a disposition (merge, close or postpone) → merged/closed |
| **dotnet/runtime API review** | api-suggestion → api-ready-for-review → api-approved (or api-needs-work) |
| **A pragmatic internal default** | Draft → In review (decision due) → Approved / Approved with conditions / Rejected / Withdrawn → Implemented → Superseded |

Rules that make the header work:

1. **Exactly one approver (or one named deciding body).** "The team" is not a decider. If several parties must agree, list them as *required reviewers whose concerns must be resolved*, and still name who makes the final call (Concept 33).
2. **A decision date.** Reviews expand to fill the time available. A date creates a forcing function and lets reviewers prioritize.
3. **A visible status.** Readers landing on the doc must instantly know whether it's a draft to shape, a proposal to evaluate or a record of a decision. Oxide puts the state in the document's metadata; dotnet/designs uses folders and PR state.
4. **Links to the problem source** (PRD, incident, OKR) and **to the decision records** it produces.
5. **A changelog** once review starts, so reviewers can see what changed since they last read (Concept 37).

**The interview-grade sentence:** *"Every design doc I write opens with a small contract: owner, one named approver, required reviewers with the concern each one owns, a tier, links to the problem source and resulting ADRs, a visible status from a defined lifecycle — like Oxide's RFD states or Kubernetes' provisional, implementable, implemented — and a decision date, because the two common failures are nobody knowing who decides and reviews that never end."*

---

## Concept 7 — The summary: write it last, put it first

The summary (TL;DR, executive summary, overview) is the most-read and least-revised part of most docs. It should let a busy approver understand **the problem, the recommendation, the main trade-off and the decision being requested** in under a minute.

A strong summary has four sentences — sometimes five:

1. **Problem, with a number:** "Order confirmation emails are sent synchronously inside the checkout transaction; SMTP timeouts caused 0.8% of checkouts to fail last quarter (≈ 11,000 orders)."
2. **Recommendation:** "We propose moving notifications to a separate service that consumes order events from Azure Service Bus via the transactional outbox."
3. **Main trade-off:** "This removes notification failures from checkout entirely, at the cost of notifications becoming eventually consistent (typically under 5 seconds, bounded by an alerting SLO) and one more service to operate."
4. **Cost and plan:** "Estimated 6 engineer-weeks; ≈ €300/month run cost; rollout behind a flag over three weeks with instant rollback."
5. **Decision requested:** "Approval by 2026-10-21 of the event-driven approach (§6) and the outbox choice over direct publishing (§7.2)."

Principles:

- **Write it last.** HashiCorp's templates say this explicitly: the summary is read first but authored last, after you know what the document concludes. Drafts often change their conclusion during writing; summaries written first describe a design that no longer exists.
- **Bottom line up front (BLUF).** Lead with the conclusion, not the history. Readers who want the history will read §3.
- **No new information.** Everything in the summary is supported later in the doc.
- **State the trade-off, not just the benefit.** A summary with only upside reads as advocacy and triggers skeptical review.
- **Make the ask explicit.** What exactly must reviewers approve, and by when?

**The interview-grade sentence:** *"I write the summary last and put it first: one sentence on the problem with a number, one on the recommendation, one on the main trade-off, one on cost and plan, and the explicit decision I'm asking for and by when — bottom line up front, no new information, and always the cost alongside the benefit, because a summary with only upside reads as advocacy and invites the skeptical review I'm trying to earn my way past."*

---

## Concept 8 — Context and problem statement

This section establishes **why anything should change** — and most rejected design docs fail here, not in the design. A reviewer who isn't convinced the problem matters will critique the solution's cost forever.

Contents:

1. **Current state.** A short description (and usually a diagram) of how things work today, at the altitude relevant to the change. Reviewers from other teams need this; your team will skim it.
2. **The problem, with evidence.** Incidents, metrics, support tickets, customer complaints, cost reports, developer-productivity data, audit findings. *"Checkout error rate attributable to notifications: 0.8% (Application Insights, Q3 2026); 3 Sev-2 incidents in 90 days; median time to recover 47 minutes."*
3. **Why now.** What changed: growth, a deadline, a regulation, a new product, an end-of-support date, an incident. "Why now" distinguishes urgent problems from perennial complaints.
4. **The cost of doing nothing.** Quantify the status quo's trajectory: "At projected Q2 volume, notification-induced checkout failures reach ≈ 25,000 orders per quarter." Doing nothing is always an option (Concept 14), so its cost is the baseline every alternative is measured against.
5. **Who is affected** — users, tenants, teams, operators.
6. **Prior work** — earlier attempts, related docs, what other teams or companies have done (Rust RFCs have a "Prior art" section for exactly this).

Common failures:

- **Solution-shaped problem statements.** "The problem is that we don't use Kafka." That's a solution. The problem is the consequence you'd measure: lost events, latency, cost.
- **Anecdote without data.** "Customers complain about slow emails" → how many, how slow, compared to what?
- **Problems nobody owns.** If the business doesn't care about the problem, no design will be prioritized. Link the PRD or the goal it serves.

**The interview-grade sentence:** *"Most design docs are rejected in the problem statement, not the design, so I establish the current state, the problem with evidence — incidents, metrics, tickets, cost — why it matters now, and a quantified cost of doing nothing, which becomes the baseline every option is compared against. And I keep the problem free of solutions: 'we don't use Kafka' is a solution; 'we lose 0.3% of events during failover' is a problem."*

---

## Concept 9 — Goals and non-goals

**Goals** state what must be true when the work is done. **Non-goals** state what this work deliberately will *not* address, even though a reader might reasonably expect it to.

**Good goals are outcome-based and measurable:**

| Weak goal | Strong goal |
|---|---|
| "Make notifications reliable" | "Zero checkout failures caused by notification delivery" |
| "Improve performance" | "Checkout p99 latency drops from 1.9 s to under 800 ms at 1,200 checkouts/min" |
| "Scalable design" | "Handles 5× current peak (10,000 notifications/min) without architectural change" |
| "Better observability" | "Every notification traceable end-to-end by order ID within 1 minute of the event" |

**Non-goals are scope weapons.** They prevent the three things that sink reviews:

1. **Scope creep during review.** "While you're there, could it also support SMS?" — *Non-goal: new channels; the design leaves a seam for them (§6.4).*
2. **Misplaced criticism.** A reviewer critiquing the design for not handling multi-region failover is answered by *Non-goal: multi-region active-active; tracked separately in RFC-0139.*
3. **Ambiguous success.** Implementers know what not to gold-plate.

Non-goals are not things nobody would expect ("not building a spaceship"); they're things a reasonable reader *might* assume are in scope. Some teams distinguish **non-goals** (not now) from **anti-goals** (things the design must actively avoid, e.g., "must not introduce a new datastore the team doesn't operate").

**Prioritize goals.** When two goals conflict — and in real designs they do — the doc should already say which wins. "Goal 1 (checkout never fails because of notifications) takes priority over Goal 3 (notifications within 5 seconds)." This turns a future argument into a lookup (Concept 38).

**The interview-grade sentence:** *"I write goals as measurable outcomes — 'zero checkout failures caused by notification delivery', 'p99 under 800 ms at 1,200 checkouts a minute' — prioritized so that conflicts are already settled, and non-goals as scope weapons naming the things a reasonable reader might assume are included but aren't, like new channels or multi-region active-active, because that stops scope creep and misplaced criticism during review and tells implementers what not to gold-plate."*

---

## Concept 10 — Requirements and quality attribute scenarios

Functional requirements say what the system does; **quality attribute requirements** (non-functional requirements, Module 4) say how well — and they drive architecture far more than features do. The trouble with "-ilities" is that they're untestable as written. "The system shall be highly available" can't be designed for or verified.

The SEI's **quality attribute scenario** (Bass, Clements and Kazman, *Software Architecture in Practice*) makes them concrete with six parts:

| Part | Meaning | Example (availability) | Example (modifiability) |
|---|---|---|---|
| **Source** | Who or what generates the stimulus | The SMTP provider | A product manager |
| **Stimulus** | The event | Becomes unreachable for 30 minutes | Requests a new notification channel (push) |
| **Artifact** | What is affected | Notification sender | Notification service |
| **Environment** | Conditions at the time | Normal operation, peak hour | During normal development |
| **Response** | What the system does | Checkout continues unaffected; notifications queue and are retried; operators are alerted | Developers add a channel adapter without changing order or checkout code |
| **Response measure** | How we know it's good enough | 0 checkout failures; 100% of queued notifications delivered within 10 minutes of recovery; alert within 5 minutes | Done in ≤ 5 engineer-days, touching only the notification service |

Scenarios are how you make a reviewer's question — "what happens when the provider is down?" — part of the requirements rather than an attack on the design. Two or three scenarios per important quality attribute are usually enough; prioritize them (ATAM's utility tree in Concept 32 does this formally by rating importance and difficulty).

**Precise requirement language.** Many teams borrow **RFC 2119** keywords (clarified by RFC 8174 to apply only when capitalized): **MUST / MUST NOT** (absolute), **SHOULD / SHOULD NOT** (strong recommendation; deviation needs justification), **MAY** (optional). In a design doc they turn vague intentions into reviewable commitments — "notifications MUST NOT be sent for orders that fail payment; they SHOULD be delivered within 5 seconds at p95." (Cloudflare's 2026 AI reviewer enforces exactly these MUST statements from its RFCs — a sign of how seriously some organizations take the wording.)

**Link requirements to their origin** — the PRD, the SLO, the regulation, the contract — so reviewers know which are negotiable. A requirement from a regulation and a requirement from someone's preference deserve different treatment.

**The interview-grade sentence:** *"I turn quality attributes into six-part scenarios — source, stimulus, artifact, environment, response and response measure — so 'highly available' becomes 'when the SMTP provider is down for 30 minutes at peak, checkout is unaffected, every queued notification is delivered within 10 minutes of recovery and on-call is alerted within 5', which is both designable and testable. I write commitments with RFC 2119 MUST and SHOULD, prioritize the scenarios, and link each requirement to its source so reviewers know which ones are negotiable."*

---

## Concept 11 — Constraints and assumptions

Two kinds of givens, often confused:

**Constraints** are fixed — the design must work within them, and the doc should say where each comes from:

| Kind | Examples |
|---|---|
| **Technical** | Must run on Azure; must integrate with the existing SQL Server 2022 orders database; .NET 10 LTS; existing Entra ID tenant |
| **Organizational** | Owned by a team of four; no dedicated SRE; must fit the platform team's supported service list |
| **Regulatory / contractual** | EU data residency; PCI DSS scope must not expand; customer contract requires 99.9% monthly availability |
| **Time** | Must ship before the November peak; code freeze on 2026-11-15 |
| **Budget** | Run cost under €1,000/month; no new licenses |
| **Compatibility** | Existing email templates and the public webhook contract must not change |

**Assumptions** are things you believe but haven't proven — and each is a risk if wrong. State them, and say **how and when each will be validated**:

| Assumption | Why it matters | Validation | Owner / date |
|---|---|---|---|
| Peak volume stays ≤ 2,000 notifications/s through 2027 | Sizes Service Bus tier and partitioning | Compare with the 2026 growth forecast; load test at 3× | Ivan / 2026-10-15 |
| The email provider's API supports idempotency keys | Enables exactly-once-effect delivery | Read API docs; test in sandbox | Ana / 2026-10-12 |
| SRE will accept on-call for one more service | Operability model | Confirm with SRE lead | Ivan / 2026-10-10 |

Why this section matters disproportionately:

- **Constraints legitimize choices.** Many "why didn't you use X?" questions are answered by a constraint: "PCI scope must not expand" kills a whole class of options. If the constraint isn't written down, the reviewer reasonably assumes you didn't consider X.
- **Assumptions are where designs die.** Postmortems often trace back to an unstated assumption ("we assumed the provider retried"). Writing them down invites the reviewer who *knows* the assumption is false.
- **Challenging constraints is a senior move.** Sometimes the best design requires relaxing a constraint ("if we could move the deadline two weeks, option B removes a whole class of risk"). Say so explicitly; let the decider choose.

**The interview-grade sentence:** *"I separate constraints — fixed technical, organizational, regulatory, time, budget and compatibility limits, each with its source — from assumptions, which are beliefs that become risks if wrong, each with an owner, a validation method and a date. Constraints answer many 'why not X?' questions before they're asked, written assumptions invite the reviewer who knows one is false, and when a constraint blocks a clearly better design I say so and let the decider choose whether to relax it."*

---

## Concept 12 — Estimation and capacity

Module 5's back-of-envelope math belongs in the doc, **visibly**. Showing the arithmetic does three things: it lets reviewers check it (they will), it exposes which inputs matter most, and it signals that the design is sized rather than guessed.

```text
Volume
  Orders/day (peak season):            250,000   (2025 peak 180k × 1.4 growth)
  Notifications per order:             3         (confirmation, shipped, delivered)
  → notifications/day:                 750,000
  Peak hour share:                     12%  → 90,000/hour → 25/s average in peak hour
  Burst factor (flash sales):          ×20  → 500/s design point; 2,000/s ceiling (×4 headroom)

Message size and broker tier
  Event payload: ~1.5 KB  → 500/s × 1.5 KB ≈ 0.75 MB/s at design point; 3 MB/s at ceiling
  Design point: Service Bus Standard is plausible, but it is shared capacity with throttling
  and no throughput guarantee. Ceiling (or if predictable latency matters): Premium with
  1 messaging unit. Tier is the sensitive decision → §7.3; verify current limits and prices.

Delivery
  Provider latency p95: 400 ms; 1 concurrent call per in-flight message
  Concurrency needed at 500/s: 500 × 0.4 s ≈ 200 concurrent sends (Little's Law, Module 6)
  → 4 consumer replicas × MaxConcurrentCalls = 64 (Service Bus processor) → 256 slots

Storage
  Outbox rows: 750k/day × 2 KB = 1.5 GB/day; retained 7 days → ~10 GB, purged by job
```

Rules for estimation in a doc:

1. **Cite every input's source** (telemetry, forecast, product assumption) and mark guesses as guesses.
2. **Design for a stated headroom** (e.g., 4× peak), and say what breaks first beyond it.
3. **Use the numbers in the design.** If the estimate says 200 concurrent sends, the design should show where that concurrency lives (replicas × concurrency setting) — reviewers notice when the math and the design never meet.
4. **Identify the sensitive inputs.** "If sustained load passes ~500/s or bursts reach ×50, we move from Standard to Premium (1–2 messaging units) and run cost roughly triples; nothing else in the design changes." That's a sensitivity statement (Concept 23) and it pre-empts the "what if it's bigger?" question.
5. **Cost follows from capacity** (Concept 18, Module 33) — compute run cost from the same numbers.
6. **Verify vendor limits** at the time of writing; service limits change, and the doc should say "as documented on <date>."

**The interview-grade sentence:** *"I put the back-of-envelope estimate in the doc with the arithmetic visible — volumes, peaks, burst factors, payload sizes, Little's-Law concurrency, storage growth — each input with its source and guesses marked as guesses, a stated headroom with what breaks first beyond it, and the numbers actually carried into the design and the cost estimate. Then I name the sensitive inputs, so the inevitable 'what if it's ten times bigger?' is already answered."*

---

## Concept 13 — The proposed design

The heart of the doc — but in strong docs it's often *shorter* than people expect, because the surrounding sections carry the reasoning. Present it top-down, so each reader can stop at their altitude:

1. **Overview** — one paragraph and one diagram showing the main components and the end-to-end flow (a C4 container-level diagram is usually the right altitude — Module 31).
2. **Components and responsibilities** — each new or changed component, what it owns, where it runs (compute choice, Module 26), who owns it.
3. **Interfaces and contracts** — APIs (OpenAPI excerpt), message schemas (with versioning rules), events, configuration. Contracts are the most expensive things to change, so they deserve precision.
4. **Data model** — entities, ownership, storage choice, partition keys, indexes, retention (Modules 8, 12, 27).
5. **Key flows** — sequence diagrams for the happy path *and* the important failure paths.
6. **State, consistency and concurrency** — what's strongly consistent, what's eventual, how idempotency and ordering are achieved (Modules 7, 11).
7. **Failure behavior** — what happens when each dependency fails (a failure-mode table; Concept 15).
8. **Technology choices** — with a sentence of rationale each, linking to alternatives where they were contested.

Altitude discipline:

- **Include** anything expensive to reverse, anything another team depends on, anything a reviewer must evaluate: contracts, data models, consistency semantics, failure behavior, security boundaries.
- **Exclude** class hierarchies, method names, framework wiring, anything that can change in a code review without anyone outside the team noticing.
- **Use code only where code *is* the decision** — an interface signature, a message schema, an idempotency algorithm, a SQL index definition.

A .NET-flavored example of "code as the decision" — the contract and the idempotency rule matter; the implementation doesn't:

```csharp
// Integration event published via the outbox (contract: versioned, additive-only changes).
public sealed record OrderConfirmedV1(
    Guid EventId,            // idempotency key for consumers
    Guid OrderId,
    string TenantId,
    string CustomerEmail,    // classified PII — see §8.1
    decimal Total,
    string Currency,
    DateTimeOffset OccurredAt);

// Rule (MUST): the notification service records EventId in a processed-events table
// in the same transaction as the "notification requested" row; duplicates are acknowledged
// and dropped. Delivery to the provider uses EventId as the provider's idempotency key.
```

**The interview-grade sentence:** *"I present the design top-down so each reader stops at their altitude: an overview diagram at container level, components with owners and where they run, precise contracts and schemas with versioning rules, the data model with partitioning and retention, sequence diagrams for the happy path and the important failure paths, consistency and idempotency semantics, failure behavior per dependency, and technology choices with a sentence of rationale each. I include what is expensive to reverse or that others depend on, leave class-level detail to code review, and use code only where code is the decision, like a contract or an idempotency rule."*

---

## Concept 14 — Alternatives considered

This is the section that separates senior design docs from everything else. **A choice is only as credible as the comparison behind it.** A proposal without serious alternatives tells reviewers either that the author didn't look, or that they're hiding the comparison — and either way the reviewers will now do the comparison themselves, in the review, at length.

What to include:

1. **Do nothing (status quo)** — with the cost from Concept 8. Sometimes it wins; saying so honestly is credibility.
2. **The minimal change** — the cheapest thing that addresses most of the problem ("move the email call after commit and add a retry"). Reviewers will always ask about it.
3. **Buy / managed service / SaaS** — even if you reject it (Module 33).
4. **The genuinely different approach** — a different architecture style, not a variation on yours.
5. **The option a senior reviewer would propose** — you can usually predict it; include it.
6. **Variants within your proposal** — sub-decisions (outbox vs direct publish; Functions vs Container Apps for consumers).

How to present them:

- **Same criteria for all options**, derived from the goals and quality scenarios — not criteria invented to favor the winner.
- **A comparison table** for the summary, prose for the reasoning.
- **Steelman each alternative** — describe it as its best advocate would. A strawman comparison destroys trust instantly when a reviewer who likes that option reads it.
- **Say why each was rejected, and under what conditions you'd choose it instead.** That last part is gold: it tells the future reader when to revisit, and it's the "rebuttal" of the argument (Concept 5).

| Criterion (from goals) | A. Do nothing | B. Async after commit + retry in-process | C. Outbox + Service Bus + notification service (**proposed**) | D. Buy: SaaS notification platform |
|---|---|---|---|---|
| Checkout failures from notifications | 0.8% | ~0% | 0% | 0% |
| Lost notifications on crash | Rare | **Possible** (in-memory queue lost on restart) | None (durable outbox) | None after handoff; outbox still needed |
| Latency to send | Immediate | < 1 s | p95 < 5 s | p95 < 10 s (vendor) |
| Effort | 0 | 1 week | 6 weeks | 4 weeks integration + procurement |
| Run cost/month | €0 | €0 | ≈ €300 | ≈ €1,800 at our volume (quote) |
| New operational surface | None | None | One service, one queue | Vendor dependency, data processing agreement |
| Future channels (SMS, push) | Hard | Hard | Adapter per channel | Built in |
| **Why not chosen** | Fails Goal 1 | Fails "no lost notifications"; crash during deploy drops queue | — | Cost 6× at our volume; data-processing review adds 2 months; reconsider if we add ≥ 3 channels |

Two senior subtleties:

- **The "do nothing" and "minimal" options are the ones approvers care most about**, because they're cheapest. If your proposal isn't clearly better than the minimal change on the goals that matter, the minimal change should win — and recommending it is a strong signal of judgment.
- **Record options you *abandoned during the design*.** HashiCorp's RFC template keeps an "abandoned ideas" section for this reason: reviewers who think of them see they were already considered.

**The interview-grade sentence:** *"The alternatives section is where a design earns trust: I always include doing nothing with its quantified cost, the minimal change, a buy option, a genuinely different approach and the option I expect a senior reviewer to propose, compare them on the same criteria drawn from the goals, steelman each one, and say why each was rejected and under what conditions I'd choose it instead. If my proposal isn't clearly better than the minimal change on the goals that matter, I recommend the minimal change."*

---

## Concept 15 — Cross-cutting concerns

These are the sections reviewers from outside your team read first, and they're where "looks good to me" turns into "this will page us every night." Each deserves a short, specific subsection — or an explicit "not applicable because…".

| Concern | What the section must answer | Links |
|---|---|---|
| **Security** | Trust boundaries, authentication and authorization at each hop, secrets (ideally none), data classification, threat model summary with top threats and mitigations | Module 29 (STRIDE, managed identity, least privilege) |
| **Privacy and compliance** | What personal data is processed, retention and deletion, residency, consent, audit requirements, regulatory scope changes (PCI, GDPR DPIA) | Module 29 (LINDDUN) |
| **Reliability** | SLOs for new and affected paths, failure-mode table, retries/timeouts/circuit breakers, idempotency, backpressure, DR (RTO/RPO) | Modules 13, 25, 28 |
| **Observability** | Traces across the async boundary, key metrics, logs (with redaction), dashboards, alerts tied to SLOs, how on-call diagnoses a stuck message | Module 28 |
| **Performance and scalability** | Latency budget, throughput at design point and ceiling, what saturates first, load test plan | Modules 5, 6, 17 |
| **Cost** | Build effort and run cost, cost drivers and how they scale, cost guardrails | Module 33 |
| **Operability** | Who owns it and is on call, deployment model, configuration, runbooks, toil introduced | Module 26 |
| **Data** | Ownership, schema evolution, migration and backfill, retention, backup/restore | Modules 12, 19 |
| **Testing** | How correctness, contracts, failure modes and performance are tested; what is tested in production (canaries) | Module 36 |
| **Compatibility** | API and event versioning, consumers affected, deprecation plan | Module 31 |
| **Accessibility, localization** | If user-facing: templates, languages, accessibility standards | — |

**The failure-mode table** is the single most effective reliability artifact in a design doc:

| Dependency / component | Failure | Detection | Effect on users | Mitigation | Residual risk |
|---|---|---|---|---|---|
| Email provider | Unavailable 30 min | Error-rate alert on send failures (5 min) | Emails delayed; checkout unaffected | Retry with backoff and jitter; DLQ after 24 h; circuit breaker | Emails > 24 h late go to DLQ → manual replay runbook |
| Service Bus namespace | Regional outage | Platform health + publish failures | Notifications delayed | Outbox keeps accumulating; relay resumes on recovery | Delay equals outage; acceptable per Goal 3 priority |
| Outbox relay | Crashes / stuck | Outbox age metric > 2 min | Delayed notifications | Two replicas with lease; restart; alert | None if alert works — alert tested in game day |
| Notification service | Poison message | Delivery count > 10 | One notification lost to DLQ | DLQ + alert + replay tool | Requires operator action |
| SQL (orders DB) | Failover | Connection errors | Checkout errors (existing behavior) | Existing retry policy; outbox write is in the same transaction | Unchanged from today |

The security subsection in particular should not be "we'll follow security best practices." It should be a compressed version of Module 29's threat model: the data flow diagram, the trust boundaries crossed, the top threats and their mitigations, the identities used ("the notification service uses a user-assigned managed identity with Azure Service Bus Data Receiver on one subscription and no other roles").

**The interview-grade sentence:** *"Cross-cutting sections are what outside reviewers read first, so each gets a specific answer or an explicit 'not applicable because': security as a compressed threat model with boundaries, identities and top mitigations; privacy and compliance scope; reliability as SLOs plus a failure-mode table — dependency, failure, detection, user effect, mitigation, residual risk; observability across async boundaries with alerts tied to SLOs; performance with what saturates first; build and run cost; ownership and on-call; data migration and retention; testing; and compatibility."*

---

## Concept 16 — Rollout, migration and rollback

A design that can't be shipped safely isn't a good design. This section answers **how the change reaches production, how existing data and consumers move, and how it un-ships** if it goes wrong. Reviewers with operational scars read it closely; it's where many approvals become "yes, if."

**Rollout** — expose the change gradually and observably:

| Technique | What it buys | .NET/Azure example |
|---|---|---|
| **Feature flags** | Decouple deploy from release; instant off switch | Azure App Configuration feature manager with `Microsoft.FeatureManagement`; percentage or tenant-targeted filters |
| **Dark launch / shadow mode** | Run new path without user effect, compare results | Publish events and consume them, but don't send; compare with legacy sends |
| **Canary / ring deployment** | Limit blast radius by population | Container Apps revisions with traffic splitting; App Service deployment slots; internal tenants first |
| **Progressive percentage** | Watch SLOs at each step | 1% → 10% → 50% → 100% with explicit go/no-go criteria |
| **Bake time** | Catch slow-burn issues | 48 hours at each ring before proceeding |

**Migration** — when data or consumers must move (Module 32 goes deep):

- **Expand / migrate / contract** for schema changes: add the new shape alongside the old, migrate readers and writers, then remove the old.
- **Dual write or change-data-capture with reconciliation**, never dual write without verification (Module 11's dual-write problem).
- **Backfill** plan with throttling, idempotency and progress tracking.
- **Consumer migration** with versioned contracts and a deprecation window.

**Rollback** — the question reviewers always ask: *"It's 2 a.m. and this is broken. What do we do?"*

1. **Rollback mechanism** for each stage: flag off, traffic back to old revision, redeploy previous version.
2. **Data rollback** — the hard part. Which steps are reversible? After the contract step of a schema migration, rollback means a forward fix. Say so explicitly, and put irreversible steps *last*, behind explicit go/no-go decisions.
3. **Rollback triggers** — objective criteria ("checkout error rate +0.1% over baseline for 10 minutes, or any lost notification detected by reconciliation").
4. **Who decides** and who executes.

A useful device is a **reversibility table**:

| Step | Reversible? | How | Point of no return? |
|---|---|---|---|
| 1. Deploy notification service (flag off) | Yes | Delete deployment | No |
| 2. Write to outbox (shadow) | Yes | Flag off; outbox purged | No |
| 3. Send via new path for internal tenant | Yes | Flag off; legacy path resumes | No |
| 4. 100% via new path, legacy code still present | Yes | Flag off | No |
| 5. Remove legacy SMTP code | Redeploy old version | Hours | **Soft** — requires redeploy |

**The interview-grade sentence:** *"I don't consider a design reviewable until it says how it ships and how it un-ships: rollout behind feature flags with dark launch, canary rings and percentage steps gated by explicit SLO criteria; migrations as expand, migrate, contract with reconciled dual writes and throttled idempotent backfills; and a rollback plan with objective triggers, a named decider and a reversibility table — with irreversible steps, especially data ones, placed last behind explicit go/no-go decisions."*

---

## Concept 17 — Risks and open questions

The honest section — and, counterintuitively, the one that most *increases* reviewer confidence. Reviewers know every design has risks; a doc that lists none tells them the author either didn't look or is hiding them.

**Risks** — things that might go wrong *with the design or its delivery*:

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Trigger to act |
|---|---|---|---|---|---|---|
| R1 | Provider rate limits lower than documented | Medium | Delayed emails at peak | Load test with provider sandbox; negotiate limit; client-side rate limiter | Ana | Load test < 500/s |
| R2 | Team has no production Service Bus experience | High | Misconfiguration, slow incident response | Pair with Payments team; game day before 100% | Ivan | Game-day findings |
| R3 | Outbox table growth hurts orders DB | Low | Checkout latency | Purge job; index on `ProcessedAt`; monitor table size | Marko | Table > 20 GB |
| R4 | November freeze compresses the schedule | Medium | Ship without game day | Cut push-notification seam to v2 | Ivan | Behind by > 1 week on 2026-10-30 |

**Open questions** — things not yet decided, each with an owner and a date:

- *Q1: Should transactional and marketing emails share the pipeline?* — Owner: Product; decide by 2026-10-14; default if undecided: **no** (non-goal).
- *Q2: DLQ retention: 7 or 14 days?* — Owner: SRE; decide during review.

Principles:

1. **Distinguish risks from open questions from assumptions** (Concept 11). Risks might happen; questions need answers; assumptions need validation.
2. **Give every open question a default.** "If undecided by X, we'll do Y." This prevents open questions from blocking approval.
3. **Separate design risks from delivery risks** (skills, schedule, dependencies) — both are real; approvers care about both.
4. **Move items out as they resolve**, with a changelog note, so the section reflects the current state.
5. **Never hide a known risk to get approval.** If it materializes, the cost isn't just the incident — it's every future doc you write being read with suspicion.

**The interview-grade sentence:** *"I list risks honestly because it raises reviewer confidence rather than lowering it: each with likelihood, impact, mitigation, an owner and a trigger to act, separating design risks from delivery risks like skills and schedule. Open questions get an owner, a decision date and a default if nobody decides, so they don't block approval — and I never omit a known risk to get a yes, because if it materializes every future document I write gets read with suspicion."*

---

## Concept 18 — Plan, effort, cost and staffing

Approvers decide on cost and time as much as on architecture. This section turns the design into a commitment they can evaluate.

1. **Milestones with exit criteria**, not just dates: *M1 (2026-10-24): service deployed, shadow mode, traces visible end-to-end. M2 (2026-11-07): internal tenants live, game day passed. M3 (2026-11-14): 100%, legacy path behind flag.*
2. **Effort estimate** with ranges and confidence: "6 engineer-weeks (range 5–9; main uncertainty: provider integration)."
3. **Staffing** — who, with what skills, and what they stop doing to do this.
4. **Dependencies** — other teams, approvals, procurement, platform features — each with an owner and a date.
5. **Cost** — build cost (people × time) and **run cost** (monthly, computed from the capacity estimate, with drivers): "Service Bus Standard at our volume ≈ €X; Container Apps consumption ≈ €Y; Application Insights ingestion ≈ €Z; total ≈ €300/month at the design point; ≈ €1,000/month at the ceiling, where Premium becomes necessary." Use the Azure pricing calculator and say when you priced it (Module 33).
6. **Phasing and cut lines** — what ships in v1, what's deferred, and what you'd cut first if time compresses. Cut lines decided in advance are much better than cut lines decided in a panic.

Don't let this section turn the doc into a project plan; the detailed tasks belong in the backlog. The doc needs enough for the approver to judge feasibility and the cost of the commitment.

**The interview-grade sentence:** *"I give approvers what they actually decide on: milestones with exit criteria rather than just dates, effort as a range with its main uncertainty, staffing and what those people stop doing, dependencies with owners and dates, build and monthly run cost computed from the capacity estimate with its drivers and the pricing date, and cut lines agreed in advance so a compressed schedule removes scope deliberately instead of in a panic."*

---

## Concept 19 — Appendices, glossary, references and FAQ

The appendix is how you **add depth without hurting readability**: everything a specialist reviewer might want, out of the main reader's way.

Typical contents:

- **Detailed calculations** behind the estimate and cost.
- **Benchmark and spike results** with methodology (BenchmarkDotNet output, load-test reports — Module 17).
- **Full schemas and API definitions** (OpenAPI, AsyncAPI, JSON Schema, SQL DDL).
- **The full threat model** (Module 29) and data flow diagram.
- **Detailed comparison data** for alternatives.
- **Glossary** — especially for cross-team or domain-heavy docs (DDD's ubiquitous language, Module 22).
- **References** — prior docs, ADRs, incidents, vendor documentation with dates.

**The FAQ** deserves special mention: it's where you **pre-answer the objections you expect** (Concept 35). "Why not Event Grid?" "Why not just retry inside the request?" "Why a new service instead of a background job in the Orders API?" Each answer is two or three sentences with a link to the relevant section. A good FAQ often shortens a review meeting by half, because the most predictable debates are already resolved in writing — and the reviewer who raises one sees it was taken seriously.

**The interview-grade sentence:** *"I use appendices to add depth without hurting readability — calculations, benchmark and spike results with methodology, full schemas, the full threat model, detailed comparison data, a glossary and dated references — and I add an FAQ that pre-answers the objections I expect, like 'why not Event Grid?' or 'why a new service rather than a background job?', in two or three sentences each, which often halves the review meeting because the predictable debates are already settled in writing."*

---
# Part C — Writing a design document that works

## Concept 20 — The writing process

Strong design docs are rarely written top to bottom in one pass. A process that works for most significant designs:

| Step | What you do | Why |
|---|---|---|
| **1. Frame the problem** | Write the problem statement, goals and non-goals first — half a page | If you can't state the problem crisply, you're not ready to design; this is also the part you should validate with product and the decider first |
| **2. Talk before you write too much** | 2–4 short conversations with the people most likely to object or to know something you don't | Cheapest way to discover a hidden constraint or a fatal objection (Concept 29's "pre-wiring") |
| **3. Outline the argument** | Headings plus one sentence per section: claim, evidence, alternatives, risks | Exposes gaps in reasoning before prose hides them |
| **4. Write a one-pager** | Problem, proposal, main alternative, main risk | Share with 2–3 trusted reviewers; kill or redirect early. Many Tier 1 changes stop here |
| **5. Gather evidence** | Spikes, benchmarks, cost estimates, queries against production telemetry | Converts claims into grounds (Concept 24) |
| **6. Draft the full doc** | Alternatives and failure modes *before* polishing the design section | These are where the author finds the flaws |
| **7. Self-review as a hostile reviewer** | Read it as the most skeptical stakeholder; list the ten questions they'll ask; answer them in the doc or FAQ | Concept 35 |
| **8. Cut** | Remove anything no reader needs; move depth to appendices | Every extra page dilutes attention on the parts that matter |
| **9. Write the summary** | Last — now that you know what the doc concludes | Concept 7 |
| **10. Open for review** | With named reviewers, specific asks and a decision date | Concept 29 |

Habits that make the difference:

- **Write the alternatives section early.** If you can't find a credible alternative, you don't understand the problem well enough — or the decision is genuinely obvious and a Tier 1 doc will do.
- **Keep a "questions I can't answer yet" list** while writing. It becomes the open-questions section and the spike backlog.
- **Iterate in the open, but label maturity.** A draft marked "early — looking for direction, not detail" gets very different (and more useful) feedback than one that looks finished. Oxide's *prediscussion* and *ideation* states exist for this.
- **Time-box.** A design doc that takes longer than the work it describes is mis-tiered. Most Tier 2 docs should take days, not weeks — the spikes take longer than the writing.

**The interview-grade sentence:** *"I write design docs in a sequence: frame the problem, goals and non-goals and validate them with the decider; talk to the likely objectors before writing much; outline the argument; circulate a one-pager to kill or redirect early; gather evidence through spikes and telemetry; draft alternatives and failure modes before polishing the design; self-review as the most hostile stakeholder; cut; write the summary last; then open review with named reviewers and a date — labeling maturity throughout so people know whether I want direction or detail."*

---

## Concept 21 — Clarity: writing that busy experts can review

Reviewers are senior, busy and reading many documents. Every unclear sentence costs a comment thread. The techniques that matter most:

**1. Bottom line up front and the pyramid principle.** Barbara Minto's *pyramid principle*: state the conclusion, then the supporting arguments, then the details — at every level: document, section, paragraph. The first sentence of each section should tell the reader what the section concludes. A reviewer skimming only first sentences should get the whole argument.

**2. Concrete over abstract.** Numbers, names and examples instead of adjectives.

| Vague | Concrete |
|---|---|
| "significantly faster" | "p99 drops from 1.9 s to 750 ms" |
| "a small number of tenants" | "3 of 1,240 tenants (0.2%), all on the Enterprise plan" |
| "highly scalable" | "linear to 8 replicas; then SQL connection pool saturates" |
| "some risk of data loss" | "messages in flight during a crash (≤ 64) may be redelivered, never lost" |
| "the system" | "the Orders API" / "the outbox relay" |

**3. Precise modal verbs.** MUST / SHOULD / MAY (Concept 10). Avoid "will try to", "ideally", "where possible" unless you say what happens when it isn't possible.

**4. Define terms once, use them consistently.** If "order" means a submitted cart in one section and a fulfilled shipment in another, reviewers will argue past each other. A glossary and DDD's ubiquitous language (Module 22) help.

**5. Active voice with a subject.** "The relay publishes the event" instead of "the event is published" — the second hides *who* is responsible, which is exactly what reviewers need to know.

**6. One idea per paragraph; short paragraphs; descriptive headings.** Headings like "Why not Event Grid" beat "Discussion."

**7. Tables for comparisons, lists for sequences, prose for reasoning.** Bullet lists are good at enumerating and bad at explaining *why*. The rationale for a choice usually needs sentences with "because."

**8. Remove hedging and filler.** "It could be argued that perhaps…" → state the claim and its qualifier. Remove buzzwords ("cloud-native", "robust", "seamless") that carry no checkable content.

**9. Show uncertainty explicitly.** Confidence is a property of each claim, not a tone. "We are confident (load-tested) that…" versus "We believe (not yet tested) that…" — the Azure WAF ADR guidance recommends recording the **confidence level** of each decision for the same reason.

**10. Write for the reader who disagrees with you.** If that reader can follow your reasoning and see where they'd diverge, the doc is clear — even if they still disagree.

Free resources that teach this well: Google's *Technical Writing One and Two* courses, the Microsoft Writing Style Guide, and plain-language guidelines (all in the resources section).

**The interview-grade sentence:** *"I write for busy expert reviewers: conclusion first at every level in the pyramid-principle sense so first sentences carry the argument; numbers and names instead of adjectives; MUST, SHOULD and MAY instead of 'ideally'; terms defined once and used consistently; active voice so responsibility is visible; tables for comparisons and prose with 'because' for reasoning; no buzzwords; and confidence stated per claim — tested or believed — so the reader who disagrees can see exactly where they diverge."*

---

## Concept 22 — Diagrams that earn their place

Diagrams are where reviewers build their mental model, so a bad diagram creates a bad review. Rules:

1. **One question per diagram.** "What are the components and how do they connect?" (structure), "What happens when an order is confirmed?" (behavior), "Where does it run?" (deployment), "Where are the trust boundaries?" (security). A single diagram trying to answer all four answers none.
2. **Choose the right kind:**

| Question | Diagram | Notes |
|---|---|---|
| Who uses the system and what does it talk to? | System context (C4 level 1) | Module 31 |
| What are the deployable pieces? | Container diagram (C4 level 2) | The usual altitude for a design doc overview |
| What happens in a flow, including failures? | **Sequence diagram** | The most underused and most valuable diagram in design docs |
| What states can an entity be in? | State diagram | Orders, payments, sagas (Module 12) |
| Where does data cross trust boundaries? | Data flow diagram | Module 29 |
| Where does it run, in which regions? | Deployment diagram | Networking, zones, regions |
| How does it change over the rollout? | Before / after (or per-phase) diagrams | Brownfield work, Module 32 |

3. **Every diagram has a title, a legend and labeled arrows.** An unlabeled arrow is ambiguous: sync or async? Request or response? Which protocol? Who initiates? Label with verb + protocol: "publishes OrderConfirmed (AMQP)".
4. **Consistent notation across the doc** — same shape and color for the same kind of thing; mark new and changed elements (e.g., "NEW", "CHANGED") in before/after diagrams.
5. **Show the failure path,** not just the happy path, for the flows that matter.
6. **Readable at the size it's viewed** — if it needs zooming, split it.
7. **Diagrams as code** where possible, so they're versioned and reviewed with the doc: **Mermaid** (rendered natively in GitHub and Azure DevOps Markdown and wikis), **PlantUML**, **Structurizr DSL** (C4 models), **D2**. They trade some visual polish for diffability and longevity.

A sequence diagram that answers the reviewer's first question — *"what happens if the process dies between the database commit and the publish?"*:

```mermaid
sequenceDiagram
    autonumber
    participant API as Orders API
    participant DB as Orders DB (SQL)
    participant Relay as Outbox relay
    participant SB as Service Bus topic
    participant NS as Notification service
    participant P as Email provider

    API->>DB: BEGIN; insert Order; insert OutboxMessage(OrderConfirmedV1); COMMIT
    Note over API,DB: One local transaction — no dual write
    Relay->>DB: poll unprocessed outbox rows (lease)
    Relay->>SB: publish OrderConfirmedV1 (MessageId = EventId)
    Relay->>DB: mark row processed
    Note over Relay,SB: Crash after publish, before mark → republish;<br/>duplicate detection + consumer idempotency absorb it
    SB->>NS: deliver (peek-lock)
    NS->>NS: EventId already processed? → complete & drop
    NS->>P: send(idempotencyKey = EventId)
    alt provider error / timeout
        NS-->>SB: abandon → redelivery with backoff; DLQ after max delivery count
    else success
        NS->>SB: complete
    end
```

**The interview-grade sentence:** *"Every diagram in my docs answers one question — structure, behavior, deployment or trust boundaries — at the right altitude, usually a C4 container view for the overview and sequence diagrams for the flows, including the failure path. Each has a title, a legend and arrows labeled with a verb and a protocol, uses consistent notation with new and changed elements marked, and lives as code — Mermaid, PlantUML, Structurizr or D2 — so it's versioned and reviewed with the text."*

---

## Concept 23 — Making trade-offs explicit

Architecture is the art of choosing what to give up. A design doc that hides its trade-offs invites reviewers to discover them — usually the worst way for an author to have them surfaced. Tools, from simplest to most formal:

**1. The trade-off statement.** For each significant decision, one sentence of the form: *"We choose X over Y, gaining A at the cost of B, because goal G outranks goal H."* "We choose eventual consistency for notifications, gaining checkout independence from the provider at the cost of up to minutes of delay during outages, because Goal 1 (checkout never fails due to notifications) outranks Goal 3 (fast notifications)."

**2. The trade-off table** — options against quality attributes, with honest entries including the cells where your option loses (Concept 14).

**3. The weighted decision matrix** — criteria with weights, options scored, weighted sum. Widely used, often misused. Its traps:

| Trap | What goes wrong | Fix |
|---|---|---|
| **False precision** | A 7.42 vs 7.38 "win" from subjective 1–10 scores | Use coarse scales (e.g., −/0/+ or 1–3); treat close scores as a tie |
| **Post-hoc weights** | Weights adjusted until the preferred option wins | Agree on weights *before* scoring, ideally with the decider |
| **Compensation hides deal-breakers** | A great score on cost compensates for failing a hard requirement | Apply **must-have gates** first; only score options that pass every gate |
| **Correlated criteria** | "Scalability", "performance" and "throughput" triple-count the same property | Merge correlated criteria |
| **Missing criteria** | Operability and team skills absent; the "best" option is unrunnable | Derive criteria from goals and quality scenarios, plus delivery and operations |
| **Single-point scores** | No sense of uncertainty | Note confidence; run a **sensitivity check**: does the winner change if a weight moves ±20%? |

Use the matrix to *structure discussion*, never as the decision itself. The prose rationale decides.

**4. ATAM vocabulary** (Concept 32), which is worth borrowing even informally:
- A **sensitivity point** is a decision that strongly affects one quality attribute ("the Service Bus `MaxConcurrentCalls` setting determines throughput").
- A **trade-off point** is a decision that affects *several* attributes in opposite directions ("partitioning the topic improves throughput but loses global ordering").
- **Risks** are decisions that may cause a quality goal to fail; **non-risks** are decisions you've verified are safe.

Naming trade-off points explicitly is one of the clearest senior signals in a doc.

**5. Reversibility as a dimension.** For each decision, say whether it's a one-way or two-way door and what reversing would cost. Reviewers rightly tolerate more uncertainty in reversible decisions.

**6. Make the normative choice explicit.** Many trade-offs are value choices ("we prefer managed services even at 2× cost because the team has no ops capacity"). Writing the principle lets reviewers challenge the principle directly instead of nitpicking every consequence.

**The interview-grade sentence:** *"I make trade-offs explicit with one-sentence statements — we choose X over Y, gaining A at the cost of B, because goal G outranks goal H — and tables that show where my option loses. I'll use a weighted decision matrix only to structure discussion: must-have gates first, coarse scores, weights agreed before scoring, correlated criteria merged, and a sensitivity check on whether the winner survives a weight change. And I borrow ATAM's language: sensitivity points that drive one attribute and trade-off points that move several in opposite directions, plus reversibility for every decision."*

---

## Concept 24 — Quantification and evidence

A design doc's persuasive power comes from the quality of its grounds. Think of evidence as a ladder — the higher the rung, the more weight a claim can bear:

| Rung | Evidence | Strength | Use for |
|---|---|---|---|
| 1 | Intuition, "best practice", "everyone does it" | Weakest — and senior reviewers discount it | Nothing load-bearing |
| 2 | Vendor documentation, published limits, reference architectures | Moderate; may not match your workload; dated | Initial sizing; feasibility |
| 3 | Back-of-envelope estimate with sourced inputs | Moderate; transparent | Capacity, cost, latency budgets |
| 4 | Others' published experience (engineering blogs, case studies, papers) | Moderate; different context | Pattern selection; known pitfalls |
| 5 | **Your own spike or prototype** | Strong for the question it answers | Feasibility, integration, developer experience |
| 6 | **Benchmark or load test** with stated methodology | Strong if methodology is sound | Performance and capacity claims |
| 7 | **Production telemetry** from your own system | Strongest for current-state claims | Problem statement, baselines, traffic shape |
| 8 | **Production experiment** (canary, A/B, shadow traffic) | Strongest for behavior claims | Validating the riskiest assumptions during rollout |

Practices:

1. **Put the strongest evidence behind the most load-bearing claims.** If the whole design rests on "Cosmos DB's change feed keeps up at 3,000 writes/s," that claim needs a rung-5 or rung-6 result, not a vendor page.
2. **State methodology.** "BenchmarkDotNet, .NET 10, Release, 3 runs, Server GC, `[MemoryDiagnoser]`" or "Azure Load Testing, 30-minute ramp to 1,200 RPS against the staging environment with production-shaped data" (Modules 17 and 5). Reviewers can't trust a number whose provenance they can't see.
3. **Report distributions, not averages** — p50/p95/p99, and the shape at saturation.
4. **Label confidence.** "Measured", "estimated", "assumed". Many reviews stall because reviewers can't tell which numbers were measured.
5. **Spikes answer specific questions.** "Can the relay sustain 500 msg/s with one replica?" — a spike that tries to "evaluate Service Bus" in general produces opinions, not evidence. The ISE playbook's spike template forces a question, a time box and a conclusion.
6. **Cheap prototypes on .NET.** A throwaway solution with an Aspire AppHost wiring up a SQL container, a Service Bus emulator or a dev namespace and two small services can answer integration questions in a day; record the result and delete the code.
7. **Don't overclaim from small evidence.** A local benchmark doesn't prove production latency; say what it does prove.

**The interview-grade sentence:** *"I think of evidence as a ladder — intuition, vendor docs, back-of-envelope math, others' experience, my own spike, a benchmark or load test with stated methodology, production telemetry, and finally a production experiment — and I put the strongest rungs under the most load-bearing claims. Every number states its methodology and percentiles, every claim is labeled measured, estimated or assumed, and spikes answer one specific question in a timebox; in .NET an Aspire-wired throwaway prototype can answer an integration question in a day."*

---

## Concept 25 — Length, altitude and tiers

The right length is the shortest that lets every required reviewer evaluate their concerns. Longer is not more rigorous; it's usually less reviewed.

| Tier | Typical length | Contents | Example |
|---|---|---|---|
| **1 — One-pager** | ≤ 2 pages | Problem; proposal; one main alternative; key risk; rollout/rollback in two lines | Add a Redis cache in front of a read-heavy endpoint with HybridCache (Module 10) |
| **2 — Standard doc** | ~4–10 pages + appendices | Full anatomy, compressed where not applicable | New notification service (Worked example 1) |
| **3 — Major proposal** | 10–20+ pages, or a parent doc linking child docs | Full anatomy, formal evaluation (ATAM-style), staged decisions | Moving from single-region to multi-region active-active; changing the tenancy model |

**Amazon's six-page narrative** is worth knowing as a format: a prose memo (no slide decks), read silently by everyone at the start of the meeting, followed by discussion. Bezos's shareholder letters describe the reasoning — narrative structure forces better thinking than bullet points — and the silent reading guarantees that everyone in the room has actually read the document. Many companies have adopted the "read first, then discuss" pattern even without the six-page limit.

**Altitude** is the companion to length: write at the level where decisions are being made. Signs you're too low: method names, class diagrams, library configuration snippets. Signs you're too high: no contracts, no failure behavior, nothing an implementer could act on. A useful test: *every paragraph should help a reviewer evaluate the decision or an implementer avoid a mistake.*

**Split large designs.** A Tier 3 change often works best as a short **parent doc** (problem, strategy, decomposition, cross-cutting decisions) plus **child docs** for each component, each reviewed by the people who care. This mirrors how Kubernetes KEPs and Rust RFCs keep individual proposals focused.

**The interview-grade sentence:** *"The right length is the shortest that lets every required reviewer evaluate their concerns: a one-pager for moderate single-team changes, four to ten pages plus appendices for a standard design, and for major one-way-door proposals a short parent document with focused child documents. I keep the altitude where decisions are made — every paragraph should help a reviewer judge the decision or an implementer avoid a mistake — and I like Amazon's read-first narrative pattern because it guarantees the room has actually read the document."*

---

## Concept 26 — Docs-as-code and tooling

Where the doc lives shapes how it's reviewed, found and maintained. Two families:

| | **Collaborative editors** (Google Docs, Microsoft Word/Loop on SharePoint, Confluence, Notion) | **Docs-as-code** (Markdown/AsciiDoc in Git, reviewed by pull request) |
|---|---|---|
| Strengths | Low friction; inline comments; suggestions; easy for non-engineers | Versioned with history and blame; reviewed like code; lives next to the code; links to source; diagrams-as-code; automatable checks |
| Weaknesses | Discoverability (docs scattered in drives); stale copies; weak status tracking | Higher friction for non-engineers; review comments tied to lines, less to ideas; rendering limitations |
| Who uses it | Google (design docs as Docs), HashiCorp (with its Hermes document system), many enterprises | Oxide RFDs (Git repo + PR discussion), Kubernetes KEPs, Rust RFCs, Go proposals, dotnet/designs, Microsoft ISE (async design reviews via PRs) |

Practical guidance:

1. **Choose by audience.** Engineering-only, long-lived decisions → docs-as-code. Cross-functional docs with product, legal, finance → a collaborative editor, with the final version (and ADRs) committed or linked from the repo.
2. **Make docs discoverable.** A single index (a `docs/rfcs/` folder with a numbered list, an RFD site, an index page) with number, title, status, owner and date. Uber built a dedicated tool for this; Oxide built an RFD site with search and inline discussion — because the hardest problem at scale is *finding* the doc.
3. **Number them.** "RFC-0142" is linkable from code comments, ADRs, tickets and incident reports.
4. **Status in metadata** (front matter), not only in someone's head.
5. **Link both directions** — code → doc (comments, READMEs) and doc → code (as-built links after implementation).
6. **Keep the review record.** Whether in PR comments or doc comments, the discussion is part of the decision history; resolved comments should say *how* they were resolved.

A docs-as-code layout that works well in a .NET repository:

```text
/docs
  /rfcs
    0000-template.md
    0142-event-driven-notifications.md
    index.md                      # number, title, status, owner, date
  /adr
    0031-outbox-for-integration-events.md
  /architecture
    workspace.dsl                 # Structurizr C4 model (Module 31)
  /diagrams
    notifications-sequence.mmd
/src
  Orders.Api/...
```

Add light automation: a Markdown linter, a link checker, a CI check that every RFC has the required front-matter fields, Mermaid rendering in the Azure DevOps or GitHub wiki, and perhaps a doc site generated with DocFX or MkDocs (the ISE playbook has recipes for both).

**The interview-grade sentence:** *"I choose the doc's home by audience: engineering-only, long-lived designs as Markdown in the repo reviewed by pull request — the way Oxide RFDs, Kubernetes KEPs, Rust RFCs and dotnet/designs work — and cross-functional ones in a collaborative editor with the final decisions committed as ADRs. Either way, docs are numbered, carry status in metadata, are listed in one searchable index, link to and from the code, and keep their review discussion, because at scale the hardest problem is finding the doc and knowing whether it's current."*

---

## Concept 27 — AI and design documents in 2026

AI writing assistants changed design-doc practice in two years more than templates changed it in twenty. The senior position is neither "never use AI" nor "let it write the doc"; it's understanding what changed economically.

**What changed:** producing a fluent, well-structured design doc used to take days, so its existence was a *costly signal* that someone had thought hard. Now a plausible doc can be generated in an hour. The cost of *reviewing* one hasn't changed — senior reviewers still need hours to evaluate a design against reality — and their attention was already the scarcest resource. Commentators in 2026 call the result an **"RFC glut"**: more documents, more polish, the same review capacity, and a harder time telling the thought-through from the prompted.

**Where AI genuinely helps the author:**

| Use | Why it works |
|---|---|
| **Adversarial review of your draft** ("act as a skeptical principal engineer; list the ten strongest objections") | Generates the objection list for Concept 35 cheaply; you still judge which are real |
| **Completeness checks** against a template or checklist | Catches missing sections, unstated rollback, missing failure modes |
| **Consistency checks** | Numbers that differ between sections; terms used inconsistently |
| **Clarity edits** | Shortening, active voice, removing hedges |
| **Diagram drafting** (Mermaid/PlantUML from a description) | Fast first draft; you correct it |
| **Summarizing prior art and long threads** | Faster context gathering — verify the claims |
| **Turning a decided design into tasks** | Spec-driven tooling (GitHub Spec Kit's *specify → plan → tasks → implement*) feeds coding agents from a reviewed spec |

**Where it fails, and why reviewers care:**

- **Fluent emptiness.** Generated docs state benefits fluently and hedge everything else. They rarely make the *specific, checkable, falsifiable* claims that are the point of a design doc.
- **Invented facts.** Service limits, pricing, API behaviors and "industry benchmarks" that are plausible and wrong. Every number in a doc must have a source you checked.
- **Generic alternatives.** Strawmen chosen because they're common, not because they're the real contenders in your context.
- **No ownership.** The author can't defend a claim they didn't reason through — which shows within two questions of a review (and within two minutes of an interview).

**Policies that work** (and are good answers when an interviewer asks how you'd run a design process today):

1. **The author owns every claim**, regardless of who or what drafted it; "the AI suggested it" is not a rationale.
2. **Require a few human-written anchors** at the top — one practitioner proposal is three: *the trade-off that kept you up at night, the strongest argument against this design, and what you'll measure to know it failed.* These are hard to fake and quick to review.
3. **Triage review visibly** — when the queue exceeds the review budget, docs wait or get downgraded in tier explicitly rather than rotting in a queue everyone pretends is being read.
4. **Use AI on the review side too, as a first pass**: Cloudflare described in 2026 an AI reviewer that checks specs and code against the company's own RFC corpus, producing non-blocking findings from approved RFCs and blocking only on explicitly *enforced* MUST requirements — a sensible split between machine consistency checking and human judgment.
5. **Distinguish a design doc from an agent spec.** A spec-driven development artifact (Spec Kit's `spec.md` / `plan.md` / `tasks.md`) is an *input to implementation*; a design doc is an *input to a decision*. The design doc's alternatives, trade-offs and risks are exactly what an agent spec omits.

**The interview-grade sentence:** *"AI made writing a polished design doc nearly free but left reviewing one as expensive as ever, so the bottleneck is senior review attention. I use AI to red-team my drafts, check completeness and consistency, tighten prose and draft diagrams — never as the source of numbers or alternatives I haven't verified, because the author owns every claim. As a process owner I'd require a few human-written anchors like the hardest trade-off, the strongest argument against and the failure metric, triage the review queue visibly, use machine review for consistency against our own standards the way Cloudflare does, and keep design docs, which serve decisions, distinct from agent specs, which serve implementation."*

---
# Part D — The review process

## Concept 28 — Review models

Organizations review design docs in a handful of recognizable ways. Each makes a different trade-off between speed, rigor, breadth and fairness — and as an architect you'll often be the person choosing or reshaping the process.

| Model | How it works | Strengths | Weaknesses | Seen at |
|---|---|---|---|---|
| **Async comment review** | Doc shared with named reviewers; comments inline; author resolves | Scales across time zones; reviewers go deep on their area | Threads sprawl; silence mistaken for approval; slow without a deadline | Google, most companies |
| **Review meeting** | Doc pre-read (or read silently at the start), then discussion | Fast resolution of contested points; shared understanding | Dominated by loud voices; limited to who's in the room | Amazon's narrative meetings; many teams |
| **Architecture review board (ARB)** | Standing group of senior architects approves significant designs | Consistency across the org; expertise concentrated | Bottleneck; ivory-tower risk; "approval theater"; authors design to pass the board | Enterprises, regulated industries |
| **Review council with "Yes, if"** | Senior engineers review in depth, defaulting to conditional approval ("yes, if you add X") rather than veto | Keeps momentum; feedback becomes conditions; respects the author's work | Can centralize power in a few people; critics argue it suppresses legitimate blocking objections | Squarespace (Tanya Reilly's write-up, 2019) |
| **PR-based RFC repository** | Doc is a PR; discussion in review comments; merge = accepted | Docs-as-code benefits; transparent; durable history | Less friendly to non-engineers; long PR threads | Oxide RFDs, Rust RFCs (with a final comment period), Kubernetes KEPs, dotnet/designs |
| **API review board** | Specialists review every public API against design guidelines | Consistency of public surface; catches irreversible mistakes | Narrow scope; a queue | .NET's FXDC for dotnet/runtime; ASP.NET Core's weekly API review |
| **Advice process** | Anyone may make an architectural decision, provided they seek advice from those affected and those with expertise; supported by ADRs and an advisory forum | Decentralized and fast; builds architectural skill across teams | Requires trust and good ADR discipline; advice can be ignored | Described by Andrew Harmel-Law on martinfowler.com |
| **Lightweight peer review** | One or two peers approve a one-pager | Proportionate for Tier 1 | Too light for one-way doors | Most teams for small changes |

The **.NET API review process** is a useful concrete example to know as a .NET engineer: proposals are filed as issues in dotnet/runtime using an API-proposal template; area owners label ready ones `api-ready-for-review`; the FXDC reviews them in regular (livestreamed) meetings; the outcome is `api-approved` with the approved shape recorded on the issue, or `api-needs-work`; and pull requests adding public API shouldn't be submitted before approval. It shows two principles that generalize: **irreversible surfaces get specialist review**, and **the approved artifact is recorded where implementers will see it.**

Choosing a model is itself a trade-off. A healthy organization usually combines them by tier: peer review for Tier 1, async review plus a meeting for Tier 2, a board or formal session for Tier 3, specialist boards for public APIs and security-sensitive designs.

Two failure modes to name in an interview:

- **Approval theater:** the board approves whatever arrives because it lacks context or time, while teams treat approval as a box to tick. Symptom: boards that rarely change designs but always delay them.
- **Design by committee:** every reviewer's preference gets added, nobody owns the coherence, and the design becomes the union of everyone's concerns. Symptom: designs that grow during review. The cure is a named decider and non-goals (Concepts 9 and 33).

**The interview-grade sentence:** *"I'd pick the review model by tier: peer review for one-pagers, async review plus a meeting for standard docs, a formal session or board for one-way doors, and specialist boards for public APIs — the way .NET's FXDC reviews every API in dotnet/runtime before implementation — possibly within an advice-process culture where teams decide after seeking advice. Squarespace's 'Yes, if' default keeps momentum by turning objections into conditions, and the failure modes I watch for are approval theater, where boards delay without improving, and design by committee, where nobody owns coherence."*

---

## Concept 29 — Preparing for review

Most design reviews are won or lost before they start.

**1. Pre-wire the key stakeholders.** Before the doc is formally opened, talk to the people whose objections would be fatal or whose knowledge you need: the decider, the team that will operate it, the owner of the system you're changing, the security reviewer. In Japanese management practice this is *nemawashi* — preparing the roots before moving a tree. The goal isn't to lobby; it's to **surface objections while they're cheap** and to make sure no key stakeholder is surprised in a meeting. Surprised people defend their position; consulted people improve the design.

**2. Choose reviewers deliberately.**
- **Required reviewers**: one per major concern (operability, security, the dependent team's contract, cost), each told what to look at.
- **Include a likely skeptic.** A review that only includes allies produces approval without improvement — and the skeptic will find the problem in production instead.
- **Keep it small.** Five focused reviewers beat twenty passive ones. Wider audiences can be *informed* rather than asked to approve.

**3. Say what feedback you want.** "I'm confident about the data model; I'd like the most scrutiny on the outbox relay's failure handling (§6.3) and the rollback plan (§9). I'm not looking for naming feedback yet." This focuses attention where it matters and signals that you know where your weak spots are.

**4. Set expectations on timing.** "Comments by Friday; review meeting Tuesday; decision by the following Monday." Give reviewers enough time to actually read — a 10-page doc deserves at least a few working days of async review.

**5. Make the doc reviewable.** Line numbers or section numbers; a changelog once it starts changing; a clear status; comments enabled; diagrams readable without zooming.

**6. Prepare the FAQ** (Concept 19) with the objections pre-wiring surfaced.

**7. Know your walk-away positions.** Before the review, decide which parts you'd concede easily, which you'd argue for, and which are genuinely non-negotiable (and why — usually a constraint or a goal priority). This prevents both caving on the important point and fighting to the death on a trivial one.

**The interview-grade sentence:** *"I prepare a review before opening it: I pre-wire the decider, the operators, the owners of affected systems and security in short conversations so objections surface while they're cheap and nobody is surprised in the meeting; I pick a small set of required reviewers, one per concern, including a likely skeptic; I tell each what I want scrutinized; I give real reading time with a decision date; and I go in knowing what I'd concede, what I'd argue for and what's non-negotiable because of a constraint."*

---

## Concept 30 — Running a design review meeting

When a synchronous review is needed, structure turns an hour of opinion into decisions.

**Roles:**

| Role | Responsibility |
|---|---|
| **Author / presenter** | Owns the doc; frames the questions; answers; does *not* chair the debate about their own design if avoidable |
| **Facilitator** | Keeps time and focus; ensures quiet voices are heard; parks tangents |
| **Decider** | Makes the call at the end, or states what's needed to make it |
| **Scribe** | Records decisions, conditions, action items and open questions *in the doc* |
| **Reviewers** | Bring prepared concerns, ranked by importance |

**A 60-minute agenda that works:**

| Time | Activity |
|---|---|
| 0–10 (or 0–20 for longer docs) | **Silent reading** if not pre-read — Amazon's practice; everyone writes questions as they read |
| 10–15 | Author: the decision requested and the two or three areas needing the most scrutiny — *not* a re-presentation of the doc |
| 15–45 | **Risk-first discussion**: the highest-impact concerns first (each reviewer's top one), then down the list; facilitator timeboxes each |
| 45–55 | Outcomes: decision or "yes, if" conditions; open questions with owners and dates |
| 55–60 | Confirm what will change in the doc and when; who signs off on conditions |

Practices:

- **Don't present the slides version of the doc.** Re-presenting rewards people who didn't read and wastes the time of those who did. Present only what the doc can't: the question, the stakes, the uncertainty.
- **Start with the top risks, not the first section.** Meetings that walk the doc in order spend 40 minutes on context and 5 on the failure modes.
- **Separate "I don't understand" from "I disagree."** The first is fixed by the doc; the second needs discussion.
- **Capture outcomes in the doc, live.** A meeting whose conclusions live only in people's memories didn't happen.
- **Possible outcomes are explicit:** *approved*, *approved with conditions* ("yes, if"), *needs another iteration* (with what must change), *rejected* (with why), *needs a spike* (with the question).
- **Escalate unresolved value disagreements** to the decider rather than litigating them in the room (Concept 38).

**The interview-grade sentence:** *"I run a design review with distinct roles — author, facilitator, decider, scribe — and a short agenda: silent reading if the doc wasn't pre-read, the author framing only the decision requested and the areas needing scrutiny, a risk-first discussion where each reviewer's top concern comes before any walk through the sections, and explicit outcomes — approved, yes-if with conditions, another iteration, a spike or rejected — captured live in the doc with owners and dates, escalating value disagreements to the decider instead of litigating them in the room."*

---

## Concept 31 — Being a good reviewer

Architect interviews often test the *reviewer* side: "Here's a design doc — review it." And at work, the quality of your reviews shapes the engineering culture around you.

**Review in three passes:**

1. **The problem pass** — Is the problem real, evidenced and worth solving now? Do goals and non-goals match it? Is the decision requested clear? *If the problem pass fails, stop: detailed design feedback is wasted.*
2. **The decision pass** — Are the alternatives real (including do-nothing and minimal)? Are the criteria derived from the goals? Does the proposal actually meet the goals? Are the trade-offs stated? Is the riskiest assumption validated?
3. **The detail pass** — Failure modes, consistency, security, operability, rollout and rollback, cost, contracts, migration. Use a checklist (Appendix B).

**Prioritize and label your feedback.** The author can't tell which of your 30 comments matter unless you say so. **Conventional Comments** offers a simple scheme — a label plus an optional decoration:

```text
issue (blocking): The relay marks rows processed after publishing, but there's no duplicate
detection on the topic and the consumer's idempotency check (§6.4) isn't in the same
transaction as the side effect. A crash between send and record will double-send. Suggest
recording EventId and the "send requested" row in one transaction before calling the provider.

question (non-blocking): Why Container Apps over Functions for the consumer? Not opposed —
I'd like the reasoning in §7.

suggestion (non-blocking): A sequence diagram for the DLQ replay path would help on-call.

nitpick: "Notifcation" in §4 heading.

praise: The failure-mode table is excellent — I'd like to reuse its shape.
```

Principles of good review feedback:

- **Anchor to goals.** "This doesn't meet Goal 2 because…" is far more actionable than "I don't like this."
- **Ask before asserting.** "What happens if the relay crashes after publishing?" lets the author show they've thought of it — or discover they haven't — without a fight.
- **Distinguish preference from problem.** "I'd have used Functions" is a preference; "unbounded Functions scale-out would exceed the provider's concurrent-connection limit" is a problem. Label preferences as such, and let them go.
- **Propose, don't just object.** A blocking comment should suggest what would unblock it — that's the essence of **"Yes, if"**.
- **Review the doc you were given**, not the doc you'd have written. The bar is "good and safe enough to proceed," not "what I'd do."
- **Respect the author's time and effort.** Specific praise for good sections is not fluff; it tells the author what to keep.
- **Be explicit about your verdict.** "Approve with two conditions (comments 1 and 4)" — not a silence that the author must interpret.
- **Know when to talk instead.** If a thread has gone back and forth three times, switch to a call and write the outcome in the doc.

**The interview-grade sentence:** *"I review in three passes — the problem, then the decision and its alternatives, then the details against a checklist — and stop early if the problem pass fails. I label every comment by type and whether it blocks, using something like Conventional Comments; anchor objections to the doc's own goals; ask before asserting; separate my preferences from real problems; make every blocking comment say what would unblock it, in the 'Yes, if' spirit; review the doc as written rather than the one I'd have written; and end with an explicit verdict."*

---

## Concept 32 — Structured evaluation methods

For high-stakes designs, ad hoc review isn't enough. Several structured methods exist; a senior architect knows what each is for and how to run a lightweight version.

**ATAM — the Architecture Tradeoff Analysis Method** (Software Engineering Institute, Carnegie Mellon; Kazman, Klein and Clements). The canonical method for evaluating an architecture against multiple competing quality attributes. Its nine steps, in two phases with stakeholders:

1. Present the ATAM. 2. Present business drivers. 3. Present the architecture. 4. Identify architectural approaches. 5. **Generate the quality attribute utility tree.** 6. Analyze architectural approaches. 7. **Brainstorm and prioritize scenarios** (with the wider stakeholder group). 8. Analyze architectural approaches again against the new scenarios. 9. Present results.

The **utility tree** is the most reusable artifact: *Utility* at the root, quality attributes beneath it, refinements beneath those, and concrete scenarios (Concept 10) as leaves — each leaf rated **(importance, difficulty)** as H/M/L. The (H,H) leaves are where analysis time goes.

```text
Utility
├── Availability
│   ├── Provider outage
│   │   └── (H,M) Provider down 30 min at peak → checkout unaffected; all mail delivered ≤ 10 min after recovery
│   └── Region outage
│       └── (M,H) Primary region lost → notifications resume ≤ 1 h after failover; none lost
├── Performance
│   └── Burst
│       └── (H,H) Flash sale ×20 burst → p95 send latency ≤ 30 s; no checkout impact
├── Modifiability
│   └── New channel
│       └── (M,L) Add push channel in ≤ 5 engineer-days without touching Orders
└── Security
    └── PII exposure
        └── (H,M) Notification payloads in logs/telemetry contain no customer email in clear text
```

ATAM's outputs are exactly the vocabulary from Concept 23: **risks**, **non-risks**, **sensitivity points**, **trade-off points** and **risk themes** — recurring patterns of risk that point at systemic issues ("every risk traces to the team's lack of messaging experience").

A full ATAM is heavyweight (days, with an evaluation team). Its **ideas** scale down to an afternoon: build a small utility tree with the team, pick the (H,H) scenarios, walk each through the design, and record risks, sensitivity and trade-off points.

**Lighter-weight methods:**

| Method | What it is | Use it for |
|---|---|---|
| **ARID** (Active Reviews for Intermediate Designs, SEI) | Reviewers actively *use* a partial design (write code against an interface, walk scenarios) instead of passively reading | Reviewing interfaces and APIs before they're finished |
| **Quality Attribute Workshop** (SEI) | Stakeholder workshop to elicit and prioritize quality attribute scenarios before design | Early requirements for a new system |
| **Pre-mortem** (Gary Klein, *HBR* 2007) | "Imagine it's a year from now and this project failed. Write down why." Then mitigate the most plausible causes | Overcoming optimism and groupthink; surfacing risks people hesitate to raise. Klein cites research showing that this "prospective hindsight" improves people's ability to identify reasons for future outcomes by roughly 30% |
| **Risk storming** (Simon Brown) | Each participant independently marks risks on the architecture diagram (often with likelihood × impact), then the group converges | Quick, visual, collaborative risk identification |
| **Threat modeling** (Module 29) | STRIDE over a data flow diagram | Security review of the design |
| **Azure Well-Architected Review** | Free assessment questionnaire across the five pillars, plus each pillar's design review checklist and documented trade-offs | Checking an Azure workload design for gaps; a shared vocabulary with Microsoft and partners |
| **Production readiness review (PRR)** / launch checklist | Google SRE's practice of checking a service's operational readiness before SRE takes it on; the SRE book publishes Google's original launch coordination checklist | The gate between "designed" and "in production" |
| **Game day / chaos experiment** | Inject the failures the design claims to handle (e.g., Azure Chaos Studio) | Turning the failure-mode table from claims into evidence |

**The interview-grade sentence:** *"For high-stakes designs I borrow from ATAM even when I don't run the full nine-step method: a utility tree with quality attributes refined down to concrete scenarios rated by importance and difficulty, analysis concentrated on the high-high leaves, and results expressed as risks, non-risks, sensitivity points, trade-off points and risk themes. Around it I use lighter tools by purpose — a pre-mortem against optimism, risk storming on the diagram, ARID for interfaces, STRIDE for security, the Azure Well-Architected Review and pillar checklists for Azure gaps, a production readiness review before launch and a game day to turn the failure-mode table into evidence."*

---

## Concept 33 — Decision rights and closing the review

A review without a clear decision process doesn't end; it decays. Before a review starts, everyone should know **who decides, how, and what happens when people disagree.**

**Decision-rights frameworks** you'll meet:

| Framework | Roles | Notes |
|---|---|---|
| **DACI** (popularized by Atlassian) | **D**river (runs the process — usually the author), **A**pprover (one person who decides), **C**ontributors (consulted), **I**nformed | Simple; maps naturally onto a design doc header |
| **RAPID** (Bain) | **R**ecommend, **A**gree (veto holders, e.g., security or legal), **P**erform, **I**nput, **D**ecide | Makes veto rights explicit — useful when compliance or security can block |
| **RACI** | Responsible, Accountable, Consulted, Informed | Better for task ownership than for decisions |

**Decision rules** — how the group reaches the decision:

| Rule | Meaning | When it fits |
|---|---|---|
| **Autocratic / single decider** | The approver decides after input | Most design decisions — fast, clear accountability |
| **Consultative** | Decider must seek and consider input from named parties | The default for design docs |
| **Consent** | Proceed unless someone has a *reasoned, paramount* objection ("is it safe enough to try?") | Reversible decisions; sociocracy-style teams |
| **Consensus** | Everyone agrees | Rarely appropriate — slow, favors the status quo, gives everyone a veto |
| **Rough consensus** | IETF's tradition: not unanimity, but all objections addressed (not necessarily accommodated) | Standards and open-source bodies |
| **Final comment period** | Last call with a stated disposition; silence = consent | Rust RFCs; many PR-based processes |

**Closing practices:**

1. **Record the decision in the doc** — status, decision, conditions, date, decider — and extract the durable decisions into **ADRs** (Module 31).
2. **"Disagree and commit."** Bezos's 2016 letter popularized the phrase (the idea is older; it's usually associated with Intel under Andy Grove): once a decision is made, people who disagreed commit fully to making it work. It only works if (a) dissent was genuinely heard and (b) the disagreement is *recorded*, so that if the risk they predicted materializes, the organization learns rather than blames. Record dissent in the doc: "Payments team prefers Event Grid (see thread); decided against because of ordering requirements; revisit if…".
3. **Timebox indecision.** If the decider can't decide, ask what information would let them, get it (often a spike), and set a new date.
4. **Escalate cleanly.** When peers can't agree, write a **joint escalation**: a short note *both* sides agree describes the disagreement fairly, the options and each side's reasoning — then the next level decides. Escalating together, not separately, keeps relationships intact and decisions fast.
5. **Decide what "approved" commits to.** Approval usually covers the direction and the one-way-door elements; details can still change in implementation, with significant deviations coming back as a doc update.
6. **Oxide's example**: its RFD 113 ("Engineering Determination") describes how RFDs are closed out — the point being that *how a discussion ends* deserves as much process design as how it begins.

**The interview-grade sentence:** *"Before review starts, I make decision rights explicit — usually DACI, with me as driver and one approver, or RAPID when security or compliance hold a veto — and the decision rule, which for design docs is typically consultative: the approver decides after named parties are heard, not consensus, which hands everyone a veto. Then I close deliberately: record the decision, conditions and any dissent in the doc and in ADRs, ask dissenters to disagree and commit with their concern on record, timebox indecision with a spike, and escalate peer deadlocks jointly with a description both sides agree is fair."*

---
# Part E — Defending the design

## Concept 34 — The defense mindset

"Defending" a design doc sounds adversarial. The best architects treat it as **collaborative stress-testing of an argument**, in which the author has a specific job: help the group reach the best decision, which will often — but not always — be the one in the doc.

Five principles:

1. **Defend the reasoning, not the design.** You are responsible for making the argument (Concept 5) as strong and as honest as it can be. If a better argument appears, the design should change. Authors who defend the *design* treat every objection as a threat; authors who defend the *reasoning* treat every objection as data.
2. **Separate identity from proposal.** A rejected design is not a rejected engineer. Senior people have many rejected docs behind them — often the most instructive ones. In interviews, "tell me about a design of yours that was rejected" is a common question precisely because the answer reveals this separation.
3. **Assume good faith and shared goals.** Most objections come from someone who sees a risk you don't, holds a different priority, or misunderstood something. All three are useful to know. Treating a reviewer as an opponent makes even good objections harder to hear.
4. **Be strongly reasoned, not strongly attached.** "Strong opinions, weakly held" (Paul Saffo's formulation) is often quoted; the critique of it is fair — it can license loud overconfidence. The better version: *state your confidence honestly and update in proportion to evidence.* Being confidently wrong and refusing to change is worse than being tentatively right.
5. **The goal is a decision that sticks.** Winning an argument by exhausting the room produces a decision people undermine later. Losing a point gracefully while getting the important things right produces a decision people execute.

What a strong defense *looks like* from the reviewer's side: the author listens fully, restates objections accurately, answers with evidence and reasoning, concedes precisely when wrong, says "I don't know — here's how we'd find out" when they don't know, and leaves the room with the doc better than it entered.

What a weak defense looks like: interrupting, answering a different question, appeals to authority ("this is how Netflix does it"), escalating detail to overwhelm, dismissing concerns as out of scope without a non-goal to point to, or conceding everything to end the discomfort.

**The interview-grade sentence:** *"I treat defending a design as collaborative stress-testing of an argument: I defend the reasoning rather than the design, keep my identity separate from the proposal, assume reviewers share the goal and see something I don't, state my confidence honestly and update in proportion to evidence rather than holding strong opinions loosely, and aim for a decision that sticks — conceding precisely when I'm wrong, because winning by exhausting the room produces decisions that people quietly undermine."*

---

## Concept 35 — Anticipating objections

The best defense is the objection already answered in the doc. Before review, **red-team your own design** systematically.

**The standard challenge list** — nearly every design review hits most of these. Prepare an answer (in the doc or FAQ) for each:

| # | Challenge | What the reviewer is really asking | Where the answer lives |
|---|---|---|---|
| 1 | "Why not just ___?" (the simpler option) | Is the complexity justified? | Alternatives (minimal change, Concept 14) |
| 2 | "Why not ___?" (their favorite technology) | Did you consider my option fairly? | Alternatives; FAQ |
| 3 | "Does this scale?" / "What about 10×?" | Is the capacity model real; what breaks first? | Estimation with sensitivity (Concept 12) |
| 4 | "What happens when ___ fails?" | Is failure designed or hoped for? | Failure-mode table (Concept 15) |
| 5 | "Isn't this over-engineered?" | Are we solving problems we don't have? | Goals, non-goals, evidence for each complexity |
| 6 | "Isn't this under-engineered?" | Will we rebuild it in a year? | Headroom, extension seams, revisit triggers |
| 7 | "Who runs this at 3 a.m.?" | Operability and ownership | Operability, observability, runbooks |
| 8 | "How do we roll this back?" | Reversibility | Reversibility table (Concept 16) |
| 9 | "What does it cost?" | Build and run cost, and its curve | Cost section (Concept 18, Module 33) |
| 10 | "Is it secure?" / "Where's the PII?" | Threat model, data handling | Security and privacy sections (Module 29) |
| 11 | "Can the team build this?" | Skills, timeline, risk | Plan, staffing, delivery risks |
| 12 | "Why now?" / "Why us?" | Priority versus other work | Problem statement, cost of doing nothing |
| 13 | "What did you assume?" | Hidden assumptions | Assumptions table with validation |
| 14 | "How will we know it worked?" | Success measures | Goals with metrics; what you'll monitor |
| 15 | "What would make you abandon this?" | Falsifiability; intellectual honesty | Revisit triggers, kill criteria (Concept 37) |

**Red-teaming techniques:**

1. **The hostile-reader pass.** Re-read the doc as each stakeholder in Concept 4, most skeptical first. Write their top three questions.
2. **The pre-mortem** (Concept 32), alone or with a teammate: "It's a year later, and this failed. Why?" — list five reasons, then check the doc addresses the plausible ones.
3. **The "what would a principal engineer from team X say?" exercise** — name the specific person whose priorities differ most from yours and argue their case.
4. **The cheaper-option challenge.** Force yourself to write the best possible version of the minimal alternative. If it's nearly as good, reconsider.
5. **The 10× and 0.1× check** — does the design make sense if load is ten times higher? If it's ten times *lower*, is it absurd overkill?
6. **The "explain it to on-call" test** — can you explain how to diagnose the most likely failure in two minutes?
7. **AI-assisted adversarial review** (Concept 27) — useful for generating candidate objections; you decide which are real.

**Answer in the doc, not just in your head.** Every objection you anticipate and address in writing is one fewer argument in the meeting — and evidence of thoroughness to the reviewer who raises it.

**The interview-grade sentence:** *"I red-team my own doc before anyone else reads it: I walk the standard challenge list — why not the simpler option, why not your favorite technology, what at ten times the load, what when each dependency fails, over- or under-engineered, who runs it at 3 a.m., how we roll back, what it costs, is it secure, can the team build it, why now, what did we assume, how we'll know it worked, and what would make us abandon it — run a pre-mortem, argue the case of the stakeholder whose priorities differ most from mine, and write the answers into the doc or its FAQ so the meeting can spend its time on the questions I couldn't predict."*

---

## Concept 36 — Responding to challenges in the moment

Even a well-prepared doc meets unexpected objections. A response protocol keeps you effective under pressure — in a review and especially in an interview panel.

**The protocol:**

1. **Listen to the end.** Don't start answering the objection you expect; answer the one being asked. Interrupting signals defensiveness.
2. **Restate it.** "So the concern is that if the relay is down for an hour, the outbox table grows enough to hurt checkout latency — is that right?" Restating confirms understanding, often reveals a misunderstanding on either side, and buys you a few seconds to think.
3. **Classify it** — silently — using the argument model (Concept 5) and the disagreement types (Concept 38):

| The challenge is… | Tell-tale | Response |
|---|---|---|
| **A misunderstanding** | Based on something the doc doesn't say | Clarify; fix the doc so the next reader doesn't misunderstand |
| **A gap you covered** | The answer is in the doc | Point to it briefly; don't lecture |
| **A gap you missed** | Valid and new | Acknowledge it plainly; assess impact live; propose a mitigation or a follow-up |
| **A factual disagreement** | "That won't handle 2,000/s" | Ask for or offer evidence; propose a test if neither side has data |
| **A prediction disagreement** | "This will be a maintenance burden" | State your reasoning, propose an observable early indicator or a spike |
| **A value / priority disagreement** | "Consistency matters more than availability here" | Name it as a priority question; point to the goal ranking; if unresolved, it's the decider's call |
| **Out of scope** | "What about SMS?" | Point to the non-goal and the extension seam; offer to track it |

4. **Answer with reasoning and evidence, at the right altitude.** Lead with the conclusion, then the *one* strongest reason, then offer depth. "No — at peak the outbox grows about 90,000 rows an hour; the purge job and the `ProcessedAt` index keep the active set small, and we tested 2 million rows with no change to checkout p99. Happy to show the numbers."
5. **Concede precisely.** If they're right, say exactly what you're conceding and what changes: "You're right that we don't handle relay downtime longer than a day — I'll add an alert on outbox age and a capacity bound. The rest of the design doesn't change." Precise concessions build credibility; vague ones ("good point, we'll look into it") make reviewers wonder what else is shaky.
6. **"I don't know" — then how you'd find out.** "I don't know the provider's rate limit under burst. I'll confirm with them by Thursday; if it's below 500/s, we'll add a client-side token bucket and the design otherwise holds." This is far stronger than bluffing — and in an interview, bluffing is detected quickly and penalized heavily.
7. **Park what can't be resolved in the time.** "That's a real question about cross-region ordering that deserves more than five minutes. Can I take it as an open question with you and bring a proposal to the next review?"
8. **Check closure.** "Does that address the concern?" If not, you misunderstood it or they're not convinced — both worth knowing.

**Phrases that work, and ones that don't:**

| Instead of… | Try… |
|---|---|
| "That's not a problem." | "Here's why I think that case is covered — tell me if I'm missing something." |
| "We already thought of that." | "That's in §6.3 — the short version is…" |
| "That's out of scope." | "We made that a non-goal for this phase because…; the design leaves a seam for it at…" |
| "Netflix does it this way." | "This pattern works for workloads like ours because…" |
| "Good point, we'll look into it." | "You're right about X. I'll change Y by Friday; Z is unaffected." |
| "Trust me." | "Here's the measurement / here's how we'll measure it." |

**The interview-grade sentence:** *"When I'm challenged I listen to the end, restate the concern to confirm it, and classify it — misunderstanding, gap already covered, gap I missed, factual or predictive disagreement, value disagreement, or out of scope — because each needs a different answer. Then I lead with the conclusion and the single strongest reason with evidence, concede precisely what I got wrong and what changes, say 'I don't know — here's how I'll find out by when' rather than bluffing, park what can't be resolved in the time, and check that the concern is actually addressed."*

---

## Concept 37 — Changing your mind well

Updating a design in response to review is not losing; it's the process working. How you change your mind affects your credibility as much as whether you do.

**State in advance what would change your mind.** Strong docs contain **revisit triggers** or **kill criteria**: "We would choose Event Hubs instead if sustained volume exceeds 10,000 events/s or we need replay of history beyond 14 days." "We'll abandon the self-hosted approach if the spike can't reach 1,000 req/s on two nodes." This does three things: it makes the argument falsifiable (Concept 5's rebuttal), it shows reviewers you've thought about the boundary of your claim, and it turns future arguments into checks.

**Update proportionally.** One anecdote shouldn't flip a well-evidenced decision; a measured result contradicting your key assumption should. Say which kind of evidence you're responding to.

**Update visibly.** Maintain a changelog in the doc once review starts:

```markdown
## Changelog
- v0.5 (2026-10-14): Consumer moved from Functions to Container Apps after review (thread #12):
  the provider's rate limit needs a shared limiter across a bounded set of instances, simpler
  with fixed replica bounds plus a KEDA queue-length scale rule. §7.2 updated; cost +€40/month.
- v0.4 (2026-10-10): Added outbox-age alert and 24-hour capacity bound (R5) after SRE review.
- v0.3 (2026-10-07): Opened for review.
```

Reviewers can then see exactly what changed and why, and they trust that their input was used.

**Credit the reviewer.** "Thanks to Marko's point about the provider's rate limit, the consumer now runs on Container Apps with bounded replicas." Crediting costs nothing and builds the culture you want reviewers to keep participating in.

**Know the difference between updating and capitulating.** Changing the design because the evidence or the goals require it is updating. Changing it because a senior person pushed hard, without a new reason, is capitulating — and it produces worse designs and less respect. If you're overruled on a value question by the decider, that's legitimate: record it, disagree and commit (Concept 33).

**When the design dies,** write it up anyway: mark the doc *Rejected* or *Withdrawn* with the reason. Rejected designs are valuable organizational memory — they stop the next person proposing the same idea without the context, and they're excellent interview stories.

**The interview-grade sentence:** *"I change my mind well by deciding in advance what would change it — revisit triggers and kill criteria written into the doc, like 'we'd switch to Event Hubs above 10,000 events a second or if we need replay beyond 14 days' — updating in proportion to the strength of the evidence, keeping a changelog that shows what changed, why and who raised it, and distinguishing updating from capitulating: a new reason changes the design, pressure alone doesn't. And when a design dies I mark it rejected with the reason, because rejected docs are some of the most useful memory an organization has."*

---

## Concept 38 — Kinds of disagreement, and how to resolve each

Many design arguments go in circles because the participants are having *different kinds* of disagreement without realizing it. Diagnosing the kind tells you the resolution tool.

| Kind | Example | Resolution tool | Who settles it |
|---|---|---|---|
| **Understanding** | "Wait, I thought the relay ran inside the API." | Clarify; improve the doc | The author |
| **Facts (current)** | "Our peak is 300/s, not 500/s." | Look at the data | Telemetry |
| **Predictions (future)** | "This will need a rewrite at 10×." / "The team will struggle to operate Kafka." | Reason about mechanisms; find leading indicators; run a spike; choose the more reversible option | Evidence, or the decider under uncertainty |
| **Values / priorities** | "Consistency matters more than latency for this feature." | Refer to the goal ranking, principles, product owner | The decider (or whoever owns the goal) |
| **Risk appetite** | "A 1-in-1,000 duplicate notification is unacceptable." vs "It's fine." | Quantify the impact; compare to the cost of prevention; check policy | The risk owner |
| **Scope** | "This should also handle SMS." | Non-goals; separate proposal | Product / decider |
| **Ownership / incentives** | "My team doesn't want to be on call for this." | Make the cost explicit; negotiate support model; escalate | Management |
| **Taste** | "I'd name it differently" / "I prefer minimal APIs" | Defer to the owner's choice or a team convention | The owning team |

Patterns:

- **Value disagreements disguised as technical ones** are the most common cause of endless reviews. Two engineers arguing about whether to use Cosmos DB's strong or session consistency (Module 7) may actually disagree about how much a rare stale read matters to users. Surface it: *"I think we agree on the mechanics; we disagree on how bad a five-second stale read is. That's a product question — can we ask?"*
- **Prediction disagreements** are often best resolved by **choosing the more reversible path** or by **buying information**: a spike, a load test, a small production experiment. "We don't agree whether the relay will keep up; let's spend two days measuring it rather than two weeks debating it."
- **Ownership disagreements** must not be resolved by stealth. If a team doesn't want to operate a component, a design that assumes they will is a design that fails in production. Make it explicit and escalate.
- **Taste** should be settled quickly and without ego — usually in favor of whoever owns the code.

**When to escalate:** after the kind is identified and the right tool has been tried, if peers still disagree on a value, risk or ownership question, escalate *jointly* (Concept 33). Escalation is not failure; it's what decision rights are for.

**The interview-grade sentence:** *"I diagnose which kind of disagreement we're having before trying to resolve it: misunderstandings get clarified, factual ones get settled by data, predictions by mechanisms, leading indicators, spikes or picking the more reversible option, value and risk-appetite disagreements by the goal ranking or the person who owns that goal or risk, scope by non-goals, ownership by explicit negotiation, and taste by the owning team. The most common trap is a value disagreement disguised as a technical one — two people arguing consistency levels when they really disagree about how bad a stale read is for users — and surfacing that usually ends the circle."*

---

## Concept 39 — Defending to different stakeholders

The same design is defended differently to different audiences — not by changing the facts, but by leading with what each one decides on.

| Stakeholder | What they decide on | Lead with | Avoid | Example framing |
|---|---|---|---|---|
| **Executive / VP** | Value, cost, risk, time, options | The business outcome, the cost, the main risk, and 2–3 options with a recommendation | Technical detail they can't evaluate; a single option ("approve or reject") | "This removes the cause of ~11,000 failed checkouts a quarter for 6 engineer-weeks and €300/month. The cheaper fix leaves a crash-loss risk; the SaaS option costs 6× more. I recommend the middle option." |
| **Principal / senior engineers** | Technical soundness, long-term consequences, consistency with the platform | Trade-offs, failure modes, evidence, alternatives | Hand-waving; appeals to authority | "The trade-off point is ordering versus throughput; here's why per-order ordering is enough." |
| **Implementing team** | Feasibility, clarity, workload, operability | Scope, contracts, plan, what's hard, what they'll own | Over-specifying how they code it; hidden work | "Here's the contract and the failure rules; how you structure the service is yours." |
| **SRE / operations** | On-call burden, observability, rollback | SLOs, alerts, dashboards, runbooks, rollback, game day | "We'll add monitoring later" | "Here's what pages, why, and the runbook for each alert." |
| **Security / compliance** | Risk and regulatory exposure | Threat model, data flows, identities, data classification | "It's internal, so it's fine" | "No new secrets — managed identity with one data-plane role; PII redacted from telemetry." |
| **Product** | User value, timeline, scope | What users get when; what's cut; trade-offs in user terms | Engineering jargon | "Emails arrive within seconds normally and within minutes during provider outages — checkout never fails." |
| **Finance / FinOps** | Cost and its trajectory | Run-rate, drivers, scaling curve, guardrails (Module 33) | Single-point costs with no drivers | "€300/month at today's volume, roughly linear with orders, ceiling €1,000 when we need Premium." |
| **Dependent teams** | Impact on their roadmap | What changes for them, when, with what support | Surprises | "Your webhook contract is unchanged; the new event is opt-in." |

Two senior skills:

1. **Translation without distortion.** "Eventually consistent, typically under 5 seconds" becomes "emails arrive within seconds" for product — but *not* "emails arrive instantly." Simplifying is fine; changing the claim isn't.
2. **Options, not ultimatums, for decision-makers.** Executives and principals both respond better to "here are three options with their costs and risks; I recommend B" than to "approve this." It respects their decision rights and shows you understand the trade-off space. Always include the cheap option and say honestly what it doesn't buy.

**The interview-grade sentence:** *"I defend the same facts differently to each stakeholder by leading with what they decide on: business outcome, cost, risk and options for executives; trade-offs, failure modes and evidence for principal engineers; contracts, scope and ownership for the implementing team; SLOs, alerts and rollback for SRE; the threat model and data flows for security; user-visible behavior and timeline for product; run-rate and its drivers for finance. I simplify without changing the claim, and I give decision-makers options with a recommendation rather than an ultimatum."*

---

## Concept 40 — The document's afterlife

Approval is the midpoint of a design doc's life, not the end.

**1. Implementation drift.** Implementation always discovers things. The rule: **significant deviations update the doc** (or produce a new ADR) — anything that changes a contract, a data model, a failure behavior, a security property, a cost assumption or a non-goal. Minor deviations don't need ceremony. A doc that silently diverges from reality is worse than no doc, because it misleads.

**2. Freeze, then record as-built.** Many organizations freeze the design doc after approval (its job — the decision — is done) and add a short **"as-built" note** after implementation: what shipped, what changed and why, links to the ADRs, the code and the runbooks. The living description of the system belongs in the architecture documentation (arc42, C4 — Module 31), not in an ever-growing design doc.

**3. Extract the durable decisions into ADRs.** The doc may contain ten decisions; the three that future maintainers will ask about ("why outbox?", "why Service Bus over Event Hubs?", "why a separate service?") become ADRs with context and consequences. The Azure WAF architect guidance treats the ADR log as append-only, recording each decision's confidence level and superseding rather than editing old records.

**4. Supersede, don't delete.** When a later design replaces this one, mark it *Superseded by RFC-0187* and link both ways. Deleted docs leave unexplained decisions behind.

**5. Close the loop with a retrospective.** A few weeks or months after launch, compare the doc's predictions with reality:

| Prediction in the doc | Reality | Lesson |
|---|---|---|
| p95 notification latency < 5 s | 2.1 s | Estimate conservative; fine |
| 6 engineer-weeks | 8.5 | Provider integration underestimated — again; add buffer for third-party integrations |
| Run cost €300/month | €410 | Telemetry ingestion higher than estimated; add sampling (Module 28) |
| Zero checkout failures from notifications | 0 since launch | Goal met |
| Risk R2 (team inexperience) | Two config incidents in week 1 | Game day caught one; the other was DLQ alerting threshold |

This is how an organization — and an architect — gets better at estimating, at spotting risks and at writing docs. It's also the raw material for the strongest behavioral interview stories (Modules 34–35): *"Here's what I predicted, here's what happened, here's what I changed in how I design."*

**6. Make it findable.** The doc's number should appear in code comments at the important seams, in the service README, in runbooks and in incident reports. A design doc that nobody can find when the system misbehaves has lost most of its value.

**The interview-grade sentence:** *"After approval I keep the doc honest: significant deviations — contracts, data, failure behavior, security, cost or scope — come back as doc updates or new ADRs; the doc is then frozen with an as-built note linking ADRs, code and runbooks, while the living system description moves to the architecture documentation; durable decisions become append-only ADRs that are superseded, never deleted; and a retrospective compares the doc's predictions — latency, effort, cost, risks — with reality, which is how I improve my own estimates and where my best interview stories come from."*

---
# Part F — The design-document interview

## Concept 41 — The design-document round: formats and what's scored

Architect loops (and many staff loops) contain a round built around a design document. Five common formats — ask your recruiter which one you'll get, because preparation differs:

| Format | How it runs | What's really scored | Preparation |
|---|---|---|---|
| **A. Take-home + panel defense** | You get a prompt (often brownfield, often vague) and a few days; submit a doc; 60–90 min panel challenges it | Problem framing, judgment on scope, quality of alternatives and trade-offs, honesty about risks, defense under pressure | Concept 44; practice exercise 1 |
| **B. Past design deep dive** | "Bring a design you led" — present for 10–20 min, then questions for 40+ | Ownership, decision quality in a real context, what you learned, ability to criticize your own work | Concept 42; prepare two designs |
| **C. Live critique** | They hand you a design doc (often deliberately flawed); you review it in 30–45 min, then discuss | Reviewer skill: prioritization, depth, tact, constructive alternatives | Concept 43; Worked example 4 |
| **D. Live design, doc-shaped** | A system design round where you're expected to structure your answer like a doc (requirements → options → decision → risks) | Same as Module 3's rubric, plus explicit trade-offs and alternatives | Module 3 framework narrated as a doc |
| **E. Stakeholder role-play** | Interviewers play a skeptical VP, a product manager or an opposing principal | Influence, translation, handling disagreement, options framing | Concepts 36, 38, 39; Worked example 3 |

**What the rubric typically looks for** (the shape of most architect rubrics, consistent with Module 1):

| Dimension | Below bar | At bar | Above bar |
|---|---|---|---|
| **Problem framing** | Jumps to solution | States problem, goals, non-goals | Reframes the problem; finds the real constraint; questions the prompt's assumptions |
| **Alternatives & trade-offs** | One option | Several options, pros and cons | Options compared on goal-derived criteria; trade-off points named; recommends the simpler option when it's enough |
| **Technical depth** | Boxes without mechanisms | Correct mechanisms for main flows | Failure modes, consistency, capacity and cost reasoned with numbers |
| **Pragmatism** | Ideal-world design | Accounts for team, time, cost | Phased plan with reversibility and cut lines; brownfield-aware |
| **Risk honesty** | No risks | Lists risks | Ranks risks, names its own design's weakest point, states kill criteria |
| **Communication** | Disorganized | Clear structure | Layered for audiences; concise; diagrams that earn their place |
| **Defense** | Defensive or capitulates | Answers questions | Restates, classifies, uses evidence, concedes precisely, updates the design live |
| **Influence** | Ultimatums | Explains reasoning | Frames options for decision-makers; finds common ground; knows when to escalate |

A recurring theme in hiring guidance for architects: **ask candidates for a design doc they wrote, then focus the conversation on what they got wrong and would change.** Candidates who can't criticize their own past design signal that they won't adapt to the new context. Prepare that self-critique deliberately.

**The interview-grade sentence:** *"Design-document rounds come in five shapes — a take-home doc defended to a panel, a deep dive on a design I led, a live critique of a doc they give me, a doc-structured live design and a stakeholder role-play — and they all score the same things: framing the real problem, comparing genuine alternatives on criteria derived from the goals, technical depth on failure, consistency, capacity and cost, pragmatism about team, time and reversibility, honesty about the design's weakest point, clear layered communication, and a defense that restates, uses evidence, concedes precisely and updates the design."*

---

## Concept 42 — Presenting a past design

For format B, prepare **two** designs: one that went well and one that went badly or was rejected. For each, a 10-minute presentation with this structure:

| Minute | Section | Content |
|---|---|---|
| 0–1 | **Context** | The business, the system, the scale (anonymized), your role — what *you* owned versus the team |
| 1–2 | **Problem** | What was wrong, with evidence; why it mattered then |
| 2–3 | **Constraints and goals** | The two or three constraints that shaped everything; goals and non-goals |
| 3–5 | **Options** | Two or three alternatives, how you compared them, and the deciding trade-off |
| 5–7 | **The design** | One diagram; the key mechanisms; the riskiest part and how you de-risked it |
| 7–8 | **Getting to a decision** | Who disagreed, about what kind of thing, and how it was resolved |
| 8–9 | **Outcome** | What shipped, with numbers; predictions versus reality |
| 9–10 | **What I'd change** | Two or three specific things, with reasons — and what you now do differently in every design |

Guidance:

1. **Anonymize carefully.** Many candidates can't (or prefer not to) name past employers or reveal confidential details. Describe the domain and scale in relative terms ("a B2B SaaS platform in logistics, ~2,000 tenants, peak ~3,000 requests per second"). Interviewers care about the reasoning, not the logo — and handling confidentiality carefully is itself a professional signal.
2. **Be precise about your role.** "I wrote the doc and led the review; the team implemented; I owned the migration plan." Inflated ownership is easy to detect in follow-up questions.
3. **Bring the numbers.** Scale, latency, cost, effort, incident counts, before and after. Approximate is fine; absent is not.
4. **Show the trade-off you'd make differently now.** "We chose a shared database for speed of delivery; it became our biggest coupling problem eighteen months later. Today I'd accept the extra two weeks to give each module its own schema" (Module 21). Self-critique that names the *mechanism* of the mistake is a strong signal.
5. **For the failed or rejected design,** be generous to the people who rejected it and specific about what you learned: "They were right that our team couldn't operate Kafka at the time; I'd framed it as a technology decision when it was an ownership decision."
6. **Prepare for the drill-down.** Expect "why not X?" on every option you rejected, "what happened when Y failed?", "what would you do at 10×?", "who disagreed and why?". Rehearse with someone who will push.

**The interview-grade sentence:** *"When I present a past design I take ten minutes: anonymized context and my exact role, the problem with evidence, the two or three shaping constraints, the options and the deciding trade-off, one diagram with the riskiest mechanism, how disagreement was resolved, the outcome with numbers and predictions versus reality, and two or three things I'd change, explained by mechanism. I prepare one design that went well and one that failed or was rejected, because the self-critique is what interviewers are really listening for."*

---

## Concept 43 — Critiquing a design doc live

For format C, use a **systematic pass** so you don't spend 30 minutes on the first section you find interesting.

**Step 1 — Skim for shape (3–5 minutes).** Read the summary, goals and non-goals, headings and diagrams. Ask: what decision is requested? Is the problem stated? Are there alternatives? Is there a rollout and a risk section? Missing sections are findings.

**Step 2 — The problem pass (5 minutes).** Is the problem evidenced? Do the goals match the problem? Are non-goals sensible? Is "do nothing" quantified?

**Step 3 — The decision pass (10 minutes).** Are alternatives real, including the minimal option? Are criteria derived from goals? Does the proposal meet its own goals — check each one? What's the riskiest assumption, and is it validated?

**Step 4 — The detail pass (10–15 minutes),** guided by Appendix B's checklist, spending time where the design is riskiest:
- consistency and the dual-write problem; idempotency and ordering;
- failure modes for each dependency; retries, timeouts, backpressure;
- capacity math, if any, and the sensitive inputs;
- security boundaries, identities, secrets, PII;
- data model and migration; rollback and reversibility;
- operability, observability, ownership;
- cost.

**Step 5 — Prioritize and present (5–10 minutes).** Interviewers want judgment, not a list of 25 nits. Present:

1. **A one-sentence overall verdict.** "The direction is sound, but I wouldn't approve it yet: two blocking issues, three important ones."
2. **Blocking issues** (correctness, data loss, security, irreversible decisions) — each with *why it matters* and *what would fix it* ("yes, if").
3. **Important but non-blocking issues.**
4. **Questions** you'd ask the author.
5. **What's good** — specific; it shows you can recognize quality and are not just a critic.
6. **Smaller notes** — mentioned briefly, or offered in writing.

Signals that impress:

- **Finding the unstated assumption** ("this assumes the provider is idempotent — is it?").
- **Checking the doc against its own goals** ("Goal 2 says no lost notifications; §6 uses an in-memory queue").
- **Spotting the missing alternative** ("the minimal option — moving the call after commit with a durable retry — isn't considered and might be enough").
- **Thinking about the humans** ("who's on call for this? The plan adds a service to a team with no messaging experience").
- **Tact** — critiquing the design, never the author, and framing fixes as conditions.

**The interview-grade sentence:** *"When I critique a design doc live I work in passes so I don't sink into the first interesting section: skim for shape and missing sections, check that the problem is evidenced and the goals match it, check that the alternatives are real and that the proposal meets its own goals, then go deep where the design is riskiest — consistency, failure modes, capacity, security, migration and rollback, operability and cost. I present a one-line verdict, blocking issues each with why it matters and what would fix it, then important issues, questions and specific strengths — critiquing the design, never the author."*

---

## Concept 44 — The take-home design doc

Format A is the closest thing to real architect work in an interview loop, and the most common way to fail it is to write the wrong *kind* of document.

**Budget your time** (for a typical "a few days, aim for about 4–6 hours" take-home — respect any stated limit; over-investing is noticed and isn't always rewarded):

| Share | Activity |
|---|---|
| 15% | Read the prompt closely; list ambiguities; decide your assumptions (and write them down) |
| 15% | Problem framing: goals, non-goals, quality scenarios, constraints |
| 15% | Estimation and options; choose |
| 25% | Design, failure modes, cross-cutting concerns |
| 15% | Rollout/migration, risks, plan, cost |
| 15% | Summary, cut, polish, diagrams; re-read as the panel |

**Structure** — the anatomy from Part B, compressed. Usually 4–8 pages plus a short appendix. The panel will read it in 15–30 minutes; length is a cost to them.

**What graders look for, in order:**

1. **Did you solve the problem they asked — or the one you wanted to?** Address the prompt's actual constraints (often hidden in one sentence: "the team is five engineers," "the existing system is a monolith on SQL Server," "budget is tight").
2. **Assumptions stated** where the prompt is ambiguous — and, where possible, a note on how the design changes if an assumption is wrong. Prompts are often deliberately vague to see whether you notice.
3. **Real alternatives**, including the simplest viable one.
4. **A design proportional to the problem.** The most common failure is a microservices-Kafka-Kubernetes design for a problem a modular monolith and a queue would solve (Modules 21 and 26). Over-engineering reads as poor judgment, not as skill.
5. **Brownfield awareness** if there's an existing system: migration path, coexistence, rollback (Module 32).
6. **Honest risks and the weakest point** of your own design, named by you before they find it.
7. **Clarity**: summary first, diagrams with legends, numbers.

**Prepare for the defense:** re-read your doc the night before as the panel; list the ten hardest questions (Concept 35); decide in advance what you'd concede. Expect the panel to change a constraint mid-defense ("now assume 50× traffic" or "the team just lost two engineers") to watch you adapt — treat it as a new qualifier on your argument: say what holds, what changes and what you'd do first.

**Common take-home traps:**

| Trap | Better |
|---|---|
| Technology catalogue ("we'll use Kafka, Redis, Cosmos, AKS…") | Each technology justified by a requirement; fewer moving parts |
| No numbers | Back-of-envelope estimate with sourced inputs |
| One option | Two or three, including the minimal one |
| Ignores the existing system | Migration plan and coexistence |
| Ignores team and budget constraints | Design that a team of the stated size can build and run |
| 20 pages | 4–8 pages, appendix for depth |
| Generated-looking prose with no specific claims | Specific, checkable claims you can defend line by line |
| No risks | A risk section that names your design's weakest point |

**The interview-grade sentence:** *"For a take-home design doc I spend as much time framing the problem as designing — reading the prompt for the hidden constraint, stating assumptions where it's vague and how the design would change if they're wrong — then compare real options including the simplest viable one, size the design to the problem and the team rather than to my résumé, handle the existing system's migration and rollback, name my design's weakest point before the panel does, and keep it to four to eight readable pages. In the defense I treat a changed constraint as a new qualifier: what still holds, what changes and what I'd do first."*

---
# Worked examples

## Worked example 1 — A complete Tier 2 design doc (condensed): event-driven order notifications

This is the running example from Part B assembled into one document, condensed to about the length a real Tier 2 doc would have. Notice how short the design section is relative to the reasoning around it.

---

> **RFC-0142: Event-driven notification service for Orders**
>
> | Status | Approver | Required reviewers | Tier | Decision due |
> |---|---|---|---|---|
> | In review (v0.5) | Principal engineer, Commerce | Payments, SRE, Security, FinOps | 2 | 2026-10-21 |
>
> ### 1. Summary
> Order notifications (confirmation, shipped, delivered) are sent by SMTP **inside the checkout transaction**. Provider timeouts caused **0.8% of checkouts to fail in Q3 2026 (≈ 11,000 orders)** and three Sev-2 incidents. We propose writing an `OrderConfirmedV1` integration event to a **transactional outbox**, relaying it to an **Azure Service Bus** topic, and sending notifications from a new **Notification service** on **Azure Container Apps**. This removes notifications from the checkout path entirely, at the cost of **eventual delivery (p95 ≤ 5 s normally; minutes during provider outages)** and **one new service** to operate. Estimate: **6 engineer-weeks (5–9)**, **≈ €300/month** run cost at the design point. Rollout behind a feature flag over three weeks with instant rollback. **Decision requested:** approve the event-driven approach (§6) and the outbox over direct publishing (§7.2) by 2026-10-21.
>
> ### 2. Context and problem
> Today `CheckoutHandler` (Orders API, ASP.NET Core on App Service) commits the order, then calls the email provider synchronously before returning; a provider timeout throws and the request fails, and after client retries some customers receive duplicate confirmation emails or none. Evidence: Application Insights failure attribution (Q3), incidents INC-2207/2241/2270, 420 support tickets. Q2 2027 forecast volume roughly doubles this cost if nothing changes.
>
> ### 3. Goals (in priority order) and non-goals
> **G1** No checkout fails because of notification delivery. **G2** No notification is lost (every confirmed order produces exactly one confirmation email, or an alert). **G3** p95 delivery ≤ 5 s in normal operation. **G4** A new channel can be added without touching the Orders API.
> **Non-goals:** new channels in this phase (seam only, §6.4); marketing email; multi-region active-active (RFC-0139); changing email templates.
>
> ### 4. Quality scenarios (top three)
> - **Provider outage** — provider down 30 min at peak → checkout unaffected; all notifications delivered ≤ 10 min after recovery; alert ≤ 5 min.
> - **Crash mid-flight** — relay or consumer killed during a deploy → no notification lost; duplicates suppressed (≤ 1 email per event).
> - **Flash-sale burst** — ×20 burst for 15 min → no checkout impact; p95 delivery ≤ 30 s during the burst.
>
> ### 5. Constraints, assumptions, estimate
> Constraints: Azure; SQL Server orders DB unchanged except the outbox table; .NET 10; team of four, SRE on-call shared; ship before 2026-11-15 freeze. Assumptions (with validation in §10): provider supports idempotency keys; peak ≤ 2,000/s through 2027. Estimate: design point 500 notifications/s, ceiling 2,000/s; ~200 concurrent sends at 500/s (Little's Law, p95 provider latency 400 ms); outbox ≈ 10 GB retained for 7 days.
>
> ### 6. Proposed design
> *(Container diagram and sequence diagram as in Concept 22.)*
> 1. **Outbox write.** In the same EF Core transaction as the order, insert an `OutboxMessage` row containing `OrderConfirmedV1` (Module 11). No direct publish from the request.
> 2. **Relay.** A background service (two replicas, lease-based) polls unprocessed rows, publishes to the `orders-events` topic with `MessageId = EventId` (duplicate detection enabled, 10-minute window), and marks rows processed.
> 3. **Notification service.** Container Apps, KEDA scale rule on subscription backlog, bounded at 8 replicas; Service Bus processor with `MaxConcurrentCalls = 32`; records `EventId` in `ProcessedEvents` in the same transaction as the `NotificationRequested` row, then calls the provider with `EventId` as its idempotency key; client-side token bucket matched to the provider's rate limit.
> 4. **Failure handling.** Abandon on transient error → redelivery; max delivery count 10 → DLQ; DLQ alert and replay tool. Circuit breaker on the provider (Polly, Module 25).
> 5. **Channel seam.** `INotificationChannel` with an email adapter; new channels are new adapters plus a subscription filter.
> 6. **Contracts.** `OrderConfirmedV1` is additive-only; breaking changes create `V2` published side by side.
>
> ### 7. Alternatives
> *(Table as in Concept 14: do nothing, async-after-commit with in-process retry, outbox + Service Bus (proposed), SaaS platform.)*
> **7.2 Outbox vs direct publish after commit:** direct publish loses events when the process dies between commit and publish — the dual-write problem — violating G2. Outbox adds a table and a relay; accepted.
> **7.3 Service Bus Standard vs Premium:** Standard at the design point (shared capacity, throttling possible); move to Premium (1 MU) if sustained load passes ~500/s or the flash-sale load test shows throttling. Revisit trigger recorded.
> **7.4 Event Grid / Event Hubs:** Event Grid's push delivery and retry model fit fan-out notifications but not our need for peek-lock, sessions-if-needed and DLQ replay tooling the team already uses; Event Hubs is built for high-throughput streams and replay, not per-message settlement — overkill at this volume (Module 27).
>
> ### 8. Cross-cutting
> **Security:** relay and service use separate user-assigned managed identities — `Azure Service Bus Data Sender` on the topic and `Data Receiver` on one subscription respectively; local (SAS) auth disabled on the namespace; provider API key in Key Vault (cannot be eliminated), read via the configuration provider with reload. **Privacy:** customer email is PII — never logged; traces carry `OrderId` and `TenantId` only. **Reliability:** SLO — 99.5% of notifications delivered ≤ 60 s, monthly; failure-mode table in Appendix C. **Observability:** OpenTelemetry trace context propagated through the message (`Diagnostic-Id`/W3C), outbox age, backlog, DLQ count, send latency and failures; alerts tied to the SLO's burn rate (Module 28). **Cost:** §11.
>
> ### 9. Rollout and rollback
> Flag `notifications.eventDriven` (App Configuration). Shadow mode (events published and consumed, sending suppressed, results compared with legacy sends) for one week → internal tenant → 10% → 50% → 100%, each step gated on zero checkout regressions and reconciliation showing no missing notifications. Rollback at every step: flag off (legacy path remains until M3 + 2 weeks). Removing the legacy code is the only soft point of no return and is a separate PR after a game day.
>
> ### 10. Risks and open questions
> R1 provider rate limits (load test with sandbox by 2026-10-15); R2 team's Service Bus inexperience (pair with Payments, game day); R3 outbox growth (purge job, alert at 20 GB); R4 schedule (cut the channel seam first). Q1 DLQ retention 7 vs 14 days — SRE, default 14.
>
> ### 11. Plan and cost
> M1 shadow mode (2026-10-24) · M2 internal + game day (2026-11-07) · M3 100% (2026-11-14). Effort 6 engineer-weeks (5–9). Run cost ≈ €300/month at the design point; ≈ €1,000/month at the ceiling on Premium (priced 2026-10-06).
>
> ### FAQ
> *Why not a background job inside the Orders API?* It would share the API's deployment and scaling and couple channel changes to checkout releases (G4); the outbox still applies — the relay could live in-process in v1 if we wanted to defer the new service, and this is our fallback if the schedule slips.
> *Why not just retry inside the request?* Retries extend checkout latency and still fail when the provider is down for longer than the retry budget (G1).

---

What makes this doc strong, mapped to the concepts:

| Feature | Concept |
|---|---|
| The summary states problem with a number, recommendation, trade-off, cost and the exact decision requested | 7 |
| Goals are prioritized, so G1 beating G3 is already decided | 9 |
| Quality scenarios make the reviewers' predictable questions part of the requirements | 10 |
| Alternatives include do-nothing, the minimal change and buy; sub-decisions have their own rationale and a revisit trigger | 14, 37 |
| The security section names identities and roles rather than "best practices" | 15 |
| Rollout has gates; rollback exists at every step; the point of no return is identified | 16 |
| The FAQ answers the two objections a senior reviewer would raise first, and offers a fallback | 19, 35 |

---

## Worked example 2 — Quality attribute scenarios and a utility tree from a vague requirement

**The vague requirement from the PRD:** *"Notifications must be reliable and fast, and the system should be easy to extend."*

**Step 1 — Ask what each word means to whom:**

| Word | Stakeholder | What they actually mean |
|---|---|---|
| "reliable" | Product | Customers always get their confirmation |
| "reliable" | Support | No duplicates (duplicates generate tickets) |
| "reliable" | SRE | It doesn't page at night and recovers by itself |
| "fast" | Product | "Before the customer checks their inbox" — seconds, not milliseconds |
| "easy to extend" | Product roadmap | Push notifications next year |

**Step 2 — Write scenarios** (six parts each — Concept 10). Example for "no duplicates":

| Source | Stimulus | Artifact | Environment | Response | Response measure |
|---|---|---|---|---|---|
| Platform | Consumer crashes after sending, before completing the message | Notification service | Rolling deployment at peak | Message redelivered; idempotency check suppresses re-send | ≤ 1 email per event in 99.99% of events; duplicates counted by a metric |

**Step 3 — Build the utility tree and rate (importance, difficulty):**

```text
Utility
├── Reliability
│   ├── Delivery: (H,M) no confirmed order without a confirmation email or an alert
│   ├── Duplicates: (H,H) ≤ 1 email per event through crashes and redeliveries
│   └── Self-healing: (M,M) provider outage recovers without operator action
├── Performance
│   └── Latency: (M,L) p95 ≤ 5 s normal; ≤ 30 s in ×20 bursts
└── Modifiability
    └── New channel: (L,M) push channel in ≤ 5 engineer-days, no Orders change
```

**Step 4 — Spend design and review time on (H,H) first.** Here that's duplicate suppression — which is why the design's most precise rule (Concept 13's code block) is the idempotency rule, and why the reviewer in Concept 31 went straight for it. "Extensibility," which sounded important in the PRD, turns out to be (L,M): a seam, not a framework.

**Lesson:** a vague "-ility" usually hides several concrete scenarios with very different priorities, and the utility tree tells you where the design effort — and the review time — should go.

---

## Worked example 3 — A defense, exchange by exchange

A panel (or review meeting) challenges RFC-0142. Each exchange shows the classification (Concept 36) and a strong response.

**1. "Isn't this over-engineered? A new service, a topic and an outbox to send emails."**
*Classification: prediction/value — is the complexity justified?*
> "Fair challenge — the minimal option is in §7: send after commit with an in-process retry. It fixes G1 but breaks G2: a crash or a deploy loses whatever is queued in memory, and we deploy daily. The outbox is the smallest thing that gives us 'never lost'. The separate service is the part I'd give up first — the FAQ describes running the consumer inside the Orders API in v1 if we want fewer moving parts. What I wouldn't give up is the outbox."

*Why it works:* concedes the most debatable element precisely, defends the load-bearing one with the goal it serves.

**2. "Why Service Bus? We use Event Hubs everywhere else."**
*Classification: factual + consistency/value.*
> "Two reasons, both about per-message semantics: we need per-message settlement with dead-lettering and a replay tool, and our volume is three orders of magnitude below where Event Hubs' throughput model pays off. Consistency with the platform is a real cost, though — if platform standards require Event Hubs, the design survives with checkpointing and a custom poison-message store; it's about two extra weeks. That's a call for the platform owner, and I'd like it made before M1."

*Why it works:* answers with mechanism, acknowledges the reviewer's legitimate value (consistency), quantifies the alternative, routes the value question to its owner.

**3. "What happens if the relay is down for a day?"**
*Classification: a gap.*
> "The outbox grows about 750,000 rows — about 1.5 GB. Checkout is unaffected; notifications are delayed by up to a day, which violates G3 but not G1 or G2. We alert when outbox age exceeds two minutes, so a day-long outage means the alert failed. You've made me realize there's no capacity bound in the doc — I'll add one: if the table exceeds 20 GB we page, and the runbook covers manual relay restart. Thanks."

*Why it works:* answers with numbers, maps the effect onto the prioritized goals, concedes precisely and states the change.

**4. "Exactly-once delivery is impossible. Your doc promises it."**
*Classification: misunderstanding (or a wording problem in the doc).*
> "Agreed that exactly-once *delivery* isn't achievable over the network. What the design gives is at-least-once delivery with idempotent processing — exactly-once *effect* on our side, and on the provider's side to the extent its idempotency key holds. If the doc says 'exactly once' anywhere, that's my wording error — I'll fix it to 'at most one email per event, enforced by idempotency keys'."

*Why it works:* agrees with the correct principle (Module 11), clarifies the actual claim, fixes the doc.

**5. "Your 6-week estimate is optimistic; integrations always take longer."**
*Classification: prediction.*
> "You're probably right on the integration part — that's why the range goes to 9 weeks and the main uncertainty is named. Two things limit the downside: the provider sandbox test is scheduled before M1, so we'll know by the 15th, and there's a cut line — the channel seam goes first. If we're behind by more than a week on the 30th, we ship the in-process consumer and do the separate service in Q1."

*Why it works:* doesn't argue with a reasonable prior; shows the plan already contains early detection and a pre-agreed cut.

**6. (Panel changes a constraint) "Assume the business now wants SMS for high-value orders in the same release."**
*Classification: scope change — a new qualifier.*
> "The design holds: SMS is a second adapter and a subscription filter on order value. What changes is the plan and the risk: SMS adds a second provider integration — our least predictable work — and phone numbers as new PII. To keep the November date, I'd ship email first and SMS two weeks later behind its own flag, rather than coupling both launches. If both must ship together, I'd want to drop the dashboard work and accept a higher schedule risk explicitly."

*Why it works:* separates what holds from what changes, quantifies the new risk, gives the decider options.

**7. "I just don't like adding a new service for a team of four."**
*Classification: value / ownership.*
> "That's a legitimate concern, and it's about operating cost rather than correctness. The honest numbers: one more deployment pipeline, one more dashboard and a handful of alerts; no new on-call rotation because it joins ours. If that's still too much, the FAQ option runs the consumer inside the Orders API — same outbox, same guarantees, fewer moving parts, and G4 gets weaker. I'm genuinely fine with either; I'd like the decider to choose."

*Why it works:* names the kind of disagreement, gives the real cost, offers the alternative without ego, routes to the decider.

---

## Worked example 4 — Reviewing a flawed design doc

**The excerpt you're given:**

> *"Inventory Sync v2. To keep the warehouse system and our catalog in sync, the Catalog API will, on every product update, write to its SQL database and then call the warehouse REST API. If the warehouse call fails, we log the error. We'll use Kubernetes (AKS) for the new sync component so it can scale to millions of requests, and Cosmos DB for a cache of warehouse stock levels with strong consistency so data is always correct. This approach is highly available and scalable. Rollout: deploy on Friday after QA sign-off."*

**A prioritized review:**

**Verdict:** *Not ready for approval. The direction may be right, but there are two blocking correctness issues and no problem statement to judge it against.*

**Blocking**
1. **Dual write with log-and-forget** (Module 11). "Write SQL, then call the warehouse; on failure, log" guarantees silent divergence whenever the call fails or the process dies between the two steps. *Yes, if:* use an outbox (or CDC) with retries and reconciliation, and define what happens when the warehouse rejects an update.
2. **No stated problem, goals or scale.** "Millions of requests" and "highly available" are unanchored. What's the update rate today? What's the cost of divergence? Without this, AKS and Cosmos DB can't be justified. *Yes, if:* add a problem statement with current failure data, goals with numbers and a capacity estimate.

**Important**
3. **Technology without requirements.** AKS for a sync component at an unknown (probably modest) volume adds a large operational surface for a team that may not run Kubernetes (Module 26) — Container Apps or a hosted background service may suffice. Strong consistency for a *cache* is contradictory: a cache is an asynchronous replica by definition (Module 10); if stock levels must be exact at the point of sale, read the source of truth there.
4. **No alternatives considered** — at minimum: CDC from SQL, a warehouse-pull model, a scheduled reconciliation alone.
5. **No failure modes, no observability, no rollback.** What happens when the warehouse is down for an hour? How do we detect divergence? How do we undo the deploy?

**Questions**
6. Who owns the warehouse API, and does it support idempotent updates?
7. What are the warehouse API's rate limits?

**What's good**
8. Separating the sync component from the Catalog API is a sound instinct — it isolates the warehouse's availability from catalog edits.

**Minor**
9. "Deploy on Friday after QA sign-off" — prefer a flagged, staged rollout early in the week.

*Note the shape:* a one-line verdict, blocking items with conditions, then decreasing severity, and one genuine strength. In an interview, saying this in about five minutes demonstrates far more than reading out twenty comments.

---

## Worked example 5 — A ten-minute past-design presentation, scripted

**Design:** splitting reporting off an OLTP database (anonymized).

| Minute | What to say |
|---|---|
| 0–1 | "A B2B SaaS platform, ~1,500 tenants, one SQL database serving both the app and tenant reporting. I was the senior engineer on the platform team; I wrote the design doc, led review and owned the migration; three engineers implemented." |
| 1–2 | "Month-end reporting queries drove CPU to 95% and app p99 from 300 ms to 4 s for two days every month; four incidents in six months; support escalations from our largest tenants." |
| 2–3 | "Constraints: no budget for a data platform team, SQL Server skills only, three months. Goals: app p99 unaffected by reporting; reports no more than 15 minutes stale. Non-goal: ad hoc analytics." |
| 3–5 | "Options: scale up the database (buys time, doesn't fix contention); read replica for reporting (cheapest, but same schema optimized for OLTP and the heaviest queries still slow); a separate reporting store fed by change tracking with a reporting-shaped schema (more work, fixes both). We chose the replica first, then the reporting store — phased, because the replica fixed the incidents in two weeks." |
| 5–7 | "Phase 2 used change tracking into a small reporting database with pre-aggregated tables per tenant and month; the riskiest part was correctness of incremental aggregates, which we de-risked by running both pipelines in parallel for a month and diffing." |
| 7–8 | "The disagreement was with the analytics lead, who wanted a lakehouse. We agreed the disagreement was about future needs versus current skills, wrote both options up for the director, and she chose ours with a revisit trigger: if ad hoc analytics became a product requirement." |
| 8–9 | "Outcome: no reporting-related incidents since; month-end app p99 under 350 ms; reports 5–10 minutes stale. Effort was 14 weeks against an estimate of 10 — the diff tooling took longer than planned." |
| 9–10 | "What I'd change: I'd have built the reconciliation diff first, not last — it was the most valuable artifact. And I underestimated the analytics lead's point; eighteen months later ad hoc analytics did become a requirement and we built the lakehouse anyway. The phased approach still paid off, but I'd now write revisit triggers with dates, not just conditions, and review them." |

---

## Worked example 6 — A Tier 1 one-pager

> **One-pager: Cache product catalog reads with HybridCache**
> **Problem.** `GET /products/{id}` is 70% of Catalog API traffic, p99 180 ms, and drives 40% of SQL DTU; product data changes a few times per day.
> **Proposal.** Add `HybridCache` (in-memory L1 + Redis L2, Module 10) on the read path with a 10-minute expiry and tag-based invalidation on product update; stampede protection is built in.
> **Expected effect.** p99 < 30 ms for cache hits (measured in a spike: 96% hit rate on a replay of yesterday's traffic); SQL DTU −35%.
> **Alternative.** Read replica — cheaper to build but doesn't cut latency and costs more per month. Rejected.
> **Main risk.** Stale product data for up to 10 minutes if an invalidation is missed. Acceptable per product (prices are re-checked at checkout from the source of truth).
> **Rollout/rollback.** Feature flag per endpoint; rollback = flag off.
> **Reviewers.** Catalog team lead (approve), SRE (Redis capacity).

---

## Worked example 7 — A weighted decision matrix done wrong, then right

**Wrong:** six criteria, 1–10 scores, weights chosen after scoring:

| Criterion | Weight | Cosmos DB | Azure SQL | PostgreSQL |
|---|---|---|---|---|
| Scalability | 25% | 10 | 7 | 7 |
| Performance | 20% | 9 | 8 | 8 |
| Throughput | 15% | 10 | 7 | 7 |
| Cost | 15% | 5 | 7 | 8 |
| Team skills | 10% | 4 | 9 | 6 |
| Query flexibility | 15% | 5 | 9 | 9 |
| **Total** | | **7.65** | **7.75** | **7.60** |

Problems: scalability, performance and throughput are correlated (triple-counting); a 0.15 difference between 1–10 subjective scores is noise; there's no must-have gate (does each option meet the data residency requirement and the 5 ms read SLO at all?); weights were set afterwards.

**Right:**

1. **Must-have gates** (pass/fail): EU residency ✓✓✓; transactions across order and payment rows ✓ (SQL, PostgreSQL) / ✗ for Cosmos DB *without* redesigning around the partition key — so Cosmos DB either needs a data-model change or exits here.
2. **Few, independent criteria with coarse scores** (−, 0, +), weights agreed with the decider *before* scoring: *fit for the workload's access patterns*, *operability for this team*, *cost at 3× scale*, *migration effort from today's SQL Server*.
3. **Sensitivity check:** Azure SQL wins unless "cost at 3×" is weighted above 40%, in which case PostgreSQL wins — so the decision is really a cost-versus-skills question, and that's what goes to the decider.
4. **Prose rationale** decides; the matrix just organized the discussion.

---
# Common interview questions with model answers

**Q1. "Walk me through how you write a design document."**
> "I start by framing the problem with evidence and agreeing goals and non-goals with the decider, then talk to the likely objectors before writing much. I outline the argument, circulate a one-pager to kill or redirect early, and gather evidence — usually a spike and an estimate. The full doc follows the standard anatomy: summary, context, prioritized goals and non-goals, quality scenarios, constraints and assumptions, estimate, design, alternatives including doing nothing and the minimal change, cross-cutting concerns with a failure-mode table and a compressed threat model, rollout with rollback and a reversibility table, risks, plan and cost. I write alternatives and failure modes before polishing the design, red-team it as the most skeptical stakeholder, cut, write the summary last, then open review with named reviewers, specific asks and a decision date."

**Q2. "What makes a design doc good?"**
> "It makes a decision easy to evaluate. Concretely: the problem is evidenced and the cost of doing nothing is quantified; goals are measurable and prioritized; quality attributes are scenarios with response measures; real alternatives are compared on the same criteria; trade-offs are stated explicitly, including where the proposal loses; failure behavior, rollout and rollback are designed; risks are honest; and the summary states the decision requested. A good doc is also proportionate — the shortest document that lets every required reviewer evaluate their concerns."

**Q3. "When would you not write a design doc?"**
> "For two-way-door decisions with small blast radius inside one team — a PR description is enough. When we don't know enough to choose, I'd run a timeboxed spike first. When the problem itself is contested, I'd write the problem statement or PRD first. And when a decision has already been made, I'd record it honestly in an ADR rather than writing a proposal with strawman alternatives."

**Q4. "Tell me about a design of yours that was rejected."**
> *Structure:* context and your proposal → who rejected it and what kind of disagreement it was → what they saw that you didn't → what happened next → what you changed in how you design. *Key signal:* generosity to the reviewers and a mechanism-level lesson ("I framed an ownership decision as a technology decision").

**Q5. "How do you handle a senior engineer who strongly disagrees with your design?"**
> "First I make sure I understand the objection — I restate it until they agree I've got it — and I work out what kind of disagreement it is. If it's factual, we get data; if it's a prediction, we look for a cheap test or choose the more reversible option; if it's a value or priority question, I point to the goal ranking or take it to whoever owns that goal. I concede precisely where they're right and update the doc with credit. If we still disagree on a value question, we write a joint escalation that both of us agree is fair and let the decider choose — and whoever 'loses' disagrees and commits with the concern recorded."

**Q6. "How do you get a decision when reviews drag on?"**
> "Usually it's missing decision rights or a deadline. I name one approver up front, list required reviewers with the concern each owns, and set a decision date. When it stalls, I ask the approver what information would let them decide, get it — often a two-day spike — and reset the date. Open questions get defaults: if nobody decides by a date, we proceed with the default. And for reversible decisions I'd use consent rather than consensus: proceed unless someone has a reasoned objection."

**Q7. "What's the most important section of a design doc?"**
> "The alternatives section, because a choice is only as credible as the comparison behind it. It should include doing nothing, the minimal change, a buy option and the alternative a senior reviewer would propose, compared on goal-derived criteria, steelmanned, with reasons for rejection and the conditions under which I'd choose each instead. A close second is the problem statement — most docs are rejected there, not in the design."

**Q8. "How do you document non-functional requirements so they're testable?"**
> "As six-part quality attribute scenarios — source, stimulus, artifact, environment, response and response measure. 'Highly available' becomes 'when the provider is down for 30 minutes at peak, checkout is unaffected, all notifications are delivered within 10 minutes of recovery and on-call is alerted within 5.' I prioritize them by importance and difficulty, utility-tree style, and spend design and review time on the high-high ones first."

**Q9. "How do you make trade-offs explicit?"**
> "One-sentence trade-off statements — we choose X over Y, gaining A at the cost of B, because goal G outranks goal H — plus a comparison table that shows where my option loses, ATAM-style sensitivity and trade-off points for the decisions that move several qualities at once, and reversibility for each decision. If I use a weighted matrix, it's with must-have gates first, coarse scores, weights agreed before scoring and a sensitivity check — and it only structures the discussion; the prose rationale decides."

**Q10. "How would you review someone else's design doc?"**
> "In three passes: problem, decision, detail. I stop early if the problem isn't evidenced or the goals don't match it. Then I check that alternatives are real and the proposal meets its own goals, and only then go deep on consistency, failure modes, capacity, security, migration, rollback, operability and cost. My feedback is labeled — blocking or not, issue versus question versus suggestion — anchored to the doc's goals, framed as 'yes, if' with what would unblock it, and ends with an explicit verdict. I review the design as written, not the one I'd have written."

**Q11. "How do you handle a question in a review you can't answer?"**
> "I say I don't know, then how and by when I'll find out, and what changes depending on the answer: 'I don't know the provider's burst limit; I'll confirm by Thursday; if it's below 500 a second we add a client-side limiter and nothing else changes.' Bluffing is both detectable and expensive — a wrong confident answer poisons trust in everything else in the doc."

**Q12. "How do you keep design docs from going stale?"**
> "I don't try to keep the design doc itself alive — it's a decision record, frozen after approval with an as-built note. Significant deviations during implementation come back as doc updates or new ADRs; the durable decisions are extracted into append-only ADRs that are superseded rather than edited; and the living description of the system lives in the architecture docs — C4 models and arc42-style sections — maintained with the code. Docs are numbered and linked from code, READMEs and runbooks so they're found when needed."

**Q13. "How do AI tools change how you write and review design docs?"**
> "They make polished writing nearly free while leaving review as expensive as ever, so the risk is a glut of fluent documents and the bottleneck is senior attention. I use AI to red-team drafts, check completeness and consistency and tighten prose — never as the source of numbers or alternatives I haven't verified, because I own every claim. As a process owner, I'd require a few human-written anchors such as the hardest trade-off, the strongest argument against and the metric that would show failure, triage the review queue visibly, and use automated review for consistency against our own standards while keeping judgment human."

**Q14. "How would you set up a design review process for an engineering organization of 80?"**
> "Tiered by reversibility and blast radius: PR descriptions for two-way doors, one-pagers with a peer reviewer for moderate changes, standard docs with async review and a short meeting for cross-team work, and a formal session — utility tree, pre-mortem, threat model — for one-way doors. A light template, docs in the repo or a single indexed space, numbered, with status and a named approver and decision date on every doc. A small rotating review group of senior engineers with a 'yes, if' default rather than a gatekeeping board, specialist review for public APIs and security, ADRs for durable decisions, and a quarterly look at metrics like time to decision and how often reviews change designs — so we can tell useful review from approval theater."

**Q15. "What would make you abandon your own design?"**
> "Whatever I wrote down as its kill criteria — that's part of the doc. For the notification design: if the provider can't support idempotency keys and duplicates exceed our tolerance, if the load test shows Standard throttling and Premium breaks the budget, or if platform standards require Event Hubs and the operating cost of a second messaging technology outweighs the benefits. Stating these in advance keeps the argument falsifiable and turns future debates into checks."

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Purpose | Treats the doc as documentation of a decision already made | Treats it as a decision instrument written while change is cheap |
| When to write | A doc for everything, or for nothing | Tiered by reversibility, blast radius, cross-team impact, novelty |
| Summary | Background first; decision buried on page 4 | Problem with a number, recommendation, trade-off, cost and decision requested on page one |
| Problem | Solution-shaped ("we don't use Kafka") | Evidenced consequence, "why now", quantified cost of doing nothing |
| Goals | Adjectives ("fast, scalable") | Measurable, prioritized goals; non-goals as scope weapons |
| NFRs | "-ilities" | Six-part quality scenarios with response measures |
| Assumptions | Unstated | Listed with owner, validation method and date |
| Estimation | Absent or asserted | Arithmetic visible, sourced inputs, headroom, sensitive inputs named |
| Design | Class-level detail or boxes without mechanisms | Top-down; contracts, data, flows, failure behavior; code only where code is the decision |
| Alternatives | None, or strawmen | Do-nothing, minimal, buy, different approach, steelmanned on the same criteria |
| Trade-offs | Benefits only | "We choose X over Y, gaining A at the cost of B, because G outranks H"; trade-off points named |
| Decision matrix | Precise-looking totals decide | Gates first, coarse scores, weights before scoring, sensitivity check; prose decides |
| Evidence | "Best practice" | Strongest evidence under the most load-bearing claims; confidence labeled |
| Security | "We'll follow best practices" | Identities, roles, boundaries, top threats and mitigations named |
| Reliability | "Highly available" | SLOs plus a failure-mode table with detection and residual risk |
| Rollout | "Deploy Friday after QA" | Flags, shadow mode, rings, gates; reversibility table; irreversible steps last |
| Risks | None listed | Honest, ranked, owned, with triggers; the design's weakest point named by the author |
| Length | 25 pages, or one paragraph for a one-way door | Shortest doc that lets every required reviewer evaluate their concern |
| Review prep | Doc dropped on a mailing list | Pre-wired stakeholders, chosen reviewers incl. a skeptic, specific asks, decision date |
| Meetings | Re-presenting the doc as slides | Silent reading or pre-read; risk-first discussion; outcomes captured live |
| Reviewing | 30 unlabeled comments; the doc they'd have written | Three passes, labeled and prioritized, "yes, if", explicit verdict |
| Decision | "The team" decides; reviews never end | One approver, decision rule, deadline, recorded dissent, disagree and commit |
| Defense | Defends the design; interrupts; bluffs | Defends the reasoning; restates; classifies; evidence; concedes precisely; "I don't know — here's how I'll find out" |
| Disagreement | Argues values as if they were facts | Diagnoses the kind of disagreement and uses the matching tool |
| Stakeholders | One pitch for everyone; ultimatums | Leads with what each audience decides on; options with a recommendation |
| AI | Generated doc with unverified numbers | AI for red-teaming and consistency; human-owned, checkable claims |
| Afterlife | Doc silently diverges from the system | Frozen with as-built note, ADRs extracted, retrospective of predictions vs reality |
| Interview | Technology catalogue, no numbers, no risks | Proportionate design, stated assumptions, real alternatives, self-critique |

---
# Practice exercises

1. **Take-home rehearsal (4–6 hours).** Prompt: *"Our B2B SaaS invoicing platform (ASP.NET Core monolith, Azure SQL, ~800 tenants) must add e-invoicing to government portals in three EU countries within six months. Portals are slow, sometimes down for hours, and each has a different API. Team: five engineers."* Write a Tier 2 design doc using Appendix A, at most 8 pages. Then have someone (or an AI acting as a skeptical principal engineer) challenge it for 45 minutes and record which challenges you hadn't anticipated.
2. **Rewrite a summary.** Take any design doc or RFC you've written (or a public KEP or Rust RFC) and rewrite its summary as the five sentences from Concept 7. Compare with the original.
3. **Scenarios from adjectives.** Convert "secure, scalable, maintainable and highly available" for a system you know into at least eight six-part quality scenarios, then build a utility tree and rate each leaf (importance, difficulty).
4. **Alternatives drill.** For a decision you made recently, write the strongest possible case for the option you rejected — steelmanned well enough that its advocate would sign it. Did your decision survive?
5. **Failure-mode table.** For a service you operate, write the table from Concept 15 for every dependency. Mark which rows you've actually tested.
6. **Live critique (40 minutes).** Pick a public design proposal — a KEP, a Rust RFC, a dotnet/designs proposal or a .NET API proposal marked `api-ready-for-review` — and review it with Concept 43's passes. Write a one-line verdict and at most five prioritized findings.
7. **Past-design presentations.** Script two ten-minute presentations (one success, one failure or rejection) using Concept 42's structure. Record yourself; check that "what I'd change" names mechanisms, not platitudes.
8. **Disagreement classification.** Recall three design arguments you've been in. For each, identify the kind (Concept 38). Was the tool used appropriate? What would have resolved it faster?
9. **Defense role-play.** Have a partner play, in turn, a skeptical VP (cost and risk), an SRE lead (operability) and an opposing principal engineer (technology choice). Defend the same doc to each in 10 minutes. Note how your opening changes.
10. **Process design.** Write a one-page proposal for a design-review process for an 80-engineer organization (Q14): tiers, template, decision rights, review group, AI policy and the metrics you'd track.
11. **Retrospective.** Find a design doc from at least six months ago (yours or your team's). Build the predictions-versus-reality table from Concept 40. What does it teach about your estimates?
12. **AI red-team.** Ask an AI assistant to produce the ten strongest objections to one of your docs. Classify each: real and already handled, real and new, or wrong. Add the real-and-new ones to the FAQ.

---
# Free resources and learning material

All free to read online (a few HBR and vendor pages may ask for a free registration). Grouped by purpose; start with the ★ items.

### How real organizations write and review design docs
- ★ [Design Docs at Google — Malte Ubl (Industrial Empathy)](https://www.industrialempathy.com/posts/design-docs-at-google/) — the most widely cited description of a design-doc culture: sections, length, lifecycle and when not to write one.
- ★ [Companies Using RFCs or Design Docs, and Examples — The Pragmatic Engineer](https://blog.pragmaticengineer.com/rfcs-and-design-docs/) — a catalogue of public templates and companies' practices.
- [Scaling Engineering Teams via RFCs: Writing Things Down — The Pragmatic Engineer](https://blog.pragmaticengineer.com/scaling-engineering-teams-via-writing-things-down-rfcs/) — how Uber's planning-doc process evolved as it grew.
- ★ [RFD 1: Requests for Discussion — Oxide Computer](https://rfd.shared.oxide.computer/rfd/0001) — a complete, public, Git-based process with explicit states.
- [RFD 1 announcement — Oxide blog](https://oxide.computer/blog/rfd-1-requests-for-discussion) and [A Tool for Discussion — Oxide blog](https://oxide.computer/blog/a-tool-for-discussion) — why they built an RFD site for search and inline discussion.
- [Oxide and Friends podcast episode on RFDs](https://oxide.computer/podcasts/oxide-and-friends/2065190) — how 500+ RFDs actually get written and closed.
- [Requests for Discussion — Bryan Cantrill (2015)](https://bcantrill.dtrace.org/2015/09/16/requests-for-discussion/) — the origin of the RFD idea and why issue trackers don't fit design discussion.
- ★ [The Power of "Yes, if" — Squarespace Engineering (Tanya Reilly)](https://engineering.squarespace.com/blog/2019/the-power-of-yes-if) — conditional approval, review councils and the RFC template they use.
- [RFCs and review councils — Brandur Leach](https://brandur.org/fragments/rfcs-and-review-councils) — a sharp critique of committee-style review; read it right after the Squarespace post.
- [Writing Practices and Culture — HashiCorp](https://www.hashicorp.com/how-hashicorp-works/articles/writing-practices-and-culture), [RFC template](https://www.hashicorp.com/how-hashicorp-works/articles/rfc-template) and [PRD template](https://www.hashicorp.com/how-hashicorp-works/articles/prd-template) — the PRD → RFC pair, with "write the summary last."
- [Hermes — HashiCorp's open-source document management system](https://github.com/hashicorp-forge/hermes) — tooling for authoring, reviewing, approving and finding docs at scale.
- [Painless Functional Specifications, Part 1 — Joel Spolsky](https://www.joelonsoftware.com/2000/10/02/painless-functional-specifications-part-1-why-bother/) — old, still the best short argument for writing before building.
- [How to write a good software design doc — Angela Zhang](https://www.freecodecamp.org/news/how-to-write-a-good-software-design-document-66fcf019569c/) — practical and concise.

### Public templates and proposal processes to study
- ★ [Rust RFC template](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) and [the Rust RFC process](https://github.com/rust-lang/rfcs) — motivation, guide- and reference-level explanation, drawbacks, alternatives, prior art, unresolved questions, final comment period.
- ★ [Kubernetes KEP template](https://github.com/kubernetes/enhancements/blob/master/keps/NNNN-kep-template/README.md) — goals/non-goals, risks and mitigations, test plan, graduation criteria, version skew and a production readiness questionnaire.
- [PEP 1 — Python Enhancement Proposal purpose and guidelines](https://peps.python.org/pep-0001/) — statuses, roles and how decisions are made.
- [Go proposal process](https://github.com/golang/proposal) — issue → discussion → design doc → review committee.

### .NET-specific
- ★ [dotnet/designs — .NET design proposals](https://github.com/dotnet/designs) and [its template](https://github.com/dotnet/designs/blob/main/meta/template.md) — how the .NET team writes designs for the runtime, libraries and SDK.
- ★ [dotnet/runtime API review process](https://github.com/dotnet/runtime/blob/main/docs/project/api-review-process.md) — `api-ready-for-review` → `api-approved`, the FXDC and what a strong API proposal looks like.
- [dotnet/apireviews — published API review notes](https://github.com/dotnet/apireviews) and [apireview.net](https://apireview.net) — the backlog, schedule and past review recordings.
- [ASP.NET Core API review process](https://github.com/dotnet/aspnetcore/blob/main/docs/APIReviewProcess.md) — the same principle for ASP.NET Core.
- [dotnet/csharplang — proposals and Language Design Meeting notes](https://github.com/dotnet/csharplang) — some of the best public examples of recorded design decisions with rationale.
- [.NET Framework Design Guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/) — the conventions API reviews hold proposals to.
- [.NET application architecture guides](https://learn.microsoft.com/dotnet/architecture/) — reference designs to borrow structure and vocabulary from.
- [Aspire documentation](https://aspire.dev) — the quickest way to wire a throwaway multi-service prototype for a spike.
- [BenchmarkDotNet](https://benchmarkdotnet.org) — evidence for performance claims (Module 17).
- [Feature management in Azure App Configuration](https://learn.microsoft.com/azure/azure-app-configuration/feature-management-overview) — the flagging mechanism most .NET rollout plans rely on.

### Microsoft and Azure architecture guidance
- ★ [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/) and [its pillars](https://learn.microsoft.com/azure/well-architected/pillars) — design principles, review checklists and documented trade-offs per pillar.
- ★ [Architecture design specification — WAF architect role](https://learn.microsoft.com/azure/well-architected/architect-role/architecture-design-specification) — Microsoft's view of what an architect's design spec must contain.
- [Architecture decision record — WAF architect role](https://learn.microsoft.com/azure/well-architected/architect-role/architecture-decision-record) — append-only log, recording confidence level, superseding rather than editing.
- [Architecture design diagrams — WAF architect role](https://learn.microsoft.com/azure/well-architected/architect-role/design-diagrams) — which diagrams to produce and for whom.
- [Azure Well-Architected Review (free assessment)](https://learn.microsoft.com/assessments/azure-architecture-review/) — a structured checklist to run against your design.
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/), [technology choices](https://learn.microsoft.com/azure/architecture/guide/technology-choices/technology-choices-overview) and [cloud design patterns](https://learn.microsoft.com/azure/architecture/patterns/) — sources for alternatives sections.
- [Mission-critical design methodology](https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-design-methodology) — what a Tier 3 design's depth looks like on Azure.
- [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/) — run-cost estimates; record the date you priced.
- [Azure Load Testing](https://learn.microsoft.com/azure/load-testing/) and [Azure Chaos Studio](https://learn.microsoft.com/azure/chaos-studio/) — turning capacity and failure-mode claims into evidence.

### Microsoft ISE Code-With Engineering Playbook (design reviews)
- ★ [Design reviews](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/) — goals, measures (cost of change, time to decision) and facilitation.
- [Design review recipes](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/recipes/) and [async design reviews via pull requests](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/recipes/async-design-reviews/).
- [Feature/story design review template](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/recipes/templates/feature-story-design-review/) and [milestone/epic template](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/recipes/templates/milestone-epic-design-review/).
- [Trade study template](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/trade-studies/template/) — a structured alternatives comparison.
- [Decision log](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/decision-log/) — ADRs in practice.
- [Engineering feasibility spikes](https://microsoft.github.io/code-with-engineering-playbook/design/design-reviews/recipes/engineering-feasibility-spikes/) — how to buy evidence before deciding.
- [Non-functional requirements capture guide](https://microsoft.github.io/code-with-engineering-playbook/design/design-patterns/non-functional-requirements-capture-guide/).

### Architecture evaluation, description and readiness
- ★ [ATAM: Method for Architecture Evaluation — SEI technical report (PDF)](https://resources.sei.cmu.edu/asset_files/TechnicalReport/2000_005_001_13706.pdf) — utility trees, scenarios, sensitivity and trade-off points, risk themes.
- [Architecture tradeoff analysis method — Wikipedia](https://en.wikipedia.org/wiki/Architecture_tradeoff_analysis_method) — a two-minute overview of the steps.
- [ISO/IEC/IEEE 42010 information site (Rich Hilliard)](http://www.iso-architecture.org/42010/) — stakeholders, concerns, viewpoints and views explained without buying the standard.
- [arc42](https://arc42.org), [docs.arc42.org (tips and examples per section)](https://docs.arc42.org/home/) and [the arc42 quality model](https://quality.arc42.org) — the free architecture-description template and a catalogue of quality attributes with example scenarios.
- [The C4 model](https://c4model.com) — diagram altitudes (Module 31).
- [Risk-storming — Simon Brown](https://riskstorming.com) — visual, collaborative risk identification.
- [Performing a Project Premortem — Gary Klein, HBR](https://hbr.org/2007/09/performing-a-project-premortem).
- [Google SRE: Launch Coordination Checklist](https://sre.google/sre-book/launch-checklist/), [Reliable Product Launches at Scale](https://sre.google/sre-book/reliable-product-launches/) and [The Evolving SRE Engagement Model (production readiness reviews)](https://sre.google/sre-book/evolving-sre-engagement-model/).
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) and [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework) — cross-cloud vocabulary for the same review questions.

### Decisions, influence and the architect's role
- ★ [Scaling the Practice of Architecture, Conversationally — Andrew Harmel-Law (martinfowler.com)](https://martinfowler.com/articles/scaling-architecture-conversationally.html) — the advice process.
- [Who Needs an Architect? — Martin Fowler (PDF)](https://martinfowler.com/ieeeSoftware/whoNeedsArchitect.pdf) — architecture as the decisions that are hard to change.
- [Amazon shareholder letters](https://ir.aboutamazon.com/annual-reports-proxies-and-shareholder-letters/default.aspx) — 2015 (Type 1 vs Type 2 decisions), 2016 (disagree and commit; deciding with ~70% of the information), 2017 (six-page narratives).
- [DACI decision-making framework — Atlassian Team Playbook](https://www.atlassian.com/team-playbook/plays/daci).
- [Conventional Comments](https://conventionalcomments.org) — labeled, prioritized review feedback.
- [How to write code review comments — Google Engineering Practices](https://google.github.io/eng-practices/review/reviewer/comments.html) — courtesy and clarity that transfer directly to design review.
- [Writing an engineering strategy — Will Larson](https://lethain.com/eng-strategies/) — "write five design docs, then synthesize"; the bridge from docs to strategy. (His 2025 book *Crafting Engineering Strategy* expands this; not free.)
- [StaffEng — stories and guides for staff-plus engineers](https://staffeng.com/).
- [Being Glue — Tanya Reilly](https://noidea.dog/glue) — the invisible work (docs, reviews, alignment) that senior roles depend on.
- [The Architect Elevator — Gregor Hohpe](https://architectelevator.com/) — essays on communicating architecture up and down an organization.

### Writing clearly
- ★ [Google Technical Writing One and Two (free courses)](https://developers.google.com/tech-writing).
- [Microsoft Writing Style Guide](https://learn.microsoft.com/style-guide/welcome/).
- [Federal Plain Language Guidelines](https://www.plainlanguage.gov/guidelines/).
- [RFC 2119 — key words for requirement levels](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174 — uppercase-only clarification](https://www.rfc-editor.org/rfc/rfc8174).
- [Diátaxis](https://diataxis.fr) — where design docs fit among tutorials, how-tos, reference and explanation.
- [Docs as Code — Write the Docs](https://www.writethedocs.org/guide/docs-as-code/).

### Diagrams as code
- [Mermaid](https://mermaid.js.org) — sequence, flow, state and C4-style diagrams, rendered natively in GitHub and Azure DevOps Markdown.
- [Structurizr](https://structurizr.com) — C4 models as code (Module 31).
- [PlantUML](https://plantuml.com) and [D2](https://d2lang.com) — alternatives with different layout trade-offs.

### AI, specs and review in 2026
- [GitHub Spec Kit](https://github.com/github/spec-kit) — spec-driven development for coding agents; compare its artifacts with a design doc's.
- [How Cloudflare enforces engineering standards using AI](https://blog.cloudflare.com/engineering-standards-enforcement/) — machine review of specs and code against an RFC corpus.
- [The RFC Glut: design review when writing is free](https://tianpan.co/blog/2026/07/02/the-rfc-glut-design-review-when-writing-is-free) — the review-attention argument and the "three human-written anchors" proposal.

### Talks
- [A Philosophy of Software Design — John Ousterhout (Talks at Google)](https://www.youtube.com/watch?v=bmSAYlu0NcY) — "design it twice," the habit behind every good alternatives section.

---
# Quick-recall sheet

**Definition.** A design doc is a pre-implementation argument that a specific change is the best answer to a stated problem under explicit constraints. Decision instrument first; record second.

**Write one when** the decision is a one-way door, has a large blast radius, crosses teams, is novel, costly or contested. Tier it: PR description → one-pager → 4–10 pages → parent + child docs.

**Toulmin map.** Claim = proposal · Grounds = evidence · Warrant = rationale · Backing = patterns/benchmarks · Qualifier = scope/assumptions · Rebuttal = alternatives/risks/kill criteria.

**Anatomy.** Header (owner, one approver, reviewers, status, decision date) → Summary (problem + number, proposal, trade-off, cost, ask; written last) → Context (evidence, why now, cost of doing nothing) → Goals (measurable, prioritized) & non-goals → Quality scenarios (source, stimulus, artifact, environment, response, measure) → Constraints & assumptions (with validation) → Estimate (arithmetic visible) → Design (overview → components → contracts → data → flows → failure behavior) → Alternatives (do nothing, minimal, buy, different, the reviewer's favorite) → Cross-cutting (security, privacy, reliability + failure-mode table, observability, performance, cost, operability, data, testing, compatibility) → Rollout/migration/rollback (flags, rings, gates, reversibility table) → Risks & open questions (owners, dates, defaults) → Plan & cost (milestones with exit criteria, ranges, cut lines) → Appendix, glossary, FAQ.

**Trade-offs.** "We choose X over Y, gaining A at the cost of B, because G outranks H." Matrices: gates first, coarse scores, weights before scoring, merge correlated criteria, sensitivity check; prose decides. ATAM words: sensitivity point, trade-off point, risk, non-risk, risk theme.

**Evidence ladder.** Intuition < vendor docs < estimate < others' experience < spike < benchmark/load test < production telemetry < production experiment. Label: measured / estimated / assumed.

**Review.** Pre-wire (nemawashi); small reviewer set incl. a skeptic; specific asks; reading time; decision date. Meeting: author, facilitator, decider, scribe; silent read; risk-first; outcomes live in the doc. Reviewer: problem → decision → detail; labeled comments; "yes, if"; explicit verdict.

**Decision rights.** DACI (Driver, Approver, Contributors, Informed) or RAPID when vetoes exist; consultative by default, consent for reversible, not consensus. Disagree and commit with dissent recorded; joint escalation.

**Defense protocol.** Listen → restate → classify (misunderstanding, covered gap, new gap, fact, prediction, value, scope) → conclusion + strongest reason + evidence → concede precisely → "I don't know — here's how and when I'll find out" → park → check closure.

**Disagreement → tool.** Fact → data · Prediction → mechanism, spike, reversible option · Value/risk → goal ranking, owner, decider · Scope → non-goals · Ownership → explicit negotiation · Taste → owning team.

**Stakeholders.** Exec: outcome, cost, risk, options. Principal: trade-offs, failure, evidence. Team: contracts, scope, ownership. SRE: SLOs, alerts, rollback. Security: threat model. Product: user behavior, timeline. Finance: run-rate and drivers.

**AI.** Writing is cheap, review is not. AI for red-teaming, completeness, consistency, prose; never for unverified numbers or alternatives. Human anchors: hardest trade-off, strongest argument against, failure metric. Design doc ≠ agent spec.

**Afterlife.** Significant deviations → doc update or ADR; freeze with as-built note; extract ADRs; supersede, don't delete; retrospective of predictions vs reality.

**Interview.** Formats: take-home + panel, past-design deep dive, live critique, doc-shaped live design, stakeholder role-play. Present a past design in 10 minutes ending with "what I'd change" by mechanism. Critique: verdict → blocking (with fix) → important → questions → strengths. Take-home: hidden constraint, stated assumptions, proportionate design, weakest point named, 4–8 pages.

---
# Appendix A — A design document template

```markdown
# RFC-NNNN: <Short, specific title>

| Field | Value |
|---|---|
| Status | Draft · In review (decision due YYYY-MM-DD) · Approved · Approved with conditions · Rejected · Withdrawn · Implemented · Superseded by RFC-NNNN |
| Owner | <name, team> |
| Approver (decider) | <one name or one named body> |
| Required reviewers | <name/team — the concern each owns> |
| Tier | 1 / 2 / 3 — <one-line justification: reversibility, blast radius> |
| Related | <PRD, incidents, ADRs, prior RFCs> |
| Changelog | <bottom of doc> |

## 1. Summary
<Problem with a number. Recommendation. Main trade-off. Effort and run cost. Decision requested and by when.>

## 2. Context and problem
- Current state (diagram if useful)
- Problem with evidence
- Why now
- Cost of doing nothing (quantified)
- Prior work / prior art

## 3. Goals and non-goals
- Goals (measurable, in priority order): G1 … G2 … G3 …
- Non-goals (things a reader might reasonably assume are in scope): …

## 4. Requirements
- Functional requirements (MUST / SHOULD / MAY)
- Quality attribute scenarios (source, stimulus, artifact, environment, response, response measure), prioritized

## 5. Constraints and assumptions
| Constraint | Source |
| Assumption | Why it matters | Validation | Owner / date |

## 6. Estimates
<Volumes, peaks, sizes, concurrency, storage — with sources; headroom; what breaks first; sensitive inputs>

## 7. Proposed design
7.1 Overview (container-level diagram)
7.2 Components and ownership
7.3 Interfaces and contracts (schemas, versioning rules)
7.4 Data model (storage, partitioning, retention)
7.5 Key flows (sequence diagrams — happy and failure paths)
7.6 Consistency, idempotency, ordering
7.7 Technology choices (one sentence of rationale each)

## 8. Alternatives considered
| Criterion (from goals) | Do nothing | Minimal change | Proposed | Buy | Other |
<For each rejected option: why, and when we would choose it instead.>

## 9. Cross-cutting concerns
9.1 Security (boundaries, identities and roles, secrets, top threats → mitigations)
9.2 Privacy and compliance (data classification, retention, residency)
9.3 Reliability (SLOs, failure-mode table, DR)
9.4 Observability (traces, metrics, logs, alerts tied to SLOs)
9.5 Performance and scalability
9.6 Cost (build and run, drivers, guardrails)
9.7 Operability (ownership, on-call, runbooks)
9.8 Data migration and backfill
9.9 Testing strategy
9.10 Compatibility and versioning

## 10. Rollout, migration and rollback
- Stages and gates (flags, shadow, rings, percentages)
- Migration steps (expand / migrate / contract)
- Rollback triggers, mechanism, decider
- Reversibility table (step · reversible? · how · point of no return?)

## 11. Risks and open questions
| ID | Risk | Likelihood | Impact | Mitigation | Owner | Trigger |
| Q# | Question | Owner | Decide by | Default if undecided |

## 12. Plan, effort and cost
- Milestones with exit criteria
- Effort (range, main uncertainty), staffing
- Dependencies (owner, date)
- Cut lines

## 13. Revisit triggers / kill criteria
<Conditions under which we would choose differently.>

## Appendix
Calculations · Spike and benchmark results (methodology) · Full schemas · Threat model · Glossary · References (dated) · FAQ

## Changelog
- vX.Y (YYYY-MM-DD): <what changed, why, who raised it>
```

---
# Appendix B — Checklists

**Author's pre-review checklist**
- [ ] Summary states problem with a number, recommendation, trade-off, cost and decision requested
- [ ] Problem is evidenced; cost of doing nothing quantified
- [ ] Goals measurable and prioritized; non-goals listed
- [ ] Top quality attributes written as scenarios with response measures
- [ ] Constraints sourced; assumptions have validation plans
- [ ] Estimate shows arithmetic and sensitive inputs; numbers used in the design and cost
- [ ] Alternatives include do nothing, minimal change and buy; steelmanned; rejection reasons and revisit conditions
- [ ] Failure-mode table covers every dependency
- [ ] Security names identities, roles, boundaries and top threats; PII handling stated
- [ ] Observability and alerts tied to SLOs
- [ ] Rollout gated; rollback at every step; points of no return identified and last
- [ ] Risks ranked and owned; open questions have defaults
- [ ] Effort as a range; run cost with pricing date; cut lines
- [ ] Every number labeled measured / estimated / assumed and sourced
- [ ] FAQ answers the objections I can predict
- [ ] One approver, required reviewers with specific asks, decision date
- [ ] Re-read as the most skeptical stakeholder; cut what nobody needs

**Reviewer's checklist**
- Problem pass: Is the problem real and evidenced? Do goals match it? Is the ask clear? *(Stop here if not.)*
- Decision pass: Are alternatives real, including minimal and do-nothing? Criteria from goals? Does the proposal meet its own goals? Is the riskiest assumption validated?
- Detail pass:
  - Consistency: any dual writes? Transactions and boundaries correct? Idempotency and ordering defined?
  - Failure: what happens when each dependency fails, is slow, or returns garbage? Retries bounded with backoff and jitter? Backpressure?
  - Capacity: does the math hold? What saturates first? Headroom?
  - Security: authentication and authorization at every hop? Secrets eliminated? Least privilege? PII in logs?
  - Data: ownership, schema evolution, migration, backfill, retention, restore?
  - Rollout: flags, gates, rollback, irreversible steps last?
  - Operability: who's on call, what pages, runbooks, dashboards?
  - Cost: build and run, drivers, guardrails?
  - People: can this team build and run it on this timeline?
- Feedback: labeled (issue / question / suggestion / nitpick / praise; blocking or not), anchored to goals, "yes, if" with what would unblock, explicit verdict.

---

*Next: **Module 31 — ADRs and the C4 model**, the durable forms of the decisions and diagrams a design document produces.*
