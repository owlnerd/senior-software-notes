# Module 2 — Senior IC vs. Architect: Calibrating Your Prep to Your Actual Target
*Phase 1: Foundations & Calibration · Senior/Architect Interview Prep for .NET & C#*

> **State of the world verified on October 8, 2026.** The reasoning in this module is stable — it is job analysis and resource allocation. What moves is the vocabulary employers use and the shape of particular loops. The facts that calibrate the module:
>
> - **Nobody has standardised the senior-plus loop, and the people who design them say so.** Will Larson's StaffEng guide on staff-plus interview processes opens by stating that no one is confident their staff-plus loop works well, and names the recurring failure modes: treating a staff candidate as "a senior engineer, but better" (faster at everything), "a senior engineer, but worse" (forgiving slower coding without testing what staff engineers are actually good at), and title inflation without changed expectations. Practical meaning: you can't infer the loop from the title — you have to ask.
> - **Vendor-side "architect" titles are customer-facing roles.** Microsoft's *Cloud Solution Architect – Cloud & AI* postings (May 2026) describe a **senior, customer-facing** role that leads strategic technical engagements, **advises C-level executives** and partners with Customer Success leaders; they ask for **6+ years in a customer-facing role** and a cloud certification, and are levelled **IC4/IC5** in a "Cloud Solution Architecture" job family. AWS Solutions Architect loops are described (Exponent's guide; many candidate reports) as **4–6 interviews plus a ~30-minute technical presentation on a problem you solved**, heavy on Amazon's Leadership Principles, with technical rounds on cloud architecture rather than algorithmic coding. These are different jobs from an "architect" in a product company's engineering org.
> - **Certifications are track signals, not track requirements.** **Azure Solutions Architect Expert** = pass **AZ-305** *and* hold **Azure Administrator Associate (AZ-104)** — AZ-305 has no registration prerequisite, but the Expert title isn't awarded without AZ-104. **AZ-204 retired on 31 July 2026** (replaced by **AI-200**). **TOGAF Standard, 10th Edition** certifications are *TOGAF Enterprise Architecture Foundation* and *Practitioner* (a combined Part 1/2 exam exists). **iSAQB CPSA-F** (software architecture, strong in German-speaking Europe) runs on curriculum **2025.1** (mandatory for courses since 1 April 2025), and the Advanced Level now includes an **SWARC4AI** (Software Architecture for AI Systems) module — 18 Advanced modules in total.
> - **The market is described as bifurcated.** Several 2026 market reports agree directionally that **senior and specialised roles are in strong demand while entry-level hiring has contracted**, and that AI skills appear in a large and growing share of postings (one aggregator says ~42%; Dice's May 2026 report, using a broader definition, ~71%). These are low-grade sources (C–D in Module 39's terms) and the numbers disagree — use the *direction*, not the figures. The practical consequence is concrete: both loop types now contain AI questions — AI-assisted coding rounds in some IC loops; "how would you govern AI-generated code / architect an LLM feature / control its cost" in architect loops.
> - **The .NET support cliff shapes architect conversations this quarter.** **.NET 8 and .NET 9 reach end of support on 10 November 2026**, as does the **Azure Functions in-process model**; **.NET 10** is the LTS. Enterprise solution-architect loops in .NET shops are disproportionately about modernisation right now — Module 32's material is unusually likely to be the interview.
>
> Titles, certifications and market numbers change; the method — find the job the loop samples, then spend your hours where that loop scores — doesn't.

## Orientation

Here is the sentence to carry through the whole module: **senior IC loops and architect loops sample different jobs — the IC loop predicts whether you can design and build a system well; the architect loop predicts whether you can frame, make, record and get an organisation to execute a technical decision — so the first act of preparation is naming which job you're targeting, at what level, in what kind of organisation, and then allocating your preparation hours in proportion to what that loop actually samples.**

The curriculum entry reads: *Senior IC vs. Architect: calibrating your prep to your actual target.* Part 0 of the curriculum gave the one-table version:

| | Senior / Staff IC | Architect |
|---|---|---|
| Core rounds | Coding screen, 1–2 system design rounds, a deep technical round, behavioral | Design-document review, a brownfield/incremental-redesign exercise, a round with the engineers who'd implement the decision, a stakeholder trade-off conversation |
| What's judged | Can you design and build it | Can you decide, document it, and get people to align behind it |
| Common failure mode | Weak trade-off reasoning, no numbers to back decisions | A technically sound design that ignores team size, delivery constraints, or cost |

This module unpacks that table from first principles: what each title actually means, why each loop has the rounds it has, how the same rubric is weighted differently, how to locate yourself honestly, and how to turn all of it into an hour-by-hour preparation plan.

**Why this module exists.** Three facts make calibration worth doing before anything else:

1. **Titles don't identify jobs.** "Architect" can mean a staff engineer at a product company, a solution architect on a bank's modernisation programme, an enterprise architect who owns the application portfolio, or a pre-sales engineer at a cloud vendor. "Senior" means three different scopes at three different companies. If you prepare for the title, you prepare for the wrong loop about half the time.
2. **The loops really are different instruments.** A Big Tech staff loop can contain two coding rounds; an enterprise-architect panel usually contains none. An AWS SA loop puts a presentation and Leadership Principles at the centre; a product company's senior loop puts system design depth there. The same candidate, with the same skills, can be a clear hire in one and a clear no in another.
3. **Your preparation hours are finite, and misallocation is the most common silent failure.** Forty hours of timed coding practice is excellent preparation for a staff loop at a large tech company and nearly worthless for an enterprise-architect panel — where the same forty hours spent on a design-document exercise, a brownfield migration story and a stakeholder role-play would have changed the outcome.

How this connects to the rest of the curriculum:

- **Module 1** (rubric literacy) told you *what* senior loops score. This module tells you *how the weights shift* between the IC and architect families, and how to find out which family you're in.
- **Modules 3–5** (the system design method) are shared by both tracks — but the architect version of each step looks different (Concept 11).
- **Phase 7 (Modules 30–33)** is the architect-specific track: design documents, ADRs and C4, brownfield, cost and build-vs-buy. This module decides how much of your time goes there.
- **Modules 34–35** (STAR and the story bank) are where level is decided; this module tells you *which altitude* to tell stories at.
- **Module 36** (coding rounds) — how much of it you need depends almost entirely on the target chosen here.
- **Module 38** (mocks and the six scoring dimensions) and **Module 39** (company research and the loop map) are the execution layer for the decision you make here.

Why it matters in interviews:

1. **"Why architect?" / "Why not management?" / "Are you still hands-on?" are standard questions** — and the strong answers come from someone who has actually thought about the differences between the tracks.
2. **Level is decided by evidence at the right altitude.** Telling senior-shaped stories in a staff or architect loop is the most common reason experienced candidates are down-levelled; telling architect-shaped stories in a senior IC loop reads as "doesn't build any more".
3. **The deliverable differs inside the round.** In an IC design round the deliverable is a design; in an architect round it is a decision with its path to adoption. Candidates who deliver the wrong artefact score poorly even when the content is good.

This module has five jobs:

1. **Define the roles precisely** — what architecture is, the IC ladder, the architect family, where each role lives organisationally, and the adjacent tracks loops confuse you with (Part A).
2. **Explain the two evaluation shapes** — why loops contain the rounds they do, the anatomy of each loop family, the same problem answered both ways, the weighting of the six dimensions, and the characteristic failure modes (Part B).
3. **Locate yourself** — an evidence inventory, an honest fit check, the hands-on question, and the 2026 market reality (Part C).
4. **Turn it into a plan** — the target statement, preparation allocation by target, dual-targeting without dilution, positioning, and re-calibration (Part D).
5. **Give you the tools** — worked examples, model answers, a prep-allocation tool in C#, worksheets and checklists (examples and appendices).

Seven framings to carry through:

1. **A loop is a work sample.** It samples the tasks of the job. Prepare for the job it samples, not the title it carries.
2. **Read verbs, not titles.** Scope, ambiguity and influence describe a job; titles only label it.
3. **Same dimensions, different weights.** Both families score framing, depth, judgement, communication, collaboration and level — in very different proportions.
4. **Different deliverables.** IC: a design that would work. Architect: a decision that would be adopted.
5. **Technical credibility is non-negotiable on both tracks.** Architects lose it in interviews faster than they expect — through one vague answer about how something actually works.
6. **Choose a primary target.** Hedge deliberately, with a stated plan — never by default.
7. **The level you can defend is set by your evidence.** Calibrate to what your stories prove, not what your title says.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What architecture is | The decisions that are expensive to change — and whoever owns them is doing architecture |
| 2 | The IC ladder | Senior → staff → principal: scope, ambiguity and influence grow; archetypes differ |
| 3 | The architect family | Software, solution, enterprise, domain and vendor architects are different jobs |
| 4 | Where the role lives | Organisation type decides which "architect" exists and what it does |
| 5 | Adjacent tracks | Tech lead, engineering manager, TPM — know why you're not those |
| 6 | Loops as work samples | Rounds follow from the job's core tasks; "watch them do the work" |
| 7 | The senior/staff IC loop | Coding, design, depth, behavioral; level set by design and behavioral |
| 8 | The architect loop | Design doc, brownfield, implementers, stakeholders, presentation, credibility probe |
| 9 | Vendor and consultancy loops | Customer communication, presentation, breadth, business outcomes |
| 10 | Hybrid and ambiguous loops | Hands-on architect, staff-with-architect-title; detect by asking |
| 11 | Same prompt, two deliverables | A design vs. a decision with an adoption path |
| 12 | Same dimensions, different weights | D1–D6 re-weighted per target |
| 13 | Failure modes and crossover errors | What sinks each track — and what sinks you in the other track's loop |
| 14 | Your evidence inventory | Six axes; the level you can defend is where your stories are |
| 15 | Fit and energy | Time split, meetings, authority vs. influence, horizon, customers |
| 16 | The hands-on question | Technical currency, demonstrated, not claimed |
| 17 | The 2026 market | Bifurcated demand; AI questions in both loop families |
| 18 | The target statement | One sentence that names role, level, org, archetype, loop and location |
| 19 | Allocating preparation | Relevance × gap, per curriculum phase |
| 20 | Dual-targeting without dilution | Shared core first; track-specific layers; altitude switching |
| 21 | Positioning and narrative | CV, pitch, title translation, cross-track levelling |
| 22 | Re-calibration | Use real outcomes to move the target, not just the prep |

---
# Part A — What the titles actually mean

## Concept 1 — What architecture is: the decisions that are expensive to change

Start from first principles, because every later distinction depends on it.

A software system is the accumulated result of thousands of decisions. Most are cheap to reverse: a variable name, a method's internal algorithm, the order of fields in a DTO. Some are expensive: the boundaries between services, the database engine, the consistency model, the identity provider, the message contract between teams, the hosting platform, the tenancy model. The difference is the **cost of change** — how much work, risk and coordination it takes to reverse the decision later.

Martin Fowler's well-known IEEE Software column *Who Needs an Architect?* builds on an exchange with Ralph Johnson, who rejected the usual definitions ("the highest-level components", "the decisions made early") and offered two better ones: architecture is **the shared understanding the expert developers have of the system design**, and it is about **the decisions you wish you could get right early**. His summary — *architecture is about the important stuff, whatever that is* — sounds glib but carries the whole idea: the core architectural skill is **recognising which decisions are important** (expensive to change, high in consequence) and keeping those in good condition.

Derive the work from the definition. If architecture is the set of expensive decisions, then "doing architecture" consists of five activities:

| # | Activity | What it means in practice |
|---|---|---|
| 1 | **Identify** | Notice which decisions are expensive and consequential — often before anyone else does |
| 2 | **Frame and decide** | Turn a vague pressure into explicit options, criteria and trade-offs; make or facilitate the decision |
| 3 | **Record** | Write it down so the reasoning survives (Modules 30–31: design docs, ADRs) |
| 4 | **Ensure execution** | Get the decision implemented as intended — by teams that weren't in the room |
| 5 | **Revisit** | Detect when the context has changed enough that the decision should be reopened |

This list is the key to the whole module, because **different roles own different slices of it**:

