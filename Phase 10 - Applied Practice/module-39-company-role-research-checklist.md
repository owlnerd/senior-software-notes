# Module 39 — Company/Role Research Checklist
*Phase 10: Applied Practice · Senior/Architect Interview Prep for .NET & C# · the final module*

> **State of the world verified on October 8, 2026.** The method in this module is stable — it is ordinary decision analysis and source criticism. What moves fast is the information environment you research *in*. The facts that calibrate the module:
>
> - **Loops differ more than they look.** interviewing.io's senior-engineer guide to Microsoft describes a **team-dependent** process: recruiter call (plus a hiring-manager call at principal and above), a coding phone screen *or* an asynchronous Codility quiz, an onsite of roughly **one behavioral, two or three coding, one system design and one domain-specific round**, then **team matching** — and Microsoft lets you interview with **several teams at once**. The same source describes **Google and Meta** as **centralised** processes (one loop, team matching afterwards, no parallel team interviews), with Meta the most standardised. "Big Tech interviews" is not one thing; it is several different instruments.
> - **AI rules are per company, and sometimes per round.** Microsoft's **Candidate Code of Conduct** encourages AI for *preparation* but expects candidates to complete assessments and interviews **without AI unless specifically instructed**. Google's virtual-interview guidance says using AI during the interview leads to **disqualification**. Module 38 covered the other side — Meta's AI-enabled coding round, Canva's AI-expected round, Anthropic's prep-yes/interview-no rule. The AI mode is a research item for every round.
> - **Pay transparency is arriving unevenly.** The **EU Pay Transparency Directive (2023/970)** had a transposition deadline of **7 June 2026**; by late summer only **five member states** (Slovakia, Lithuania, Italy, Malta, Greece) had fully transposed it, **Sweden** halted transposition, and the **Netherlands** pushed entry into force to **1 January 2027**. In the **US**, about **16 states plus DC** require pay disclosure (Delaware joins in 2027), and many apply to remote postings performable in those states. Practical meaning: you can increasingly *see* or *ask for* the range — but the right is jurisdiction-specific.
> - **The .NET support cliff is this month's brownfield question.** **.NET 8 (LTS) and .NET 9 (STS) both reach end of support on 10 November 2026**; **.NET 11** (STS) ships the same day; **.NET 10** is the LTS, supported to **November 2028**. The **Azure Functions in-process model** also ends support on 10 November 2026. Any .NET team you research right now is either done with this migration, mid-migration, or about to have a problem — and the answer tells you a lot.
> - **Certification signals moved.** **AZ-204** (Azure Developer Associate) **retired on 31 July 2026**, replaced by **AI-200** (Azure AI Cloud Developer Associate). **AZ-305** (Azure Solutions Architect Expert, with AZ-104 as prerequisite) remains. A job description still asking for "AZ-204 required" is a minor staleness signal, not a disqualifier — existing AZ-204 credentials remain valid.
> - **Work-model policies are volatile.** Microsoft moved to **three days a week on-site** for staff within 50 miles of an office, starting with Puget Sound at the end of February 2026 and rolling out to other US and then international locations during 2026. Whatever a company's careers page said a year ago, verify it now.
> - **The raw data is biased, and the bias is measurable.** Glassdoor's own economists (Marinescu et al., NBER 2018) showed **voluntary employer reviews are distributed differently from incentivised ones** — people with extreme opinions post more. Since 2024 Glassdoor has required **real-name verification** for accounts (reviews remain anonymous on the page), which changed who posts. Analyses of hiring-platform data put **"ghost" postings** (no current intent to hire) at roughly **one in five** listings; there are no official statistics.
>
> Products, policies and laws change; the discipline of asking which decision a fact changes, and how reliable its source is, doesn't.

## Orientation

Here is the sentence to carry through the whole module: **company and role research is uncertainty reduction in service of three decisions — whether to pursue an opportunity, how to prepare for its specific loop, and whether to accept the offer — so every item worth researching must be able to change one of those decisions, must come from a source whose reliability you've graded, and must be confirmed by a second independent source before you act on it.**

The curriculum entry reads: *Company/role research checklist.* Module 38 ended with the promise that this module would cover how to find out what a specific company's loop contains (round types, AI policy, in-person rounds, levelling), map its values and rubric vocabulary to the six dimensions, tailor your story bank and mock catalogue to it, and prepare the questions you'll ask your interviewers. All of that is here — plus the decision at the end, which most preparation guides skip.

**Why this module exists.** Modules 1–38 made you a stronger candidate *in general*. But you never interview in general. You interview for one team, at one company, at one level, in one loop with its own round types, its own vocabulary and its own rules — and then you decide whether to spend the next several years there. Three facts make research worth doing properly:

1. **The same candidate scores differently in different loops.** A loop with a domain-specific round on Azure networking, a loop built around Leadership Principles and a loop with an AI-enabled coding stage are different tests. Preparing for "the interview" instead of *this* interview wastes your strongest material and leaves you exposed where this loop actually probes.
2. **Fit is scored, and fit is mostly preparation.** "Why do you want to work here?", "What do you know about our product?", "What questions do you have?" — every senior loop asks some version of these, and generic answers read as low motivation or low judgement. Specific answers are only possible if you did the work.
3. **The decision at the end is high-stakes and hard to reverse.** A job shapes years of your career, compensation and learning. Most engineers spend more time researching a laptop than the employer they'll give 2,000 hours a year to. For an architect especially, joining an organisation whose decision-making culture you misread is the most common way to have a bad two years.

How this connects to earlier modules:

- **Modules 1–2** (what's scored; IC vs architect loops) — research tells you which loop shape *this* company runs and which rubric vocabulary it uses.
- **Modules 4 and 30–33** (requirements gathering; design documents, ADRs, brownfield, cost) — company research is requirements gathering applied to a career decision, and an architect's first weeks in a new job are exactly this research done from the inside.
- **Modules 34–35** (STAR, story bank) — research decides which stories to lead with and how to tag them in the company's vocabulary.
- **Module 37** (worked design problems) — research tells you what a *company-shaped* design problem looks like, so you can rehearse one.
- **Module 38** (mocks and rubrics) — research sets the mock catalogue, the priority weights and the AI mode for each mock.

Why it matters in interviews:

1. **Motivation and fit questions are scored**, explicitly at some companies (Amazon's Leadership Principles appear in every round) and implicitly everywhere (a hiring manager's "would this person choose us for the right reasons?").
2. **Your questions are evaluated.** The questions you ask at the end of each round reveal what you care about and how you think. For staff and architect candidates, they are often the best evidence of judgement the interviewer gets.
3. **Level is decided by evidence and negotiated by information.** Down-levelling is common for experienced hires moving between companies; knowing the ladder and what each level means there lets you present evidence at the right altitude.
4. **Architect roles are research roles.** "What would you do in your first 90 days?" is a standard architect question, and the strong answer is a research plan — stakeholders, systems, constraints, decision history. Showing that you researched *them* before the interview is a live demonstration.

This module has six jobs:

1. **Treat research as decision support** — the three decisions, the value of information, grading sources, the biases in the data, triangulation and the dossier (Part A).
2. **Research the company** — business, financial health, engineering culture, stack and maturity (with .NET/Azure signals), work model and employment terms (Part B).
3. **Research the role and the loop** — the job description, the level, the team and manager, the loop structure, the AI rules and the recruiter call (Part C).
4. **Turn research into preparation** — values mapped to the six dimensions, story-bank tagging, a company-shaped mock catalogue, the "why us" answer, your questions, the per-round card (Part D).
5. **Decide** — reading the loop itself as evidence, evaluating and comparing offers, and making a reversible-or-not decision well (Part E).
6. **Give you the tools** — a worked dossier, templates, question banks, a recruiter-call script, an offer comparison tool in C#, and checklists (appendices).

Seven framings to carry through:

1. **Research is for decisions, not for knowing things.** If a fact can't change whether you apply, how you prepare or whether you accept, stop collecting it.
2. **Questions first, sources second.** Write the question you need answered before you open a browser; otherwise you'll read the careers page and believe it.
3. **Grade every source.** Separate *how reliable the source is* from *how plausible the claim is*. An official page is reliable about policy and unreliable about culture.
4. **The data is biased in known directions.** Reviews over-represent the angry and the delighted; interview reports over-represent the memorable; careers pages over-represent the aspirational. Correct for it.
5. **Two independent sources before you act.** One anonymous post is a hypothesis. A recruiter's statement plus a matching engineer's account is a fact you can plan around.
6. **The loop is data.** How a company runs its interview process — organisation, honesty, respect for your time, the questions interviewers ask and dodge — is the most direct evidence of how it runs everything else.
7. **Depth scales with commitment.** Thirty minutes before applying, a few hours before the loop, as long as it takes before signing.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What research is for | Three decisions — pursue, prepare, accept — and only facts that change them matter |
| 2 | The research question tree | Write questions before opening sources; four branches: company, role, loop, offer |
| 3 | Grading sources | Reliability of the source × credibility of the claim, graded separately |
| 4 | Biases in the data | Selection, survivorship, recency, branding, ghost postings, ignored base rates |
| 5 | Triangulation and the dossier | Two independent sources; a claims log with confidence and an expiry date |
| 6 | Business model and strategy | How the company makes money decides where engineering sits and how it's paid |
| 7 | Financial health and stability | Public filings, funding and runway, registries, layoffs, reorganisations |
| 8 | Engineering culture | Westrum, DORA, Conway, team types, incidents and decision-making |
| 9 | Stack and maturity — the .NET/Azure lens | Versions, hosting models, the November 2026 cliff, delivery and operations signals |
| 10 | Work model and employment terms | Hybrid/remote reality, hiring entity, employee vs contractor, time zones, pay rights |
| 11 | Decoding the job description | Must-haves vs wish list, level signals, staleness, archetype |
| 12 | Level research and calibration | Ladders, cross-company mapping, down-levelling, how to raise it |
| 13 | The team and the manager | Why the role is open, team type, first-year problems, manager style |
| 14 | Mapping the loop | Round types, owners, media, centralised vs team-dependent, matching, cooldowns |
| 15 | AI policy and round rules | No AI / permitted / expected — per round, in writing |
| 16 | The recruiter call as instrument | A scripted call that fills the loop map and protects your position |
| 17 | Values → the six dimensions | Translate company vocabulary into your rubric; retag the story bank |
| 18 | Company-shaped mocks | Domain rounds, company-shaped design problems, re-weighted priorities |
| 19 | "Why us, why this role, why now" | A specific, evidenced, two-sided answer in under 90 seconds |
| 20 | Your questions | Designed to extract decision-relevant information, per interviewer type |
| 21 | The per-round card | One page per round: rules, focus, stories, questions |
| 22 | Reading the loop as evidence | Process quality, interviewer signals, red and green flags |
| 23 | Evaluating an offer | Components, vesting, level, risk, scenarios — computed, not eyeballed |
| 24 | Making the decision | Weighted criteria, premortem, reversibility, closing the log |

---
# Part A — Research as decision support

## Concept 1 — What research is for: three decisions

Start from first principles. Research costs time — usually your scarcest resource during a job search, when you are also preparing, interviewing and possibly working. It is only worth doing to the extent it improves a decision. In a job search you make three decisions per opportunity, at three different moments, with three different budgets:

| Decision | When | Question | Typical budget | Cost of getting it wrong |
|---|---|---|---|---|
| **D1 — Pursue?** | Before applying or replying to a recruiter | Is this worth a loop? | 15–45 min | Wasted loop (10–20 hours of prep and interviews); cooldown if it goes badly |
| **D2 — Prepare how?** | After the recruiter call, before the loop | What exactly will this loop test, and in what vocabulary? | 2–6 hours | Misdirected preparation; a weaker loop than you're capable of |
| **D3 — Accept?** | Offer in hand | Is this the right job, at the right level, on the right terms? | As long as it takes (often 5–15 hours) | Years in the wrong role; comp left on the table; a hard-to-reverse move |

Two consequences follow immediately.

**First, research is staged.** Doing D3-depth research before D1 is waste — most opportunities won't reach an offer. Doing D1-depth research before D3 is reckless — the stakes are highest at the end. The budget should grow as your commitment grows, and you should carry the dossier forward so each stage builds on the last.

**Second, every research item has an owner decision.** "The company was founded in 2009" changes nothing. "The team is mid-migration from .NET Framework 4.8 on IIS to .NET 10 on Container Apps" changes D2 (prepare a brownfield story and a strangler-fig design) and D3 (is this the kind of work you want?). If you can't name the decision a fact feeds, it's trivia.

### The value of information

Decision analysis gives a clean way to think about *how much* research a question deserves: the **expected value of information** is how much better your decision is expected to be with the information than without it. It is high only when three conditions hold together:

1. **You're genuinely uncertain** — you don't already know the answer with high confidence.
2. **The answer could flip the decision** — some plausible answers lead to a different choice.
3. **The stakes are material** — the difference between the choices matters.

Apply it to common research questions:

| Question | Uncertain? | Could flip a decision? | Stakes | Worth researching? |
|---|---|---|---|---|
| "Is the role really remote from my country?" | Often | Yes — D1 outright | High | **Yes, first** — before anything else |
| "Does the loop include a domain-specific round, and on what?" | Usually | Changes D2 a lot | Medium–high | **Yes**, at the recruiter call |
| "What's the AI policy for each round?" | Often | Changes D2 practice mode | Medium; disqualification risk | **Yes**, in writing |
| "What's the founding story?" | Rarely matters | No | Low | No |
| "Is the company profitable or how long is the runway?" | For startups, yes | Flips D3 | High | **Yes** for startups; less for large public companies |
| "Exact office layout" | — | Rarely | Low | Only if it matters to you personally |

The test is quick, and it is the main defence against the most common failure: hours spent reading a company's blog and none spent finding out whether the role is at your level.

### What a decision-oriented researcher produces

At each stage, research should end with a written output that *is* the decision:

- **D1 output:** a one-paragraph go/no-go with the two or three facts that drove it, and the open questions for the recruiter.
- **D2 output:** a loop map (Concept 14), a prep plan (Concept 18) and a set of per-round cards (Concept 21).
- **D3 output:** an offer comparison (Concept 23) and a decision record (Concept 24) — an ADR for your career, in the same shape as Module 31.

**Interview-grade sentence** *(if asked how you approach a new company, a new domain or a vendor evaluation — the method transfers)*: *"I research in stages tied to decisions — enough to decide whether to pursue it, then enough to prepare for exactly what it'll test, then as much as it takes before committing — and for each question I ask whether the answer could actually change what I do; if it can't, I don't spend time on it."*

---

## Concept 2 — Questions before sources: the research question tree

The second most common failure is **source-driven research**: open the careers page, read what it says, open Glassdoor, read what it says, and end up with a pile of impressions shaped by whatever those sources chose to tell you. The fix is the same as in Module 4 (requirements before design): **write the questions first**, then go looking for answers, and notice which questions no source answers — those become your questions for the recruiter and interviewers.

### The four branches

```text
                              OPPORTUNITY
        ┌──────────────┬──────────────┬──────────────┬──────────────┐
     COMPANY          ROLE           LOOP          OFFER/TERMS
  how it makes      what the job    what's tested,  what you'd get,
  money; health;    really is; the  how, by whom,   on what terms,
  culture; stack;   team; the level; under which    with what risk
  work model        why it's open   rules
     (D1, D3)        (D1, D2, D3)     (D2)           (D1 range, D3)
```

### The core questions per branch

**Company (feeds D1, D3)**
1. How does the company make money, and is engineering a profit centre or a cost centre here? (Concept 6)
2. Is it financially healthy and stable — growing, flat, shrinking, reorganising? (Concept 7)
3. How does engineering actually work — information flow, incidents, decisions, on-call? (Concept 8)
4. What's the stack and how mature is the delivery and operations practice? (Concept 9)
5. Can I work there from where I am, on what terms? (Concept 10)

**Role (feeds D1, D2, D3)**
6. What is this job day to day, and what does the job description really require? (Concept 11)
7. What level is it, and what does that level mean at this company? (Concept 12)
8. Who is the team, who is the manager, and why is the role open? (Concept 13)
9. What would success at 6 and 12 months look like?

**Loop (feeds D2)**
10. What are the rounds — type, length, interviewer, medium, order? (Concept 14)
11. What are the rules — AI mode, language, take-home, in-person? (Concept 15)
12. What vocabulary do they score in — values, principles, competencies? (Concept 17)
13. How is the decision made, and what happens after (team matching, committee, timeline)?

**Offer/terms (feeds D1 range check, D3)**
14. What's the compensation structure and band for this level and location? (Concept 23)
15. What are the employment terms — entity, contract type, notice, IP, non-compete, probation?
16. What risk sits in the package — equity liquidity, currency, bonus variability?

### Which questions sources can answer — and which only people can

| Question type | Best answered by | Why |
|---|---|---|
| Facts the company must publish (financials, legal entity, pay range in some jurisdictions) | **Official / regulatory sources** | Legal obligation; high reliability |
| Facts the company chooses to publish (values, process, benefits) | **Careers pages, engineering blog** — then verify | Reliable about *stated* policy; aspirational about practice |
| Practice and culture (on-call reality, decision-making, how incidents go) | **People** — current and recent engineers, the interviewers themselves | Not written down anywhere reliable |
| This specific team, manager, role | **The recruiter and hiring manager**, then the loop | Varies team to team; public sources rarely reach this level |
| Compensation for this level/location | **Aggregated self-reported data** + the recruiter's range | Individual reports are noisy; ranges are often required to be disclosed |

The pattern: **public sources answer the company questions, people answer the team questions.** Plan accordingly — the recruiter call and the last five minutes of every round are research time.

**Interview-grade sentence:** *"I start with written questions in four branches — the company, the role, the loop and the terms — before I look at any source, because otherwise the sources set the agenda; and I keep track of which questions public material can't answer, because those become the questions I ask the recruiter and the interviewers."*

---

## Concept 3 — Grading sources: reliability and credibility are different things

Intelligence analysts have used a simple discipline for decades: the **Admiralty system** (also known as the NATO system) grades each piece of information on **two independent axes** — how reliable the **source** is (A–F) and how credible the **information** is (1–6). A completely reliable source can relay a false claim; an unknown source can be right. Collapsing the two into one "trustworthiness" feeling is how researchers get fooled.

Adapted for job research:

| Source reliability | Meaning | Examples |
|---|---|---|
| **A — Authoritative** | Legally accountable or directly responsible for the fact | Regulatory filings, a public company's annual report, a company registry, the company's own statement of its *policy*, the recruiter's written statement of the loop |
| **B — Usually reliable** | Knowledgeable, generally careful, may lag or simplify | Established engineering press, well-maintained compensation aggregators, guides written with many current interviewers |
| **C — Fairly reliable** | Knowledgeable but partial | An individual engineer you can verify works there; a hiring manager describing their own team |
| **D — Not usually reliable** | Often partial, motivated or out of date | A single anonymous review; a forum thread; content farms summarising other sources |
| **E — Unreliable** | Known to be wrong or manipulated | Paid "employer reputation" content; scraped job boards with stale listings |
| **F — Cannot judge** | No basis to assess | A screenshot with no context |

| Information credibility | Meaning |
|---|---|
| **1 — Confirmed** | Independently confirmed by a second source |
| **2 — Probably true** | Consistent with other information, not yet confirmed |
| **3 — Possibly true** | Plausible, no corroboration |
| **4 — Doubtful** | Possible but inconsistent with other information |
| **5 — Improbable** | Contradicted by better information |
| **6 — Cannot judge** | No basis |

### Reliability depends on the question

The key subtlety for job research: **a source's reliability is not a property of the source alone — it's a property of the source *for a kind of question*.**

| Source | Reliable (A–B) for… | Unreliable (D–E) for… |
|---|---|---|
| Careers page / values page | What the company *says* it values; official process steps; stated AI policy | Whether the values are lived; team-specific reality |
| Annual report / regulatory filing | Revenue, segments, risk factors, headcount trends | Engineering culture |
| Recruiter | Loop structure, timeline, range, level being considered | Team culture, manager quality (they're incentivised to close) |
| Hiring manager | The team's problems, priorities, why the role is open | How they themselves are as a manager (self-report) |
| Current engineer (verified) | Day-to-day work, on-call, tooling, decision-making | Company-wide strategy |
| Anonymous review sites | Recurring themes across many reviews | Any single review's claim |
| Interview-report sites | Round types and topics, broadly | The exact question you'll get; difficulty (reports skew to memorable rounds) |
| Compensation aggregators | Ranges by level and location, with enough data points | A single offer; small offices with few data points |

### How to use the grades

Write the grade next to every claim in your dossier (Concept 5): `B2: domain round is customised to the team's area (interviewing.io guide, consistent with recruiter mention)`. Then apply three rules:

1. **Act on A1–B2.** Plan preparation around these.
2. **Investigate C3–D3.** Turn them into questions for the recruiter or an engineer.
3. **Discard E/5 and below**, unless several independent D-sources point the same way (then it's a theme worth asking about).

**Interview-grade sentence** *(useful for any question about evaluating vendors, technologies or incident reports)*: *"I grade information on two separate axes — how reliable the source is for this kind of question, and how credible the specific claim is given everything else — so an official page counts as authoritative about policy but weak about culture, and a single anonymous review counts as a hypothesis to check rather than a fact to act on."*

---

## Concept 4 — The biases in job-market data

Every source you'll use is a filtered view of reality, and the filters are **systematic** — they push in known directions. Knowing the direction lets you correct for it.

### 1. Selection bias in reviews

Who writes an employer review? Mostly people with strong feelings — very satisfied or very dissatisfied — plus people prompted to post. Marinescu, Klein, Chamberlain and Smart (NBER Working Paper 24372, 2018), using Glassdoor's own data, compared **voluntary** reviews with reviews **incentivised** by the site's "give-to-get" model and found the distributions differ — consistent with voluntary reviewers being more extreme — and that the bias was large enough to **change industry rankings** by average satisfaction. Their follow-up experiment found that incentives reduced the bias. Product-review research shows the same shape: Hu, Pavlou and Zhang (2009) described the **J-shaped distribution** of online ratings, where extreme scores are over-represented relative to the underlying opinions.

**Corrections:**
- Read **the distribution and the themes**, not the average rating. Recurring, specific complaints across many reviews and years ("on-call is unpaid and heavy", "reorgs every six months") are signal; the average is mostly noise.
- **Filter by role and recency** — engineering reviews from the last 12–18 months, ideally from the same org.
- Weight **specific** claims over adjectives. "Deploys take two days because of a manual change board" is informative; "toxic" is not.
- Remember the 2024 change: real-name verification for Glassdoor accounts may have shifted who posts. Treat older and newer reviews as slightly different samples.

There is also evidence the aggregate carries *some* signal: Green, Huang, Wen and Zhou (2019) found that changes in crowdsourced employer ratings predicted stock returns. So the right stance is neither "reviews are useless" nor "reviews are truth" — they're a **biased, noisy instrument with real signal in the trends**.

### 2. Survivorship bias

Current employees who answer your questions are, by definition, people who stayed. People who left fastest — often the best informants about the problems — are invisible unless you look for them. **Correction:** seek **recent leavers** (people who left in the last year) as well as current staff; ask current staff "who left recently and why?".

### 3. Recency and decay

Companies change faster than their reputations. A culture article from 2021, a pre-reorganisation team structure, a pre-RTO remote policy — all can be confidently wrong. **Correction:** every claim in the dossier gets an **as-of date** and an **expiry** (Concept 5). As defaults: policies (remote, AI, process) decay in **6–12 months**; team structure in **3–6 months** (often less after reorganisations); compensation bands in **6–12 months**; financial health in **one quarter** for public companies.

### 4. Employer branding

Careers pages, engineering blogs and conference talks are marketing — usually honest, but selected. They show the most impressive projects, the newest stack and the aspirational culture. Engineering blogs over-represent the 5% of the codebase that's interesting. **Correction:** use them for **what the company wants to be seen as** (which tells you what it values and what vocabulary to use) and verify **practice** with people. A useful question in the loop: *"The blog post on X was great — how much of the codebase looks like that today?"*

### 5. Interview-report bias

Interview reports over-represent **memorable** rounds — very hard, very strange or very bad — and candidates who failed are more motivated to vent. Reports also mix levels, teams and years. **Correction:** use them for **round types and topic areas**, never for the exact question; weight reports matching your **level and org** and the last 12 months.

### 6. Ghost postings

A share of postings don't correspond to an active, funded intent to hire now — evergreen pipelines, paused headcount, compliance postings, or roles with an internal candidate already chosen. Analyses of hiring-platform data put it around **one in five** active listings; there are no official statistics, and estimates vary widely by method. **Corrections:** prefer roles where a **named recruiter or hiring manager** reached out or responds; check whether the posting has been **re-posted repeatedly** for months; ask the recruiter directly *"Is this headcount approved and open now, and what's the target start date?"* — a vague answer is information.

### 7. Ignoring base rates (the inside view)

Kahneman and Lovallo's distinction between the **inside view** (reasoning from the specifics of your case) and the **outside view** (reasoning from the base rate of similar cases) applies directly. Inside view: "the hiring manager loved me, the team is great, this will be my job for five years." Outside view: how long do engineers typically stay at this company? What fraction of senior hires there are down-levelled? How often do startups at this stage raise their next round? **Correction:** for every important prediction, write the base rate first, then adjust for specifics — this is **reference-class forecasting**.

### Summary of corrections

| Bias | Direction | Correction |
|---|---|---|
| Selection (reviews) | Extremes over-represented | Themes and distributions, recent, same role |
| Survivorship | Problems under-represented among current staff | Talk to recent leavers |
| Recency | Old truths persist | As-of dates and expiries |
| Branding | Aspirational over actual | Blog = values; people = practice |
| Interview reports | Memorable and failed over-represented | Round types, not questions; match level and org |
| Ghost postings | Opportunity overstated | Named contacts; approved-headcount question |
| Inside view | Optimism about your case | Base rates first, then adjust |

**Interview-grade sentence:** *"I treat review sites and interview reports as biased instruments with known directions — extreme opinions and memorable rounds are over-represented, current staff are survivors, and branding is aspirational — so I read for recurring specific themes in recent, role-matched data, talk to people who left as well as people who stayed, and anchor predictions on base rates before adjusting for the specifics."*

---

## Concept 5 — Triangulation and the dossier

### Triangulation

**Triangulation** means confirming a claim through **independent** sources — sources that didn't get the information from each other. Two blog posts quoting the same Glassdoor review are one source, not two. A recruiter's statement and a current engineer's description of the same round *are* two.

The independence test: **"Could both sources be wrong for the same reason?"** If yes (they share an origin, an incentive or a bias), they don't triangulate.

| Claim | Weak triangulation | Strong triangulation |
|---|---|---|
| "The loop has a domain-specific round" | Two interview-report sites | Interview guide + recruiter's written loop description |
| "On-call is heavy" | Three anonymous reviews in one month (could be one bad incident) | Reviews across two years + an engineer's answer to "how many pages did you get last rotation?" |
| "The team is migrating to .NET 10" | The job description + the engineering blog (both company-authored) | The job description + the hiring manager's answer + a public repo or conference talk showing the work |
| "Comp band for this level is X–Y" | One aggregator | Aggregator with enough data points + the recruiter's disclosed range |

### The dossier: a claims log, not a pile of notes

Keep one file per opportunity. Its core is a **claims log** — every fact you might act on, with its grade, its source, an as-of date and an expiry. This is the research equivalent of an ADR log (Module 31): it makes your reasoning auditable later, including by you.

```text
CLAIM                                         GRADE  SOURCE(S)                          AS-OF       EXPIRES   FEEDS
Loop: recruiter → coding screen → onsite      B2     interviewing.io guide              2026-10-08  2027-04   D2
  (1 behav, 2–3 coding, 1 SD, 1 domain)
Recruiter confirms 4 onsite rounds incl.      A1     recruiter email 2026-10-14         2026-10-14  loop      D2
  "domain: storage internals"
AI not permitted in any round                 A1     candidate code of conduct + recruiter 2026-10-14 loop    D2
Team on .NET 8, migrating to .NET 10 by Nov   C3     hiring manager (call)              2026-10-16  2027-01   D2, D3
On-call: 1 week in 6, follow-the-sun          C2     engineer (recent leaver)           2026-10-18  2027-04   D3
```

Three design choices make the log useful:

- **One claim per row.** Easy to grade, easy to contradict, easy to update.
- **An expiry date on every row.** A stale claim should look stale when you open the file again in three months for a different team at the same company.
- **A "feeds" column.** Forces the value-of-information question (Concept 1) every time you write something down.

### The dossier's other sections

Beyond the claims log, the dossier holds: the **question tree** with open questions (Concept 2), the **loop map** (Concept 14), the **story mapping** (Concept 17), your **question bank** for each interviewer (Concept 20), the **per-round cards** (Concept 21), **post-round notes** (Concept 22), and finally the **offer analysis and decision record** (Concepts 23–24). Appendix A is a full template.

### Time budgets

| Stage | Budget | What gets done |
|---|---|---|
| **D1 (pursue?)** | 15–45 min | Location and work-model check; business model in one line; level and range sanity check; two or three culture themes; go/no-go |
| **Recruiter call prep** | 30–60 min | Question tree; the recruiter-call script (Concept 16) |
| **D2 (prepare)** | 2–6 hours | Loop map; values mapping; story retagging; company-shaped mocks; per-round cards; question bank |
| **Between rounds** | 15 min per round | Update the loop map and claims log; adjust the next card |
| **D3 (accept?)** | As needed, often 5–15 hours | Engineer conversations; financial check; offer model; decision record |

**Interview-grade sentence:** *"I confirm anything I'm going to act on with two independent sources — independent meaning they couldn't be wrong for the same reason — and I keep a claims log per company with a reliability grade, an as-of date and an expiry on each claim, so my preparation rests on facts I can trace and I notice when they've gone stale."*

---
# Part B — Researching the company

## Concept 6 — Business model and strategy: where engineering sits

The single most predictive fact about an engineering job is **how the company makes money and where engineering sits relative to that money.** It determines budgets, headcount stability, how architecture decisions are justified, what "senior" means and how you're paid.

### Profit centre or cost centre

| | Engineering as **product** (profit centre) | Engineering as **IT** (cost centre) |
|---|---|---|
| What engineering builds | The thing customers pay for | Systems that support the business that customers pay for |
| How investment is justified | Revenue, growth, retention | Cost reduction, risk, compliance, efficiency |
| Typical architecture conversation | "What does this enable?" | "What does this cost and what risk does it remove?" |
| Who decides | Product and engineering leadership | Business owners, often through a governance board |
| Pressure in a downturn | Cut speculative bets; protect the core product | Cut projects; outsource; freeze headcount |
| Architect role shape | IC-track architect / staff engineer close to the product | Solution and enterprise architects, governance, vendor management |

Neither is better — but they are different jobs, and Module 33's language (cost, build-vs-buy, debt) lands very differently in each. A solution architect at an insurer and a staff engineer at a SaaS company both "do architecture", but the first spends much of the time on integration, vendors, compliance and governance, and the second on product scaling and technical bets.

### The compensation tiers follow the same logic

Gergely Orosz's widely cited **trimodal** model of software engineering pay (first published for the Netherlands and Europe in 2021, revisited since for other markets) describes three loose tiers of employers by **who they benchmark compensation against**:

1. **Tier 1** — companies benchmarking against local peers in their own industry; engineering often seen as IT; salary-only packages.
2. **Tier 2** — companies benchmarking against *all* local employers; often funded scale-ups and strong local tech companies; bonuses and some equity.
3. **Tier 3** — companies benchmarking regionally or globally (Big Tech, top scale-ups, trading firms); large equity and bonus components.

Ranges between tiers barely overlap for the same title, and public salary portals mostly capture the lower tiers. Two research consequences: **identify the tier before comparing numbers**, and **expect work-life trade-offs to correlate with tier** — the same article notes that higher tiers often mean higher performance pressure, global time zones and on-call expectations.

### Questions to answer about the business

| Question | Where to look | Why it matters |
|---|---|---|
| Who pays, for what, how (subscription, usage, transaction, licence, ads, services)? | Website pricing, annual report, investor material | Tells you what "scale" and "reliability" mean in money — e.g. usage-based pricing makes metering and billing accuracy core |
| What are the business segments and which is growing? | Annual report segment data; earnings calls | Growing segments get headcount and interesting problems; flat ones get maintenance |
| Where does this team's product sit — core, adjacent, experimental? | Org charts, hiring manager | Experimental teams are exciting and the first cut in a downturn |
| Who are the competitors, and what's the stated differentiation? | Annual report, analyst coverage, product comparison pages | Gives you substance for "why us" and for design-round domain framing |
| What's the strategy for the next 1–2 years? | Shareholder letters, keynotes, CEO interviews | Lets you connect your experience to their direction |
| Is AI a product, a productivity bet, or both? | Product announcements, earnings calls | In 2026 this shapes many teams' roadmaps and many interviews' follow-ups |

### For Microsoft-ecosystem roles specifically

Three common employer types hire senior .NET/Azure engineers and architects, and each has a different business model:

| Employer type | How it makes money | What the senior/architect job emphasises |
|---|---|---|
| **Product companies on .NET/Azure** (SaaS, fintech, marketplaces) | Their product | Scaling, reliability, cost per tenant, product velocity |
| **Enterprises** (banks, insurers, retailers, public sector) | Their core business; software is internal | Integration, modernisation, governance, compliance, vendor management |
| **Consultancies and Microsoft partners** | Selling engineers' time and delivery outcomes | Client-facing design, pre-sales, estimation, many short engagements; certifications and partner status often matter commercially |
| **Microsoft itself** | Cloud, software, devices, services | Product engineering at very large scale; compliance and operational rigour; team-specific domains |

For a consultancy, ask how utilisation is measured and what share of time is billable — it determines whether "architect" means designing systems or selling them.

**Interview-grade sentence** *(for "what do you know about our business?")*: *"Before anything technical I want to know how the company makes money and where this team sits relative to that — whether engineering is the product or supports it — because that decides how architecture decisions get justified, what the team is funded to do and what 'good' looks like in this role."*

---

## Concept 7 — Financial health and stability

You don't need to be an analyst. You need answers to three questions: **Is the company growing, stable or shrinking? Is the money to pay for this team secure for the next two years? Is a reorganisation likely to change the job you're joining?**

### Public companies

| Source | What to read | What it tells you |
|---|---|---|
| **Annual report / 10-K** (US: SEC EDGAR) | Business description; segment revenue and growth; **risk factors**; headcount | Strategy in the company's own legally accountable words; which segments matter |
| **Quarterly results and earnings calls** | The CEO/CFO's priorities; analyst questions | Where investment is going; whether "efficiency" is the theme (often precedes cuts) |
| **Investor relations page** | Shareholder letters, investor-day decks | The strategic narrative; key metrics |

Read **risk factors** for engineering-relevant items: dependence on a single cloud, regulatory exposure (data residency, AI regulation), large migrations, security incidents. These often map directly to the problems the team you're joining exists to solve.

### Private companies and startups

| Question | Where to look | Why |
|---|---|---|
| When was the last funding round, how much, from whom? | Funding databases (Crunchbase, Dealroom — partly paid), press releases | Recency and size of the last round bound the runway |
| How long is the runway? | **Ask** — founders and hiring managers at funded startups usually answer | Under ~18 months means the next round or profitability is a near-term existential question |
| Is it profitable or on a path to profitability? | Ask; registry filings where available | Profitability makes headcount far more stable |
| Who are the investors and what do they expect? | Funding databases | Growth-stage investors push for growth; this shapes the pace |
| Is equity real? | The offer documents (Concept 23) | Valuation, preference stack and exit horizon decide whether options are worth anything |

### Company registries — underused and highly reliable

Many countries publish companies' **filed financial statements** for free. They are **A-grade** sources for revenue, profit and often employee counts:

- **UK — Companies House**: filed accounts, officers, ownership, for every UK company.
- **US — SEC EDGAR**: filings for public companies (and some private ones with registered securities).
- **Serbia — APR (Agencija za privredne registre)**: the Registry of Financial Statements publishes companies' annual financial statements, searchable by registration number (*matični broj*), tax ID (PIB) or name, including the reported **number of employees**. For a local subsidiary of a foreign company or a local product company, this tells you whether the entity is growing and roughly how many people it employs.
- **OpenCorporates** aggregates registry data across many jurisdictions.

A subtlety for multinationals: the **local entity** that would employ you may be a cost-plus service company whose own accounts show modest, stable profit by design. Its growth in **headcount and revenue** over three years is more informative than its profit.

### Layoffs, reorganisations and leadership change

| Signal | Where | Interpretation |
|---|---|---|
| Recent layoffs | Layoff trackers, news, company announcements | Not disqualifying — many healthy companies have cut — but ask how this org was affected and whether the role is a backfill |
| Frequent reorganisations | Reviews, LinkedIn tenure patterns, engineers | Your team, manager and mandate may change within a year; weigh the company over the team |
| New CTO/VP Engineering in the last 12 months | Press, LinkedIn | Expect strategy changes — sometimes the reason the role exists |
| Hiring freeze rumours | Recruiter, forums | Ask about approved headcount explicitly (Concept 4) |

### A quick stability score

For D1, rate each 0–2 and sum (max 10): **revenue trend** (shrinking 0, flat 1, growing 2), **funding/profitability** (under 12 months' runway 0, unclear 1, profitable or well-funded 2), **org churn** (reorganised twice in a year 0, once 1, stable 2), **team centrality** (experimental 0, adjacent 1, core 2), **leadership stability** (turnover plus layoffs 0, one of them 1, neither 2). Below 5: proceed only if you have specific reasons. This is crude — it's a forcing function to look, not a model.

**Interview-grade sentence** *(asked, in some form, by thoughtful hiring managers: "what made you look at us?")*: *"I looked at where the company is investing — the segment growth, what leadership says the priorities are, and where this team sits relative to them — because I want to join work that's funded for the next few years, and from what I could see this team is close to the centre of that."*

---

## Concept 8 — Engineering culture: what you can measure from outside

"Culture" sounds unresearchable. It isn't, if you replace the word with **specific, observable practices**. Research on software delivery gives you the vocabulary.

### Westrum's typology — information flow

Sociologist Ron Westrum classified organisational cultures by **how they process information**. DORA's research programme adopted his typology and found that a high-trust, generative culture predicts software delivery and organisational performance.

| | **Pathological** | **Bureaucratic** | **Generative** |
|---|---|---|---|
| Orientation | Power | Rules | Performance |
| Cooperation | Low | Modest | High |
| Bad news (messengers) | Punished | Neglected | Trained (encouraged) |
| Responsibilities | Shirked | Narrow | Shared risks |
| Cross-team bridging | Discouraged | Tolerated | Encouraged |
| Failure leads to | Scapegoating | Justice (find who) | Inquiry (find why) |
| New ideas | Crushed | Lead to problems | Implemented |

DORA measures it with six survey statements about the team: information is actively sought; messengers aren't punished; responsibilities are shared; cross-functional collaboration is encouraged; failures are treated as chances to improve the system; new ideas are welcomed. **Turn each into a question you can ask an engineer** (Concept 20 and Appendix C):

| Westrum dimension | Interview question that reveals it |
|---|---|
| Information sought | *"When you start a piece of work, how do you find out what other teams depend on?"* |
| Messengers | *"Tell me about the last time someone raised a problem late in a project — what happened next?"* |
| Shared responsibility | *"Who's on call for the services this team builds?"* |
| Bridging | *"How do you get something done that needs another team's change?"* |
| Failure → inquiry | *"What happened after the last significant incident? Was there a written review, and what changed?"* |
| Novelty | *"What's something the team changed in how it works in the last six months, and who proposed it?"* |

Ask for **the last time** something happened, not how it generally works. People describe general practice aspirationally and specific events accurately.

### DORA metrics as conversation, not interrogation

The four classic DORA delivery measures — **deployment frequency, lead time for changes, change failure rate, time to restore service** — are useful anchors because good teams know their numbers roughly and are happy to discuss them. *"Roughly how often does this service deploy to production, and how long does a typical change take from merge to production?"* An answer like "every day, about an hour" versus "monthly release train, two weeks of regression" tells you more than a page of culture statements.

### Conway's law — reading architecture from the org chart

Melvin Conway's 1968 observation — organisations design systems that mirror their communication structure — gives you predictive power in both directions:

- **From org to system:** if the company has separate frontend, backend and database teams, expect handoffs and layered architectures; if teams own vertical slices, expect services aligned to business capabilities.
- **From system to org:** if the job description lists twelve microservices for a team of four, expect a lot of operational load per engineer; if a "platform team" owns everything shared, expect queues for platform changes.

### Team types (Team Topologies)

Skelton and Pais's four team types are a quick way to classify the team you'd join, and each implies a different job:

| Team type | Purpose | What the senior/architect job is like |
|---|---|---|
| **Stream-aligned** | Delivers a product or value stream end to end | Product-facing; speed and quality of delivery; most common |
| **Platform** | Provides internal services that reduce other teams' cognitive load | Internal customers; APIs, paved roads, reliability; adoption is the success metric |
| **Enabling** | Helps other teams adopt capabilities, then steps back | Coaching, prototypes, standards; influence without authority |
| **Complicated-subsystem** | Owns a part needing deep specialist knowledge | Depth (search, payments engine, ML model serving); fewer stakeholders |

Ask the hiring manager: *"Who are the team's customers — end users or other teams?"*

### Decision-making — the architect's culture question

For architect and staff roles, the most important cultural fact is **how technical decisions are made and recorded**:

| Signal | Healthy | Warning |
|---|---|---|
| Design documents / RFCs | Written for significant changes; reviewed asynchronously | "We discuss it in a meeting" with no record |
| ADRs | Exist and are referred to | Nobody knows why the current architecture is the way it is |
| Architecture review | Advisory, fast, with clear criteria | A gate that takes weeks and decides by seniority |
| Disagreement | Resolved by evidence; "disagree and commit" | Resolved by escalation or by whoever is loudest |
| Who decides | Clear owner per decision | Unclear — or always the same senior person |

Questions: *"How was the last significant technical decision made? Is there a written record I could read on day one?"* *"What happens when two senior engineers disagree?"*

### Incidents and on-call

Incident culture is where Westrum's "failure → inquiry" becomes concrete. Ask: rotation length and frequency; pages per rotation; whether on-call is paid or compensated with time off (varies by country and company); whether post-incident reviews are blameless and **produce changes**; whether developers own their services in production.

**Interview-grade sentence:** *"I research culture through specific practices rather than adjectives — how bad news travels, what happened after the last incident, how the last big technical decision was made and recorded, who's on call — using Westrum's typology and the DORA measures as a checklist, and I ask for the last time something happened rather than how it's supposed to work."*

---

## Concept 9 — Stack and engineering maturity: the .NET/Azure lens

The stack matters less than people think for *whether* to join — good engineers learn stacks — but it matters a lot for **D2** (which deep dives and stories to prepare) and for **D3** (what kind of work fills your days). For a .NET/Azure engineer, specific signals carry specific meaning.

### Where to find stack signals

| Source | What it reveals | Reliability |
|---|---|---|
| Job descriptions (current *and* past) for this and adjacent teams | Versions, frameworks, cloud services, tooling | B for "what they use"; D for "what you'll mostly do" |
| Engineering blog, conference talks, meetup talks | Architecture, scale, recent migrations | B for existence; C for how widespread |
| Public GitHub organisation | Languages, libraries, CI setup, code quality, activity | A for open-source code; says little about internal code |
| Website technology fingerprinting (e.g. BuiltWith, Wappalyzer) | Front-end stack, CDN, analytics, sometimes hosting | B for the public site only |
| Stack-sharing sites | Self-declared tools | C–D; often stale |
| Engineers' public profiles | Technologies they list for the team | C; aggregate several |

### .NET version signals — and the November 2026 cliff

| Signal | Likely meaning | Questions it raises |
|---|---|---|
| **.NET Framework 4.x** (WCF, Web Forms, ASP.NET MVC 5, IIS) | Brownfield; long-lived line-of-business systems; possibly Windows-only hosting | Is modernisation funded? Strangler fig (Module 32) or "keep the lights on"? .NET Framework 4.8.x support follows the Windows lifecycle — there's no hard cliff, which can remove urgency |
| **.NET 8 or 9** today | Modern but **reaching end of support on 10 November 2026** | Is the move to .NET 10 done, planned or forgotten? A team with no answer has a security-patching problem a month from now |
| **.NET 10** | Current LTS (to November 2028); healthy upgrade discipline | What drove the upgrade — features, policy, or the cliff? |
| **Talks of .NET 11** | Early adopters; STS release from 10 November 2026 | Is that deliberate (a specific capability) or novelty-seeking? Default for stability is LTS |
| **Azure Functions in-process model** | Its support also ends **10 November 2026**; migration to the isolated worker model is required | Has the team migrated? Isolated-worker migration touches bindings, middleware and DI |
| **Mixed versions across services** | Normal in larger estates | Is there an upgrade policy and an owner (often a platform team)? |

This is unusually good material right now: it is a **real, dated, verifiable** engineering problem every .NET team faces, and asking about it in the loop shows you track the platform without needing to perform.

### Azure and hosting signals

| Signal | What it suggests |
|---|---|
| App Service + Azure SQL + Service Bus | Conventional, pragmatic PaaS; strong fit for Modules 26–27 patterns |
| AKS everywhere | Platform team likely; Kubernetes operational depth expected; ask who owns the cluster |
| Container Apps | Recent architecture choices; serverless containers; Dapr or KEDA possibly |
| Functions-heavy | Event-driven; ask about the isolated model migration and cold-start strategies |
| Cosmos DB | Ask about partition-key design and RU cost management (Module 27) — a great deep-dive topic |
| On-premises + Azure (hybrid) | Integration, networking, identity federation; enterprise context |
| Multi-cloud | Usually historical (acquisitions) or regulatory; ask which is primary |

### Engineering maturity signals

| Area | Mature | Immature |
|---|---|---|
| Delivery | CI on every PR; automated deploys; feature flags; trunk-based or short-lived branches | Manual releases; long-lived branches; change board for every deploy |
| Infrastructure | Infrastructure as code (Bicep, Terraform); reproducible environments | Portal clicking; snowflake environments |
| Observability | OpenTelemetry or equivalent; SLOs; dashboards owned by teams (Module 28) | Logs searched by hand during incidents |
| Testing | Fast unit and integration tests; contract tests between services | Manual QA phase; flaky end-to-end suite |
| Security | Managed identities; Key Vault; dependency scanning; threat modelling for new systems (Module 29) | Secrets in config files; annual pen test only |
| Dependencies | Regular upgrades; owner for shared libraries | "We can't upgrade X because Y" |

**Immaturity is not a red flag on its own.** For a senior engineer or architect, an immature but *improving* organisation that wants your help can be the best job on the market — it's where the impact is. The red flag is immaturity **plus** no funding or mandate to improve it. The question that separates them: *"What's the plan for improving X, and is anyone's time allocated to it?"*

### Certifications in job descriptions

Microsoft certifications appear often in .NET/Azure job descriptions, especially at consultancies and partners, where they can matter commercially. As of now: **AZ-305** (Azure Solutions Architect Expert, with **AZ-104** as prerequisite) remains current; **AZ-204** retired on 31 July 2026 in favour of **AI-200**. Two reading rules: a stale certification code in a JD means the description wasn't recently revised — check other details for staleness too; and "required" certifications at a consultancy are often a client- or partner-programme requirement, so ask whether you'd be expected to obtain them after joining (and whether the company pays).

**Interview-grade sentence** *(for "what do you know about our stack?" or as your own question)*: *"From the job description and the team's talks it looks like you're on .NET 8 with Functions and Service Bus — so I'd be curious how the move to .NET 10 and the isolated worker model is going, since both the .NET 8 runtime and the in-process model go out of support on November 10; that's usually a good window into how the team handles platform upgrades in general."*

---

## Concept 10 — Work model, location and employment terms

These are often the facts that **end** an opportunity at D1 — and the ones candidates research last, after investing a loop. Check them first.

### Remote, hybrid, on-site — and the reality behind the label

Labels drift. A "hybrid" role may mean a fixed three days per week, a "flexible" policy that's being tightened, or remote with quarterly travel. Policies at large companies have moved toward more on-site time in the last two years — Microsoft's three-day expectation, phased in from February 2026, is a current example — and team-level practice can differ from company policy in both directions.

Questions that cut through:
- *"Is this role open to candidates in [country]? Which legal entity would employ me?"*
- *"What's the on-site expectation for this team specifically, and has it changed in the last year?"*
- *"Where are the other team members, and what are the core overlap hours?"*
- *"How often would I be expected to travel, and is that funded?"*

### Hiring entity and contract type

"Remote" can mean several legally different arrangements:

| Arrangement | What it is | Things to check |
|---|---|---|
| **Local employee** of the company's entity in your country | Ordinary employment under local law | Notice periods, benefits, local equity plan eligibility |
| **Employee via an Employer of Record (EOR)** | A third party employs you on the company's behalf | Equity eligibility (sometimes excluded or different); benefits; how promotion and performance work for EOR staff |
| **Contractor** (B2B, through your own business entity) | You invoice; the client isn't your employer | Rates vs. employee comp (contractors carry their own costs and gaps); local rules on when a contractor relationship is really employment; termination terms; IP clauses; whether you can be promoted at all |
| **Employee of the foreign entity** | Rare without local presence | Tax and social-security implications; usually not offered |

Employment, tax and contractor-classification rules vary a lot by country and change; this is an area to **verify with a local accountant or employment lawyer** rather than generalise from forums. For research purposes, the point is to **find out which arrangement is on the table before the loop**, because it changes the comparable compensation (an employee salary and a contractor rate are not comparable numbers) and sometimes the role itself (contractors are often excluded from on-call, leadership or equity).

### Time zones

For architects, time-zone overlap is not a comfort issue — it is a job-design issue. Architecture work is mostly synchronous discussion with stakeholders. A role whose stakeholders are eight or nine hours away will have early mornings or late evenings as a structural feature. Ask where the **decision-makers** are, not just the team.

### Your information rights on pay

Depending on jurisdiction, you may be entitled to see or request the pay range:

- **US**: about 16 states plus DC require ranges in postings or on request; many laws cover remote roles that could be performed in the state.
- **EU**: the Pay Transparency Directive gives candidates the right to receive pay information (initial level or range) before the interview and bans employers from asking about pay history — but only where it has been transposed; as of late summer 2026, few member states had fully done so.
- **Elsewhere**: no right, but **asking for the range at the recruiter call is normal** in tech hiring and the answer itself is informative (Concept 16).

### Employment-terms checklist (before D3)

Probation period · notice period (both directions) · non-compete and non-solicitation scope and duration · IP assignment (does it reach side projects? — important if you build your own products) · moonlighting/outside-activities policy · equity plan eligibility for your location · on-call compensation · relocation obligations or clawbacks · sign-on bonus clawback terms.

**Interview-grade sentence** *(recruiter call, early)*: *"Before we go further, I want to make sure the basics fit — is the role open to someone based in my country, which entity would employ me, and what's the on-site and time-zone expectation for this team in practice? I'd rather know now than after we've both invested in a loop."*

---
# Part C — Researching the role and the loop

## Concept 11 — Decoding the job description

A job description is a **negotiated document**: written by a recruiter from a hiring manager's notes, often adapted from an older posting, sometimes padded to filter applicants. Read it the way you'd read a requirements document from a stakeholder (Module 4): separate the real constraints from the wishes, find what's missing, and ask about the ambiguities.

### The four layers

| Layer | Where it appears | How to read it |
|---|---|---|
| **Hard requirements** | "Required", "must have", years of experience, specific domains | Usually 3–5 real ones are hidden among 10 listed. The real ones repeat across the hiring manager's other postings and in the summary paragraph |
| **Wish list** | "Nice to have", long technology lists | Signals the team's stack and the shape of an ideal candidate; rarely all required |
| **Level signals** | Scope verbs, years, "lead", "own", "influence", "set direction" | Tell you the level more reliably than the title |
| **Context signals** | Team mission, "fast-paced", "greenfield", "modernise", "regulated" | Tell you the work and the stage |

### Level signals in the verbs

| Verbs and phrases | Usually indicates |
|---|---|
| "Implement", "contribute to", "work with the team" | Mid-level |
| "Design and deliver features end to end", "own services", "mentor" | Senior |
| "Set technical direction for a group of teams", "drive cross-team initiatives", "influence roadmap" | Staff / principal |
| "Define architecture standards", "govern", "evaluate vendors", "work with business stakeholders on target state" | Solution / enterprise architect |
| "Hands-on" + "set direction" | Player-coach staff role; ask how the time splits |

A **title–verb mismatch** is the most useful thing to find. "Senior Engineer" with staff verbs means either a generous scope you can grow into or a down-levelled role (the company wants staff output at senior pay). "Architect" with mid-level verbs may be a title used for an implementation role. Either way, it's a question for the recruiter.

### The four staff archetypes

For staff-plus and architect roles, Will Larson's **staff archetypes** describe four common shapes, and job descriptions usually lean toward one:

| Archetype | What they do | JD phrases |
|---|---|---|
| **Tech Lead** | Guides one team's approach and execution | "Lead the technical direction of the X team", "partner with the EM" |
| **Architect** | Owns direction and quality across a critical area | "Own the architecture for…", "set standards", "cross-team design" |
| **Solver** | Goes deep on hard problems wherever they are | "Tackle our hardest technical challenges", "investigate", "unblock" |
| **Right Hand** | Extends an executive's reach across the organisation | "Partner with the VP/CTO", "operate across the org" |

Knowing the archetype tells you which stories to lead with (Concept 17) and which questions to ask.

### Context signals and what they cost

| Phrase | Often means | Ask |
|---|---|---|
| "Fast-paced", "dynamic" | Frequent priority changes; possibly understaffing | *"How often did priorities change last quarter?"* |
| "Greenfield" | New system; also a new team, an unproven product, possibly unclear ownership | *"Who are the first users and when?"* |
| "Modernise", "transform", "migrate" | Brownfield (Module 32); the legacy is the job | *"What's the current state, the target, and the funded timeline?"* |
| "Wear many hats" | Small team; on-call, support, possibly DevOps | *"What did the last person in this role spend their time on?"* |
| "Regulated", "compliance", "audit" | Change control, documentation, possibly slower delivery | *"How does a change get to production here?"* |
| "Strong communication with business stakeholders" | Architect in an enterprise; meetings are the job | *"What's the split between design work and stakeholder work?"* |

### Staleness signals

A JD that hasn't been revised in a while tends to show it: retired certifications (AZ-204 after July 2026), end-of-support frameworks listed as the target stack, product names that have changed, a team size that doesn't match LinkedIn. Staleness doesn't make the role bad — but it means the **real** requirements are in someone's head, and the recruiter call matters more.

### Apply when you meet the core, not the list

A short practical rule: if you meet the **real hard requirements** and most of the core stack, apply; the long tail of a wish list is rarely binding. Then use the gap list to prepare a short, honest *"I haven't used X in production; here's what I've done that's closest, and how I'd ramp up"* answer.

**Interview-grade sentence:** *"I read a job description like a stakeholder's requirements: I separate the few real constraints from the wish list, read the level from the verbs rather than the title, look for the archetype and the context signals — greenfield, modernisation, regulated — and take every mismatch or stale detail to the recruiter as a question."*

---

## Concept 12 — Level research and calibration

Level is the most consequential and least researched variable in a senior job search. It sets compensation, scope, expectations at the first performance review and the pace of your next promotion. And titles are unreliable across companies.

### Why titles mislead

- **Different ladders.** Microsoft uses numbered levels (59 upward); Google, Meta and Amazon use L- or E-numbers with different starting points; most European companies use titles with their own definitions. "Senior" at one company maps to "mid" at another and "staff" at a third.
- **Sources disagree even about one company.** Public descriptions of Microsoft's ladder agree that **63** is a senior software engineer level, but disagree on where "Principal" starts — some put it at **64**, while levels.fyi's breakdown lists **63–64 as Senior and 65–67 as Principal**. When public sources disagree about the *naming* of levels inside one well-documented company, the lesson is: **ask the recruiter which level number the role is, and what that level's expectations document says.**
- **Bands span levels.** Posted ranges in US listings often cover more than one level, so a range alone doesn't identify the level.

### Cross-company mapping

Use level-comparison data (levels.fyi's leveling comparisons are the standard public reference) to translate. As orientation only — mappings vary by source and change:

| Rough tier | Microsoft (numbered) | Typical meaning |
|---|---|---|
| Mid | 61–62 | Independent on well-scoped work |
| Senior | 63–64 | Owns features or services end to end; mentors |
| Principal (Microsoft's term) / Staff-equivalent elsewhere | 65–67 | Cross-team scope; sets direction for an area |
| Partner and above | 68+ | Organisation-wide scope |

For orientation on pay at those levels, levels.fyi's self-reported US medians for Microsoft software engineers (as of mid-2026) were roughly **$251k at 63, $275k at 64 and $318k at 65** in total compensation. Pay in other countries is substantially lower and varies by location — use location-filtered data for your market.

### Down-levelling — the default risk for experienced hires

Companies often hire experienced external candidates **one level below** their current title, because the interview loop gives limited evidence of scope and internal leveling bars feel safer that way. You can't eliminate this, but research changes the odds:

1. **Know the target level's expectations before the loop.** Many companies publish engineering ladders or competency frameworks; for others, the recruiter can describe them. Map your stories to **that** level's scope verbs (Concept 17).
2. **Ask early which level(s) the loop is calibrated for.** *"Is this loop calibrated for one level, or will the panel decide between two?"*
3. **Present evidence at the right altitude.** For staff/principal, stories must show cross-team scope, direction-setting and organisational impact — not just strong execution (Modules 34–35).
4. **Ask after the loop, before the offer**: *"What level is the panel recommending, and what evidence would have supported the level above?"* — and be prepared with additional evidence (a reference, a design document, a follow-up conversation) if the answer is close.

### Is the level right for you?

The highest offer isn't always the right one. A level above your real current scope can produce a hard first review; a level below can cost years of compensation and scope. The decision question is: *"Can I meet this level's expectations within the first review cycle, and is the next level reachable from here in a reasonable time?"* Ask the hiring manager what someone at this level did last year that earned a strong rating.

**Interview-grade sentence** *(at the recruiter call or with the hiring manager)*: *"Titles don't translate cleanly between companies, so I'd like to understand the level itself — which level number or band this role is, what the expectations for it say, and whether the loop is calibrated for one level or deciding between two — so I can give you evidence at the right scope."*

---

## Concept 13 — The team and the manager

Most of what makes a job good or bad happens at the team level: the manager, the problems, the peers, the on-call, the pace. Company research can't reach it; you have to ask people.

### Why is the role open?

This is the single most informative question, and it has a small number of answers:

| Answer | What it implies | Follow-up |
|---|---|---|
| **Growth** — new headcount, expanding scope | Funded mandate; usually good | *"What's the scope growing into, and what's funded?"* |
| **Backfill** — someone left | Normal; but why did they leave? | *"What did the previous person work on, and why did they move?"* |
| **New team** | Exciting; also unproven | *"Who sponsors it, and what's the 6-month goal?"* |
| **Replacing a contractor or vendor** | Insourcing; possibly legacy knowledge gaps | *"Is there handover, or is the knowledge gone?"* |
| **Several backfills on one team** | A pattern worth understanding | *"How long have current team members been here?"* |
| **Vague** | Possibly unapproved headcount, or a reorganisation | Ask about approval and start date (Concept 4) |

### The team's real work

| Question | What it reveals |
|---|---|
| *"What are the two or three biggest problems the team will face in the next year?"* | The actual job; material for your "why this role" answer and for design-round framing |
| *"What would make you say, a year from now, that hiring for this role was a great decision?"* | Success criteria — and whether the manager has thought about them |
| *"What does the team own in production, and how many people are on call for it?"* | Operational load per engineer |
| *"How does work arrive — roadmap, tickets from other teams, incidents?"* | Planned vs. interrupt-driven work |
| *"What's the split between new development and maintenance?"* | Day-to-day reality |

### The manager

You can't research a manager's quality from public sources, but you can observe it in conversation and in how they run the hiring process. Ask:

- *"How do you run one-to-ones, and what do you use them for?"*
- *"Tell me about someone on your team who grew a lot recently — what did you do to help?"*
- *"How do you handle it when a senior engineer disagrees with a decision you've made?"*
- *"How is performance evaluated, and how would I know mid-cycle where I stand?"*

And observe: did they prepare for the conversation? Do they describe the team's problems candidly, including their own role in them? Do they ask about *your* goals? Can they name what they don't know?

### Talk to an engineer on the team

Before D3 it's normal to ask the recruiter for a conversation with a future peer (not an interviewer). Most companies agree. It's the highest-value research hour you'll spend. Ask the Westrum questions (Concept 8), the on-call questions, and two open ones: *"What surprised you after joining?"* and *"What would you change if you could?"*

**Interview-grade sentence:** *"The team matters more than the company, so I always ask why the role is open, what the team's biggest problems are for the next year and what success would look like, and before deciding I ask to speak with a future peer — the answers to 'what surprised you after joining?' are usually the most honest description of a job you'll get."*

---

## Concept 14 — Mapping the interview loop

The **loop map** is the main output of D2 research: a table of every stage, what it tests, how and by whom, under which rules. Without it you prepare for an imagined interview; with it you prepare for the real one, and Module 38's mock programme can be weighted to match.

### What to capture for each stage

```text
STAGE      FORMAT        LENGTH  INTERVIEWER        MEDIUM              AI MODE   TESTS                       PREP SOURCE
Recruiter  call          30      recruiter          video               —         motivation, logistics, range Concept 16
Screen     coding        60      engineer           Codility / CoderPad none      DS&A medium, C#              Module 36
Onsite 1   coding        45      engineer (other)   shared editor       none      DS&A + code quality          Module 36
Onsite 2   system design 60      senior/HM          Excalidraw / Canvas none      SD + compliance angle        Modules 3–13, 37
Onsite 3   domain        60      team senior        conversation+code   none      storage/networking/etc.      dossier → targeted study
Onsite 4   behavioral    45      HM or senior       conversation        —         ownership, growth, conflict  Modules 34–35
Matching   HM calls      30 ea   hiring managers    video               —         mutual fit                   Concept 20
Decision   —             —       HM / committee     —                   —         —                           ask timeline
```

### Two structural families

From interviewing.io's comparison of large tech companies, a useful distinction:

| | **Centralised** loop | **Team-dependent** loop |
|---|---|---|
| Examples (per that guide) | Google, Meta | Microsoft, Amazon, Apple, Netflix |
| Who interviews you | A pool of trained interviewers, mostly not your future team | Often your future team and adjacent teams |
| Question style | More standardised; question banks | Varies by team; customised to the team's domain |
| After the loop | **Team matching** — can take weeks or longer | Offer for the team (sometimes with matching within the org) |
| Parallel attempts | One loop at a time; cooldown after failure | Can often interview with several teams concurrently |
| Prep implication | General excellence; rubric-driven; predictable | **Research the team's domain**; expect variance; prepare 2–3 projects to discuss in depth |

The same guide characterises Microsoft's process as relatively unpredictable from candidate to candidate (team-defined rounds, interviewer training and question standardisation varying by team) and singles out the **domain-specific round** — customised to the team's technical area, sometimes including coding or language-specific questions, sometimes "walk me through a complex problem you solved". Interviewers it quotes also describe a strong emphasis on **compliance and auditability** in system design at Microsoft. These are B-grade claims (an aggregator of interviewer anecdotes) — exactly the kind to confirm with the recruiter.

### Questions that fill the map

Ask the recruiter (Concept 16), ideally getting the answer in writing:

1. *"Could you walk me through every stage after this call, with the format and length of each?"*
2. *"For each onsite round, what's the focus — and is there a domain-specific round? If so, what area?"*
3. *"Who are the interviewers — the team, or a pool?"*
4. *"What tools will we use for coding and for design — and may I choose the language? Is C# fine?"*
5. *"What's the AI policy for each round?"* (Concept 15)
6. *"Is any round in person?"*
7. *"Is there a take-home or a presentation? If so, how long, and how is it assessed?"*
8. *"How is the decision made — hiring manager, panel debrief, committee? And is there team matching afterwards?"*
9. *"What's the typical timeline from onsite to decision, and to offer?"*
10. *"If it doesn't work out, is there a waiting period before I could interview again, and does it apply company-wide or per team?"*

### Architect loops: map the stakeholders, too

Architect loops (Module 2) often include rounds with non-engineers — a product lead, an enterprise architect, a security lead, sometimes a business owner. For each, note **what that person cares about** (their likely evaluation criteria): product wants delivery and trade-offs in user terms; security wants threat models and controls; finance wants cost drivers; implementers want feasibility and ownership. Your per-round card (Concept 21) should carry that audience's vocabulary.

**Interview-grade sentence:** *"After the recruiter call I write a loop map — every stage, its format, length, interviewer, tools, AI rules and what it tests — and whether the loop is centralised or team-dependent, because a team-dependent loop with a domain round needs targeted study of that team's area, while a centralised loop rewards general, rubric-shaped preparation and adds a team-matching stage."*

---

## Concept 15 — AI policy and round rules

Module 38 covered *how to practise* for each AI mode. This concept is about **finding out which mode applies** — and getting it right, because the downside of guessing wrong is disqualification.

### The three modes

| Mode | Meaning | Where it's common now |
|---|---|---|
| **No AI** | Assistants, copilots and outside help not permitted | The default at most companies for live interviews — e.g. Microsoft's Candidate Code of Conduct (no AI in assessments and interviews unless instructed); Google's virtual-interview guidance (AI use leads to disqualification); Amazon and Anthropic for live interviews |
| **AI permitted** | Allowed; you won't be penalised for not using it | Some take-homes; some practical rounds |
| **AI expected** | The round is designed around an assistant | Meta's AI-enabled coding round; Canva's AI-assisted coding interview |

Policies distinguish **preparation** (widely encouraged — Microsoft and Anthropic say so explicitly) from **live interviews and assessments** (usually prohibited unless stated).

### The research rules

1. **Read the company's candidate guidance** — careers pages increasingly state the policy.
2. **Ask per round**, because one loop can mix modes (an AI-enabled coding round alongside a no-AI coding round).
3. **Get it in writing** — a line in the recruiter's email is enough.
4. **Ask about adjacent tools too**: IDE autocomplete, documentation lookup, compiler and test runner. "No AI" doesn't always mean "no IntelliSense", and the answer changes how you practise (Module 38's blank-editor drill).
5. **Ask about take-homes separately** — policies for take-homes are the least consistent, and some explicitly allow AI as long as you disclose it.
6. **When in doubt, don't.** If the policy is ambiguous and you can't get clarity, treat the round as no-AI.

A sentence for the recruiter: *"Could you confirm the AI and tooling rules for each round — whether assistants are not allowed, allowed or expected, and whether normal IDE features like autocomplete are fine — so I practise in exactly the right setup?"* It also signals integrity, which is being evaluated anyway.

### Other round rules worth confirming

| Rule | Why it matters |
|---|---|
| Programming language choice | Using C# is usually fine and often preferred at .NET shops; confirm for screens with fixed language lists |
| In-person vs remote | Whiteboard handwriting is a different skill (Module 38, Concept 21) |
| Take-home time limit and scope | Over-delivering on an unbounded take-home is a common trap; ask what "done" looks like |
| Presentation rounds | Audience, length, whether slides are expected, whether it's your past work or a set problem |
| Accommodations | If you need any, ask early; companies generally have a process |

**Interview-grade sentence:** *"I confirm the AI and tooling rules per round, in writing, before I start practising — whether assistants are prohibited, permitted or expected, and what counts, like IDE autocomplete — because loops now mix modes, the rules differ between companies, and guessing wrong is disqualifying."*

---

## Concept 16 — The recruiter call as a research instrument

The first recruiter call is usually framed as the company screening you. It is equally **your best research opportunity before the loop** — the recruiter knows the process, the level, the range and the timeline, and is motivated to help you succeed once you look like a fit. Treat it as a structured interview you're running in the second half.

### Structure (30 minutes)

```text
0–3    Rapport; confirm the role and team being discussed
3–15   Their questions: your background, motivation, logistics
        → a 90-second career summary pitched at the target level
        → a specific "why this company/role" (Concept 19)
15–27  Your questions — the research script below
27–30  Next steps; ask for the loop description in writing
```

### Your research script (prioritise; you won't get through everything)

**Fit and logistics (D1 blockers first)**
1. Location and entity: *"Is this role open to someone based in [country]? Which entity would employ me?"*
2. Work model: *"What are the on-site and overlap-hour expectations for this team?"*
3. Headcount: *"Is the headcount approved and open now? What's the target start date?"*

**Role and level**
4. *"Which level is this role, and is the loop calibrated for one level or deciding between two?"*
5. *"Why is the role open — growth, backfill, new team?"*
6. *"Who's the hiring manager, and what's the team's main focus this year?"*

**The loop (fills the loop map)**
7. Stages, formats, lengths, interviewers, tools, language choice.
8. Domain round: area and depth.
9. AI and tooling rules per round.
10. In-person rounds; take-homes; presentations.
11. Decision process, team matching, timeline, cooldown.
12. *"Is there any preparation material you'd recommend?"* (Many companies have official prep guides; recruiters will often share them.)

**Compensation**
13. *"What's the range for this level and location?"*

### Handling the compensation questions

Two different questions get asked, and they deserve different handling:

- **"What are your salary expectations?"** — Early, before you know the level and scope, naming a number anchors the negotiation (often against you) and can be used to screen you out. Widely recommended practice is to **defer politely and ask for their range**: *"I'd like to understand the level and full scope first. Could you share the range you've budgeted for this level and location? I'm sure we'll find a fit if the role is right."* Where pay-transparency laws apply, the range may already be in the posting or be disclosable on request.
- **"What are you earning now?"** — Pay-history questions are prohibited in a growing number of jurisdictions (many US states; the EU directive where transposed). Elsewhere, you can still decline: *"I'd prefer to focus on the value of this role rather than my current package — what range are you working with?"*

If pressed hard, you can give a **well-researched range anchored at the top of the market for the level** (Concept 23) rather than a single number — but do so only after you know the level.

### What not to share early

Where else you're interviewing and how far along (beyond "I'm in other processes and expect decisions in about N weeks", which helps with timing), your current compensation, and your lowest acceptable number. These reduce your options without improving the process.

### After the call

Within the hour: update the loop map and claims log (grade the recruiter's statements as A for process facts, C for culture claims), list the open questions, and send a short thank-you email that **restates the loop as you understood it** — *"To confirm: four onsite rounds — coding ×2, system design, and a domain round on storage — all without AI tools, C# fine…"* — which both creates a written record and invites corrections.

**Interview-grade sentence:** *"I treat the recruiter call as research as much as screening: I check the blockers first — location, work model, approved headcount — then the level and why the role is open, then the full loop including the AI rules, and I follow up in writing with my understanding of the process so there's a record and a chance to correct it."*

---
# Part D — Turning research into preparation

## Concept 17 — Translating values and rubric vocabulary to the six dimensions

Module 38 established that most loops score six universal dimensions under different names: **D1 framing and scoping, D2 technical depth (plus D2b stack precision), D3 trade-offs and judgement, D4 driving and communication, D5 collaboration and coachability, D6 level signal.** Company research supplies the *local vocabulary* and the *local weights*. The work in this concept is translation.

### Step 1 — Collect the company's vocabulary

Sources, in order of reliability: published values or principles pages; official interview-preparation guides; engineering ladders or competency frameworks (some companies publish them); hiring-process guides written with current interviewers; the recruiter's description of "what we look for".

Examples of how vocabularies differ:

| Company style | Vocabulary | Where it shows up |
|---|---|---|
| **Principle-driven** (e.g. Amazon's 16 Leadership Principles) | Customer Obsession, Ownership, Dive Deep, Are Right A Lot, Bias for Action, Earn Trust, Deliver Results… | Every round, including technical; behavioral questions are explicitly mapped to principles |
| **Mindset-driven** (e.g. Microsoft's emphasis on growth mindset, alongside respect, integrity and accountability in its candidate guidance) | Learning from failure, openness to ideas, collaboration across teams ("One Microsoft") | Behavioral round; also how you respond to hints and pushback |
| **Competency-framework** (many European scale-ups and enterprises) | "Technical excellence", "Impact", "Collaboration", "Ownership", "Communication" per level | Rubric rows, often published internally and described by recruiters |
| **Enterprise architecture** | Alignment to target state, governance, risk, stakeholder management, standards | Architect panels, often with business representatives |

### Step 2 — Build the translation table

Map each local term to the universal dimensions, and note what **evidence** satisfies it at your target level. Example for a principle-driven loop:

| Local term | Maps mostly to | Evidence that scores at senior/staff |
|---|---|---|
| Customer Obsession | D1, D6 | Started a design from a customer problem with a metric; changed a technical plan because of customer evidence |
| Ownership | D6 | Took responsibility beyond your remit; long-term over short-term; "that's not my job" never appears |
| Dive Deep | D2 | Found root cause by going into data or code personally; knew numbers without notes |
| Are Right, A Lot | D3 | A decision under uncertainty, the alternatives, how you checked it, and when you were wrong and changed your mind |
| Have Backbone; Disagree and Commit | D3, D5 | Disagreed with someone senior on evidence, then committed fully once decided |
| Earn Trust | D5 | Admitted a mistake publicly; gave credit; listened to a critic |
| Deliver Results | D6 | Delivered against a hard constraint, with a quantified outcome |
| Insist on the Highest Standards | D2, D6 | Raised a quality bar for a team (tests, reviews, SLOs) and kept it there |

For a growth-mindset culture, the translation emphasises **D5**: how you describe a failure and what changed; how you respond in the loop to a hint ("that's useful — so…"); evidence of learning a new area quickly. interviewing.io's guide describes the soft skills Microsoft behavioral rounds screen for as positivity, ownership and communication — which is the same message from a different source (triangulation).

### Step 3 — Re-weight the dimensions

Use what you learned about the loop map to adjust Module 38's priorities:

| Research finding | Re-weighting |
|---|---|
| Principles assessed in every round | D5/D6 evidence must appear even in technical rounds — prepare one-sentence "principle tags" for design decisions |
| Team-dependent loop with a domain round | D2/D2b deep in the team's area outweighs breadth |
| Interviewer anecdotes emphasise compliance and auditability | D3 failure-and-risk reasoning plus audit logging, data residency and retention in every design |
| Architect panel with business stakeholders | D4 (audience-appropriate communication) and D6 (cost, risk, migration) dominate |
| Startup with a small team | Breadth, pragmatism, speed (D3 "good enough", D6 ownership across the stack) |

### Step 4 — Retag the story bank

Module 35's story bank has 6–8 core stories. For each target company, add a column with **local tags** and a **lead order**:

```text
STORY                        UNIVERSAL TAGS     COMPANY TAGS                          LEAD FOR QUESTION TYPE
Payment retries outage       failure, D3, D5    Ownership, Dive Deep, Earn Trust      "tell me about a failure"
Strangler migration          ambiguity, D6      Think Big, Deliver Results            "biggest project", "long-term thinking"
Disagreed with principal     conflict, D3, D5   Have Backbone; Disagree and Commit    "disagreed with someone senior"
Mentored two engineers       mentoring, D5      Hire and Develop the Best             "helped someone grow"
...
```

Check **coverage**: every local principle that the loop is known to emphasise should have at least one strong story, and no single story should be your lead for more than two principles (interviewers in the same loop compare notes, and repetition looks thin).

**Interview-grade sentence:** *"I prepare by translating a company's own vocabulary — its principles or competencies — into the handful of things every loop actually scores, then re-weighting based on what the loop contains and retagging my stories so each emphasised principle has a strong, specific example in their language."*

---

## Concept 18 — Company-shaped mocks and problem banks

Research should change **what you practise**, not just what you know. Module 38's mock catalogue becomes a company-specific plan.

### 1. Re-weight the mock schedule

Apply Module 38's priority formula — `priority = rounds_in_loop × gap + 0.5` — using the **loop map's** round counts. A loop with two coding rounds, one design round, one domain round and one behavioral round gets a different schedule from a loop with one coding round and two architect rounds.

### 2. Build a company-shaped design problem

Generic design problems (URL shortener, rate limiter) are good for method. A **company-shaped** problem is better for the last two weeks: take the company's real product area and design a plausible subsystem with its likely constraints.

How to construct one:

1. **Pick a capability** the team plausibly owns (from the JD, blog, product docs).
2. **Write requirements** from what the product actually does: users, scale order of magnitude (from public figures where available), latency and consistency needs, compliance constraints.
3. **Add the company's known emphasis** — e.g. audit logging and data residency for an enterprise cloud service; idempotency and reconciliation for payments; multi-tenancy and noisy neighbours for SaaS.
4. **Prepare a probe ladder** (Module 38, Concept 7) where rung 6 is the company's own stack.

Examples:

| Company type | Company-shaped problem |
|---|---|
| Azure service team | "Design the control-plane API for provisioning a managed resource across regions — idempotent creation, long-running operations, quota enforcement, audit trail" |
| Fintech on .NET | "Design the ledger posting service for card transactions — double-entry, idempotency, reconciliation with the processor, EU data residency" |
| B2B SaaS | "Design per-tenant usage metering and billing — accuracy, late events, tenant isolation, cost per tenant" |
| Insurer modernising | "Strangle the policy-administration monolith's quoting module onto Azure — dual-run, data sync, cut-over, rollback" |

These are not predictions of the question — they're **rehearsal of the domain**, so whatever you're asked, the vocabulary and constraints are warm.

### 3. Prepare for the domain round

If the loop has a domain-specific round, research tells you the area. Prepare in three layers:

- **Fundamentals of the domain** (networking, storage, identity, data pipelines, front-end performance — whichever applies): the 20% of concepts that come up 80% of the time.
- **Two or three of your projects** told as technical deep dives: problem, constraints, design, what broke, numbers. Some domain rounds are exactly "walk me through something complex you built" — and the follow-ups go to rung 5–6.
- **The company's own public material** in that area: product documentation, architecture centre articles, published postmortems. For Azure roles, the Azure Architecture Center and the relevant product's docs are the obvious reading — not to recite, but to understand the vocabulary the team uses.

### 4. Language and stack mocks

If research says C# is welcome (common at .NET shops; interviewing.io's Microsoft guide relays interviewers recommending C#, Java or Python), do all coding mocks in C#, in the **actual tool** where known (Codility, CoderPad, shared editor) and with the **confirmed tooling rules** (Concept 15).

### 5. Behavioral mock in the local vocabulary

Have your mock interviewer ask questions phrased in the company's terms (*"Tell me about a time you had to disagree and commit"*), and score with Module 38's behavioral rubric plus a "local-term fit" check: did the story clearly demonstrate the principle asked about, in a way a trained interviewer could write down?

**Interview-grade sentence:** *"In the last couple of weeks before a loop I switch from generic practice to company-shaped practice: I weight mocks by the actual round counts, rehearse a design problem built from the team's real product area and constraints, prepare the domain round in their technical area with two or three of my own projects as deep dives, and run behavioral mocks in the company's own vocabulary."*

---

## Concept 19 — "Why us, why this role, why now"

Almost every loop asks some version of "why do you want to work here?" — the recruiter, the hiring manager, often the behavioral interviewer. A generic answer ("great culture, interesting problems, I love your product") scores low because it could be said about any company. A researched answer scores well because it's **specific, evidenced and two-sided** — it shows you understand what they need and what you bring.

### The structure (60–90 seconds)

```text
1. THEM     What you understand about where they are and where they're going        (specific, researched)
2. ROLE     What this role is for, in their terms                                   (from the JD + recruiter + HM)
3. FIT      Why your experience matches that need — one or two concrete proofs       (from your story bank)
4. YOU      What you want that this role offers — growth, problem type, scope        (honest; shows it's mutual)
5. (NOW)    Why this move makes sense at this point in your career                   (optional; for "why leave?")
```

### Example (senior .NET/Azure role on a platform team, details illustrative)

> *"From your engineering talks and the job description, the platform team is moving the service estate from App Service onto Container Apps and standardising observability on OpenTelemetry, while every team has to be off .NET 8 by November. That's a platform-adoption problem as much as a technical one — the hard part is getting thirty teams to move without stopping feature work. I've done that shape of work twice: I led a .NET Framework-to-.NET 8 migration across eleven services using a strangler approach, and I built the shared telemetry library that four teams adopted, mostly by making the paved road easier than the alternatives. What I'm looking for next is exactly this — a platform role with internal customers at larger scale, where adoption is the metric."*

What makes it work: **specific facts about them** (a reader can tell it was researched), **the role restated as a problem** (shows understanding), **two concrete proofs** (not adjectives), **a genuine personal reason** that's consistent with the role.

### Common failure modes

| Failure | Why it scores low | Fix |
|---|---|---|
| Flattery ("you're the best company") | Unfalsifiable; says nothing about you | Replace with a specific fact and why it matters to you |
| Only about you ("I want to grow") | Sounds like the company is a training programme | Lead with their need, then your fit, then your goal |
| Product-user enthusiasm only | Fine for consumer products, thin for engineering roles | Add the engineering problem behind the product |
| Reciting their website | Shows research, not understanding | Interpret: what does the fact imply for the role? |
| Negative about current employer | Reads as a pattern | "Why now" in terms of what you're moving toward |

### Variants you'll be asked

- **"Why are you leaving your current role?"** — toward, not away: *"I've done the migration I was hired for; the next problem I want is…"*
- **"Why this team over others here?"** (team-dependent loops) — the team's problems from the hiring-manager conversation.
- **"Why not management?"** (staff/architect) — what you've learned about where your leverage is, with an example.
- **"Where else are you interviewing?"** — honest at the level of category ("a couple of product companies on similar stacks"), without names or stages unless it helps timing.

**Interview-grade sentence** *(the frame itself, for coaching others)*: *"A strong 'why us' answer has four parts — what you understand about where they are, what this role is for in their terms, one or two concrete proofs that you've done that kind of work, and what you genuinely want that this role offers — and it should be specific enough that it couldn't be said about any other company."*

---

## Concept 20 — Your questions: designed to extract information

"What questions do you have for me?" ends almost every round. It's both **evaluated** (your questions show what you think about and at what altitude) and **your main research channel** for team-level facts no public source has. Design questions to do both.

### Properties of a good question

1. **Decision-relevant** — the answer feeds D2 or D3 (Concept 1).
2. **Specific and behavioural** — asks about **the last time** something happened, not general policy.
3. **Right for the interviewer** — asks what this person is best placed to answer.
4. **Shows altitude** — senior questions are about systems and teams; staff and architect questions are about decisions, direction and organisation.
5. **Not answerable from public sources** — asking what's on the careers page signals no research.

### Per interviewer type

| Interviewer | Best placed to answer | Example questions |
|---|---|---|
| **Recruiter** | Process, level, range, logistics | Concept 16 script |
| **Hiring manager** | The team's problems, success criteria, why the role is open, how they manage | *"What are the two biggest problems this team needs solved in the next year?"* · *"What would make this hire a clear success at twelve months?"* · *"How did the team make its last significant technical decision?"* · *"What do you wish the team did better?"* |
| **Peer engineer** | Day-to-day reality, tooling, on-call, culture | *"What did you work on last week?"* · *"What happened after the last real incident?"* · *"How long does a change take from merge to production?"* · *"What surprised you after joining?"* |
| **Senior/staff engineer** | Architecture, technical direction, debt | *"What's the most expensive technical decision the team is living with, and would you make it again?"* · *"Where's the architecture heading in the next year, and what's blocking it?"* · *"How are cross-team design decisions recorded?"* |
| **Architect / principal** | Governance, standards, decision rights | *"How much of the target architecture is decided centrally vs by teams?"* · *"How do you handle a team that wants to diverge from a standard?"* · *"What does the architecture review process actually change, typically?"* |
| **Product / business stakeholder** (architect loops) | Priorities, how engineering is perceived, pressure points | *"Where does engineering most often surprise you, positively or negatively?"* · *"What trade-off between speed and robustness did you face recently, and how was it decided?"* |
| **Skip-level / director** | Strategy, org stability, investment | *"How does this team's work connect to the org's priorities this year?"* · *"What would make you increase or decrease investment in this area?"* |

### Probing questions — the follow-up matters more

The first answer is often polished. A follow-up gets to reality:

- After *"we have blameless postmortems"* → *"What changed as a result of the last one?"*
- After *"we deploy continuously"* → *"What was the last deploy that went wrong, and how was it rolled back?"*
- After *"great work-life balance"* → *"When did you last work late, and why?"*
- After *"we're moving to microservices"* → *"What's the boundary of the first service you extracted, and how did data ownership work out?"*

### Questions that reveal red flags — gently

You can ask hard things politely: *"What's one thing about this team you'd want a new person to know before joining?"* · *"What's the main reason people have left the team?"* · *"If you could change one thing about how the team works, what would it be?"* Evasive, rehearsed or contradictory answers across interviewers are data (Concept 22).

### How many and when

Prepare **five or six per interviewer type**; you'll ask **two or three**. Don't ask the same question to everyone — except one deliberate **triangulation question** (e.g. "what happened after the last incident?") asked of two different people, to compare answers. Write the answers down immediately after each round.

**Interview-grade sentence** *(when asked, as some interviewers do, "what would you want to know before accepting?")*: *"I'd want to understand the team's biggest problems for the next year, how decisions get made and recorded, and what happens after things go wrong — and I ask about the last time something happened rather than the policy, because specific events are much more informative than descriptions of how things are supposed to work."*

---

## Concept 21 — The per-round card

All this research must be usable at minute zero of each round, under pressure. Module 38 introduced the one-page card for the day; here it becomes **one card per round**, built from the dossier.

### Card template

```text
ROUND: Onsite 3 — Domain (storage)        INTERVIEWER: senior engineer, team X (LinkedIn: 7 yrs, storage)
RULES: 60 min · conversation + some code · C# · NO AI, IDE autocomplete OK (recruiter email 10-14)
LIKELY: storage fundamentals; "walk me through a complex system you built"; language questions on C#
LOCAL EMPHASIS: compliance/auditability (B2 — interviewing.io anecdote); growth mindset in follow-ups
MY OPENING MOVES: restate → scope → ask about scale and durability needs → check what they'd like to focus on
DEEP DIVES READY: (1) event store on Cosmos (partitioning, RU, change feed) (2) ledger reconciliation (idempotency)
PROJECT TO TELL: blob-tier migration — 40 TB, cost −38%, lifecycle policies, the rehydration incident
NUMBERS COLD: Cosmos 20 GB logical partition limit; blob tiers & rehydration hours; my project's figures
STORIES (if behavioral bits): rehydration incident (ownership, learning) · disagreed on consistency level
QUESTIONS (2–3): last significant incident & what changed · how are storage-format decisions recorded ·
                 [triangulation] how long from merge to production
RECOVERY: blank on a number → assume, flag, continue · mistake → correct aloud
```

### Rules

- **One page, large font.** Read it in the ten minutes before the round, then put it away (no notes during the round unless explicitly allowed).
- **Every line traces to the dossier** — the rules line to the recruiter's email, the local emphasis to graded claims.
- **Update between rounds** (Concept 22): what you learned in round 2 — a confirmed focus, a hint about the team's pain — goes into round 3's card.

**Interview-grade sentence:** *"Before each round I read a one-page card built from my research — the confirmed rules, what the round is likely to probe, the company's emphasis, my opening moves, the deep dives and stories I'd lead with, and the two or three questions I want to ask this particular interviewer — and I update the next card with whatever I learned in the previous round."*

---
# Part E — Decision time

## Concept 22 — Reading the loop as evidence

The interview process is the only part of the company you get to observe directly, end to end, before joining. Companies run hiring the way they run everything else — with the same planning discipline, the same honesty, the same respect (or not) for people's time. Treat the loop as a sample.

### What to observe

| Area | Green signals | Red signals |
|---|---|---|
| **Organisation** | Clear schedule in advance; interviewers know which round they're running; the loop matches what the recruiter described | Rounds changed without notice; interviewers unsure what to assess; long silences between stages |
| **Honesty** | Straight answers to hard questions ("yes, we had layoffs; this team wasn't affected; here's why the role is open") | Evasion on the role's history, the team's problems, on-call, the level; answers that contradict each other across interviewers |
| **Interviewer preparation** | Read your CV; ask follow-ups that build on your answers; probe for depth | Reading questions off a screen; no follow-ups; checking phones |
| **Respect** | Start on time; leave time for your questions; explain next steps | Late; no time for questions; dismissive of answers |
| **Problem fit** | Questions resemble the actual work (a domain round about what the team does) | Questions disconnected from the role with no explanation (sometimes fine at centralised loops — calibrate to the loop type) |
| **How they treat disagreement** | Pushback in the design round is collaborative; they engage with your reasoning | Pushback is a test of submission; interviewer insists on "their" answer |
| **Candour about weaknesses** | Interviewers name real problems ("our test suite is slow; we're fixing it") | Everything is "great"; no one can name a challenge |

Calibrate for loop type: in a **centralised** loop, interviewers are not your future team and know little about it — a weak answer about the team means little. In a **team-dependent** loop, the interviewers *are* the team, and what you observe is close to what you'd get.

### Triangulate across interviewers

This is where Concept 20's deliberate repeated question pays off. If the hiring manager says "incidents lead to blameless reviews and real fixes" and a peer engineer, asked independently, describes the last incident review and the change that came out of it — that's a confirmed (A1–B1) claim. If the peer says "we don't really do reviews; we just fix it and move on", you've found the gap between stated and actual practice.

### Post-round notes (immediately after each round)

```text
ROUND 2 — System design — interviewer: senior engineer (team-adjacent)
Problem: ________   Twist: ________   Where they pushed: ________
What I learned about the team/company: ________   (grade: C2)
Answers to my questions: ________ (incident question → "postmortem doc, two action items, both done")
Signals: green ________ / red ________
Adjust next card: ________
Self-score (memory; flag as memory): ________   (Module 38 — don't post-mortem further during the loop)
```

Write the research notes and move on. Analysing your *performance* mid-loop hurts the next round (Module 38, Concept 24); recording the *information* you gathered doesn't.

**Interview-grade sentence** *(for "what did you think of our process?" — sometimes asked at the end)*: *"Honestly, I treat a loop as the best sample of how a company works that I'll get before joining — so I noticed that it was well organised, that the interviewers built on my answers, and that two people independently described the same incident review in the same way, which told me more about the engineering culture than anything I'd read."*

---

## Concept 23 — Evaluating an offer

An offer is a **multi-component, multi-year, partly uncertain cash-flow** — and should be analysed like one, not compared by its headline number.

### Components

| Component | What to find out | Common traps |
|---|---|---|
| **Base salary** | Amount; review cycle; typical raise; currency | Comparing gross across countries with very different tax and social-security systems |
| **Bonus** | Target %; what drives payout (company, team, individual); historical payout range | Treating target as guaranteed; some bonuses have been paid well below target in weak years |
| **Sign-on bonus** | Amount; when paid; clawback if you leave within N months | It's year-one only — it doesn't recur |
| **Equity — public company RSUs** | Grant value or units; **vesting schedule** (even, back-loaded, cliff); refresh practice | Vesting schedules vary a lot — some are back-loaded, so years 1–2 are much lower than the "annualised" figure; refreshers are discretionary |
| **Equity — private company options/RSUs** | Number; strike price; latest valuation and **preference stack**; total shares outstanding (your %); exercise window after leaving; expected liquidity | Valuing paper equity at the latest round price; ignoring dilution and liquidation preferences; short post-departure exercise windows |
| **Benefits** | Pension contributions; health insurance; leave; parental leave; learning budget; equipment | Often worth more than they look in some countries, nothing in others — convert to money where possible |
| **Level** | Exact level and its expectations | Accepting a lower level for a higher number costs more over time (Concept 12) |
| **Terms** | Notice, non-compete, IP, probation, on-call pay, remote terms | Discovering a broad IP clause after you've signed |

### Value equity by scenarios, not by the grant price

Equity value is uncertain; model it explicitly:

- **Public RSUs:** apply bear/base/bull scenarios to the share price over the vesting period. A 30% drop in the share price cuts the equity line by 30% for vests after the drop.
- **Private equity:** include a meaningful-probability scenario where it's worth **nothing** (no liquidity event, or preferences absorb the exit), plus scenarios for modest and good outcomes. Unless you have specific reasons, treat private equity as a lottery ticket with a positive expected value but a **large chance of zero**, and make sure the cash component is acceptable on its own.

Appendix F contains a small C# tool that models offers year by year with vesting schedules, refreshers, bonus payout assumptions, price scenarios and FX, and reports **expected** and **bear-case** totals — the two numbers you actually need.

### Compare like with like

| Comparison issue | Adjustment |
|---|---|
| Different countries | Compare **net** (after tax and mandatory contributions) and adjust for cost of living, or compare within one market only |
| Employee vs contractor | A contractor rate must cover unpaid leave, sickness, pension, equipment, accounting and gaps between contracts — the gross numbers aren't comparable |
| Different currencies | Convert at a stated rate and note the currency risk — a salary in a foreign currency moves with the exchange rate |
| Different levels | Compare trajectory: where each puts you in 2–3 years |
| Front-loaded vs back-loaded equity | Compare **year-by-year**, not just four-year totals — leaving in year 2 is a real possibility |

### Market research for negotiation

You negotiate with information. Gather, for the **level and location**: aggregated compensation data with enough data points; the recruiter's disclosed range; any competing offers (the strongest information you can have). The standard advice from widely read negotiation guides (Patrick McKenzie's and Haseeb Qureshi's are the classics) converges on a few principles: be enthusiastic about the role, avoid naming the first number early, negotiate on the whole package (level, equity and sign-on often have more room than base), use competing offers honestly, and get the final offer in writing.

**Interview-grade sentence** *(for the offer conversation with a recruiter)*: *"I'm genuinely excited about the role. To evaluate the offer properly I'd like to see it year by year — base, target bonus and how it's typically paid out, the equity grant with its exact vesting schedule, how refreshers usually work at this level, and any sign-on terms — so I'm comparing like with like rather than headline numbers."*

---

## Concept 24 — Making the decision

You have a dossier, a loop's worth of observations and one or more offers. The remaining risk is a **bad decision process**: deciding on the last conversation you had, the biggest number, or how much you liked the hiring manager.

### 1. Set criteria and weights *before* the offers arrive

If you define what matters after you see the numbers, you'll unconsciously weight whatever favours the option you already prefer. Write the criteria at D2 time, in the dossier. Typical criteria for a senior/architect move:

| Criterion | Example weight | How you'll score it (evidence, not feeling) |
|---|---|---|
| Problem and scope fit | 20 | Team's next-year problems vs what you want to work on |
| Learning and growth | 15 | People you'd learn from; new domains; promotion path |
| Team and manager | 15 | Peer conversation; manager signals; Westrum answers |
| Compensation (expected and bear case) | 20 | Appendix F output |
| Stability | 10 | Concept 7 score |
| Work model and time zones | 10 | Concept 10 facts |
| Career optionality | 10 | What doors this opens in 3–5 years |

Score each option 1–5 per criterion with a one-line evidence note, compute weighted totals, and **then** look at the result and ask whether it matches your gut. If it doesn't, find out why — usually a criterion is missing or a weight is wrong. That's useful; adjust deliberately and record why.

### 2. Run a premortem

Gary Klein's **premortem** reverses the usual question: *"It's eighteen months from now and taking this job was a mistake. What happened?"* Write five reasons. Common ones for engineers: the level was wrong and the first review went badly; the team was reorganised and the mandate disappeared; the on-call load was worse than described; the "modernisation" was never funded; the time-zone overlap meant every evening was meetings. Then check each against your dossier: **is there evidence it won't happen?** The unchecked ones are your remaining questions — ask them before signing.

### 3. Reversibility

Amazon's 2015 shareholder letter popularised the distinction between **one-way doors** (consequential, hard to reverse — decide carefully) and **two-way doors** (reversible — decide quickly). Job decisions are partly reversible — you can leave — but carry real costs: unvested equity, sign-on clawbacks, the time to re-search, a short tenure on your profile, cooldowns at the companies you declined. Identify what makes *this* decision less reversible (relocation, a long notice period, a non-compete) and give those factors extra weight.

### 4. Handle time pressure honestly

Offers sometimes come with short deadlines, especially when you're in other processes. It's normal to ask for a reasonable extension to finish another process — most companies agree to a short one, and asking politely reveals how they treat people under pressure. An offer that is **withdrawn** because you asked for a few days is itself information.

### 5. Close the loop

- **Write a decision record** — an ADR (Module 31) with context, options, criteria, the decision and the conditions under which you'd revisit it (*"If the platform mandate isn't funded by the second quarter, re-evaluate"*).
- **Decline other offers promptly and graciously.** You'll meet these people again.
- **Log outcomes for calibration** — real loop results against your mock scores (Module 38, Concept 15) and your research predictions against what you find after joining. The second is the best way to get better at this.
- **Keep the dossier.** Your first 90 days in the new role are the same research done from inside — and the dossier is your starting map.

**Interview-grade sentence** *(often useful in staff/architect behavioral rounds — "how do you make high-stakes decisions?")*: *"For consequential decisions I set the criteria and weights before I see the options, score each option with evidence rather than feeling, run a premortem — imagine it failed and list why — and check what makes the decision hard to reverse; then I record the decision and the condition that would make me revisit it, which is the same discipline I use for architecture decisions."*

---
# Worked example A — A senior .NET role on an Azure service team (Microsoft)

**Candidate:** you — a senior .NET/Azure engineer targeting a **Senior Software Engineer** role on an Azure service team at Microsoft, at a European engineering office. Facts marked *(verified)* were checked against public sources on October 8, 2026; facts marked *(illustrative)* are invented for the example and would come from your own recruiter and interviewer conversations.

### D1 — Pursue? (35 minutes)

```text
LOCATION/WORK MODEL   Role listed for a European office. Company policy: 3 days on-site for staff within
                      50 miles of an office, phased internationally during 2026 (verified, A1). Team practice
                      unknown → recruiter question.
BUSINESS              Azure is core to the company's growth and AI strategy; a service team inside Azure
                      is central, not experimental (A1 from annual report and earnings commentary).
LEVEL/RANGE           Title "Senior" → likely 63 or 64; sources disagree on naming at 64/65 (verified) → ask
                      the number. Location-specific pay data needed; US medians are not transferable.
LOOP (prior)          Team-dependent; recruiter → coding screen or Codility quiz → onsite with behavioral,
                      2–3 coding, system design, domain round → team matching; can interview with several teams
                      at once (verified, B2 — interviewing.io guide). C# welcome (B2).
AI POLICY             No AI in assessments/interviews unless instructed (verified, A1 — Candidate Code of Conduct).
STABILITY             Revenue growing; large org with periodic reorganisations and layoffs → ask about this
                      org specifically (B2).
DECISION              GO. Open questions for the recruiter: level number, team-level on-site practice,
                      domain round area, tooling rules, timeline.
```

### The recruiter call — what came back *(illustrative)*

```text
Level:            63, loop calibrated for 63 with possibility of 64 "if the panel sees it" → ask again after loop
Why open:         growth — the team is taking over a control-plane component from another org
Team practice:    3 days on-site, core hours overlap with a US team for 2–3 hours/day
Loop:             screen (60 min, Codility, C# allowed) → 4 onsite rounds: coding ×2 (shared editor),
                  system design (60), domain round on "distributed storage and data durability" (60)
                  + behavioral elements in each round; then team matching within the org
AI/tooling:       no AI anywhere; editor autocomplete is fine (in writing, follow-up email)
Timeline:         onsite in ~3 weeks; decision within ~1 week of onsite
Range:            provided for the office's market (record in dossier, A1)
```

### Loop map and re-weighted preparation

| Round | Tests | Prep (with Module 38 priority) |
|---|---|---|
| Screen | DS&A medium, correctness | 3 timed C# Codility-style sets (Module 36); blank-editor drill off (autocomplete allowed) |
| Coding ×2 | DS&A + code quality | 4 coding mocks — highest round count (priority ≈ 2 × gap + 0.5) |
| System design | SD, "compliance details" emphasis (B2) | Company-shaped problem: *control-plane API for provisioning a regional resource* — long-running operations, idempotency keys, quotas, **audit trail and data residency** — plus Module 37 problems |
| Domain: storage & durability | Depth in the team's area | Replication and consensus (Modules 8–9), durability and integrity (checksums, scrubbing), consistency trade-offs (Module 7); two own projects as deep dives with numbers |
| Behavioral (embedded) | Growth mindset, ownership, collaboration | Retag stories: failure → what changed; disagreement → how you learned from the other view; cross-team work |

### Claims log excerpt

```text
CLAIM                                                    GRADE  SOURCE                         AS-OF   EXPIRES
Candidates must not use AI in interviews unless told     A1     Candidate Code of Conduct      10-08   2027-04
Loop: 2 coding, SD, domain (storage) + embedded behav.   A1     recruiter email                10-14   loop
SD interviewers emphasise compliance/auditability        B2     interviewing.io anecdotes      10-08   2027-04
                                                         → confirm by asking the SD interviewer what "done" means for audit
Level 63, possible 64                                    A1     recruiter                      10-14   offer
Team takes over control-plane component                  A2     recruiter + HM (pre-loop chat) 10-16   2027-01
On-site 3 days; 2–3 h US overlap                         A1     recruiter                      10-14   2027-04
```

### Questions planned per interviewer

- **Hiring manager:** *"What does the handover of the control-plane component involve — code, on-call, people?"* · *"What would make this hire a clear success at twelve months?"* · *"What would the panel need to see for 64 rather than 63?"*
- **Peer (domain round):** *"What happened after the last durability-related incident?"* (triangulation question) · *"How are storage-format changes reviewed and rolled out?"*
- **Senior (system design):** *"How are compliance requirements — audit, residency — captured when a new feature is designed?"* · *"What's the most expensive decision in this component you'd revisit?"*

### After the loop *(illustrative)*

The peer and the hiring manager both described the same incident review and the two fixes that came out of it (**triangulated, A1**). The system-design interviewer spent ten minutes on audit-log design and retention — confirming the B2 claim (now **B1**). The recruiter reported the panel recommending **63**, with "strong on depth, less evidence of cross-team scope" — exactly the evidence gap you'd expect for 64. You supply a short written summary of a cross-team migration you led, and ask whether it changes the assessment; it doesn't this cycle, but the hiring manager names it as the path to 64. Decision matrix (Concept 24) scores the offer against an alternative; you accept 63 with a documented expectation, and an ADR noting the condition to revisit: *"If the control-plane handover hasn't happened by month six, reassess scope and level."*

---
# Worked example B — Solution architect at an insurer (compressed, illustrative)

**Role:** "Solution Architect — Claims Platform Modernisation" at a mid-size EU insurer *(fictional)*. The point of this example is a **D1 decision with yellow flags**, and how research turns those flags into specific questions rather than a vague worry.

```text
JD SIGNALS     "Modernise the claims platform", ".NET Framework 4.8 / WCF and .NET 8 Azure Functions",
               "AZ-204 required", "work with business stakeholders on target state", "architecture governance"
READING        Brownfield (Module 32) + enterprise architect shape (governance, stakeholders).
               .NET 8 and the Functions in-process model both hit end of support on 10 Nov 2026 (verified) —
               the JD doesn't mention it. AZ-204 retired 31 Jul 2026 (verified) → JD likely reused, not revised.
WORK MODEL     Remote within the EU; employment via an EOR for non-EU-entity countries (recruiter, A1)
               → equity n/a; check benefits and how EOR staff are promoted.
BUSINESS       Insurer; engineering is a cost centre; modernisation funded by a multi-year programme
               (annual report mentions the programme, A1). Stability score: 7/10.
YELLOW FLAGS   (1) EOL cliff unmentioned (2) stale JD (3) "governance" may mean a review board that gates
               every change (4) programme funding beyond next year unknown
QUESTIONS      → "What's the plan for the .NET 8 / in-process Functions services before 10 November?"
               → "Is the modernisation programme funded beyond next year, and who sponsors it?"
               → "How does a design get approved here — what does the architecture board change, typically?"
               → "What did the previous architect on this programme spend most of their time on?"
DECISION       GO, conditional on answers to the first two — they test whether this is a modernisation role
               or a maintenance role with a modernisation title.
```

At the recruiter call, the answer to the first question is *"we're aware; there's a plan to move to .NET 10 next quarter"* (next quarter is after the cliff), and the programme is funded through next year with renewal "expected". That's not a no-go — it's **exactly the job**: an architect who can make the urgent case, sequence the migration and handle the governance. It also tells you what to lead with in the architect rounds: a risk-first migration plan (Module 32), the cost-of-delay argument in executive language (Module 33) and an ADR for the upgrade path (Module 31). Research didn't just inform the decision; it handed you the content of the interview.

---
# Common interview questions with model answers

Two kinds of questions live here: **the motivation-and-fit questions** research is meant to answer well, and **questions about research and decision-making itself**, which staff and architect candidates are asked surprisingly often.

### Motivation and fit

**Q1. "Why do you want to work here?"**
> Concept 19's four-part structure, specific to them. *"From what I've read and heard from your team, you're consolidating three payment flows onto one ledger service this year, on .NET with Cosmos DB, and the hard part is moving live traffic without double-posting. I've done that shape of work — I led a dual-run migration of our order ledger with reconciliation gates, and we cut over with zero balance discrepancies. That's the kind of problem I want next, at your scale."*
*Key signal:* specific facts about them, the role restated as a problem, concrete proof, a genuine reason.

**Q2. "What do you know about us?"**
> Business model, where this team sits, one recent development, one engineering observation — interpreted, not recited. *"You make most of your revenue from usage-based subscriptions, which makes metering accuracy and cost per tenant central — and from your engineering talk on tenant isolation, it sounds like noisy neighbours were a real problem last year. I'd be curious how that's evolved."*
*Key signal:* understanding, not memorisation; ends with genuine curiosity.

**Q3. "Why are you leaving your current role?"**
> Toward, not away. *"I was hired to lead the migration off .NET Framework, and that's now done — the remaining work is incremental. I'm looking for a larger-scale platform problem with internal customers, which is what this team is."*
*Key signal:* no negativity; a coherent career narrative.

**Q4. "Why this team, rather than others here?"** *(team-dependent loops)*
> The team's specific problems from your hiring-manager conversation, matched to your experience. If you're interviewing with several teams, say so honestly and explain what draws you to each.
*Key signal:* you've differentiated the teams; you're not just looking for any job here.

**Q5. "What level do you see yourself at?"**
> *"Based on your ladder, the scope I've had — owning the architecture of the order platform across three teams and setting the migration direction — matches what you describe at the senior/staff boundary. I'd like the panel to assess that, and I'm happy to share more evidence of the cross-team work if it helps."*
*Key signal:* you know their ladder; you point to evidence; you don't fight the process.

**Q6. "What are your salary expectations?"** *(recruiter, early)*
> Defer politely and ask for the range: *"I'd like to understand the level and full scope first. Could you share the range for this level and location?"* If pressed after the level is known, give a researched range anchored at the top of the market for that level.
*Key signal:* professionalism; not anchoring yourself low.

**Q7. "Where else are you interviewing?"**
> Category and timing, not names: *"A couple of product companies on similar stacks; I expect decisions in about three weeks, so it would help to know your timeline."*
*Key signal:* honesty without weakening your position.

**Q8. "What would you do in your first 90 days?"** *(architect and staff roles)*
> A research plan, made specific by what you already learned. *"First, listen: meet the engineering leads, product and operations for each team that touches the claims platform; read the existing ADRs and the last few incident reviews; and map the current state at C4 container level. Second, find the urgent constraints — from what you told me, the .NET 8 and in-process Functions end of support is one — and get a decision on that within the first month. Third, propose a target state and a sequenced migration plan with measurable gates, and review it with the teams who'll build it before it goes to the architecture board. I'd want early wins that reduce risk rather than a big plan nobody owns."*
*Key signal:* listening first; specificity from your research; ownership by the implementers; urgency where it's real.

**Q9. "Our stack is X and you've mostly used Y. How would you ramp up?"**
> *"Honestly, I haven't run X in production. The closest I've done is Y, which shares the core ideas — [name them]. In my first weeks I'd pair on real tickets, read the service's runbooks and incident history, and pick a small change to take all the way to production. I learned Z the same way in about a month last year."*
*Key signal:* honesty, a concrete plan, evidence you've done it before.

**Q10. "Do you have any concerns about joining us?"**
> A real, mild, researched concern, framed as a question: *"The main thing I'd want to understand better is the on-site expectation in practice and the overlap with the US team — how many hours a day of overlap does the team actually use?"*
*Key signal:* candour; you're evaluating them, maturely.

### Questions about research and decision-making

**Q11. "How do you evaluate a new technology or vendor?"**
> The same method as job research: *"I start from the decision it supports and the criteria — what would make us choose or reject it — before I read any marketing. I grade sources: vendor documentation is authoritative about features and weak about operational reality, so I look for independent production experience, ideally from teams at our scale, and confirm anything important twice. I check the base rate — how often do teams like ours regret this kind of choice? — and I run a premortem on adoption. Then I write it up as an ADR with the conditions that would reopen it."*
*Key signal:* criteria first, graded sources, base rates, a recorded decision.

**Q12. "How do you get up to speed on an unfamiliar codebase or organisation?"**
> *"Questions before sources: who are the users, what does the system do that matters most, where does it break, and how are decisions made. Then sources in order of reliability for each question — the code and production telemetry for behaviour; incident reviews for failure modes; ADRs and design docs for intent; people for the history that isn't written down. I write a short map of the current state and share it so people can correct it — the corrections are the most useful part."*
*Key signal:* structured, source-aware, and collaborative.

**Q13. "How do you make a high-stakes decision with incomplete information?"**
> Concept 24's sentence, with an example. Mention the value-of-information test: *"I ask whether more information could actually change the decision — if not, I decide now and record the conditions that would make me revisit it."*
*Key signal:* bounded research; reversibility; a recorded decision.

**Q14. "What questions do you have for me?"**
> Two or three questions from Concept 20 matched to *this* interviewer — and at least one follow-up on their answer. For a senior engineer: *"What's the most expensive technical decision the team is living with — and would you make it the same way again?"* Then follow up on the answer.
*Key signal:* questions that reveal judgement and seek decision-relevant information, plus genuine listening.

**Q15. "What did you think of the interview process?"** *(occasionally asked)*
> Honest and specific, positive where deserved: what was well organised, which round felt closest to the real work, and — if there's a gentle improvement to suggest — say it constructively.
*Key signal:* candour and the habit of observing processes as systems.

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Purpose | Collecting facts about the company | Research tied to three decisions — pursue, prepare, accept |
| Order | Opening the careers page first | Questions written first; sources chosen per question |
| Budget | Same depth for every opportunity | Staged: minutes at D1, hours at D2, days at D3 |
| Blockers | Discovering the location or entity problem after the onsite | Location, entity, work model and approved headcount checked first |
| Sources | "I read it on Glassdoor" | Source reliability and claim credibility graded separately |
| Reviews | Trusting the average rating | Recent, role-matched themes; extremes discounted |
| Survivors | Only talking to current staff | Recent leavers as well |
| Freshness | Year-old policies treated as current | As-of dates and expiries on every claim |
| Confirmation | Acting on one anonymous post | Two independent sources before acting |
| Business | No idea how the company makes money | Business model and where engineering sits, in one sentence |
| Stability | Ignoring runway and reorganisations | Funding/profitability, layoffs and org churn checked; registry filings read |
| Culture | "Great culture" from the careers page | Westrum and DORA questions; "the last time" events |
| Stack | Ignoring versions | .NET/Azure version signals read — e.g. the 10 November 2026 cliff |
| JD | Taking the list literally | Real requirements, level verbs, archetype, staleness |
| Level | Accepting the title | Level number and expectations confirmed; down-levelling anticipated |
| Team | "What's the team like?" | Why the role is open; next-year problems; success criteria; peer conversation |
| Loop | Preparing for a generic interview | Loop map: rounds, interviewers, tools, rules; centralised vs team-dependent |
| AI rules | Assuming | Per round, in writing, including IDE features |
| Recruiter call | Passive screening | Scripted research; written follow-up restating the loop |
| Compensation | Naming a number before the level is known | Deferring politely; asking for the range; researched anchor |
| Vocabulary | Generic STAR stories | Stories retagged in the company's principles; coverage checked |
| Mocks | Generic problems until the end | Company-shaped design problem; domain-round prep; weighted schedule |
| "Why us" | Flattery or self-focus | Them → role → proof → you, specific and two-sided |
| Questions | "What's the culture like?" | Decision-relevant, per interviewer, with follow-ups and one triangulation question |
| During the loop | Only performing | Observing the process as evidence; post-round notes |
| Offer | Comparing headline totals | Year-by-year components, vesting, scenarios, net and FX |
| Equity | Private equity at the last round price | Scenarios including zero; preferences and dilution considered |
| Decision | Deciding on the last conversation | Pre-set weighted criteria, premortem, reversibility, ADR |
| Afterwards | Discarding the research | Calibration log; dossier reused for the first 90 days |

---
# Practice exercises

1. **Question tree.** Pick one real target company. Write the four-branch question tree (Concept 2) before opening any source. Mark which questions you expect public sources to answer and which only people can.
2. **Value-of-information triage.** List fifteen things you could research about that company. For each, mark uncertain? / could flip a decision? / stakes. Keep only the top six.
3. **Source grading.** Gather ten claims from five different sources and grade each on reliability (A–F) and credibility (1–6). Find at least one A-source claim that is weak for the question it's being used for.
4. **Review audit.** Read thirty recent engineering reviews for one company. Ignore ratings; extract recurring *specific* themes. How many survive a recency and role filter? What would you ask in the loop to test the top two?
5. **Registry check.** Find the filed financial statements of one employer in a public registry (Companies House, SEC EDGAR, or APR for a Serbian entity). Record revenue and headcount trends for three years. What does it tell you that the careers page doesn't?
6. **Stack reading.** From job descriptions, talks and public repos, infer the .NET version and Azure hosting model of one team. Write the questions you'd ask about the November 2026 end-of-support date.
7. **JD decoding.** Take three senior/architect job descriptions. For each: real requirements, level from the verbs, staff archetype, context signals, staleness signals. Compare your level estimate with the title.
8. **Level mapping.** For two target companies, find the level number for the role you'd target and that level's public expectations. Map three of your stories to the expectations' scope verbs.
9. **Recruiter-call rehearsal.** Run the Concept 16 script with a partner playing a recruiter who deflects the range question and is vague about the level. Practise the polite deferral and the written follow-up.
10. **Loop map.** For one real or past loop, produce a complete loop map. Mark each cell's grade. Which cells are B or worse, and how would you raise them?
11. **Translation table.** For a principle-driven company, map its principles to the six dimensions, and identify which stories cover each. Fix any uncovered principle with a new story or a re-framed one.
12. **Company-shaped design problem.** Build one from a real product area (Concept 18), with a six-rung probe ladder. Run it as a full mock (Module 38).
13. **"Why us" drafting.** Write three versions of your "why us" answer for three different companies. Ask someone to guess which company each is for. If they can't, the answers aren't specific enough.
14. **Question bank.** Write five questions per interviewer type (Concept 20), with planned follow-ups. Choose your triangulation question.
15. **Offer model.** Use Appendix F's tool with two real or hypothetical offers. Change the bear-case share-price assumption until the ranking flips. How robust is your preference?
16. **Premortem.** For your current top opportunity, write the five most likely reasons it would turn out to be a mistake. For each, find evidence in your dossier or write the question that would supply it.
17. **Decision record.** Write a career ADR for a past job decision, with hindsight. Which criteria did you weight correctly, and which did you miss?

---
# Free resources and learning material

All free to read or use unless marked *(book)*, *(paid)* or *(partly paid)*. Start with the ★ items. Fast-moving facts (policies, laws, support dates, certifications) were checked on October 8, 2026.

### Decision-making and research method
- ★ [Admiralty code (source reliability and information credibility) — Wikipedia](https://en.wikipedia.org/wiki/Admiralty_code) — the two-axis grading in Concept 3.
- [Value of information — Wikipedia](https://en.wikipedia.org/wiki/Value_of_information) — the formal version of Concept 1's test.
- [Reference class forecasting — Wikipedia](https://en.wikipedia.org/wiki/Reference_class_forecasting) — the outside view and base rates.
- [Kahneman & Lovallo — Delusions of Success (Harvard Business Review, 2003)](https://hbr.org/2003/07/delusions-of-success-how-optimism-undermines-executives-decisions) — inside vs outside view in practice.
- ★ [Gary Klein — Performing a Project Premortem (Harvard Business Review, 2007)](https://hbr.org/2007/09/performing-a-project-premortem).

### Bias in employer reviews and online ratings
- ★ [Marinescu, Klein, Chamberlain & Smart — Incentives Can Reduce Bias in Online Reviews (NBER w24372)](https://www.nber.org/papers/w24372) — Glassdoor's own data on selection bias in voluntary reviews.
- [Hu, Pavlou & Zhang (2009) — Overcoming the J-shaped distribution of product reviews](https://doi.org/10.1145/1562764.1562800).
- [Green, Huang, Wen & Zhou (2019) — Crowdsourced employer reviews and stock returns](https://doi.org/10.1016/j.jfineco.2019.03.012) — evidence that review *trends* carry signal.
- [Glassdoor — Wikipedia](https://en.wikipedia.org/wiki/Glassdoor) — history, the 2024 real-name policy change and reliability research.
- [Fortune — Glassdoor requires real names for accounts (2024)](https://fortune.com/2024/03/21/glassdoor-180-users-real-names-accounts-employers-trashed-them).

### Interview processes and loop structure
- ★ [interviewing.io — Ultimate guide to FAANG interviews for senior engineers](https://interviewing.io/guides/hiring-process) — centralised vs team-dependent loops, interviewer training, parallel team interviews.
- ★ [interviewing.io — Senior engineer's guide to Microsoft's interview process](https://interviewing.io/guides/hiring-process/microsoft) — the domain-specific round, team matching, C#.
- ★ [Microsoft Careers — How we hire / hiring tips (incl. Candidate Code of Conduct)](https://careers.microsoft.com/v2/global/en/hiring-tips) and [interview tips](https://careers.microsoft.com/us/en/interviewtips).
- ★ [Amazon — How we hire](https://www.amazon.jobs/content/en/how-we-hire) and [Leadership Principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles).
- [Amazon — SDE III interview prep](https://amazon.jobs/content/en/how-we-hire/sde-iii-interview-prep).
- [Google — How we hire](https://www.google.com/about/careers/applications/how-we-hire) and [interview tips](https://www.google.com/about/careers/applications/interview-tips).
- [Meta — Preparing for your full loop interview](https://www.metacareers.com/swe-prep-onsite).
- [Tech Interview Handbook — Interview formats at top companies](https://www.techinterviewhandbook.org/interview-formats-top-companies/).

### AI policies in interviews
- ★ [Employer AI policy tracker (checked September 29, 2026)](https://ophyai.com/blog/interview-tips/companies-that-allow-ai-in-interviews) — a secondary source; follow its links to each employer's own page.
- [Anthropic — Guidance on candidates' AI usage](https://www.anthropic.com/candidate-ai-guidance).
- [Canva Engineering — Yes, you can use AI in our interviews](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/).
- [Hello Interview — Meta's AI-enabled coding interview](https://www.hellointerview.com/blog/meta-ai-enabled-coding).

### Levels and compensation
- ★ [Levels.fyi](https://www.levels.fyi/) — self-reported compensation by company, level and location; level comparisons.
- [Levels.fyi — Microsoft software engineer levels and pay](https://www.levels.fyi/companies/microsoft/salaries/software-engineer).
- ★ [The Pragmatic Engineer — The trimodal nature of software engineering salaries](https://blog.pragmaticengineer.com/software-engineering-salaries-in-the-netherlands-and-europe/) and [the trimodal model revisited (2024)](https://newsletter.pragmaticengineer.com/p/trimodal-nature-of-tech-compensation) *(partly paid)*.
- [TechPays](https://techpays.com/) — European compensation data across tiers.
- [The Pragmatic Engineer — Equity for software engineers](https://blog.pragmaticengineer.com/equity-for-software-engineers/) — RSUs, options, vesting, cliffs, refreshers.
- [Tech Interview Handbook — Understanding compensation](https://www.techinterviewhandbook.org/understanding-compensation/).
- ★ [Patrick McKenzie — Salary Negotiation: Make More Money, Be More Valued](https://www.kalzumeus.com/2012/01/23/salary-negotiation/).
- ★ [Haseeb Qureshi — Ten Rules for Negotiating a Job Offer](https://haseebq.com/my-ten-rules-for-negotiating-a-job-offer/).
- [Blind](https://www.teamblind.com/) and [Glassdoor](https://www.glassdoor.com/) — use for themes and ranges, with Concept 4's corrections.

### Pay transparency rules
- [Directive (EU) 2023/970 — Pay Transparency Directive (EUR-Lex)](https://eur-lex.europa.eu/eli/dir/2023/970/oj).
- [Trusaic — EU Pay Transparency Directive transposition updates](https://trusaic.com/blog/eu-pay-transparency-directive/).
- [Lockton — EU Pay Transparency Directive implementation status (September 2026)](https://global.lockton.com/us/en/news-insights/eu-pay-transparency-directive-implementation-status).
- [Brightmine — US pay transparency laws by state and locality](https://www.brightmine.com/us/resources/charts/u-s-pay-transparency-laws-by-state-and-locality/).

### Company financials, stability and registries
- ★ [SEC EDGAR full-text search](https://www.sec.gov/edgar/search/) — annual reports (10-K), risk factors, segment data.
- [UK Companies House](https://find-and-update.company-information.service.gov.uk/) — filed accounts and officers for UK companies.
- [OpenCorporates](https://opencorporates.com/) — registry data across many jurisdictions.
- [APR — Serbian Business Registers Agency](https://www.apr.gov.rs/) — the Registry of Financial Statements publishes filed annual statements, including employee counts.
- [Crunchbase](https://www.crunchbase.com/) and [Dealroom](https://dealroom.co/) — funding history *(partly paid)*.
- [Layoffs.fyi](https://layoffs.fyi/) — tech layoff tracker.
- [Microsoft Investor Relations](https://www.microsoft.com/en-us/investor) — an example of an IR page worth reading for strategy and segments.

### Engineering culture and organisation
- ★ [DORA — Generative organisational culture (Westrum)](https://dora.dev/capabilities/generative-organizational-culture/) — the typology and the six survey statements.
- [DORA — Capabilities catalogue](https://dora.dev/capabilities/) and [DORA Quick Check](https://dora.dev/quickcheck/).
- [Ron Westrum (2004) — A typology of organisation cultures (BMJ Quality & Safety)](https://qualitysafety.bmj.com/content/13/suppl_2/ii22.short).
- [Mel Conway — Conway's Law](http://www.melconway.com/Home/Conways_Law.html).
- ★ [Team Topologies — Key concepts](https://teamtopologies.com/key-concepts) — the four team types.
- ★ [StaffEng — Staff archetypes](https://staffeng.com/guides/staff-archetypes/) — Tech Lead, Architect, Solver, Right Hand.
- [StaffEng — Guides](https://staffeng.com/guides/) — what staff-plus scope looks like, for level research.

### Questions to ask
- ★ [Reverse interview — questions to ask the company](https://github.com/viraptor/reverse-interview) — a large community list; also available [in Serbian (Latin)](https://github.com/viraptor/reverse-interview/blob/master/translations/SERBIAN-Latin.md).
- ★ [Tech Interview Handbook — Best questions to ask at the end of the interview](https://www.techinterviewhandbook.org/final-questions/).
- [Julia Evans — Questions I'm asking in interviews](https://jvns.ca/blog/2013/12/30/questions-im-asking-in-interviews/).
- [Joel Spolsky — The Joel Test](https://www.joelonsoftware.com/2000/08/09/the-joel-test-12-steps-to-better-code/) — dated in places, still a useful prompt list.

### Stack research and the .NET/Azure lens
- ★ [.NET and .NET Core support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core) — LTS/STS dates.
- [InfoWorld — End of support looms for .NET 8 and .NET 9](https://www.infoworld.com/article/4191704/end-of-support-looms-for-net-8-and-net-9.html) — the 10 November 2026 date and the .NET 10 recommendation.
- [Migrate .NET apps from the in-process model to the isolated worker model (Azure Functions)](https://learn.microsoft.com/azure/azure-functions/migrate-dotnet-to-isolated-model).
- [.NET Blog](https://devblogs.microsoft.com/dotnet/) — release and support announcements.
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/) — vocabulary and reference architectures for company-shaped design problems.
- [Microsoft Certified: Azure Solutions Architect Expert (AZ-305)](https://learn.microsoft.com/credentials/certifications/azure-solutions-architect/) and [retired certification exams](https://learn.microsoft.com/credentials/support/retired-certification-exams) — for reading certification requirements in JDs.
- [BuiltWith](https://builtwith.com/), [Wappalyzer](https://www.wappalyzer.com/) and [StackShare](https://stackshare.io/) — public stack fingerprints *(partly paid)*.
- [Engineering blogs list — GitHub](https://github.com/kilimchoi/engineering-blogs) — find a company's engineering blog.

### Work model
- [TechRadar — Microsoft's three-day on-site policy and phased rollout](https://www.techradar.com/pro/microsoft-issues-new-hybrid-policy-that-will-see-global-workers-in-office-3-days-per-week) — an example of why work-model facts need an as-of date.

### Regional sources (Serbia and the Western Balkans)
- [Joberty](https://www.joberty.com/) — developer reviews of tech employers, salaries and interview experiences for Serbia and the region (apply Concept 4's corrections).
- [HelloWorld.rs](https://www.helloworld.rs/) — Serbian IT job board and salary surveys.
- [APR — Registry of Financial Statements](https://www.apr.gov.rs/) — filed financials of Serbian entities, including local subsidiaries of foreign companies.

### Books
- *(book)* Matthew Skelton & Manuel Pais — *Team Topologies*.
- *(book)* Nicole Forsgren, Jez Humble & Gene Kim — *Accelerate* — the research behind the DORA measures and Westrum culture findings.
- *(book)* Will Larson — *Staff Engineer: Leadership Beyond the Management Track* — archetypes and staff-level scope.
- *(book)* Tanya Reilly — *The Staff Engineer's Path*.
- *(book)* Gergely Orosz — *The Software Engineer's Guidebook* — levels, compensation and how tech companies work.
- *(book)* Daniel Kahneman — *Thinking, Fast and Slow* — inside view, base rates.
- *(book)* Philip Tetlock & Dan Gardner — *Superforecasting* — calibrated judgement under uncertainty.
- *(book)* Gary Klein — *Sources of Power* — how experts decide; the origin of the premortem.
- *(book)* Chris Voss — *Never Split the Difference* — negotiation through questions and listening.

### Previous modules to revisit
- Modules 1–2 — what's scored; loop shapes.
- Module 4 — requirements gathering, which is the same discipline as Concept 2.
- Modules 30–33 — design docs, ADRs, brownfield, cost — the content of architect loops and the format of your career decision record.
- Modules 34–35 — STAR and the story bank, retagged in Concept 17.
- Module 37 — the method behind company-shaped design problems.
- Module 38 — mocks, rubrics, the priority formula and calibration against real outcomes.

---
# Quick-recall sheet

**One sentence.** Research reduces uncertainty for three decisions — pursue, prepare, accept; research only what can change one of them, grade every source, confirm before acting.

**Three decisions and budgets.** D1 pursue (15–45 min) · D2 prepare (2–6 h) · D3 accept (as long as it takes). Staged depth; one dossier carried forward.

**Value of information.** Uncertain × could flip the decision × stakes. All three, or skip it.

**Question tree.** Company · Role · Loop · Offer/terms. Questions before sources. Public sources answer company questions; people answer team questions.

**Grading.** Source reliability A–F × claim credibility 1–6, separately; reliability depends on the question. Act on A1–B2, investigate C3–D3, discard the rest.

**Biases.** Review selection (extremes) · survivorship (talk to leavers) · recency (expiry dates) · branding (blog = values, people = practice) · interview reports (round types, not questions) · ghost postings (~1 in 5; ask about approved headcount) · inside view (base rates first).

**Triangulation.** Two sources that couldn't be wrong for the same reason. Claims log: claim · grade · source · as-of · expires · feeds.

**Business.** How it makes money; profit centre vs cost centre; trimodal tiers (local-industry · local-all · regional/global).

**Stability.** Annual reports and risk factors; funding, runway, profitability; registries (Companies House, EDGAR, APR); layoffs, reorganisations, leadership change.

**Culture.** Westrum (pathological/bureaucratic/generative; failure → inquiry); DORA four measures; Conway; Team Topologies types; decision records; incidents and on-call. Ask for the last time.

**.NET/Azure signals.** .NET 8 and 9 end of support **10 Nov 2026**; .NET 10 LTS to Nov 2028; .NET 11 STS from 10 Nov 2026; Functions in-process ends **10 Nov 2026**; .NET Framework 4.8 follows Windows. AZ-204 retired 31 Jul 2026 → AI-200; AZ-305 current.

**Work model.** Location, entity (employee / EOR / contractor), on-site reality, time zones of decision-makers, pay-information rights (US ~16 states + DC; EU directive where transposed).

**JD.** Real requirements vs wish list · level from verbs · archetype (Tech Lead, Architect, Solver, Right Hand) · context and staleness signals.

**Level.** Ask the number and the expectations; sources disagree on titles; down-levelling is the default risk; evidence at the right altitude.

**Team.** Why open (growth/backfill/new/insourcing) · next-year problems · success at 12 months · manager signals · talk to a peer.

**Loop map.** Stage · format · length · interviewer · medium · AI mode · tests · prep source. Centralised (pool, matching, one at a time) vs team-dependent (team interviews, domain round, parallel teams).

**AI rules.** No AI / permitted / expected — per round, in writing, incl. autocomplete and take-homes. When in doubt, don't.

**Recruiter call.** Blockers → level and why open → loop → AI rules → range. Defer the number; ask for theirs. Written follow-up restating the loop.

**Preparation.** Translate values to D1–D6; re-weight; retag stories with coverage check; company-shaped design problem; domain-round prep; mocks in the real tool and rules.

**"Why us".** Them → role → proof → you (→ now). Specific enough to fit no other company.

**Questions.** Decision-relevant · "last time" · right for the interviewer · altitude · not on the website. Follow up. One triangulation question asked twice.

**Per-round card.** Rules · likely focus · local emphasis · opening moves · deep dives · stories · numbers · questions · recovery.

**Loop as evidence.** Organisation · honesty · preparation · respect · candour; calibrate for loop type; post-round notes, no performance post-mortems.

**Offer.** Base · bonus (target ≠ paid) · sign-on (year one, clawback) · equity (schedule, refreshers; private: preferences, dilution, zero scenario) · benefits · level · terms. Year by year, expected and bear, net, FX.

**Decision.** Criteria and weights set before offers · evidence-scored matrix · premortem · reversibility · ask for time · career ADR · calibration log · reuse the dossier for the first 90 days.

---
# Appendix A — Opportunity dossier template

```text
OPPORTUNITY: ____________  TEAM: ____________  ROLE/LEVEL: ____________  CREATED: ____  STATUS: D1|D2|D3|closed

1. DECISION LOG
   D1 (date): GO / NO-GO because ______________________ ; open questions: ____________
   D2 (date): prep plan summary ______________________
   D3 (date): decision ______  → see section 9

2. QUESTION TREE (✓ answered · ? open · → ask whom)
   Company: business model __ · health __ · culture __ · stack __ · work model __
   Role:    day-to-day __ · level __ · team/manager __ · why open __ · success at 12 months __
   Loop:    stages __ · rules __ · vocabulary __ · decision process __
   Terms:   comp structure/band __ · entity/contract __ · equity risk __ · notice/IP/non-compete __

3. CLAIMS LOG
   CLAIM | GRADE | SOURCE(S) | AS-OF | EXPIRES | FEEDS

4. LOOP MAP
   STAGE | FORMAT | LENGTH | INTERVIEWER | MEDIUM | AI MODE | TESTS | PREP SOURCE

5. VOCABULARY → DIMENSIONS
   LOCAL TERM | D1–D6 | EVIDENCE THAT SCORES AT TARGET LEVEL | STORY

6. STORY MAPPING
   STORY | UNIVERSAL TAGS | COMPANY TAGS | LEAD FOR

7. QUESTION BANK (per interviewer type; ★ = triangulation question)

8. PER-ROUND CARDS + POST-ROUND NOTES

9. OFFER ANALYSIS AND DECISION RECORD
   Offer model output (Appendix F) · decision matrix · premortem · reversibility notes · ADR
```

---
# Appendix B — Recruiter call script (one page)

```text
OPEN (0–3)       "Thanks for reaching out — to make sure I'm thinking about the right thing: this is the ____
                 role on the ____ team?"
THEIR PART       90-second career summary at target level · specific "why" (Concept 19) · logistics
BLOCKERS         □ Open to someone based in ____? Which entity would employ me?
                 □ On-site/overlap expectations for this team in practice?
                 □ Headcount approved and open now? Target start date?
ROLE & LEVEL     □ Which level (number/band)? Calibrated for one level or deciding between two?
                 □ Why is the role open? □ Hiring manager and the team's main focus this year?
LOOP             □ Every stage: format, length, interviewer □ Domain round? Area?
                 □ Tools; language choice (C#?) □ AI and tooling rules per round (autocomplete?)
                 □ In-person? Take-home? Presentation? □ Decision process; team matching; timeline; cooldown
                 □ Official prep material?
COMPENSATION     □ "Could you share the range for this level and location?"
                 If asked for expectations: "I'd like to understand the level and scope first — what range
                 are you working with?"   If asked for current pay: decline politely; refocus on the range.
CLOSE            "Could you send the loop details by email so I prepare for exactly the right format?"
WITHIN 1 HOUR    □ Update loop map and claims log (process facts A; culture claims C)
                 □ Thank-you email restating the loop as understood
```

---
# Appendix C — Question bank by interviewer

### Hiring manager
- What are the two or three biggest problems this team needs solved in the next year?
- What would make this hire a clear success at twelve months? What did someone at this level do last year that earned a strong rating?
- Why is the role open? What did the previous person work on?
- How did the team make its last significant technical decision, and where is it recorded?
- How do you run one-to-ones, and what do you use them for?
- What would the panel need to see for the level above? *(after the loop)*

### Peer engineer
- What did you work on last week?
- ★ What happened after the last significant incident — was there a written review, and what changed?
- How long does a typical change take from merge to production? How often does the service deploy?
- How many pages did you get on your last on-call rotation?
- What surprised you after joining? What would you change if you could?
- How do you get something done that needs another team's change?

### Senior / staff engineer
- What's the most expensive technical decision the team is living with, and would you make it again?
- Where is the architecture heading in the next year, and what's blocking it?
- How are cross-team design decisions proposed, reviewed and recorded?
- How are platform upgrades handled — for example, the move off .NET 8 before 10 November?
- What does "production-ready" mean here for a new service?

### Architect / principal
- How much of the target architecture is decided centrally vs by teams?
- What does the architecture review typically change in a proposal?
- How do you handle a team that wants to diverge from a standard?
- Which architectural decisions from the last two years would you revisit?

### Product / business stakeholder
- Where does engineering most often surprise you — positively or negatively?
- What speed-vs-robustness trade-off came up recently, and how was it decided?
- How do you and engineering agree on priorities when they conflict?

### Director / skip-level
- How does this team's work connect to the organisation's priorities this year?
- What would make you increase or decrease investment in this area?
- How has the organisation changed in the last year, and what's next?

### Westrum and DORA probes (any engineer)
- When you start a piece of work, how do you find out who depends on it? *(information sought)*
- Tell me about the last time someone raised a problem late — what happened next? *(messengers)*
- Who's on call for the services this team builds? *(shared responsibility)*
- What changed in how the team works in the last six months, and who proposed it? *(novelty)*

---
# Appendix D — Source grading quick reference

```text
RELIABILITY (of the source, FOR THIS KIND OF QUESTION)        CREDIBILITY (of the claim)
A authoritative — regulatory filing, registry, official        1 confirmed by an independent source
  policy, recruiter's written process statement                2 probably true — consistent, unconfirmed
B usually reliable — established press, aggregators with       3 possibly true — plausible, uncorroborated
  many data points, guides built from many interviewers        4 doubtful — inconsistent with other info
C fairly reliable — a verified individual insider              5 improbable — contradicted by better info
D not usually reliable — single anonymous review, forum        6 cannot judge
E unreliable — reputation-management content, stale scrapes
F cannot judge

ACT ON A1–B2 · INVESTIGATE C3–D3 (→ a question for a person) · DISCARD E/5 and below unless many independent D's agree
INDEPENDENCE TEST: "Could both sources be wrong for the same reason?"  If yes, they are one source.
DEFAULT EXPIRIES: policies 6–12 months · team structure 3–6 months · comp bands 6–12 months · public financials 1 quarter
```

---
# Appendix E — Job description decoder

```text
POSTING: ____________   DATE FIRST SEEN: ____   RE-POSTED? ____   NAMED CONTACT? ____
REAL REQUIREMENTS (3–5):        ____ ____ ____ ____ ____
WISH LIST (stack/tooling):      ____________________________________________
LEVEL FROM VERBS:               mid | senior | staff/principal | solution/enterprise architect     TITLE SAYS: ____
ARCHETYPE (staff+):             Tech Lead | Architect | Solver | Right Hand
CONTEXT SIGNALS:                greenfield | modernise/migrate | regulated | fast-paced | many hats | stakeholder-heavy
.NET/AZURE SIGNALS:             versions ____  hosting ____  data ____  messaging ____  EOL exposure (10 Nov 2026)? ____
STALENESS SIGNALS:              retired certs ____  old product names ____  team size vs LinkedIn ____
MISMATCHES → RECRUITER QUESTIONS: ____________________________________________
MY GAPS → HONEST RAMP-UP ANSWERS: ____________________________________________
```

---
# Appendix F — Offer model (C#) and decision matrix

### F1. The tool

Models each offer year by year: base with raises, bonus at an expected and a bear-case payout, sign-on in year one, the initial equity grant plus optional annual refreshers on a vesting schedule, share-price scenarios with probabilities, and an FX rate into one reporting currency. Reports **expected** and **bear-case** totals per year and in sum. Amounts are **gross**; compare net separately when tax systems differ, and convert contractor rates to an employee-equivalent first.

```csharp
// dotnet run -- offers.json
using System.Text.Json;

if (args.Length == 0) { Console.WriteLine("usage: <offers.json>"); return; }

var input = JsonSerializer.Deserialize<OfferSet>(
        File.ReadAllText(args[0]), new JsonSerializerOptions(JsonSerializerDefaults.Web))
    ?? throw new InvalidOperationException("Empty input.");

foreach (var offer in input.Offers)
{
    Validate(offer);
    decimal fx = offer.FxToReport;
    decimal worstPriceChange = offer.Scenarios.Min(s => s.AnnualPriceChange);
    decimal sumExpected = 0m, sumBear = 0m;

    Console.WriteLine($"\n{offer.Name} — gross, in {input.ReportCurrency}");
    Console.WriteLine($"{"Year",4} {"Cash exp",10} {"Equity exp",11} {"Total exp",10} {"Total bear",11}");

    for (int year = 1; year <= input.Years; year++)
    {
        decimal salary = offer.Base * Growth(offer.AnnualRaise, year - 1);
        decimal signOn = year == 1 ? offer.SignOn : 0m;
        decimal cashExpected = salary * (1 + offer.BonusTarget * offer.ExpectedBonusPayout) + signOn;
        decimal cashBear = salary * (1 + offer.BonusTarget * offer.BearBonusPayout) + signOn;

        decimal equityExpected = offer.Scenarios.Sum(s => s.Probability * EquityVesting(offer, year, s.AnnualPriceChange));
        decimal equityBear = EquityVesting(offer, year, worstPriceChange);

        decimal totalExpected = (cashExpected + equityExpected) * fx;
        decimal totalBear = (cashBear + equityBear) * fx;
        sumExpected += totalExpected;
        sumBear += totalBear;

        Console.WriteLine($"{year,4} {cashExpected * fx,10:N0} {equityExpected * fx,11:N0} {totalExpected,10:N0} {totalBear,11:N0}");
    }
    Console.WriteLine($"{"Sum",4} {"",10} {"",11} {sumExpected,10:N0} {sumBear,11:N0}");
}

// Value of equity vesting in `year`. grantYear 0 is the initial grant; grantYear k ≥ 1 is a refresher
// granted at the end of year k. Each grant is valued at its grant price, moved by the scenario's
// annual price change for the years between grant and vest.
static decimal EquityVesting(Offer offer, int year, decimal annualPriceChange)
{
    decimal value = 0m;
    for (int grantYear = 0; grantYear < year; grantYear++)
    {
        decimal grant = grantYear == 0 ? offer.InitialGrant : offer.RefreshPerYear;
        int scheduleIndex = year - grantYear - 1;
        if (grant == 0m || scheduleIndex >= offer.VestSchedule.Count) continue;
        value += grant * offer.VestSchedule[scheduleIndex] * Growth(annualPriceChange, year - grantYear);
    }
    return value;
}

static decimal Growth(decimal rate, int years)
{
    decimal factor = 1m;
    for (int i = 0; i < years; i++) factor *= 1 + rate;
    return factor;
}

static void Validate(Offer offer)
{
    decimal probabilities = offer.Scenarios.Sum(s => s.Probability);
    if (offer.Scenarios.Count == 0 || Math.Abs(probabilities - 1m) > 0.001m)
        throw new InvalidOperationException($"{offer.Name}: scenario probabilities sum to {probabilities}, expected 1.");
    if (offer.VestSchedule.Count > 0 && Math.Abs(offer.VestSchedule.Sum() - 1m) > 0.001m)
        throw new InvalidOperationException($"{offer.Name}: vest schedule sums to {offer.VestSchedule.Sum()}, expected 1.");
}

public sealed record Scenario(string Name, decimal Probability, decimal AnnualPriceChange);

public sealed record Offer(
    string Name, decimal FxToReport,
    decimal Base, decimal AnnualRaise,
    decimal BonusTarget, decimal ExpectedBonusPayout, decimal BearBonusPayout,
    decimal SignOn,
    decimal InitialGrant, decimal RefreshPerYear, List<decimal> VestSchedule,
    List<Scenario> Scenarios);

public sealed record OfferSet(string ReportCurrency, int Years, List<Offer> Offers);
```

Notes worth knowing: `JsonSerializerDefaults.Web` gives case-insensitive camelCase binding, so the JSON below maps directly onto the positional records; `decimal` avoids binary floating-point surprises in money arithmetic; an `annualPriceChange` of `-1` makes every vest worth zero, which is how to model the "no liquidity" scenario for private equity. For private equity, read the equity column as an *expected value at an eventual exit*, not cash in that year. Vesting is modelled annually; a back-loaded schedule is just a different `vestSchedule` array.

### F2. Example input

```json
{
  "reportCurrency": "EUR",
  "years": 4,
  "offers": [
    {
      "name": "A — public company",
      "fxToReport": 1.0,
      "base": 90000, "annualRaise": 0.03,
      "bonusTarget": 0.10, "expectedBonusPayout": 1.0, "bearBonusPayout": 0.5,
      "signOn": 15000,
      "initialGrant": 120000, "refreshPerYear": 20000,
      "vestSchedule": [0.25, 0.25, 0.25, 0.25],
      "scenarios": [
        { "name": "bear", "probability": 0.25, "annualPriceChange": -0.20 },
        { "name": "base", "probability": 0.50, "annualPriceChange": 0.05 },
        { "name": "bull", "probability": 0.25, "annualPriceChange": 0.20 }
      ]
    },
    {
      "name": "B — Series B startup",
      "fxToReport": 1.0,
      "base": 100000, "annualRaise": 0.04,
      "bonusTarget": 0.0, "expectedBonusPayout": 1.0, "bearBonusPayout": 1.0,
      "signOn": 0,
      "initialGrant": 200000, "refreshPerYear": 0,
      "vestSchedule": [0.25, 0.25, 0.25, 0.25],
      "scenarios": [
        { "name": "no exit", "probability": 0.6, "annualPriceChange": -1.0 },
        { "name": "flat",    "probability": 0.3, "annualPriceChange": 0.0 },
        { "name": "good",    "probability": 0.1, "annualPriceChange": 0.5 }
      ]
    }
  ]
}
```

### F3. Output for that input

```text
A — public company — gross, in EUR
Year   Cash exp  Equity exp  Total exp  Total bear
   1    114,000      30,750    144,750     133,500
   2    101,970      37,263    139,233     120,535
   3    105,029      44,646    149,675     122,815
   4    108,180      53,032    161,212     125,311
 Sum                           594,869     502,161

B — Series B startup — gross, in EUR
Year   Cash exp  Equity exp  Total exp  Total bear
   1    100,000      22,500    122,500     100,000
   2    104,000      26,250    130,250     104,000
   3    108,160      31,875    140,035     108,160
   4    112,486      40,313    152,799     112,486
 Sum                           545,584     424,646
```

What it shows: B's **cash** is higher every year from year 2, but A wins on **expected** total by ~€49k and on **bear case** by ~€78k over four years — B's equity carries a 60% chance of zero in this model. Change the startup's scenarios (Exercise 15) to see how optimistic you'd need to be for B to win; that threshold is the real question for the decision.

### F4. Weighted decision matrix

```text
CRITERION (weights set at D2, before offers)   WEIGHT   OPTION A score/evidence        OPTION B score/evidence
Problem and scope fit                            20      _ / ______________________     _ / ______________________
Learning and growth                              15      _ / ______________________     _ / ______________________
Team and manager                                 15      _ / ______________________     _ / ______________________
Compensation (expected + bear, F1)               20      _ / ______________________     _ / ______________________
Stability (Concept 7 score)                      10      _ / ______________________     _ / ______________________
Work model and time zones                        10      _ / ______________________     _ / ______________________
Career optionality (3–5 years)                   10      _ / ______________________     _ / ______________________
WEIGHTED TOTAL (Σ weight × score / 100)                  ____                            ____
GUT CHECK: does the result match intuition? If not, which criterion or weight is missing/wrong? ________
PREMORTEM (5 reasons it fails) → evidence or open question: ____________________________________
REVERSIBILITY: what makes this hard to undo? ________   DECISION + REVISIT CONDITION: ____________
```

---
# Appendix G — Loop observation checklist

```text
ORGANISATION   □ schedule sent in advance  □ loop matched recruiter description  □ interviewers knew their round
HONESTY        □ straight answer on why the role is open  □ straight answer on on-call  □ consistent across interviewers
PREPARATION    □ interviewers read my CV  □ follow-ups built on my answers
RESPECT        □ started on time  □ time for my questions  □ next steps explained
CANDOUR        □ at least one interviewer named a real problem  □ triangulation question answered consistently
DISAGREEMENT   □ design pushback was collaborative, engaged with my reasoning
CALIBRATION    loop type: centralised | team-dependent  → weight team observations accordingly
NOTES          green: ______________________   red: ______________________
```

---
# Appendix H — Stage checklists

### H1. D1 — pursue? (15–45 minutes)
```text
□ Location and hiring entity possible for me          □ Work model and time zones acceptable
□ Business model in one sentence; where the team sits □ Stability score ≥ 5 (or a specific reason)
□ Level plausibly right (verbs, not title)            □ Range plausibly acceptable (posted or aggregated)
□ Two or three culture themes from recent, role-matched sources
□ Posting looks live (named contact, not endlessly re-posted)
→ GO / NO-GO + open questions for the recruiter
```

### H2. D2 — prepare (2–6 hours)
```text
□ Loop map complete; every cell graded            □ AI and tooling rules per round, in writing
□ Level number and expectations known             □ Vocabulary → dimensions translation table
□ Stories retagged; coverage checked              □ Mock schedule re-weighted by round counts
□ Company-shaped design problem rehearsed         □ Domain round prepared (fundamentals + 2–3 projects)
□ "Why us / this role / now" drafted and spoken   □ Question bank per interviewer; triangulation question chosen
□ Per-round cards written                         □ Decision criteria and weights written (for D3)
```

### H3. D3 — accept? (as long as it takes)
```text
□ Peer conversation held (Westrum, on-call, "what surprised you?")   □ Recent leaver perspective, if possible
□ Financial health re-checked                                        □ Offer components all known, in writing
□ Offer modelled year by year, expected and bear (Appendix F)        □ Net and FX compared where relevant
□ Terms reviewed: entity, notice, non-compete, IP, probation, on-call pay, equity eligibility
□ Level confirmed; path to next level discussed                      □ Decision matrix scored with evidence
□ Premortem done; unchecked risks asked about                        □ Reversibility factors weighted
□ Decision record written with a revisit condition                   □ Other processes closed graciously
□ Calibration log updated (mock scores vs real outcomes; predictions vs reality after joining)
```

---
# Closing the curriculum

This is the last of the 39 modules. The throughline from the very first page still holds: past mid-level, interviews grade **judgement** — how you handle ambiguity, whether you back trade-offs with evidence, whether your reasoning survives challenge. This final module applied the same standard to the choice of where to use that judgement: questions before sources, evidence graded by reliability, decisions recorded with the conditions that would reopen them.

A practical way to use the whole curriculum from here:

1. **Pick the target loop** and build its dossier (this module).
2. **Let the loop map choose the modules to revisit** — the round types and the domain round point straight at Phases 3–7.
3. **Run Module 38's programme** against that loop, with its AI modes and its vocabulary.
4. **After each real loop, log the outcome** against your mock scores and your research predictions — and adjust.

The curriculum is designed to be navigated at variable depth and out of order. Go back to any module whenever a loop map points at it.