- A **senior engineer** does all five, but within one team's system and mostly for decisions that team can make alone.
- A **staff engineer** does all five across teams, with heavy emphasis on 1, 2 and 4 — finding the problems that matter and getting multiple teams to move.
- A **solution architect** concentrates on 2 and 3 for a programme or project — the target design, its integration and NFRs, its documentation — and depends on others for 4.
- An **enterprise architect** concentrates on 1 and 5 across the whole portfolio — standards, roadmaps, rationalisation — and on mechanisms for 4 (governance that teams don't route around).
- A **vendor solutions architect** does 2 for *customers'* systems, using the vendor's products, with 4 belonging to the customer.

### Concentrated vs. distributed architecture

Organisations answer one structural question differently: **who owns the expensive decisions?**

| | **Concentrated** | **Distributed** |
|---|---|---|
| Who decides | A named architect or architecture function | Teams, with staff/principal engineers as cross-team owners |
| How decisions are recorded | Architecture documents, review boards | RFCs, ADRs per team, design reviews |
| Typical in | Enterprises, regulated industries, consultancies, large programmes | Product companies, scale-ups, Big Tech engineering orgs |
| Risk if done badly | Ivory tower: decisions nobody follows; slow approval | Fragmentation: inconsistent decisions; nobody owns cross-cutting concerns |
| What "architect" means | A role with that title | Usually a *responsibility* of senior-plus ICs; the title may not exist |

Gregor Hohpe (*The Software Architect Elevator*) puts the modern stance in one line: architects make more impact by making **fewer** decisions — by making everyone else's decisions better through clear trade-offs and decision discipline, rather than by deciding everything themselves. That stance is now what good interviewers on *both* tracks look for: an architect who centralises every decision reads as a bottleneck; a staff engineer who never takes a cross-team decision reads as a senior engineer.

### Why this matters for preparation

Once you see architecture as **a set of activities that different roles slice differently**, the loops stop looking arbitrary. A loop for a role that owns activities 1, 2 and 4 across teams (staff) will test design depth and influence. A loop for a role that owns 2 and 3 for a programme (solution architect) will test framing, documentation and integration breadth. A loop for a role that owns 1 and 5 across a portfolio (enterprise architect) will test strategy, governance and stakeholder management. You can predict most of a loop from which slice the role owns.

**Interview-grade sentence** *(for "what's the difference between a senior engineer and an architect?")*: *"I think of architecture as the set of decisions that are expensive to change, and the work as five activities — spotting which decisions matter, framing and making them, recording them, making sure they're implemented as intended, and noticing when they need reopening. Every senior role does some of that; what differs is the scope and which activities you own — a senior engineer does all five inside one team's system, a staff engineer across teams, a solution architect mostly frames and documents for a programme, and an enterprise architect mostly identifies and revisits across the portfolio."*

---

## Concept 2 — The IC ladder: senior, staff, principal

The **individual contributor (IC) ladder** is the career path for engineers who grow in technical scope without managing people. Its levels are defined, at companies that publish frameworks, by **scope, collaborative reach and levers for impact** — Dropbox's public Engineering Career Framework uses almost exactly those words, and lists the ladder IC1–IC4 *Software Engineer*, IC5 *Staff*, IC6 *Principal*, IC7 *Senior Principal*. Titles vary; the progression doesn't.

Module 34 developed six axes of seniority — **scope, ambiguity, influence, impact, time horizon, leverage**. Here is the ladder on those axes, compressed:

| Axis | Senior | Staff | Principal |
|---|---|---|---|
| **Scope** | A feature, a service, a team's system | Several systems or teams; an area | Multiple orgs; the company's core technical bets |
| **Ambiguity** | Given a problem, finds the solution | Given a goal, finds the problem | Decides which problems the company should work on |
| **Influence** | Within the team; persuades peers | Across teams without authority | Executives and many teams; sometimes industry |
| **Impact** | Delivered outcomes | Outcomes across teams; changes in trajectory | Company-level outcomes; strategy others execute |
| **Horizon** | Weeks to quarters | Quarters to a year or two | Multi-year |
| **Leverage** | Does the work; mentors | Multiplies others; changes how teams work | Develops staff engineers; shapes technical culture |

Three properties of the ladder matter for interviews.

**1. Senior is often a career level.** Many companies treat senior as a level you can hold indefinitely ("terminal") — there is no expectation to progress. Staff is not "senior plus five years"; it's a different job with more organisational work and, usually, less personal coding. That's why Larson's "a senior engineer, but better" failure mode exists: loops that test staff candidates on senior work (faster coding, more components) measure the wrong thing.

**2. Staff roles come in shapes — the archetypes.** Larson's four **staff archetypes** (Modules 34 and 39 used them) describe the common shapes:

| Archetype | What they do | What their loop tends to emphasise |
|---|---|---|
| **Tech Lead** | Guides the approach and execution of one team or a few, partnering with a manager | Delivery of complex multi-team work; design depth; coaching |
| **Architect** | Owns direction and quality across a critical technical area | Cross-team design; standards; migrations; decision records |
| **Solver** | Goes deep on the hardest problems wherever they are | Deep technical rounds; debugging; incident and performance stories |
| **Right Hand** | Extends an executive's reach across the organisation | Strategy, cross-cutting programmes, organisational problem-solving |

Notice that **the staff "Architect" archetype and the job titled "architect" overlap heavily** — which is why some product companies call this role "Staff Engineer" and others call it "Software Architect", with nearly identical loops.

**3. The "IC" label is misleading at the top.** Pat Kua's **Trident model** argues that the conventional two-track ladder (management vs. "individual contributor") misnames the second track, because most organisations need *technical leadership*, not individual contribution, at senior-plus levels. He proposes three tracks by where people spend 70–80% of their time:

| Track | Time mostly spent on | Example roles |
|---|---|---|
| **Management** | Leading and supporting people; structures and processes | Engineering manager, VP Engineering |
| **Technical leadership** | Leading people on a technical topic: vision, risk, requirements, trade-offs explained to non-technical stakeholders | Tech lead, principal engineer, **software architect**; the Tech Lead, Architect and Right Hand archetypes |
| **True individual contributor** | Executing — deep, specialist doing | Performance specialist, database specialist, distinguished engineer; the Solver archetype |

The Trident model is the cleanest way to see where the two families of this module sit: **most architect roles and most staff roles are on the technical-leadership track**. They differ in scope, organisational context and deliverable — not in whether they lead.

**Interview-grade sentence** *(for "what does staff mean to you?")*: *"Staff isn't senior with more speed — it's a different job. A senior engineer owns a system and delivers within a team; a staff engineer finds the problems that matter across several teams and gets those teams to move, usually without authority, over a horizon of quarters to a couple of years. It comes in shapes — tech lead, architect, solver, right hand — and I'd want to know which one this role is, because that decides what good looks like."*

---

## Concept 3 — The architect family: five different jobs with one word in the title

"Architect" is a family of roles, not a role. Five members account for nearly all the senior architect openings a .NET/Azure engineer will see.

### 1. Software / application architect

Owns the architecture of **one product or system** (or a few closely related ones). Usually sits in or next to the engineering teams that build it, reviews designs and pull requests, writes ADRs, sets the technical direction of the codebase and often still codes. In product companies, this is effectively the staff "Architect" archetype under another title. In mid-size companies it's often the most senior technical person on a product.

### 2. Solution architect

Designs the **solution to a business problem** — typically a programme or project — across systems, integrations, vendors and non-functional requirements. Bridges business stakeholders and delivery teams: turns requirements into a target design, writes the solution architecture document, specifies integrations and NFRs, supports delivery through to go-live. Common in enterprises, consultancies and Microsoft partners. Scope is programme-shaped: it starts, runs and ends. Hands-on coding is usually limited, but technical depth in the platform (for you: .NET and Azure) is expected.

### 3. Enterprise architect

Owns the **portfolio**: business capabilities, the application landscape, technology standards, the target architecture and the roadmap to it. Works with business leaders, the CIO/CTO, procurement, risk and compliance. Frameworks such as **TOGAF** live here (its four domains — business, data, application, technology — are EA's vocabulary). Fowler's description is apt: much of enterprise architecture is understanding **what is worth the cost of central coordination, and what form that coordination should take** — too much and you get a bottleneck board; too little and you get duplication and systems that can't interoperate. Success is measured in rationalisation, risk and cost reduction, and business change enabled. Rarely hands-on.

### 4. Domain architects

Specialists across the portfolio in one domain: **data architect**, **security architect**, **integration architect**, **infrastructure / cloud / platform architect**. They combine solution-architect breadth within the domain with standards-setting across it. For a .NET/Azure engineer, *cloud architect* and *integration architect* (Service Bus, APIM, Event Grid, Logic Apps) are the most common adjacent openings.

### 5. Vendor and customer-facing architects

At cloud vendors and large partners: AWS **Solutions Architect**, Microsoft **Cloud Solution Architect (CSA)**, partner **pre-sales architect**. They design solutions for **customers'** problems using the vendor's platform, advise customer executives, run workshops and proofs of concept, and are measured partly on business outcomes such as adoption or consumption. Breadth across the vendor's portfolio and customer communication matter more than depth in one stack. Microsoft's current CSA postings explicitly ask for years in a customer-facing role.

### The family side by side

| | Software / application | Solution | Enterprise | Domain (cloud, data, security, integration) | Vendor SA / CSA |
|---|---|---|---|---|---|
| **Scope** | One product/system | A programme or project across systems | The portfolio | One domain across the portfolio | Customers' systems on the vendor's platform |
| **Horizon** | Quarters–years | Programme length (months–2 years) | 3–5 years | 1–3 years | Engagement-length; account-length |
| **Main stakeholders** | Engineers, EMs, product | Business owners, delivery teams, vendors | Business leaders, CIO/CTO, risk, procurement | Domain teams, security/risk, platform | Customer architects and executives, account team |
| **Hands-on** | Often codes; reviews code | Rarely codes; deep platform knowledge | Rarely | Varies; often prototypes | Prototypes, demos, PoCs |
| **Primary artefacts** | ADRs, design docs, code | Solution architecture documents, integration specs, NFRs | Capability maps, target architecture, roadmaps, standards | Reference architectures, standards, patterns | Reference designs, workshops, PoCs, presentations |
| **Success measured by** | System quality and evolvability; team velocity | Delivered solution meeting requirements on time and budget | Portfolio outcomes; governance that teams use | Domain risk and consistency | Customer outcomes; adoption/consumption |
| **Typical loop centre** | Design depth + decision-making | Design-document/case exercise + stakeholder rounds | Strategy case + governance + stakeholder panels | Domain depth + design | Presentation + customer scenario + behavioral |
| **Common certification signal** | Rarely required | AZ-305 (Azure Solutions Architect Expert); sometimes TOGAF | TOGAF Practitioner; sometimes ArchiMate | AZ-305, security/data certifications | Vendor certifications (AWS SA, AZ-305) |

Two cautions on that table. First, **the boundaries are soft** — a solution architect at a smaller company may do application and enterprise work at once. Second, **certifications are signals of the employer's world, not of competence**: a JD that demands TOGAF is telling you it lives in a concentrated, governance-heavy architecture culture; a product company that doesn't mention certifications is telling you architecture is a responsibility of senior engineers.

**Interview-grade sentence** *(for "what kind of architect are you?")*: *"'Architect' covers several jobs — an application architect owns one system and stays close to the code; a solution architect designs the answer to a business problem across systems for a programme; an enterprise architect owns the portfolio and the roadmap; and at vendors it's a customer-facing advisory role. My experience is mostly [X], so the closest fit is [Y] — and I'd want to confirm which one this role really is, because the day-to-day is very different."*

---

## Concept 4 — Where the role lives: organisation type decides which architect exists

The same title is a different job in different organisations, because **the organisation's structure and economics decide who owns the expensive decisions** (Concept 1). Four organisation types cover most of the market for senior .NET/Azure roles.

| | **Product company / scale-up** | **Enterprise (bank, insurer, retailer, public sector)** | **Consultancy / Microsoft partner** | **Cloud vendor** |
|---|---|---|---|---|
| How it makes money | The software itself | Its core business; software supports it | Selling engineers' time and delivery outcomes | Platform consumption and licences |
| Engineering is | The product (profit centre) | IT (often a cost centre) | The product being sold | The platform and the field organisation |
| Architecture model | **Distributed** — staff/principal engineers, RFCs, ADRs | **Concentrated** — architecture function, review board, solution architects per programme | **Engagement-based** — architect per client engagement; pre-sales | **Field + product** — customer-facing SAs/CSAs; product engineering separately |
| What "architect" usually means | Staff-level technical leader (title optional) | Solution / enterprise / domain architect | Solution architect who also sells and estimates | Customer-facing advisor |
| Typical loop | IC loop with heavier design and behavioral; sometimes a staff presentation | Architect panel, case study, stakeholder rounds; light or no coding | Case interview, client scenario, estimation, presentation | Presentation, customer scenario, cloud breadth, values/LPs |
| How architecture is justified | "What does this enable?" — product speed and scale | "What does this cost and what risk does it remove?" | "What will the client buy, and can we deliver it profitably?" | "What customer outcome does this drive?" |

Module 39 (Concept 6) made the same cut from the research side; here the point is preparation. **Before you prepare, place the target in one of these columns** — it predicts the loop shape better than the title does.

### Conway's law as a predictor

Melvin Conway's observation — organisations produce systems that mirror their communication structure — also predicts the **architect's job**. If teams are aligned to business capabilities and own services end to end, architecture is mostly about the contracts between teams and the platform beneath them (a staff-shaped job). If teams are organised by layer or by project, with a central architecture group, architecture is mostly about coordinating handoffs and governing the target state (a solution/enterprise-shaped job). In an interview, the structure you're told about tells you which answers will resonate.

### The architect's altitude range

Hohpe's **Architect Elevator** metaphor describes the modern architect as someone who rides between the **penthouse** (executives, strategy, money) and the **engine room** (code, infrastructure, operations), translating in both directions. Different roles work at different floors:

```text
PENTHOUSE   strategy, budgets, risk, business capabilities      ← enterprise architect, principal, vendor CSA with executives
  ...       programme goals, roadmaps, vendor choices          ← solution architect, staff "right hand"
  ...       cross-team contracts, platform, standards          ← staff "architect", domain architect
ENGINE ROOM code, services, data, runtime, incidents           ← senior engineer, staff "solver"/"tech lead", application architect
```

The best candidates in either track can **travel** — a staff candidate who can explain a decision in cost and risk terms to a director, an enterprise architect who can answer a precise question about how Service Bus sessions preserve ordering. Interviewers test the floors *adjacent* to your role: IC loops probe one floor up ("how would you justify this to leadership?"); architect loops probe the engine room ("how would that actually behave under load?").

**Interview-grade sentence** *(for "how do you see the architect's role here?")*: *"It depends a lot on the organisation. In a product company architecture is usually distributed — senior and staff engineers own the important decisions and the job is making those decisions good and consistent across teams. In an enterprise it's usually concentrated in an architecture function, and the job is framing decisions for programmes and running governance that teams actually use. I'd want to understand which model you run, because the same title means quite different work."*

---

## Concept 5 — Adjacent tracks: tech lead, engineering manager, TPM

Senior loops often probe whether you're really after a *different* job. Three adjacent roles get confused with yours, and knowing the boundaries lets you answer "why not X?" crisply.

| Role | Owns | Measured by | Overlap with staff/architect | The usual interview probe |
|---|---|---|---|---|
| **Tech lead** | A team's technical approach and execution | The team's delivery and technical quality | High with the staff Tech Lead archetype; often a *role* held by a senior or staff engineer, not a level | "How do you split technical decisions with your EM?" |
| **Engineering manager** | People: hiring, growth, performance, team health; delivery commitments | Team outcomes and people outcomes | Shared interest in delivery and team structure; staff partners with EMs | "Why not management?" |
| **Technical program manager (TPM)** | Cross-team programme execution: plans, dependencies, risk, status | Programme delivered on time | Shared with architects on cross-team coordination | "How is what you do different from a TPM?" |

Two pieces of reasoning help in interviews.

**The engineer/manager pendulum.** Charity Majors' widely read essay *The Engineer/Manager Pendulum* argues that the strongest technical leaders often swing between engineering and management over a career, and that the two paths reinforce each other. You don't have to have managed to be a strong staff or architect candidate — but if you have, it's an asset, framed as *why you chose the technical leadership track now*, not as a step down.

**The TPM distinction.** A TPM is accountable for *the plan being executed*; an architect is accountable for *the decision being right and staying right*. Architects who describe their work mostly as status, dependency tracking and timelines sound like TPMs. Architects who describe options, criteria, trade-offs and the mechanisms that kept the decision intact sound like architects.

### Answering "why not management?"

A strong answer has three parts: what you've learned about **where your leverage is**, a **concrete example** of having that leverage as an IC, and **respect** for the management track.

> *"I've thought about it — I've done the people side informally, mentoring two engineers to senior and running hiring for the team. What I learned is that my leverage is biggest on the decisions that are expensive to change: the async guidelines I introduced across our services removed a whole class of incidents, and that work doesn't need me to manage people to happen. I'd rather partner closely with a strong EM than be one, at least for the next few years."*

Notice it's toward something (leverage on decisions), evidenced (a mechanism with an outcome), and not dismissive of management.

**Interview-grade sentence:** *"The tech lead owns how one team builds, the engineering manager owns the people and the team's commitments, a TPM owns whether a cross-team plan is executed — and a staff engineer or architect owns whether the important technical decisions are right, recorded and stay right. I work closely with all three, but that last one is the job I want."*

---
# Part B — The two evaluation shapes

## Concept 6 — A loop is a work sample

Why does a staff loop contain a system design round and an enterprise-architect loop contain a stakeholder panel? Because a well-designed loop is an attempt to **observe a sample of the job's real work** under controlled conditions.

Larson's guidance for designing staff-plus loops states the principle directly: reason **forward from the signals you need** to the interview formats, and prefer **watching people do the work** over asking them about it — if mentoring matters, watch them mentor; if architecture matters, present your real systems and see how they react to decisions they disagree with. Personnel-selection research points the same way: structured interviews and work-sample exercises are among the better predictors of job performance, and unstructured conversation is among the weaker ones. Companies don't always design loops well, but the *intent* is almost always a work sample.

So you can derive the likely rounds from the job's core tasks:

| Core task of the job | Round that samples it | Present in |
|---|---|---|
| Write correct, maintainable code under constraints | Coding round | Senior IC (always), staff IC (usually), hands-on architect (often), others (rarely) |
| Design a system from ambiguous requirements | System design round | All IC; most software/solution architect loops |
| Know a technical area deeply | Deep technical / domain round | IC; domain architect |
| Review others' work | Code review or design review exercise | Staff; application architect |
| Frame and defend a decision in writing | Design-document exercise (take-home or pre-read + defence) | Architect; some staff |
| Change a running system safely | Brownfield / migration exercise | Architect; staff at mature companies |
| Get implementers to own a decision | Round with the engineers who'd build it | Architect; some staff |
| Negotiate trade-offs with non-engineers | Stakeholder conversation (product, finance, security) | Architect; senior staff/principal |
| Communicate to a group | Presentation of past work or a set problem | Staff (often), vendor SA (always), architect (often) |
| Demonstrate scope and behaviour over years | Behavioral / leadership / project deep-dive | All |

### The prediction you can make

Write down the role's top five tasks (from the JD verbs and the recruiter — Module 39, Concepts 11 and 16). Map each to the table. **That's your predicted loop**, and it's usually right about the round *types* even when you don't know the exact count. Then confirm with the recruiter.

### The implication for preparation

Practise the *task*, not the topic. Reading about stakeholder management is topic preparation; running a 30-minute role-play where a finance lead pushes back on your Cosmos DB cost is task preparation. Module 38's mock catalogue is built on this principle; this module tells you which mocks your target needs.

**Interview-grade sentence** *(useful when asked how you'd design a hiring loop — a common staff/architect question)*: *"I'd start from the five or six things the role actually does week to week and design a round that lets us watch the candidate do each one — a design review if they'll review designs, a session with the engineers they'd lead if adoption matters, a written exercise if decisions are made in documents — because watching someone do the work predicts much better than asking them to describe it."*

---

## Concept 7 — The senior and staff IC loop

### Anatomy

The IC loop is the most standardised of the families. A typical senior or staff loop at a product company:

```text
Recruiter call (30)        motivation, logistics, level, range                          Module 39
Technical screen (45–60)   coding, sometimes with light design                            Module 36
Onsite / virtual onsite:
  Coding ×1–3 (45–60)      correctness, code quality, testing, communication               Module 36
  System design ×1–2 (60)  framing, estimation, design, deep dive, trade-offs              Modules 3–13, 37
  Deep technical (45–60)   domain or stack depth — for .NET shops: CLR, async, EF Core,   Phase 4
                           ASP.NET Core; or the team's area (storage, networking, identity)
  Behavioral ×1–2 (45–60)  scope, ownership, conflict, failure, influence                  Modules 34–35
  (Staff) project deep-dive / presentation / code review / mentorship panel
Debrief → levelling → (centralised loops) team matching
```

### What each round contributes to the decision

| Round | Primarily tests | Where level is set |
|---|---|---|
| Coding | Can you produce production-quality code; are you fluent | **Pass/fail gate** for senior and staff; rarely raises level |
| System design | Framing, depth, trade-offs, driving | **Major level signal** — senior vs staff is often decided here |
| Deep technical | Real expertise, precision in your stack | Confirms or undermines the level claimed elsewhere |
| Behavioral | Scope, ambiguity, influence, impact, horizon, leverage | **The levelling round** at senior and above (Module 34) |
| Staff-specific formats | Structured thinking, review quality, mentoring, influence | Staff vs senior |

The asymmetry matters. **Coding is mostly a gate**: you must clear it, but clearing it brilliantly rarely raises your level. **System design and behavioral set the level.** Staff candidates who spend 70% of their preparation on coding are over-investing in the gate and under-investing in the rounds that decide the outcome.

### How expectations scale from senior to staff in the design round

Hello Interview's breakdown of system design expectations by level describes three dimensions that shift — **breadth, depth and proactivity** — and the shift matches what interviewers report:

| | Senior | Staff |
|---|---|---|
| **Driving** | Drives the early phases — requirements, API, data model, high-level design — with occasional steering | Drives the whole session; shapes the discussion; the interviewer mostly probes |
| **Depth** | Real depth in one or two areas with a "been there" quality | Depth in several areas; anticipates the probes before they come |
| **Breadth** | Knows the standard building blocks and their trade-offs | Breadth tapers in importance; judgement about *which* areas matter dominates |
| **Scope of answer** | A dependable service at scale | Whether this should be a platform; how it evolves; who owns it; what it costs |
| **Failure reasoning** | Identifies main failure modes and mitigations | Designs for operability; failure domains; blast radius; rollout |

### What a staff loop adds

Larson's list of formats that work for staff-plus — **structured presentation, code review, data modelling and architecture evolution, subject-matter expertise, mentorship panel** — is a good guide to what you may meet beyond the standard rounds, and to what each is looking for: structured thinking, empathy and clarity in review, evolving a design under changing requirements, real depth, and the ability to turn rough questions from less experienced engineers into useful discussions.

### The .NET-specific deep technical round

At .NET shops the deep technical round is frequently a .NET round. Expect questions that separate users of the platform from people who understand it: why `async void` is dangerous; what causes ThreadPool starvation and how you'd diagnose it; DI lifetime mismatches (scoped into singleton); EF Core change tracking costs and the N+1 problem; GC modes and allocation-conscious design; how ASP.NET Core's middleware pipeline is built. Phase 4 (Modules 14–19) exists for this round. For staff candidates, the twist is that answers are expected to connect to **guardrails across teams** — not just "I'd fix it" but "here's how I'd stop forty services from doing it" (Module 34's ThreadPool story).

**Interview-grade sentence** *(for "what do you expect our loop to test?")*: *"For a senior or staff IC role I'd expect coding as the gate, and system design plus behavioral to set the level — design showing I can drive an ambiguous problem to a dependable, operable system with numbers behind the trade-offs, and behavioral showing the scope I've actually worked at. At staff I'd also expect something that shows cross-team influence — a presentation, a review exercise or a project deep-dive."*

---

## Concept 8 — The architect loop

Architect loops are less standardised than IC loops, but most combine some of the following rounds. For each: what it samples, what's scored, and what failure looks like.

### Round 1 — Design-document exercise and defence

**Format.** Either a take-home ("write a two-to-four-page architecture proposal for this scenario") or a pre-read of the company's own document that you critique — followed by a 45–60-minute defence with two or three architects.

**Samples.** Activities 2 and 3 of Concept 1 — framing a decision, documenting it.

**Scored.** Problem statement and context; explicit options with consequences; decision criteria tied to business drivers; NFRs; risks and mitigations; what you'd *not* do; clarity of writing; how you handle challenges in the defence (concede the right points, hold the right ones). Module 30 is the full treatment.

**Failure looks like.** A solution with no alternatives considered; a document that is all diagram and no reasoning; defensiveness when challenged; or conceding everything.

### Round 2 — Brownfield / incremental-redesign exercise

**Format.** "Here's our current system — a .NET Framework monolith on IIS, one SQL Server, nightly batch integration with an ERP. The business needs X by Q3. What do you do?"

**Samples.** Activity 4 (execution) under real constraints.

**Scored.** Asking about the current state before proposing anything; finding the *urgent* constraint (often a support cliff, a scaling wall or a compliance date); a sequenced plan with measurable gates (strangler fig, anti-corruption layer, dual-run, data migration strategy — Module 32); keeping the business running throughout; cost of delay (Module 33).

**Failure looks like.** The greenfield answer — designing the target state as if the old system didn't exist; a big-bang rewrite; no rollback.

### Round 3 — The implementer round

**Format.** A conversation with the engineers who'd build and run your decision. They push back: "event-driven will make debugging impossible for us", "we don't have Kubernetes skills", "this doubles our on-call".

**Samples.** Getting a decision adopted by the people who weren't in the room.

**Scored.** Module 38 listed it: **listening before defending**; separating concerns that change the **decision** from those that change the **plan** ("that's a real risk; it doesn't change the choice, but it changes the order we build it in"); **visible concessions** when they're right; **ownership transfer** — guardrails and ADRs that let the team decide locally.

**Failure looks like.** A lecture. Technically correct and organisationally fatal — this is the round where most architects who fail, fail.

### Round 4 — Stakeholder trade-off conversation

**Format.** A product lead, finance partner, security lead or business owner with a conflicting goal: "we need it in three months", "cut cloud spend by 30%", "no new data store without a threat model".

**Samples.** Translating between the engine room and the penthouse.

**Scored.** Speaking the stakeholder's language (time, money, risk, customer impact); quantifying; offering options rather than a single answer; making trade-offs explicit and getting a decision; not hiding behind technical jargon; not caving on things that matter (security, data integrity).

**Failure looks like.** Explaining the technology instead of the trade-off; "it depends" without a recommendation; promising the impossible.

### Round 5 — Presentation of past architecture

**Format.** 20–30 minutes on a system you designed, followed by questions. Common at product companies hiring architects, at consultancies and at vendors.

**Scored.** Structure (context → problem → options → decision → outcome → lessons); honest attribution of your role; numbers; how you answer hard questions; what you'd do differently.

### Round 6 — The technical credibility probe

**Format.** Often embedded in other rounds; sometimes its own round — a code review, a "how would you implement this in .NET?" or a deep dive on one component of your proposal.

**Samples.** Whether you still understand the engine room.

**Scored.** Precise, current answers. An architect who proposes Azure Service Bus should know what sessions, duplicate detection and dead-lettering do and cost; one who proposes EF Core for a high-throughput write path should know where change tracking hurts. **One confidently vague answer here can undo a strong loop**, because it confirms the interviewer's biggest worry about architects: the ivory tower.

### Round 7 — Behavioral and values

As in IC loops, but the stories expected are **decision and alignment stories**: a decision you made under uncertainty, a time your design was rejected, a time you changed your mind, governance you introduced that teams actually used (Module 34, Concepts 23–24).

### Coding in architect loops

Algorithmic coding is **rare** in architect loops (and almost absent in enterprise-architect and vendor loops), but **code literacy** is common: reading code in review, sketching an interface, writing a small piece of C# to illustrate a pattern. Hands-on architect roles ("Lead .NET Developer / Architect") are the exception — they often include a full coding round. Ask (Concept 10).

**Interview-grade sentence** *(for "what are you expecting from our architect panel?")*: *"I'd expect to be judged less on whether I can produce a design and more on whether I can make a decision that survives — framing options and criteria in writing, sequencing a change to a running system safely, bringing the engineers who'll build it along with me, and negotiating the trade-offs with product, finance or security in their own terms — plus enough depth that you trust I understand how the pieces actually behave."*

---

## Concept 9 — Vendor and consultancy architect loops

Two variants are common enough for a .NET/Azure engineer to merit separate treatment.

### Vendor solutions architect (AWS SA, Microsoft CSA, partner pre-sales)

**What the job samples.** Advising customers: understanding their business problem, designing a solution on the vendor's platform, communicating it to technical and executive audiences, unblocking adoption.

**Typical loop shape** (graded B–C: guides and many consistent candidate reports; confirm with the recruiter):

```text
(Online assessment, some programmes)
Recruiter screen
Technical phone screen — cloud concepts and architecture, often rapid-fire breadth
Loop (4–6 interviews):
  Technical presentation (~30 min) — a problem you solved; questions after
  Architecture / customer-scenario whiteboard — design for a described customer
  Technical depth — services, networking, security, data, migration
  Behavioral — at Amazon, Leadership Principles in every round (recruiters often say
               two LPs per interviewer; prepare at least two stories per LP)
```

**What's weighted.** Customer communication (D4) and values (D5/D6 through the company's framework) dominate; technical breadth across the platform matters more than depth in one language; algorithmic coding is usually absent.

**Preparation shift for you.** Phase 6 (cloud and platform) moves to the top; you'll need **breadth beyond this curriculum** across the vendor's service catalogue (identity, networking, data, AI services, migration tooling). The presentation is a first-class deliverable — script it, time it, rehearse the Q&A. Phase 8's story bank needs **customer stories**: a time you changed a customer's mind, a time you said no to a customer, a time you turned a failing engagement around.

**A fit warning.** These roles are **customer-facing** and often tied to sales or consumption goals; Microsoft's CSA postings frame the role inside customer success and mention travel. If you want to own a system's architecture over years, a vendor SA role is a different job — sometimes a great one, but different.

### Consultancy and partner architect

**What the job samples.** Winning and delivering client engagements: understanding a client scenario quickly, proposing an approach, estimating, presenting to the client, leading delivery.

**Typical rounds.** A **case interview** (a client scenario with incomplete information — scope it, propose an approach, estimate effort and cost, name risks), a **pre-sales role-play** or presentation, a technical depth round, and culture/values.

**What's weighted.** Structured problem-solving under ambiguity (D1), estimation and commercial awareness (Module 33's language), client communication (D4). Certifications can matter commercially for partners (Microsoft partner programmes reward certified staff), so AZ-305 may be "required" for business reasons — ask whether you'd be supported to obtain it after joining (Module 39, Concept 9).

**Interview-grade sentence** *(for a vendor or consultancy loop)*: *"In a customer-facing architect role, I'd expect the bar to be whether I can understand a customer's real problem quickly, design something credible on the platform, explain it to both their engineers and their executives, and be honest when the platform isn't the right answer — so I've prepared a presentation of a real engagement, a few customer stories, and breadth across the services rather than depth in one."*

---

## Concept 10 — Hybrid and ambiguous loops

Many openings don't fit cleanly into one family. The three most common hybrids:

| Hybrid | What it is | Likely loop | Watch out for |
|---|---|---|---|
| **"Software Architect" at a product company** | Staff-level technical leader with the architect title | IC loop with an extra design round and heavier behavioral; sometimes a presentation | Being tested on coding speed like a senior ("a senior engineer, but better") |
| **"Lead .NET Developer / Architect" in an enterprise or agency** | Player-coach: leads a team, codes, owns the design | Coding test + system design + architecture discussion + behavioral | Under-preparing coding because the title says "architect" |
| **"Staff Engineer" at a company that uses it as title inflation** | A senior role with a staff title | A senior IC loop | Accepting a title the scope doesn't support; being measured later against expectations nobody defined |

### Detecting the real shape

Three sources, in order of reliability (Module 39, Concept 3):

1. **The recruiter's description of the loop, in writing** — the A-grade source. Ask directly: *"Is there a coding round? Is there a written exercise? Who's on the panel — engineers, architects, product, business?"*
2. **The JD's verbs** (Module 39, Concept 11) — "implement", "own services", "set direction across teams", "govern", "work with business stakeholders on target state" each point at a different family.
3. **Who the role reports to.** Reporting to an engineering manager or director of engineering → IC-shaped. Reporting to a chief architect, CTO office or head of enterprise architecture → architect-shaped. Reporting to a sales or customer-success leader → vendor-shaped.

### A quick classifier

```text
Is there a coding round?                      yes → IC or hands-on architect      no → architect, EA or vendor
Is there a written/document exercise?         yes → architect                      no → IC or vendor
Are non-engineers on the panel?               yes → architect, EA or vendor        no → IC or application architect
Is there a presentation?                      yes → staff, architect or vendor     no → senior IC or EA panel
Who does the role report to?                  eng manager → IC · chief architect/CTO office → architect/EA · sales/CS → vendor
How much time is customer-facing?             > 30% → vendor or consultancy
```

Appendix B turns this into a recruiter-call script.

**Interview-grade sentence** *(at the recruiter call)*: *"The title could mean a few different jobs, so I want to prepare for the right one — could you tell me whether the loop includes a coding round or a written exercise, who's on the panel, and who the role reports to? That tells me a lot about whether it's a hands-on technical leadership role or a more governance- and stakeholder-facing one."*

---

## Concept 11 — Same prompt, two deliverables

The clearest way to see the difference between the families is to answer **the same prompt** both ways.

> **Prompt:** *"Our order processing runs on a .NET Framework 4.8 monolith on IIS with one SQL Server. We're expanding to three more EU markets next year and expect peak orders to grow about five times. How would you approach this?"*

### The IC answer: a design

The IC candidate treats the prompt as a design problem and runs Module 3's seven steps.

1. **Requirements.** Functional: place order, pay, reserve stock, confirm, notify. Non-functional: peak 5× today (say 50 → 250 orders/s at peak), p99 checkout latency < 500 ms, no lost or double-charged orders, EU data residency, availability target 99.95%.
2. **Estimation.** Writes per second, row growth, storage per year, message volume per order.
3. **API.** `POST /orders` with an idempotency key; order status endpoint; webhook from payment provider.
4. **Data model.** Orders, order lines, payments, reservations; what's transactional together; partitioning key candidates (market, customer).
5. **High-level design.** ASP.NET Core order service; Azure SQL with read replicas or partitioning by market; Service Bus for order events; outbox for reliable publishing; saga for payment and stock reservation.
6. **Deep dive.** Idempotency and the outbox (Module 11); saga compensation (Module 12); SQL hot spots and partitioning (Module 8); EF Core write-path behaviour (Module 19).
7. **Wrap-up.** Bottlenecks, failure modes, what to do with more time.

**The deliverable is a design that would work.** It is scored on correctness, depth, numbers and trade-offs.

### The architect answer: a decision with an adoption path

The architect candidate treats the prompt as a decision problem in a running organisation.

1. **Current state and constraints first.** *"Before designing anything: how many teams work on the monolith, what's their .NET Core and Azure experience, what's the release process, what's the budget envelope, and when is the first new market live?"* Also: the 10 November 2026 end of support for .NET 8 doesn't apply to .NET Framework 4.8, so there may be no hard platform deadline — urgency comes from the market dates.
2. **Frame the decision as options.**

   | Option | What it is | Fits if | Cost and risk |
   |---|---|---|---|
   | A. Scale the monolith | Vertical scale, caching, read replicas, performance work | 5× is achievable with tuning; teams are small; deadline is close | Cheapest now; doesn't address multi-market change velocity |
   | B. Modular monolith on .NET 10 | Port to .NET 10, enforce module boundaries, extract nothing yet | Teams need to move faster but distribution isn't justified | Moderate; sets up later extraction; one deployable |
   | C. Extract order processing | Strangle the order flow into a separate service with its own data | Order flow is the scaling hot spot and changes most often | Higher; distributed transactions; needs operational maturity |

3. **Criteria tied to drivers:** time to first new market, peak capacity, team capacity and skills, run cost, risk to existing revenue, reversibility.
4. **Recommendation and sequence.** For example: *"B now, with C for the order flow behind a strangler facade only if load tests after the port show the order path is the bottleneck. First milestone: load-test the current system to 5× to find the real wall — we may be designing against a guess."*
5. **Adoption.** Who builds it; how the teams are involved in the design; an ADR per major decision (Module 31); guardrails (module-boundary tests, an architecture fitness function); what the architecture review checks.
6. **Cost and risk** (Module 33): run-cost delta, migration effort in team-quarters, cost of delay per market.
7. **Revisit triggers.** *"If the second market's regulatory requirements force per-market data isolation, reopen the data-partitioning decision."*

**The deliverable is a decision that would be adopted** — scored on framing, options, criteria, sequencing, stakeholder and team awareness, and the honesty of the trade-offs.

### Side by side

| Step | IC answer leads with | Architect answer leads with |
|---|---|---|
| Opening | Functional and non-functional requirements | Current state, teams, constraints, dates |
| Middle | Boxes, arrows, data model, deep dive | Options, criteria, recommendation, sequence |
| Numbers | QPS, storage, latency | Those **plus** cost, effort, time-to-market |
| Depth | One or two components in real depth | Enough depth to prove the options are real; deep on the riskiest |
| People | Implicit | Explicit — who builds, who runs, who decides |
| Ending | Bottlenecks and failure modes | ADRs, rollout gates, revisit triggers |

**Both answers need the other's content in small doses.** An IC staff answer gains a lot from one sentence of migration and ownership; an architect answer collapses without at least one component explored at real depth. Module 37's Concept 38 ("The architect's version") applies this to five classic problems.

**Interview-grade sentence** *(when an interviewer asks "is this a design question or an architecture question?")*: *"I'd treat it as both, in proportions that depend on what you want to see — I can design the target system in depth, but in a real organisation the harder question is which option to take given the teams, the dates and the cost, and how to get there without breaking what's running. Which would be more useful to focus on?"*

---

## Concept 12 — Same dimensions, different weights

Module 38 consolidated what loops score into six universal dimensions:

- **D1 — Framing and scoping**: turning ambiguity into a well-defined problem; asking the right questions.
- **D2 — Technical depth** (with **D2b — stack precision**: exact, current knowledge of your platform).
- **D3 — Trade-offs and judgement**: options, criteria, consequences, numbers; knowing when *not* to.
- **D4 — Driving and communication**: running the conversation; clarity for the audience.
- **D5 — Collaboration and coachability**: listening, integrating input, conceding and holding appropriately.
- **D6 — Level signal**: scope, ownership, impact — evidence of operating at the target level.

Every family scores all six. What changes is the **weight**. The table below is a calibrated estimate — synthesised from published rubrics, interview guides and interviewer reports, not a published dataset — so treat the numbers as relative emphasis, not precise percentages.

| Dimension | Senior IC (product) | Staff IC | Software / solution architect | Enterprise architect | Vendor SA / CSA |
|---|---|---|---|---|---|
| D1 Framing & scoping | 15% | 15% | 20% | 20% | 15% |
| D2 Technical depth (+D2b) | **30%** | 20% | 15% | 10% | 15% |
| D3 Trade-offs & judgement | 20% | **20%** | **25%** | 20% | 15% |
| D4 Driving & communication | 15% | 15% | 15% | 20% | **30%** |
| D5 Collaboration & coachability | 10% | 10% | 15% | 15% | 15% |
| D6 Level signal | 10% | **20%** | 10% | 15% | 10% |

Reading the table:

- **Senior IC** is the only family where **technical depth** is clearly the largest weight. Precision about how things work — including .NET runtime behaviour — is the main differentiator.
- **Staff IC** spreads weight evenly and adds a large **level signal**: the question is less "is this person good?" than "is this person operating at staff scope?".
- **Software/solution architects** are judged most on **trade-offs and judgement** — the quality of the decision.
- **Enterprise architects** shift weight to **framing** and **communication** with business stakeholders; depth matters, but at the level of "does this person understand the technology well enough to govern it?".
- **Vendor SAs** are judged most on **communication** — with customers, executives and in presentations.

### What "strong" looks like per dimension, by family

| Dimension | Strong in an IC loop | Strong in an architect loop |
|---|---|---|
| D1 | Clarifies functional and non-functional requirements; states assumptions with numbers | Clarifies current state, constraints, stakeholders and the business driver before any design |
| D2 | Explains mechanisms precisely; goes two or three levels deep on demand | Explains enough to prove the options are real; deep on the riskiest component; no vague answers |
| D3 | Compares alternatives per component with numbers | Compares whole options against business criteria, including cost, team capacity and reversibility; makes a recommendation |
| D4 | Drives the design session; manages time; checks in | Adjusts register for each audience; writes clearly; keeps a multi-party discussion on track |
| D5 | Takes hints well; integrates the interviewer's constraints | Listens to implementers and stakeholders; concedes visibly; transfers ownership |
| D6 | Stories and designs at the right scope for the level | Decisions, governance and alignment at the right organisational scope |

### Using the weights

Module 38's priority formula (`priority = rounds_in_loop × gap + 0.5`) allocates mocks by round. The weights above add a second lens: **within each mock, score yourself on the dimensions your target weights most.** A staff candidate who aces D2 but scores low on D6 is a strong senior; an architect candidate who aces D3 but fails D5 in the implementer round is a risky hire. Appendix C has a self-scoring sheet.

**Interview-grade sentence** *(for "what do you think we're evaluating?")*: *"Probably the same things every senior loop looks at — framing, depth, judgement, communication, collaboration and level — but in this role I'd expect judgement and communication to carry more weight than raw depth: whether I frame the decision well, weigh the options against what the business actually needs, and bring the people who'd build it with me."*

---

## Concept 13 — Failure modes and crossover errors

Each family has a characteristic way to fail. And each has a characteristic way for **candidates from the other family** to fail in it — the crossover errors.

### Characteristic failures

| Family | Failure | What the interviewer writes |
|---|---|---|
| **Senior IC** | Trade-offs without numbers | "Couldn't justify choices quantitatively" |
| | Breadth without depth — names every component, explains none | "Surface-level; no deep dive" |
| | Coding that works but isn't production-quality (no tests, edge cases, error handling) | "Wouldn't want this code in our codebase" |
| | Imprecise stack knowledge ("async makes it faster") | "Weak on fundamentals of their own platform" |
| **Staff IC** | Senior-shaped stories — "I built" instead of "I got four teams to" | "Strong senior; no evidence of staff scope" |
| | A great design with no ownership, migration or evolution story | "Didn't think beyond the design" |
| | Waiting to be driven in the design round | "Needed steering; not staff-level proactivity" |
| **Architect** | A technically sound design that ignores team size, delivery constraints or cost | "Ivory tower; not grounded in our reality" |
| | Lecturing the implementers | "Would struggle to get buy-in" |
| | "It depends" without a recommendation | "Can't make decisions" |
| | Framework recitation (TOGAF or pattern jargon) instead of reasoning | "Talks in buzzwords" |
| | One vague answer in the credibility probe | "Not sure they understand how it works any more" |
| **Vendor SA** | Technically excellent, poor customer communication | "Wouldn't put them in front of a customer" |
| | Pushing the vendor's service where it doesn't fit | "Doesn't earn customer trust" |

### Crossover errors — the expensive ones

These are worth special attention, because experienced engineers often apply for roles in both families:

| You are… | …in the other family's loop | The error | The fix |
|---|---|---|---|
| An architect-minded candidate | In an IC design round | Spends 20 minutes on stakeholders, options and governance; runs out of time before the deep dive | One minute of context and constraints, then run the seven steps; save adoption and ownership for the wrap-up |
| | In an IC coding round | Treats it as beneath them; writes sketchy code; argues about the question | Practise — Module 36 is a calibration pass, not a grind, but it's not optional |
| | In an IC behavioral round | Tells governance stories with no personal technical contribution | Include what *you* designed, built or debugged |
| An IC-minded candidate | In an architect design-doc or brownfield round | Jumps to boxes and arrows; designs the greenfield target; no options, no sequence | Start with current state and constraints; present options; sequence the change |
| | In an implementer round | Treats pushback as a technical debate to win | Listen, separate decision from plan, concede visibly |
| | In a stakeholder round | Explains the technology instead of the trade-off | Translate to time, money, risk and customer impact |
| | In a vendor presentation | A tour of the architecture with no business problem or outcome | Context → problem → options → decision → outcome → lessons |

**The single most useful habit** for anyone applying across families: **ask, at the start of every design round, what the interviewer wants the deliverable to be** — *"Would you like me to focus on designing the system in depth, or on the decision and how we'd get there from where we are?"* The answer tells you which column of Concept 11 you're in, and asking it is itself a senior signal.

**Interview-grade sentence** *(if an interviewer says "you spent a lot of time on context")*: *"That's fair — I tend to start with the current state and constraints because in real systems they change the answer, but I can go straight into the design. Let me take the order path in depth now: the write path, idempotency and how we publish events reliably."*

---
# Part C — Locating yourself

## Concept 14 — Your evidence inventory

The level and family you can credibly target are set by **evidence you can tell as stories** — not by your current title and not by what you believe you could do. Interviewers write down evidence; committees level from it (Module 34). So calibration starts with an inventory.

### Step 1 — List your candidate stories

Write one line for every substantial piece of work from the last five to seven years: projects, migrations, incidents, decisions, people you developed, things you started or stopped. Aim for 15–25 lines. Include work outside formal employment if it was real — a product you built and ran, open-source maintenance, consulting engagements.

### Step 2 — Score each story on the six axes

Use Module 34's six axes, scored by the **highest level the story honestly demonstrates**:

| Axis | 1 — Mid | 2 — Senior | 3 — Staff | 4 — Principal / EA |
|---|---|---|---|---|
| **Scope** | A task or feature | A service or a team's system | Several systems or teams | An organisation or the portfolio |
| **Ambiguity** | Given the solution | Given the problem | Given a goal; found the problem | Chose which problems matter |
| **Influence** | Own work | Own team | Across teams without authority | Executives, many teams, external |
| **Impact** | Output | Outcome for users | Outcome across teams; trajectory changed | Business- or company-level |
| **Horizon** | Weeks | Quarters | 1–2 years | Multi-year |
| **Leverage** | Did it | Mentored | Changed how teams work | Developed senior people; shaped culture |

### Step 3 — Read the profile

```text
STORY                                   SCOPE AMBIG INFL IMPACT HORIZ LEVER  FAMILY TAG
.NET Framework → .NET 8 migration         3     3    3     3     3     2     staff / solution-arch
ThreadPool starvation guardrails          3     2    3     3     2     3     staff (solver → architect)
Order ledger dual-run cut-over            2     2    2     3     2     1     senior
Built and ran own product end to end      3     3    2     2     3     1     founder → staff (with framing)
Vendor evaluation for identity provider   2     2    3     2     2     1     solution-arch
...
```

Then apply three rules:

1. **The level you can defend** is the highest level at which you have **at least three stories scoring at that level on at least four axes**. Two stories is thin (Module 35's story bank needs 6–8 total, and interviewers in one loop compare notes); one is an anecdote.
2. **The family you're shaped for** is visible in which axes are high. High **scope and influence with decision and alignment content** → architect family. High **depth, scope and leverage with technical mechanisms** → staff IC. High **influence with business stakeholders and customers** → solution, enterprise or vendor architect.
3. **The gaps are your preparation targets** — not technically, but in *evidence*. If you have no story above level 2 on influence, no amount of system-design practice will get you a staff offer.

### Founder, independent and consulting work

Work outside a conventional employer often scores higher on scope, ambiguity and horizon than candidates assume — and lower on influence, because there were fewer people to influence. Module 34 (Concept 34) covers translating it. The key moves: describe **the decisions** you made and the trade-offs, the **constraints** you operated under (budget, time, team of one), the **outcomes** with numbers, and the **people** you did have to align — customers, contractors, partners, investors. A product you designed, built, ran and evolved is strong evidence for D2, D3 and horizon; pair it with at least one story of cross-team influence from employed work for staff-level D6.

**Interview-grade sentence** *(for "what level do you see yourself at?")*: *"Based on the work I can actually point to — a migration I led across eleven services and four teams, the guardrails I introduced after a recurring incident class, and the decision records I set up for our platform — I'd put myself at the senior/staff boundary on your ladder, strongest on cross-team technical direction. I'd like the panel to assess that, and I'm happy to go deeper on any of those."*

---

## Concept 15 — Fit and energy: which job do you actually want?

Calibration isn't only about what you *can* be hired for. A loop also probes whether you *want* the job — and the probes are hard to fake ("tell me about a week in your last role you really enjoyed"). More importantly, a job you won pretending to want it is a bad two years. Answer these honestly, in writing.

| Question | Leans IC (senior/staff) | Leans architect (solution/enterprise) | Leans vendor/consultancy |
|---|---|---|---|
| What share of your week do you want to spend writing code? | 40–80% (senior) / 20–50% (staff) | 0–20% | 0–20%, plus prototypes and demos |
| How do you feel about a calendar full of meetings? | Want protected focus time | Accept that meetings are the work | Accept meetings and travel |
| Do you want authority or influence? | Influence through expertise and code | Influence plus formal decision rights (review boards, standards) | Influence over customers you don't control |
| What horizon do you enjoy? | Quarters; seeing your code ship | Years; seeing the portfolio change | Engagements; many customers |
| Who do you want to spend time with? | Engineers | Engineers, product, business, risk | Customers and the account team |
| What does a good day look like? | A hard problem solved; a clean design merged | A decision made and adopted; a roadmap agreed | A customer convinced; a blocker removed |
| How do you feel about being measured on outcomes you don't control? | Prefer to own the result | Accept it — teams deliver your designs | Accept it — customers adopt or don't |
| Breadth or depth? | Depth in a stack and a domain | Breadth across systems; depth in a few | Breadth across a vendor's portfolio |

Pat Kua's Trident framing (Concept 2) is useful here: **where do you want 70–80% of your time to go?** Executing, leading people on technical topics, or managing people. Most strong candidates for this curriculum's roles want the middle track — and the choice between staff IC and architect within it is mostly about **context**: product company vs. enterprise, engineers as main stakeholders vs. business as main stakeholders, close to the code vs. close to the portfolio.

### The honest "why architect?" — and "why IC?"

Whichever you choose, you'll be asked why. A good answer is **toward** something, **evidenced**, and **specific to the role's context**:

- *"Why architect?"* — *"The work I've enjoyed most and done best is making the expensive decisions well — the database and messaging choices on our platform, the migration plan, the standards that let four teams move independently. I want that to be my job rather than something I do around my job, and I want to do it close to the teams who build it."*
- *"Why staff rather than architect?"* — *"I want to stay close enough to the code that my designs are informed by it — I still review most PRs on the services I'm responsible for and I write the tricky parts myself — while working across teams. In a product company that's a staff role; the architect roles I've seen are further from the engine room than I want to be."*

**Interview-grade sentence:** *"I've thought about where I want my time to go — I want most of it on the technical decisions that are expensive to change and on getting teams aligned behind them, with enough hands-on work to keep those decisions honest — and in a company like yours that's [the staff role / the architect role], which is why I'm here."*

---

## Concept 16 — The hands-on question

Every architect loop asks some version of **"Are you still hands-on?"**, and many staff loops ask it too. It's a proxy for the interviewer's real worry: **will your decisions be grounded in how things actually behave?** You answer it with evidence, not assertion.

### What "technical currency" means for a .NET/Azure architect

| Area | You should be able to… |
|---|---|
| Runtime | Explain async/await and ThreadPool starvation, GC modes and allocation pressure, DI lifetimes — and how you'd detect each in production (Phase 4) |
| Data | Explain EF Core change tracking and query translation costs; SQL isolation levels; Cosmos DB partition-key design and RU cost (Modules 12, 19, 27) |
| Messaging | Say exactly what Service Bus sessions, duplicate detection and dead-lettering do, and when you'd choose Event Hubs or Kafka instead (Modules 11, 27) |
| Platform | Explain the trade-offs between App Service, Container Apps, AKS and Functions — including the isolated worker model and the November 2026 in-process end of support (Module 26) |
| Operations | Describe how you'd instrument a service with OpenTelemetry and set an SLO (Module 28) |
| Security | Describe managed identity, Key Vault and token flows without hand-waving (Module 29) |
| Currency | Know what's current: .NET 10 LTS, .NET 11 STS from 10 November 2026, and why that matters for an upgrade policy |

### How to demonstrate it in interviews

1. **Answer one level deeper than asked**, briefly. *"Service Bus — with sessions keyed by order ID so events for one order are processed in sequence, and duplicate detection on the message ID to absorb publisher retries."* One precise sentence buys more credibility than a minute of generalities.
2. **Have a recent, specific hands-on example**: code you wrote, a PR you reviewed, an incident you debugged, a spike or proof of concept you built. *"Last month I wrote the spike for the outbox publisher myself, because the team was worried about lock contention on the outbox table — it turned out to be fine at our volume with a covering index."*
3. **Know where your hands-on edge is, and say so.** *"I haven't run AKS in production; I've run Container Apps, and here's what I'd check before moving."* Precision about limits is a credibility signal, not a weakness.

### Keeping currency if you're targeting architect roles

If your recent work has been far from code, fix it **before** the loop: build or extend a small but real system on the current stack (.NET 10, a real Azure deployment, OpenTelemetry, a message broker), and be ready to walk through it. A personal or independent product counts — if it's real, running and you can discuss its trade-offs in depth.

**Interview-grade sentence** *(for "are you still hands-on?")*: *"Yes, deliberately — not as the main contributor, but enough that my decisions are grounded. In my last role I wrote the spikes for the riskier decisions, reviewed most PRs touching the shared libraries and debugged the hard incidents myself; the most recent was a ThreadPool starvation issue I traced to a sync-over-async call in a logging sink. I think an architect who can't do that eventually makes decisions the code doesn't agree with."*

---

## Concept 17 — The 2026 market and what it changes

Three directional facts (graded C — consistent across several 2026 market reports, numbers unreliable) shape calibration this year.

**1. Demand is bifurcated toward senior and specialised.** Reports agree that senior and specialised engineers (AI/ML, security, data, platform, cloud) remain in demand while entry-level hiring has contracted. For you this is mostly good news, with a caveat: **competition at senior-plus is for evidence, not for availability** — loops are selective, and down-levelling remains the default risk for experienced external hires (Module 39, Concept 12).

**2. AI appears in both loop families — differently.**

| Family | How AI shows up |
|---|---|
| Senior/staff IC | AI-assisted or AI-expected coding rounds at some companies (Meta, Canva — Module 38), while many others prohibit AI in live interviews (Microsoft, Google — Module 39); design questions about LLM features (latency, cost, evaluation, safety); "how do you review AI-generated code?" |
| Architect | Governance of AI coding assistants across teams; architecture of LLM-backed features (RAG, data boundaries, cost controls, observability); build-vs-buy for AI services (Module 33); risk and compliance (the EU AI Act in European enterprises) |
| Vendor SA | AI services are now a central part of the portfolio — Microsoft's current CSA postings are titled "Cloud & AI" |

The preparation implication: **one well-reasoned AI story and one AI design answer per family** — not a new specialism. The reasoning is the same as everywhere else in this curriculum: requirements, cost, failure modes, trade-offs.

**3. Modernisation is the enterprise theme.** The .NET 8/9 and Azure Functions in-process support dates (10 November 2026) and the continued move off .NET Framework mean that **solution-architect openings in .NET shops are disproportionately modernisation roles**. If you're targeting the architect family, Module 32 (brownfield, strangler fig) and Module 33 (cost of delay, business cases) are likely to *be* the interview.

**Interview-grade sentence** *(for "how do you see AI changing this role?")*: *"For the architect side, the change I see most is that AI-generated code raises the value of the decisions around it — boundaries, contracts, guardrails and review — because more code arrives faster and someone has to make sure it fits. So I'd focus on making the expensive decisions explicit and enforceable: ADRs, fitness functions, and review practices that catch the patterns assistants get wrong, like sync-over-async in .NET."*

---
# Part D — Calibrating the prep

## Concept 18 — The target statement

Everything so far reduces to **one sentence you write down before you prepare**:

```text
[Role family] at [level] in [organisation type], [archetype / role shape], with a [loop shape],
[location / work model] — e.g. as the primary target, with [secondary target] as a deliberate hedge.
```

Examples:

- *"Staff software engineer at a product company or scale-up, Architect archetype, IC loop with system design and behavioral weighted most, remote-friendly EU — primary. Software architect at a product company as a hedge (same loop shape)."*
- *"Solution architect at an enterprise or Microsoft partner in a modernisation programme, architect loop with design-document and stakeholder rounds, hybrid in Belgrade or remote EU."*
- *"Senior software engineer (.NET) at a product company, IC loop with two coding rounds and a .NET deep-technical round, remote."*
- *"Cloud Solution Architect at a cloud vendor, customer-facing, presentation-centred loop, EU."*

### Why writing it matters

1. **It forces a choice.** Most mis-preparation comes from never choosing — preparing a little for everything.
2. **It sets the allocation** (Concept 19): each phrase in the statement changes which modules get hours.
3. **It sets the narrative** (Concept 21): your CV, LinkedIn headline and 90-second pitch should all match it.
4. **It sets the research** (Module 39): the statement is the D1 filter for which openings to pursue.
5. **It makes re-calibration possible** (Concept 22): you can't adjust a target you never stated.

### Checks before you commit

- **Evidence check** (Concept 14): do you have three or more stories at this level on four or more axes?
- **Fit check** (Concept 15): does the time split match what you want?
- **Market check** (Concept 17, Module 39): are there enough openings matching the statement in your location and work model to make a search viable?
- **Loop check** (Concepts 7–10): do you know what the loop samples, and is any of it a gap you can't close in your timeline?

If a check fails, change the statement — not the evidence.

**Interview-grade sentence** *(for "what are you looking for?" — the recruiter's first question)*: *"I'm looking for a staff-level technical leadership role at a product company — owning the architecture of an area across several teams, close enough to the code to keep decisions honest — ideally on .NET and Azure, which is where most of my depth is."*

---

## Concept 19 — Allocating preparation: relevance × gap

With a target statement, allocation becomes arithmetic. Each curriculum phase has a **relevance** to your target (how much the target's loop samples it) and a **gap** (how far you are from interview-ready on it). Hours should flow to phases that are both relevant and weak — with a small floor for relevant strengths so they don't decay.

### Relevance by target

The following relevance weights (0–1) are calibrated estimates derived from the loop anatomies in Part B:

| Phase (modules) | Senior IC (product) | Staff IC | Software / solution architect | Enterprise architect | Vendor SA / CSA |
|---|---|---|---|---|---|
| 2 — System design method (3–5) | 1.0 | 1.0 | 0.8 | 0.4 | 0.8 |
| 3 — Distributed systems theory (6–13) | 0.9 | 1.0 | 0.7 | 0.3 | 0.6 |
| 4 — .NET & C# mastery (14–19) | 0.9 | 0.6 | 0.6 | 0.1 | 0.2 |
| 5 — .NET architecture patterns (20–25) | 0.7 | 0.7 | 0.9 | 0.4 | 0.4 |
| 6 — Cloud & platform (26–29) | 0.6 | 0.6 | 1.0 | 0.6 | 1.0 |
| 7 — Architect track (30–33) | 0.3 | 0.7 | 1.0 | 1.0 | 0.8 |
| 8 — Behavioral & leadership (34–35) | 0.7 | 1.0 | 0.9 | 1.0 | 1.0 |
| 9 — Coding round (36) | 1.0 | 0.8 | 0.3 | 0.0 | 0.1 |
| 10 — Applied practice (37–39) | 0.9 | 0.9 | 0.8 | 0.6 | 0.8 |

If all your gaps were equal, hours would split in proportion to relevance:

| Phase | Senior IC | Staff IC | Software / solution architect | Enterprise architect | Vendor SA / CSA |
|---|---|---|---|---|---|
| 2 — Design method | 14% | 14% | 11% | 9% | 14% |
| 3 — Distributed theory | 13% | 14% | 10% | 7% | 11% |
| 4 — .NET mastery | 13% | 8% | 9% | 2% | 4% |
| 5 — .NET patterns | 10% | 10% | 13% | 9% | 7% |
| 6 — Cloud & platform | 9% | 8% | 14% | 14% | 18% |
| 7 — Architect track | 4% | 10% | 14% | 23% | 14% |
| 8 — Behavioral | 10% | 14% | 13% | 23% | 18% |
| 9 — Coding | 14% | 11% | 4% | 0% | 2% |
| 10 — Applied practice | 13% | 12% | 11% | 14% | 14% |

*(Rounded; columns may not sum to exactly 100%.)*

Three things to notice: **coding falls from 14% to almost nothing** as you move right; **the architect track and behavioral together rise from 14% to 46%**; and **vendor roles weight cloud breadth most** — and need material beyond this curriculum (the vendor's full service catalogue).

### Adding your gaps

Score each phase's gap from 0 (interview-ready) to 3 (major gap), then:

```text
priority(phase) = relevance × (gap + floor)        floor = 0.5 keeps relevant strengths warm
hours(phase)    = total_hours × priority / Σ priority
```

This is Module 38's mock-priority idea (`rounds_in_loop × gap + 0.5`) lifted from rounds to phases. Appendix D implements it in C#; Worked example A runs it.

### Two rules that override the arithmetic

1. **Gates before differentiators.** If the loop has a coding gate and your coding gap is 2 or more, close it first — failing the screen ends the loop before your strengths are seen.
2. **Mocks are not optional.** Phase 10 (applied practice) never drops below about 15% of hours in the final two weeks, whatever the formula says — knowledge that hasn't been performed under interview conditions is not yet interview-ready (Module 38).

**Interview-grade sentence** *(useful for any "how do you prioritise?" question)*: *"I weight effort by two things together — how much it matters to the outcome and how far we are from good on it — with a small floor so strengths don't decay; and I put gates first, because a failed gate makes everything else irrelevant."*

---

## Concept 20 — Dual-targeting without dilution

Many experienced engineers reasonably pursue both families — say, staff IC at product companies and solution architect at enterprises. That's a sound hedge **if it's deliberate**. Done by default, it dilutes preparation and produces stories pitched at the wrong altitude in both loops.

### What's shared and what isn't

| Shared core (prepare once) | IC-specific layer | Architect-specific layer |
|---|---|---|
| Phase 2 (design method), Phase 3 (distributed theory), Phase 6 (cloud), Phase 8 (story bank — stories exist once, told two ways), Module 39 (research) | Phase 9 (coding gate), Phase 4 depth (.NET deep-technical round), design rounds driven at IC tempo | Phase 7 (design docs, ADRs, brownfield, cost), stakeholder and implementer mocks, a design-document exercise, a presentation |

### Sequencing

- **Prepare the IC core first if you're dual-targeting.** Coding fluency and design depth are slower to build and decay faster; architect material layers on top more easily than the reverse.
- **Schedule loops by family in blocks.** Two IC loops in one week and an architect panel the next is easier than alternating daily — the altitude switch costs more than people expect.
- **Keep a "mode card"** for each family (Module 39, Concept 21's per-round card) with the opening move that sets the altitude: *IC* — "Let me clarify requirements and get some numbers"; *architect* — "Before designing, tell me about the current state, the teams and the constraints."

### One story, two altitudes

The same story bank serves both, told differently (Module 34):

| Element | IC telling (staff) | Architect telling |
|---|---|---|
| Situation | The recurring incident class across 40 services | The same, framed as a gap in the organisation's guardrails and its cost |
| Task | Eliminate the class | Decide how the organisation should prevent it |
| Action | Root cause, the analyzer package, the pipeline stage, getting the platform team to own it | Options (training, review checklist, analyzer, platform template), criteria, the decision record, governance |
| Result | No recurrences; ~2 violations a week caught in PRs | The same, plus the review practice adopted org-wide |

### When to stop hedging

Stop dual-targeting when either family produces **two consecutive strong signals** (offers, or strong feedback at the target level) — then concentrate. Or when the evidence inventory shows one family is clearly stronger. A hedge is a tool for uncertainty; once the uncertainty resolves, it's only a cost.

**Interview-grade sentence** *(if asked "are you also interviewing for architect roles?")*: *"Yes — I'm looking at technical leadership roles in two shapes, staff engineering at product companies and solution architecture where the work is close to delivery. They overlap a lot; what I'm really optimising for is owning important technical decisions with teams that build them. This role is the first shape, and it's the one I'm most excited about because…"*

---

## Concept 21 — Positioning and narrative

Your materials and your opening minutes should be **consistent with the target statement**. Inconsistency reads as "doesn't know what they want".

### The CV and profile

| Element | IC-positioned | Architect-positioned |
|---|---|---|
| Headline | "Staff Software Engineer — .NET, Azure, distributed systems" | "Solution Architect — .NET and Azure modernisation" |
| Bullets lead with | Systems built, scale, performance, reliability, cross-team technical direction | Decisions made, programmes delivered, stakeholders aligned, cost and risk outcomes |
| Numbers | Throughput, latency, incidents, adoption by teams | Cost, time-to-market, systems rationalised, approval time, adoption |
| Technical detail | Specific: runtime, data, messaging | Specific enough for credibility; breadth across platforms |
| Certifications | Usually omit or put last | AZ-305, TOGAF where relevant, near the top |

Keep **one master CV** with all bullets and derive two views from it, rather than maintaining two drifting documents.

### Title translation

Titles carry different meanings in different organisation types (Concept 4). When you move across types, **translate explicitly** — in the CV (a short scope line under the title) and in interviews:

- *Enterprise "Architect" → product company:* "My title was Solution Architect; in product-company terms the scope was closest to staff — I owned the architecture of the claims platform across four teams and still wrote the spikes for risky decisions."
- *Product "Senior Engineer" → enterprise architect:* "My title was senior, but I owned the architecture and the decision records for our platform — which in your structure would be the solution architect's job."
- *Founder / independent → either:* "I designed, built and operated the product end to end; the closest equivalent responsibilities are [X], and the decisions I'd point to are [Y]."

### Cross-family levelling

Moving across families often shifts level:

| Move | Common outcome | How to manage it |
|---|---|---|
| Enterprise architect / solution architect → product company IC | Often levelled **senior** rather than staff, because coding and deep design are weaker evidence | Close the coding gate; bring two deep technical stories; present cross-team decisions as staff evidence |
| Product staff IC → enterprise solution architect | Often levelled well — design depth transfers; stakeholder and governance evidence is the gap | Add stakeholder and governance stories; practise the stakeholder round |
| Product staff IC → vendor SA | Level depends on customer-facing evidence | Tell customer or partner stories; prepare the presentation |
| Vendor SA → product company IC | Coding and depth gaps are common | Hands-on currency (Concept 16) before applying |

### The 90-second pitch

One per target family, built from the target statement: *who you are* (role and level in their terms) → *two proof points* (the strongest stories at that altitude) → *what you're looking for* (matching the role). Appendix E has templates.

**Interview-grade sentence** *(for "walk me through your background" in an architect loop)*: *"I'm a .NET and Azure engineer who's spent the last several years owning architecture decisions — most recently the move of a claims platform off .NET Framework across four teams, where I ran the options, the decision records and the cut-over plan, and before that the messaging architecture that let our teams deploy independently. I'm looking for a role where that is the job — framing and making the expensive decisions, close to the teams who build them."*

---

## Concept 22 — Re-calibration: let outcomes move the target

A target statement is a hypothesis. Real loops are evidence. Module 38's calibration log (mock scores vs. real outcomes) and Module 39's claims log give you the data; this concept tells you **what to change** when the data comes in.

| Signal (from feedback, recruiter debriefs or patterns) | Likely meaning | Adjust |
|---|---|---|
| "Strong technically, didn't see staff-level scope" | Evidence gap at D6 | Strengthen stories at the target altitude (Module 35) — or target senior with a growth path |
| "Great communicator, light on depth" | D2 gap — or a better fit for the architect/vendor family | Deepen Phase 3–4, or shift the target toward architect/vendor roles |
| Passed design and behavioral, failed coding twice | Gate problem | Concentrate on Module 36 for two weeks; consider architect-family loops in the meantime |
| Strong in architect panels, rejected in implementer rounds | D5 problem — lecturing | Run implementer mocks with planted objections (Module 38) |
| Offers consistently one level below target | Down-levelling pattern | Ask what evidence would have supported the level above; adjust stories or accept and negotiate (Module 39, Concept 12) |
| Recruiters keep pitching a different family | The market reads your profile differently from how you read it | Check your CV positioning — or take the hint seriously |

Two disciplines make re-calibration work:

1. **Change one thing at a time.** If you change the target, the stories and the preparation simultaneously, you won't know what worked.
2. **Distinguish noise from signal.** One loop is noise (interviewer variance is large — Module 38). A pattern across three loops is signal.

**Interview-grade sentence** *(for "what have you learned from interviewing so far?" — occasionally asked by hiring managers)*: *"That the same experience lands very differently depending on how I frame it — in staff loops I've learned to lead with the cross-team decisions rather than the systems I built, and that change made a visible difference to how my level was assessed."*

---
# Worked example A — Calibrating an experienced .NET engineer with recent independent product work

**Profile** *(illustrative, close to a common real pattern)*: 15+ years of .NET; strong algorithms and data structures; recent years spent designing, building and running an ambitious product independently, with earlier employed roles that included a cross-team migration and a platform standard. Deep in systems design and architecture; interested in both staff IC and architect roles.

### Step 1 — Evidence inventory (excerpt)

```text
STORY                                         SCOPE AMBIG INFL IMPACT HORIZ LEVER  FAMILY
Own product: architecture, build, operation      3     3    2     2     3     1     staff / app-architect
Cross-team .NET migration (employed)             3     3    3     3     3     2     staff / solution-arch
Async guardrails across services (employed)      3     2    3     3     2     3     staff
Vendor and platform selection for own product    2     3    2     2     3     1     solution-arch
Mentored two engineers to senior (employed)      2     2    2     2     2     3     staff (leverage)
```

**Reading:** three stories at staff level on four or more axes → staff is defensible. Influence is strongest in the employed stories; the independent work is strong on scope, ambiguity and horizon. Family shape: technical decisions with mechanisms → **staff IC, Architect archetype**, with software/solution architect as a credible hedge.

### Step 2 — Fit check

Wants 30–50% hands-on, cross-team technical direction, engineers as main stakeholders → **technical-leadership track, product-company context** first.

### Step 3 — Target statement

*"Staff software engineer (Architect archetype) at a product company or scale-up on .NET/Azure, IC loop with design and behavioral weighted most, remote or hybrid EU — primary. Software architect at a product company as a hedge (same loop shape, same preparation)."*

### Step 4 — Allocation (120 hours)

Gaps scored 0–3: design method 1, distributed theory 1, .NET mastery 0, .NET patterns 0, cloud 1, architect track 1, behavioral 2 (staff-altitude stories after an independent period need reframing), coding 1 (strong DS&A, out of practice under time), applied practice 2 (no mocks yet). Running Appendix D with the Staff IC relevance column:

```text
Target: Staff IC (Architect archetype), product company — 120 h
Phase                                 Rel  Gap  Prio  Hours  Share
Phase 8 — Behavioral & leadership     1.0    2  2.50   26.0    22%
Phase 10 — Applied practice           0.9    2  2.25   23.4    19%
Phase 2 — System design method        1.0    1  1.50   15.6    13%
Phase 3 — Distributed systems theory  1.0    1  1.50   15.6    13%
Phase 9 — Coding round                0.8    1  1.20   12.5    10%
Phase 7 — Architect track             0.7    1  1.05   10.9     9%
Phase 6 — Cloud & platform            0.6    1  0.90    9.4     8%
Phase 5 — .NET architecture patterns  0.7    0  0.35    3.6     3%
Phase 4 — .NET & C# mastery           0.6    0  0.30    3.1     3%
```

**What the numbers say:** the largest investment is **behavioral** (reframing the independent work and the employed cross-team stories at staff altitude) and **mocks**; the strengths (.NET depth, patterns) get a maintenance floor only. Coding gets about 12 hours — enough to clear the gate given strong fundamentals, consistent with Module 36's "calibration pass, not a grind".

### Step 5 — Narrative

Pitch: *"A .NET/Azure engineer who owns expensive technical decisions end to end — most recently as the architect, builder and operator of my own product, and before that leading a cross-team migration and the async guardrails that removed a recurring incident class across our services. Looking for a staff role owning the architecture of an area with the teams that build it."*

Prepared answers: "Why staff rather than founder?" (toward: scale and teams), "Are you still hands-on?" (yes — the product), "Tell me about cross-team influence" (the employed stories — this is where independent work alone is thin).

### Step 6 — Re-calibration plan

After three loops: if feedback says "strong, but limited recent cross-team evidence", add the strongest employed influence story as the lead in every behavioral round; if architect-titled roles at product companies respond better, swap primary and hedge — same preparation, different positioning.

---
# Worked example B — Enterprise solution architect moving to a product company *(compressed)*

**Profile:** 6 years as a solution architect at a bank, designing integration and modernisation programmes on Azure; last wrote production code 4 years ago. Target: staff engineer at a fintech.

```text
EVIDENCE      Strong: framing, options, stakeholder alignment, programme-scale decisions (D1, D3, D4)
              Weak:   recent hands-on depth (D2, D2b); coding gate
FIT           Wants to be closer to the product and the code — genuine
RISK          Down-levelling to senior is likely without hands-on evidence (Concept 21)
TARGET        "Staff (Architect archetype) at a fintech on .NET/Azure — primary; accept senior with a
              documented path to staff as the acceptable outcome"
ALLOCATION    Gaps (Staff IC column): design 1 · theory 1 · .NET mastery 2 · patterns 0 · cloud 0 ·
              architect track 0 · behavioral 1 · coding 3 · applied 2. The arithmetic puts ~36% of hours
              into Phases 4 and 9 (coding alone ~23%, the largest single phase) and only ~3% into the
              architect track the candidate knows best — gates before differentiators
CURRENCY PLAN Build a small real system on .NET 10 + Service Bus + OpenTelemetry in the first three weeks;
              use it as the hands-on story (Concept 16)
NARRATIVE     Title translation: "My title was Solution Architect; the scope was cross-team technical
              direction for a platform of four teams — staff in your terms. I'm moving to get closer to
              the product and the code, which is why I've been building [X] on .NET 10."
```

The lesson: **the strongest part of this profile (architect judgement) isn't the constraint — the gate is.** Calibration puts hours where the loop would end the process, not where the candidate is most comfortable.

---
# Worked example C — Decoding an ambiguous posting *(illustrative)*

> **"Senior .NET Architect"** — *"Hands-on architect to lead the technical direction of our order platform. You'll design and implement core services in .NET 10 on Azure, set coding standards, review PRs, mentor developers, and work with product owners and the CTO on the roadmap. Experience with Service Bus, Azure SQL and Container Apps. AZ-305 a plus."*

```text
VERBS            "design and implement", "review PRs", "set standards", "mentor" → hands-on staff / tech-lead architect
                 "work with product owners and the CTO on the roadmap" → some upward stakeholder work
REPORTS TO       CTO (implied) → small-to-mid company; the architect is the most senior technical person
FAMILY           Hybrid: application architect = staff Tech Lead/Architect archetype (Concept 10)
PREDICTED LOOP   Coding round (likely, "implement"), system design, architecture discussion with the CTO,
                 behavioral; maybe a code-review exercise
CERT             AZ-305 "a plus" → not a governance-heavy culture
PREP SHIFT       Treat as Staff IC column with Phase 7 relevance raised to 0.9; do not skip coding
RECRUITER Qs     "Is there a coding round?" · "How much of the week is hands-on?" · "How many developers
                 would I be leading?" · "Who else is on the panel?"
```

---
# Common interview questions with model answers

**Q1. "What's the difference between a senior engineer, a staff engineer and an architect?"**
> *"I think of architecture as the decisions that are expensive to change. A senior engineer makes and owns those decisions inside one team's system and delivers them. A staff engineer finds the decisions that matter across several teams and gets those teams to adopt them, usually without authority. An architect — depending on the organisation — does the framing, documenting and governing of those decisions for a programme or a portfolio. The scope and the stakeholders change; the core skill, judging which decisions matter and making them well, is the same."*
*Key signal:* a principled distinction, not title trivia; awareness that it depends on the organisation.

**Q2. "Why do you want to be an architect?"** *(or "Why staff?")*
> Concept 15's structure: toward something, evidenced, context-specific. *"The work I've done best is making the expensive decisions well and getting teams to build on them — the messaging architecture that let four teams deploy independently is the example I'm proudest of. I want that to be the job, and I want to do it close to the people who build it."*
*Key signal:* genuine motivation backed by a concrete example.

**Q3. "Why not management?"**
> Concept 5's three parts: leverage, evidence, respect. *"I've done the people side informally and enjoyed it, but my leverage is biggest on decisions — the guardrails I introduced removed a whole incident class without me managing anyone. I'd rather partner with a strong EM."*
*Key signal:* a reasoned choice, no dismissiveness.

**Q4. "Are you still hands-on?"**
> Concept 16: yes, with a specific recent example and a stated edge. *"Deliberately. I write the spikes for risky decisions and debug the hard incidents — most recently a ThreadPool starvation problem from a sync-over-async call in a logging sink. I haven't run AKS in production; I've run Container Apps, and I'd want to know these three things before moving."*
*Key signal:* recent specifics; honesty about limits.

**Q5. "How do you make sure your architecture actually gets implemented?"**
> *"By not being the only person who owns it. I involve the implementing team in the options before the decision, write the decision and its reasoning down as an ADR, turn the important constraints into things the build checks — module-boundary tests, analyzers, pipeline templates — and review a sample of the work early rather than at the end. And I treat significant deviations as information: either the decision was wrong for that context, or the guardrail was missing."*
*Key signal:* mechanisms, not authority; humility about being wrong.

**Q6. "Tell me about a time your design was rejected."**
> A real story, no villains (Module 34, Concept 32): what you proposed, why it was rejected, what you learned about the context, what happened next — ideally that you either improved the proposal or recognised the rejection was right. *"…in hindsight the team was right: the event-driven option would have doubled their on-call for a benefit we didn't need for another year. What changed in how I work is that I now bring the run cost and the on-call impact into the options table from the start."*
*Key signal:* self-awareness; a mechanism-level lesson.

**Q7. "How do you decide what to decide centrally and what to leave to teams?"**
> *"By the cost of inconsistency. If two teams choosing differently creates real cost — interoperability, security, data contracts, the identity provider — it's worth central coordination. If the cost is local — a library, an internal pattern — teams should decide, and the most I'd do is offer a paved road that's easier than the alternatives. I'd also make the central decisions few and explicit, because every one of them is a tax on every team."*
*Key signal:* a principle (Fowler's cost-of-coordination view), not a rule list.

**Q8. "What does an architect do in an agile organisation?"**
> *"The same five things — identify the important decisions, frame and make them, record them, make sure they're implemented and revisit them — but continuously and in small increments rather than up front. Architecture becomes a set of decisions taken at the last responsible moment, recorded as ADRs, protected by fitness functions, and owned by the teams as much as possible. The architect's job shifts toward making those decisions good rather than making all of them."*
*Key signal:* architecture as an ongoing activity, not a phase.

**Q9. "You've been a [solution architect / founder / senior engineer]. Why should we hire you at staff?"**
> Evidence at the level, translated (Concept 21): *"The scope I've held matches your staff description — I'll give you three examples: [cross-team decision], [mechanism that outlived me], [developing other senior people]. If the panel sees it differently, I'd like to understand what evidence would have changed the assessment."*
*Key signal:* knows the ladder; evidence, not argument.

**Q10. "How would you approach this problem?"** *(when the prompt is ambiguous between design and decision)*
> Concept 13's habit: *"Would you like me to focus on designing the target system in depth, or on which option to take and how we'd get there from the current system? I can do both, but I want to spend the time where it's most useful to you."*
*Key signal:* recognises two deliverables; asks.

**Q11. "How do you stay technical as an architect?"**
> *"Three habits: I keep a small part of the delivery that's mine — usually spikes for the riskiest decisions; I review code in the areas my decisions touch; and I build things on the current platform outside work, which is how I keep up with things like .NET 10 and the move to the isolated worker model for Functions. The risk I'm guarding against is making decisions the code doesn't agree with."*
*Key signal:* specific practices, current platform awareness.

**Q12. "What makes a good architect, in your view?"**
> *"Someone who makes the important decisions well and makes everyone else's decisions better — clear trade-offs, decisions written down with their reasoning, guardrails rather than gates, and enough hands-on contact with the system to know when a decision has stopped fitting. The bad version is someone who decides everything, writes documents nobody reads and can't answer a precise question about how the system behaves."*
*Key signal:* a stance (Hohpe's "fewer decisions") with both sides.

**Q13. "How do you see AI changing this role?"**
> Concept 17's sentence: AI-generated code raises the value of boundaries, contracts, guardrails and review; plus governance of AI tools and the architecture of LLM features (cost, data boundaries, evaluation).
*Key signal:* a reasoned position, not hype or dismissal.

**Q14. "Where do you see yourself in five years?"** *(in a staff or architect loop)*
> Consistent with the target statement and the track: *"Deeper in the same direction — owning the architecture of a larger area, with more of my impact coming through the engineers I've helped grow and the mechanisms I've left behind. Possibly principal, if the organisation needs that; I'm less attached to the title than to the scope."*
*Key signal:* coherence; staying on the chosen track; scope over title.

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Choosing a target | Preparing for "senior/architect interviews" in general | A written target statement: family, level, org type, archetype, loop, location |
| Titles | Taking "architect" or "senior" at face value | Reading verbs, reporting line and loop shape to find the real job |
| Architecture definition | Boxes and arrows; "high-level design" | The decisions that are expensive to change, and the five activities around them |
| Org context | Same answer for every company | Distributed vs. concentrated architecture; product vs. enterprise vs. vendor |
| Loop prediction | Assuming the loop | Deriving rounds from the job's tasks, then confirming with the recruiter |
| Prep allocation | Hours where you're comfortable | Relevance × gap per phase; gates first; mocks protected |
| Coding (architect targets) | Skipping it entirely for hybrid roles | Asking whether there's a coding round; preparing if so |
| Coding (staff targets) | 70% of hours on coding | Coding as a gate; design and behavioral as the level-setters |
| Design round deliverable | Wrong artefact for the family | Asking whether to design in depth or to frame the decision and path |
| Architect answers | Greenfield target state; no options | Current state → options → criteria → recommendation → sequence → ADRs |
| Implementer round | Defending the design | Listening; separating decision from plan; visible concessions; ownership transfer |
| Stakeholder round | Explaining the technology | Translating into time, money, risk and customer impact |
| Credibility | Vague answers about mechanisms | One level deeper than asked; precise and current |
| Stories | One altitude for every loop | The same story told at staff and architect altitude |
| Level | Claiming the title you had | The level your evidence supports, with three stories on four axes |
| Fit | Applying for the job that sounds senior | Matching time split, stakeholders and horizon to what you want |
| "Why not management?" | Dismissive of management | Leverage, evidence, respect |
| Dual-targeting | Preparing a little for everything | Shared core first, deliberate hedge, stop when signal arrives |
| Positioning | One generic CV | One master CV, two views; explicit title translation |
| Feedback | Treating each loop in isolation | Patterns across loops drive re-calibration, one change at a time |

---
# Practice exercises

1. **Define architecture in your own words.** Write the five activities (Concept 1) and, for your last two roles, which you owned and which others owned.
2. **Place three real openings.** For each: organisation type, concentrated or distributed architecture, family, archetype (Concepts 2–4). Note where the title misled you.
3. **Predict a loop from tasks.** Take one JD, list its top five tasks, map each to a round (Concept 6), then compare with a published interview guide or the recruiter's description.
4. **Classify with the quick classifier** (Concept 10) for five postings. Which were hybrids?
5. **Answer the same prompt twice.** Take Concept 11's order-processing prompt (or Module 37's problems) and write a one-page IC answer and a one-page architect answer. Time-box each to the round length.
6. **Re-weight a mock.** Score a recent mock on D1–D6, then compute weighted scores under two target columns (Concept 12). Which target does this mock make you look strongest for?
7. **Crossover drill.** Have a partner start a design round without saying which family it is. Practise asking about the deliverable in the first two minutes (Concept 13).
8. **Evidence inventory.** Build the full table (Concept 14) for 15–25 stories. What's the highest level with three stories on four axes?
9. **Fit questionnaire.** Answer Concept 15's table in writing. Does your answer match the target you were planning?
10. **Hands-on audit.** For each row of Concept 16's currency table, write one precise sentence. Where you can't, that's a gap.
11. **Write your target statement** (Concept 18) and run the four checks.
12. **Run the allocation.** Score your gaps and run Appendix D with the relevant column. Compare with how you'd been spending time.
13. **One story, two altitudes.** Tell your strongest story at staff altitude and at architect altitude (Concept 20), each in under three minutes. Record and compare.
14. **Two pitches.** Write and speak a 90-second pitch for each of two target families (Concept 21, Appendix E).
15. **Implementer role-play.** Have two people play engineers with three planted objections to a decision of yours. Score yourself on Module 38's implementer criteria.
16. **Re-calibration rehearsal.** Given each signal in Concept 22's table, write what you'd change — and what you'd keep the same.

---
# Free resources and learning material

All free to read or use unless marked *(book)*, *(paid)* or *(partly paid)*. Start with the ★ items. Fast-moving facts (certifications, postings, curricula) were checked on October 8, 2026.

### What architecture is, and what architects do
- ★ [Martin Fowler — Software Architecture Guide](https://martinfowler.com/architecture/) — "the important stuff", application vs. enterprise architecture, and links to everything below.
- ★ [Martin Fowler — Who Needs an Architect? (IEEE Software, PDF)](https://martinfowler.com/ieeeSoftware/whoNeedsArchitect.pdf) — the Ralph Johnson definitions behind Concept 1.
- [Martin Fowler — Is High Quality Software Worth the Cost?](https://martinfowler.com/articles/is-quality-worth-cost.html) — why architecture matters economically.
- ★ [Gregor Hohpe — The Architect Elevator (on martinfowler.com)](https://martinfowler.com/articles/architect-elevator.html) — penthouse to engine room.
- [The Architect Elevator — Gregor Hohpe's site](https://architectelevator.com/) — essays on architect skills, decisions and communication.
- [Hohpe — Think like an Architect](https://architectelevator.com/architecture/famous-architects-sketch/) and [An Architect's Path](https://architectelevator.com/architecture/architect-bookshelf/).
- [Hohpe — Agile and Architecture](https://architectelevator.com/transformation/agile_architecture/).
- [Gregor Hohpe — Modern Architect talks (YouTube playlist)](https://www.youtube.com/playlist?list=PLsuboX68NN3DL-sYP16NRyaLFRvSexa_s).
- [Rebecca Parsons — Enterprise Architects Join the Team (IEEE Software, PDF)](https://martinfowler.com/ieeeSoftware/enterpriseArchitects.pdf).
- [Kevin Hickey — The Role of an Enterprise Architect in a Lean Enterprise](https://martinfowler.com/articles/ea-in-lean-enterprise.html).
- [Sarah Taraporewalla — Creating an integrated business and technology strategy](https://martinfowler.com/articles/creating-integrated-tech-strategy.html).
- [Sriram Narayan — Products Over Projects](https://martinfowler.com/articles/products-over-projects.html) — why product vs. project organisations need different architects.
- [Mark Richards & Neal Ford — Developer to Architect (Software Architecture Monday)](https://www.developertoarchitect.com/) — short free lessons on the architect role.
- [Wikipedia — Solution architecture](https://en.wikipedia.org/wiki/Solution_architecture) and [Enterprise architecture](https://en.wikipedia.org/wiki/Enterprise_architecture) — neutral starting definitions.

### The IC ladder, staff-plus and archetypes
- ★ [StaffEng — Guides (Will Larson)](https://staffeng.com/guides/) — the full index.
- ★ [StaffEng — Staff archetypes](https://staffeng.com/guides/staff-archetypes/).
- [StaffEng — What do Staff engineers actually do?](https://staffeng.com/guides/what-do-staff-engineers-actually-do/)
- [StaffEng — Does the title even matter?](https://staffeng.com/guides/does-the-title-even-matter/)
- [StaffEng — Operating at Staff](https://staffeng.com/guides/operating-at-staff/) and [Writing engineering strategy](https://staffeng.com/guides/engineering-strategy/).
- [StaffEng — Present to executives](https://staffeng.com/guides/present-to-executives/) — the penthouse skill for IC leaders.
- [StaffEng — Staff-plus career ladders](https://staffeng.com/guides/staff-career-ladders/) — links to many public ladders.
- [StaffEng — Stories](https://staffeng.com/stories/) — first-person accounts of staff roles at many companies.
- ★ [Dropbox — Engineering Career Framework](https://dropbox.github.io/dbx-career-framework/) — a public ladder with level expectations by scope, reach and impact; see [IC5 Staff](https://dropbox.github.io/dbx-career-framework/ic5_staff_software_engineer.html) and [IC6 Principal](https://dropbox.github.io/dbx-career-framework/ic6_principal_software_engineer.html).
- [Dropbox — What is Impact?](https://dropbox.github.io/dbx-career-framework/what_is_impact.html) and [Archetypes behaviors](https://dropbox.github.io/dbx-career-framework/archetypes_behaviors.html).
- ★ [Pat Kua — The Trident Model of Career Development](https://www.patkua.com/blog/the-trident-model-of-career-development/).
- [Charity Majors — The Engineer/Manager Pendulum](https://charity.wtf/2017/05/11/the-engineer-manager-pendulum/).
- [Tanya Reilly — Being Glue](https://noidea.dog/glue) — making technical-leadership work visible.
- [Levels.fyi](https://www.levels.fyi/) — levels and compensation across companies.

### Interview loops: IC, staff and architect
- ★ [StaffEng — Interviewing for Staff-plus roles](https://staffeng.com/guides/interviewing-staff-plus-roles/).
- ★ [StaffEng — Staff-plus interview processes](https://staffeng.com/guides/staff-plus-interview-process/) — the failure modes and formats in Concepts 6–7.
- [StaffEng — Finding the right company](https://staffeng.com/guides/finding-the-right-company/) and [Deciding to switch companies](https://staffeng.com/guides/deciding-to-switch/).
- ★ [Hello Interview — The System Design Interview: What is Expected at Each Level](https://www.hellointerview.com/blog/the-system-design-interview-what-is-expected-at-each-level).
- [Hello Interview — System design in a hurry](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction).
- [interviewing.io — Senior engineer's guide to FAANG hiring processes](https://interviewing.io/guides/hiring-process).
- [interviewing.io — Microsoft's interview process](https://interviewing.io/guides/hiring-process/microsoft).
- [Tech Interview Handbook — Interview formats at top companies](https://www.techinterviewhandbook.org/interview-formats-top-companies/).
- [Tech Interview Handbook — Behavioral interviews](https://www.techinterviewhandbook.org/behavioral-interview/).
- [Lethain — Designing interview loops](https://lethain.com/designing-interview-loops/) — reasoning from signals to formats.

### Vendor and customer-facing architect roles
- ★ [Exponent — Amazon Solutions Architect interview guide](https://www.tryexponent.com/guides/amazon-solutions-architect-interview) — loop shape including the technical presentation.
- [Amazon — How we hire](https://www.amazon.jobs/content/en/how-we-hire) and [Leadership Principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles).
- [Microsoft Careers](https://careers.microsoft.com/) — search "Cloud Solution Architect" for current postings; [US corporate pay ranges](https://careers.microsoft.com/us/en/us-corporate-pay).
- [Microsoft Careers — Hiring tips](https://careers.microsoft.com/v2/global/en/hiring-tips) — including the candidate code of conduct on AI.

### Architecture practice for the architect track
- ★ [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/) — reference architectures and the vocabulary of Azure design reviews.
- ★ [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/) — the five pillars used in many Azure architect interviews.
- [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/) — the enterprise/solution architect's view of cloud programmes.
- [.NET application architecture guides](https://learn.microsoft.com/dotnet/architecture/) — microservices, cloud-native and modernisation e-books (free).
- [C4 model](https://c4model.com/) — the diagramming standard from Module 31.
- [arc42](https://arc42.org/) — a free architecture documentation template, popular in European architect roles.
- [ADR GitHub organisation](https://adr.github.io/) and [Joel Parker Henderson — Architecture Decision Records](https://github.com/joelparkerhenderson/architecture-decision-record).
- [Wikipedia — Architecture tradeoff analysis method (ATAM)](https://en.wikipedia.org/wiki/Architecture_tradeoff_analysis_method) — structured evaluation of quality-attribute trade-offs.
- [Martin Fowler — Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/) — brownfield rounds.
- [Thoughtworks Technology Radar](https://www.thoughtworks.com/radar) — current technology positions; useful for "what would you adopt?" questions.
- [InfoQ — Architecture & Design](https://www.infoq.com/architecture-design/) — free talks and articles by practising architects.

### Certifications and frameworks (track signals)
- [Microsoft — Azure Solutions Architect Expert (AZ-305)](https://learn.microsoft.com/credentials/certifications/azure-solutions-architect/) — requires AZ-104 for the Expert title.
- [Microsoft — Retired certification exams](https://learn.microsoft.com/credentials/support/retired-certification-exams) — AZ-204 retired July 2026.
- [The Open Group — TOGAF Standard](https://www.opengroup.org/togaf) — the enterprise-architecture framework; the standard is downloadable free with registration.
- [iSAQB — International Software Architecture Qualification Board](https://www.isaqb.org/) — CPSA Foundation and Advanced Level.
- [iSAQB — CPSA-F curriculum (GitHub)](https://github.com/isaqb-org/curriculum-foundation) — the full Foundation curriculum is public; a good syllabus for the software-architect role even without the exam.

### System design references (shared core)
- [System Design Primer (donnemartin)](https://github.com/donnemartin/system-design-primer).
- [ByteByteGo — System Design 101](https://github.com/ByteByteGoHq/system-design-101).
- [Martin Fowler — Catalog of Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/).
- [Team Topologies — Key concepts](https://teamtopologies.com/key-concepts) — team types and their implications for architecture.
- [Mel Conway — Conway's Law](http://www.melconway.com/Home/Conways_Law.html).

### .NET platform currency (for the hands-on question)
- [.NET and .NET Core support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core).
- [Migrate to the isolated worker model (Azure Functions)](https://learn.microsoft.com/azure/azure-functions/migrate-dotnet-to-isolated-model).
- [.NET Blog](https://devblogs.microsoft.com/dotnet/).

### Books
- *(book)* Will Larson — *Staff Engineer: Leadership Beyond the Management Track*.
- *(book)* Tanya Reilly — *The Staff Engineer's Path*.
- *(book)* Gregor Hohpe — *The Software Architect Elevator*.
- *(book)* Mark Richards & Neal Ford — *Fundamentals of Software Architecture* (2nd edition).
- *(book)* Neal Ford, Mark Richards, Pramod Sadalage & Zhamak Dehghani — *Software Architecture: The Hard Parts*.
- *(book)* Neal Ford, Rebecca Parsons, Patrick Kua & Pramod Sadalage — *Building Evolutionary Architectures* (2nd edition).
- *(book)* Matthew Skelton & Manuel Pais — *Team Topologies*.
- *(book)* Gergely Orosz — *The Software Engineer's Guidebook* — levels and career paths in tech companies.
- *(book)* Camille Fournier — *The Manager's Path* — for understanding the management track you're positioning against.
- *(book)* Gernot Starke et al. — *Software Architecture Foundation* (iSAQB-aligned).

### Previous and later modules to revisit
- Module 1 — rubric literacy; what's scored.
- Modules 3–5 — the shared design method.
- Modules 30–33 — the architect track: design documents, ADRs and C4, brownfield, cost.
- Modules 34–35 — STAR at altitude; the story bank.
- Module 36 — the coding gate, right-sized.
- Module 37 (Concept 38) — the architect's version of the classic design problems.
- Module 38 — the six dimensions, mock catalogue and the priority formula.
- Module 39 — researching the company, decoding the JD and mapping the loop.

---
# Quick-recall sheet

**One sentence.** IC loops sample designing and building; architect loops sample framing, making, recording and getting adoption for decisions — choose your target, then spend hours where its loop scores.

**Architecture.** The decisions that are expensive to change. Five activities: identify · frame and decide · record · ensure execution · revisit. Roles slice them differently.

**Concentrated vs. distributed.** Enterprise/regulated → architecture function and boards. Product companies → staff engineers, RFCs, ADRs. "Architects make more impact by making fewer decisions."

**IC ladder.** Senior (system, team) → staff (several teams; finds the problem) → principal (org; chooses problems). Six axes: scope, ambiguity, influence, impact, horizon, leverage. Senior is often terminal.

**Archetypes.** Tech Lead · Architect · Solver · Right Hand. **Trident:** management · technical leadership · true IC — most staff and architect roles are technical leadership.

**Architect family.** Software/application (one system, near code) · solution (programme, business problem, integrations) · enterprise (portfolio, standards, roadmap) · domain (cloud, data, security, integration) · vendor SA/CSA (customer-facing, platform breadth).

**Org types.** Product co · enterprise · consultancy/partner · cloud vendor — each defines "architect" differently. Conway predicts the job.

**Adjacent tracks.** Tech lead (how a team builds) · EM (people, commitments) · TPM (plan executed) · staff/architect (decisions right and staying right).

**Loops as work samples.** Rounds follow tasks. "Watch them do the work."

**IC loop.** Coding = gate. Design + behavioral = level. Deep technical confirms. Staff adds presentation, code review, mentorship panel, architecture evolution.

**Architect loop.** Design-doc exercise · brownfield · implementer round · stakeholder round · presentation · credibility probe · behavioral. Coding rare; code literacy common; hybrids may code.

**Vendor loop.** Presentation (~30 min) · customer scenario · cloud breadth · values (e.g. LPs in every round). Customer-facing; often tied to business outcomes.

**Classifier.** Coding? Written exercise? Non-engineers on panel? Presentation? Reporting line? Customer-facing share?

**Two deliverables.** IC: a design that would work. Architect: a decision that would be adopted. Ask which they want.

**Weights.** Senior IC: depth. Staff: level signal + judgement. Solution architect: judgement. EA: framing + communication. Vendor: communication.

**Crossover errors.** Architect in IC round: too much context, no deep dive. IC in architect round: greenfield answer, lecturing implementers, explaining tech to stakeholders.

**Evidence.** Defend the level where three stories score on four axes. Founder work: strong on scope/ambiguity/horizon; pair with influence stories.

**Fit.** Where should 70–80% of your time go? Code share, meetings, authority vs. influence, horizon, stakeholders.

**Hands-on.** One level deeper than asked; a recent specific example; precision about limits; currency on .NET 10, isolated Functions, Nov 2026 dates.

**2026.** Bifurcated demand (direction, not numbers). AI in both families. Enterprise .NET = modernisation.

**Target statement.** Family · level · org type · archetype · loop · location · primary/hedge. Four checks: evidence, fit, market, loop.

**Allocation.** priority = relevance × (gap + 0.5); hours ∝ priority. Gates first; mocks ≥ ~15% in the last two weeks.

**Dual-targeting.** Shared core first (IC core before architect layer); loops in family blocks; mode cards; one story, two altitudes; stop hedging after two strong signals.

**Positioning.** One master CV, two views; title translation; cross-family levelling risks; a 90-second pitch per family.

**Re-calibration.** Patterns across loops, not single loops. Change one thing at a time.

---
# Appendix A — Target statement and calibration worksheet

```text
DATE: ____                      TIMELINE TO FIRST LOOP: ____ weeks      HOURS AVAILABLE: ____

1. EVIDENCE (Concept 14)
   Highest level with ≥3 stories scoring that level on ≥4 axes: ____
   Strongest axes: ____ ____     Weakest axes: ____ ____
   Family shape from the stories: staff IC | software/solution architect | EA | vendor

2. FIT (Concept 15)
   Desired hands-on share: ____%   Main stakeholders: engineers | business | customers
   Horizon I enjoy: quarters | years | engagements     Authority vs influence: ____

3. TARGET STATEMENT (Concept 18)
   Primary: [family] at [level] in [org type], [archetype], [loop shape], [location/work model]
   ____________________________________________________________________________
   Hedge (optional, deliberate): ____________________________________________
   Stop-hedging condition: ________________________________________________

4. CHECKS
   □ Evidence  □ Fit  □ Market (enough openings?)  □ Loop (gaps closable in timeline?)

5. ALLOCATION (Concept 19, Appendix D)
   Relevance column used: ____   Gaps (P2..P10): __ __ __ __ __ __ __ __ __
   Gates identified: ________     Mock share in final two weeks: ____%

6. NARRATIVE (Concept 21)
   Headline: ____________________   90-second pitch drafted □   Title translation line: ____
   Prepared: □ "Why [architect/staff]?" □ "Why not management?" □ "Are you still hands-on?"

7. RE-CALIBRATION LOG (Concept 22)
   LOOP | FAMILY | OUTCOME | FEEDBACK THEME | CHANGE MADE (one at a time)
```

---
# Appendix B — Loop-shape classifier and recruiter questions

```text
ASK THE RECRUITER (ideally get answers in writing)
□ "Who does this role report to?"
□ "Could you walk me through the stages — is there a coding round? a written or take-home exercise?
   a presentation?"
□ "Who's on the panel — engineers, architects, product, business, security?"
□ "Roughly what share of the week is hands-on technical work vs. stakeholder or customer work?"
□ "Is the role closer to owning one system's architecture, a programme's, or standards across the
   organisation?"
□ "How are significant technical decisions made and recorded today?"
□ "Which level is the role, and is the loop calibrated for one level or deciding between two?"

CLASSIFY
                                   IC      Hybrid   Architect   EA      Vendor
Coding round                       yes     yes      rare        no      rare
Written / design-doc exercise      rare    some     yes         yes     rare
Non-engineers on panel             rare    some     yes         yes     yes (account team)
Presentation                       staff   some     often       often   yes
Reports to                         EM/Dir  CTO      Chief Arch  CIO/EA  Sales/CS
Customer-facing share              low     low      low–med     low     high

→ FAMILY: ____     → RELEVANCE COLUMN FOR APPENDIX D: ____     → ADJUSTMENTS: ____
```

---
# Appendix C — Dimension self-scoring sheet by target

Score each mock round 1–4 per dimension (Module 38's anchors), then weight by the target column (Concept 12).

```text
MOCK: ____  ROUND TYPE: ____  TARGET COLUMN: ____

DIMENSION                        SCORE (1–4)   WEIGHT   WEIGHTED
D1 Framing & scoping               ____         ____%     ____
D2 Technical depth (+D2b)          ____         ____%     ____
D3 Trade-offs & judgement          ____         ____%     ____
D4 Driving & communication         ____         ____%     ____
D5 Collaboration & coachability    ____         ____%     ____
D6 Level signal                    ____         ____%     ____
WEIGHTED TOTAL (Σ score × weight)                         ____  (out of 4.0)

WEIGHTS (from Concept 12)
                 Senior IC  Staff IC  SW/Solution Arch  EA    Vendor SA
D1                 15%        15%          20%          20%     15%
D2                 30%        20%          15%          10%     15%
D3                 20%        20%          25%          20%     15%
D4                 15%        15%          15%          20%     30%
D5                 10%        10%          15%          15%     15%
D6                 10%        20%          10%          15%     10%

LOWEST WEIGHTED DIMENSION → NEXT MOCK FOCUS: ____
CROSSOVER CHECK: did I deliver the right artefact (design vs. decision)? □ yes □ no
```

---
# Appendix D — Prep allocation tool (C#)

Computes hours per curriculum phase from a total budget, each phase's relevance to your target (Concept 19's table) and your self-scored gap. Phases with zero relevance drop out; relevant strengths keep a floor so they don't decay.

```csharp
// dotnet run -- plan.json
using System.Text.Json;

if (args.Length == 0) { Console.WriteLine("usage: <plan.json>"); return; }

var plan = JsonSerializer.Deserialize<PrepPlan>(
        File.ReadAllText(args[0]), new JsonSerializerOptions(JsonSerializerDefaults.Web))
    ?? throw new InvalidOperationException("Empty input.");

Validate(plan);

// priority = relevance × (gap + floor): relevant and weak first; relevant strengths keep a floor.
var rows = plan.Phases
    .Select(p => (Phase: p, Priority: p.Relevance * (p.Gap + plan.MaintenanceFloor)))
    .ToList();

decimal totalPriority = rows.Sum(r => r.Priority);
if (totalPriority == 0m)
    throw new InvalidOperationException("Every phase has zero relevance — check the target column.");

Console.WriteLine($"Target: {plan.Target} — {plan.TotalHours:0} h");
Console.WriteLine($"{"Phase",-37} {"Rel",3} {"Gap",4} {"Prio",5} {"Hours",6} {"Share",6}");

foreach (var (phase, priority) in rows.OrderByDescending(r => r.Priority))
{
    decimal hours = plan.TotalHours * priority / totalPriority;
    Console.WriteLine(
        $"{phase.Name,-37} {phase.Relevance,3:0.0} {phase.Gap,4} {priority,5:0.00} {hours,6:0.0} {hours / plan.TotalHours,6:P0}");
}

static void Validate(PrepPlan plan)
{
    if (plan.TotalHours <= 0m)
        throw new InvalidOperationException("totalHours must be positive.");
    if (plan.MaintenanceFloor < 0m)
        throw new InvalidOperationException("maintenanceFloor must not be negative.");
    foreach (var p in plan.Phases)
    {
        if (p.Relevance is < 0m or > 1m)
            throw new InvalidOperationException($"{p.Name}: relevance must be between 0 and 1.");
        if (p.Gap is < 0 or > 3)
            throw new InvalidOperationException($"{p.Name}: gap must be between 0 and 3.");
    }
}

public sealed record Phase(string Name, decimal Relevance, int Gap);

public sealed record PrepPlan(string Target, decimal TotalHours, decimal MaintenanceFloor, List<Phase> Phases);
```

Notes worth knowing: `JsonSerializerDefaults.Web` binds camelCase JSON to the positional records case-insensitively; `decimal` keeps the shares exact enough to sum cleanly; `OrderByDescending` is a stable sort, so phases with equal priority keep their input order; the relational patterns in `Validate` (`is < 0m or > 1m`) are C# 9+. The `P0` format follows the current culture (`22%` or `22 %`).

### Example input (Worked example A)

```json
{
  "target": "Staff IC (Architect archetype), product company",
  "totalHours": 120,
  "maintenanceFloor": 0.5,
  "phases": [
    { "name": "Phase 2 — System design method",       "relevance": 1.0, "gap": 1 },
    { "name": "Phase 3 — Distributed systems theory", "relevance": 1.0, "gap": 1 },
    { "name": "Phase 4 — .NET & C# mastery",          "relevance": 0.6, "gap": 0 },
    { "name": "Phase 5 — .NET architecture patterns", "relevance": 0.7, "gap": 0 },
    { "name": "Phase 6 — Cloud & platform",           "relevance": 0.6, "gap": 1 },
    { "name": "Phase 7 — Architect track",            "relevance": 0.7, "gap": 1 },
    { "name": "Phase 8 — Behavioral & leadership",    "relevance": 1.0, "gap": 2 },
    { "name": "Phase 9 — Coding round",               "relevance": 0.8, "gap": 1 },
    { "name": "Phase 10 — Applied practice",          "relevance": 0.9, "gap": 2 }
  ]
}
```

### Output for that input

```text
Target: Staff IC (Architect archetype), product company — 120 h
Phase                                 Rel  Gap  Prio  Hours  Share
Phase 8 — Behavioral & leadership     1.0    2  2.50   26.0    22%
Phase 10 — Applied practice           0.9    2  2.25   23.4    19%
Phase 2 — System design method        1.0    1  1.50   15.6    13%
Phase 3 — Distributed systems theory  1.0    1  1.50   15.6    13%
Phase 9 — Coding round                0.8    1  1.20   12.5    10%
Phase 7 — Architect track             0.7    1  1.05   10.9     9%
Phase 6 — Cloud & platform            0.6    1  0.90    9.4     8%
Phase 5 — .NET architecture patterns  0.7    0  0.35    3.6     3%
Phase 4 — .NET & C# mastery           0.6    0  0.30    3.1     3%
```

Σ priority = 11.55. Then apply Concept 19's two overrides by hand: if the coding gap were 2 or more, move it to the top regardless of its computed share; and protect at least ~15% for mocks in the final two weeks (here Phase 10 already has 19%).

---
# Appendix E — 90-second pitch templates

### IC (senior / staff) version

```text
WHO     "I'm a [senior/staff]-level .NET and Azure engineer focused on [distributed systems / platform /
         a domain]."
PROOF 1 "Most recently I [designed/led] [system or cross-team change] — [scale/number] — which
         [outcome with number]."
PROOF 2 "Before that I [mechanism or deep technical achievement] that [outcome across teams]."
WANT    "I'm looking for a [staff] role where I own the technical direction of [an area] with the teams
         that build it — which is why this role on [their team/problem] caught my attention."
```

### Architect version

```text
WHO     "I'm a [solution/software] architect on .NET and Azure, mostly in [modernisation / integration /
         platform] work."
PROOF 1 "On my last programme I framed and drove the decision to [X] — options, the decision records,
         the cut-over plan — and we [outcome: time, cost, risk, with numbers]."
PROOF 2 "I also introduced [governance mechanism] that teams actually used — [adoption or approval-time
         number]."
WANT    "I'm looking for a role where making the expensive decisions and getting teams behind them is the
         job — and from what I understand, your [programme/platform] is exactly that."
```

### Vendor / customer-facing version

```text
WHO     "I'm a .NET and Azure architect who's spent much of my time working directly with [customers /
         business stakeholders]."
PROOF 1 "I helped [customer type] move [workload] to Azure — [outcome] — mostly by [how you earned
         trust: a PoC, a workshop, saying no to the wrong service]."
PROOF 2 "[A presentation or executive conversation that changed a decision]."
WANT    "I want to do that across many customers, with the breadth of the platform behind me."
```

---
# Appendix F — Evidence inventory template

```text
STORY (one line)      ROLE/EMPLOYER   YEAR   SCOPE AMBIG INFL IMPACT HORIZ LEVER   FAMILY TAG    NUMBERS READY?
____________________  ___________     ____    _     _     _     _      _     _     __________    □
____________________  ___________     ____    _     _     _     _      _     _     __________    □
...

SCALE  1 = mid · 2 = senior · 3 = staff · 4 = principal / EA (Concept 14)
RULES  Defensible level = highest level with ≥3 stories at that level on ≥4 axes.
       Family shape = which axes are high, and whether stories are decisions-and-alignment or
       mechanisms-and-depth.
GAPS   Axis with no story at target level → story to develop or reframe: ____________________
```

---
# Closing the module

This module turns the curriculum from a syllabus into a plan. The throughline from Part 0 still holds: past mid-level, interviews grade **judgement**. The first judgement they reward — before any design question — is your own: knowing which job you're applying for, what its loop will sample, and how to spend your preparation where that loop decides.

A practical way to use it from here:

1. **Write the target statement** (Appendix A) and run the four checks.
2. **Run the allocation** (Appendix D) and let it reorder the modules you revisit.
3. **Research the specific loop** (Module 39) and adjust the relevance column if the loop differs from the family default.
4. **Mock the rounds that loop contains** (Module 38), scored with this module's weights.
5. **Re-calibrate after real loops** — on patterns, one change at a time.
