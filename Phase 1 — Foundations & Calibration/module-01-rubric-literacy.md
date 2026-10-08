# Module 1 — What Top Companies Actually Score (Rubric Literacy)
*Phase 1: Foundations & Calibration · Senior/Architect Interview Prep for .NET & C#*

> **State of the world verified on October 8, 2026.** The *principles* in this module are stable — they rest on decades of selection research and on how large engineering organisations have run hiring for fifteen-plus years. What moves is each company's process: round mix, AI policy, in-person requirements, committee mechanics. The facts that calibrate the module:
>
> - **Structured interviewing is the industry's foundation, and the research still backs it.** Sackett, Zhang, Berry and Lievens (2022) re-ran the classic selection meta-analyses with corrected range-restriction adjustments: **structured interviews came out as the strongest single predictor of job performance (operational validity ≈ .42)**, unstructured interviews far behind (≈ .19). That is the reason almost every company you'll interview with uses the same questions per role, a rubric, and independent written scoring.
> - **Microsoft states it outright.** Its official *How we hire* page says hiring uses **structured interviews and a consistent framework to evaluate real skills against the role's requirements**, and that it looks for **respect, integrity, accountability and growth mindset**. It describes most interviews as **2–4 conversations of up to an hour** (engineering loops commonly run 4–5), and its candidate code of conduct encourages **AI for preparation** but requires candidates to demonstrate their own skills **without outside assistance unless explicitly permitted**. Process remains **team-dependent**; a final senior interviewer (the "as appropriate", AA, interview) is widely reported but not universal.
> - **Amazon's 16 Leadership Principles** (two added in 2021: *Strive to be Earth's Best Employer* and *Success and Scale Bring Broad Responsibility*) are scored in every round. Amazon's official SDE III prep page describes a **60-minute phone screen (half Leadership Principles, half coding and system design)** and a loop of **five 55-minute interviews** where **different interviewers are assigned to evaluate different Leadership Principles**; coding must be **syntactically correct — no pseudocode** — and is judged on being **scalable, robust and well-tested**. A **Bar Raiser** from outside the hiring team takes part and, with the hiring manager, effectively holds a veto (interviewing.io reports a 5-point "Strongly Inclined … Strongly Not Inclined" scale and a live debrief).
> - **Google** still decides through a **hiring committee** of people who did not interview you, reading a written **packet**; the committee also sets the **level**, and **team matching happens after** committee approval — so approval isn't yet an offer. interviewing.io reports a **seven-point** interviewer scale (Strong No Hire … Strong Hire). Google historically described four attributes — **general cognitive ability, role-related knowledge, leadership and "Googleyness"**. In 2025 Sundar Pichai said Google would reintroduce **at least one in-person round** for engineering candidates, citing AI-assisted cheating.
> - **Meta** scores design rounds on four named focus areas — **problem navigation, solution design, technical excellence, technical communication** — and, per interviewing.io's interviewer-sourced guide, coding rounds largely decide **hire/no-hire** while **design and behavioral rounds largely decide the level**. Since late 2025 Meta has been reported to replace one traditional coding round with a **60-minute AI-enabled coding round** (E7+ reportedly get only the AI-enabled one).
> - **The generic rubrics converge.** Coding rounds across large companies score some version of **communication, problem solving, technical competency and testing** (Tech Interview Handbook's synthesis); system design rounds score **problem navigation, solution design, technical excellence, and communication/collaboration** (Hello Interview's synthesis from Meta and Amazon interviewers). At senior level, system design **carries disproportionate weight**.
>
> Specific numbers and round mixes will drift; the shape of the measurement won't.

## Orientation

Here is the sentence to carry through the whole module: **an interview loop is a measurement pipeline — each round samples a few predefined dimensions, the interviewer turns what you said into written evidence, and a decision body reads that evidence against a level-specific bar and returns two answers: hire or not, and at what level. You are not trying to be impressive; you are trying to produce legible, level-appropriate evidence on every dimension each round exists to measure.**

The curriculum entry reads: *What top companies actually score (rubric literacy).* It is Module 1 because everything that follows — the design method (Modules 3–5), the distributed-systems theory (6–13), the .NET depth (14–19), the architecture patterns (20–25), the architect track (30–33), behavioral (34–35) — is *content*. This module is about the **scoring function** that content is fed into. Knowing the scoring function changes how you deploy what you know.

**Why this module exists.** Three facts make the case:

1. **Most senior candidates who fail know enough.** They fail because the evidence they produced didn't map onto the rubric: they designed silently, never stated a trade-off, said "we" through every behavioral story, or spent 20 minutes on parts nobody scores. The knowledge was there; the *signal* wasn't.
2. **The person who decides usually never met you.** At Google, a committee reads a packet. At Meta, a committee and then directors review written feedback. At Amazon, a debrief discusses written notes. Even at team-driven companies like Microsoft, the hiring manager reads what interviewers wrote. **If it isn't in the write-up, it didn't happen.** You are, in effect, dictating the notes.
3. **There are two decisions, not one.** Passing at the wrong level is a common senior outcome — the "down-level". Rubric literacy is how you aim for the level you want rather than the one you happen to demonstrate.

How this connects to the rest of the curriculum:

- **Module 2** (Senior IC vs Architect) — takes this module's scoring model and shows how the *shape* of the loop and the weight of each dimension change between the IC and architect tracks.
- **Modules 3–5** (design method, requirements, estimation) — a method designed to produce evidence on exactly the system-design dimensions defined here.
- **Modules 34–35** (STAR calibrated to seniority, story bank) — behavioral answers built to produce the ownership, scope and judgement evidence this module defines.
- **Module 36** (coding rounds) — the coding rubric here, applied to C#.
- **Module 38** (mock interviews and self-scoring) — turns this module's dimensions into behaviourally anchored rubrics you can score yourself against; it uses the same six universal dimensions (D1–D6) introduced in Concept 20.
- **Module 39** (company/role research) — the per-company dialect work in Part D, made into a checklist.

Why it matters in interviews:

1. **It tells you where to spend minutes.** A 45-minute round has a fixed budget. Rubric literacy says which minutes buy score.
2. **It tells you what to say out loud.** Many strong behaviours are invisible unless narrated. Rubric literacy tells you which ones.
3. **It protects you from the round you're bad at.** Committees react to weaknesses more than to strengths; knowing the knockouts is worth more than polishing a strength.
4. **It is itself an interview topic.** Staff and architect candidates are asked how they'd interview, calibrate or design a loop. Knowing how hiring decisions are made makes those answers concrete.

This module has five jobs:

1. **Derive why rubrics exist** — hiring as a measurement problem under asymmetric costs, and why the industry converged on structured interviews (Part A).
2. **Show the decision pipeline** — how a loop becomes a decision and a level, under the three main decision architectures (Part B).
3. **Define what each round type scores** — coding, system design, behavioral, deep technical, architect and AI-enabled rounds (Part C).
4. **Translate the company dialects** — Google, Amazon, Meta, Microsoft and everyone else, mapped onto six universal dimensions (Part D).
5. **Turn rubric literacy into behaviour** — producing writeable evidence, level signals, knockouts, and reverse-engineering an unknown rubric (Part E).

Seven framings to carry through:

1. **The interviewer is a reporter.** Your job is to make their notes easy to write and hard to dispute.
2. **Dimensions, not impressions.** Each round scores a small, fixed set of things. Know them before you walk in.
3. **The bar is level-specific.** The same answer can be a "strong hire" at L4 and a "no hire" at L6. Aim at the level, not at "good".
4. **The weakest round matters most.** Committees discount vague strengths and amplify documented weaknesses.
5. **Narrate the signal.** Trade-offs, assumptions, failure modes and self-corrections only score if said aloud.
6. **Two decisions: hire and level.** Some rounds mostly decide one, some mostly the other.
7. **Dialects differ; the grammar doesn't.** Google, Amazon, Meta and Microsoft use different words for very similar dimensions.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Hiring as a measurement problem | Noisy instrument, asymmetric costs → companies optimise for precision |
| 2 | Why structured interviews won | Same questions, rubric, independent scoring — the best-validated method |
| 3 | Signal, evidence and the write-up | What isn't written doesn't exist at decision time |
| 4 | The rating scale | Even scales, hire/no-hire bands, negatives weigh more |
| 5 | Anatomy of a loop | Screen → loop → debrief/committee → level → team match → offer |
| 6 | Three decision architectures | Central committee · Bar Raiser debrief · hiring-manager-led |
| 7 | Leveling, the second decision | Level bands across companies; down-levels and how they happen |
| 8 | How a packet is really read | Weakest link, evidence quality, variance, knockouts |
| 9 | What coding rounds score | Communication, problem solving, technical competency, testing — plus production quality at senior |
| 10 | What system design rounds score | Navigation, solution design, technical excellence, communication — depth decides level |
| 11 | What behavioral rounds score | Competencies via past behaviour: ownership, scope, judgement, results |
| 12 | What deep technical rounds score | Role-related knowledge: precision, depth on demand, debugging reasoning |
| 13 | What architect rounds score | Decision quality, stakeholders, migration risk, cost, documentation |
| 14 | What AI-enabled rounds add | Decomposition, verification, critical reading, ownership |
| 15 | Google's dialect | GCA · RRK · Leadership · Googleyness; committee; team match |
| 16 | Amazon's dialect | 16 Leadership Principles; Bar Raiser; STAR with data |
| 17 | Meta's dialect | Four design focus areas; coding decides hire, design decides level |
| 18 | Microsoft's dialect | Team-dependent; values; growth mindset; AA interview |
| 19 | Everyone else | Mid-size, enterprise, consultancy, startup — practical and team-driven |
| 20 | The six universal dimensions | D1 framing · D2 depth · D3 judgement · D4 driving · D5 collaboration · D6 level |
| 21 | Producing writeable evidence | Quotable sentences: decision, reason, condition |
| 22 | Level signals | What moves a packet from senior to staff/architect |
| 23 | Red flags and knockouts | The behaviours that sink a packet regardless of strengths |
| 24 | Reverse-engineering an unknown rubric | Job description, recruiter questions, values pages → a predicted rubric |

---
# Part A — Why rubrics exist

## Concept 1 — Hiring as a measurement problem

Start from first principles, the way you would with any system you're asked to design. A company wants to predict one quantity — *how well will this person perform in this role over the next few years?* — and it has a few hours of conversation to do it. Treat the loop as a classifier.

### The confusion matrix of hiring

| | Candidate would actually succeed | Candidate would actually struggle |
|---|---|---|
| **Hired** | True positive — good hire | **False positive — bad hire** |
| **Rejected** | **False negative — missed good engineer** | True negative |

The two errors are not equally expensive to the company:

- **A false positive** costs salary and equity for months before the problem is recognised, a manager's time on performance management, the team's morale and velocity, possibly production incidents and a bad design that outlives the hire, then a separation and a backfill. For a senior or staff hire, whose decisions shape systems, the cost compounds.
- **A false negative** costs the difference between you and the next-best candidate — usually small for a company with a deep pipeline — plus some recruiting effort. Most large companies have many more qualified applicants than openings.

So the loss function is asymmetric, and the rational response is to **optimise for precision at the expense of recall**: accept rejecting some good engineers in order to rarely hire a bad one. Everything else in this module follows from that one fact:

- Bars are set high and **ties go to "no"**.
- **A documented weakness outweighs an undocumented strength** (Concept 8).
- Decision bodies are designed to **resist pressure to fill a seat** (Bar Raisers, committees of non-interviewers — Concept 6).
- Candidates who are probably good but produced **ambiguous evidence** are usually rejected — which is the most avoidable failure mode, and the main reason this module exists.

### The instrument is noisy

Now add measurement error. A 45-minute round samples a tiny slice of your ability: one problem, one interviewer, one day. interviewing.io's data on thousands of interviews shows the **same engineer's performance varies a lot from one interview to the next**. Interviewers differ in harshness. Problems differ in fit.

A noisy classifier with a high threshold has a predictable property: **candidates near the bar pass or fail almost at random**, and the only way to move your odds is to move your *true score* well above the threshold — or to reduce the noise in how your ability is measured. You can do the second more cheaply than you think: by making your evidence **clear, explicit and aligned with what the rubric measures**, you reduce the interviewer's uncertainty about you. That's what "rubric literacy" means operationally.

### Base rates and why loops have several rounds

If one round were a perfect test, one round would do. It isn't, so companies **average over several independent samples** — 4 to 6 rounds, different interviewers, different problems, different dimensions. Two consequences:

1. Rounds are **designed to measure different things** (coding, design, behavioral, domain), so a strength in one can't simply cover a gap in another.
2. Independence is protected deliberately: interviewers usually **write feedback before seeing each other's**, and committees read the written record rather than a hallway consensus.

**Interview-grade sentence** *(if asked how you'd think about hiring, at staff/architect level)*: *"Hiring is a noisy classification problem with asymmetric costs — a bad senior hire costs far more than a missed one when the pipeline is deep — so good loops optimise for precision, take several independent samples on different dimensions, and decide from written evidence rather than impressions."*

---

## Concept 2 — Why structured interviews won

If you were designing the measurement, you'd want it to be **valid** (measures job performance) and **reliable** (gives the same answer for the same candidate regardless of who's interviewing). The selection-research literature tested both for decades. The consistent finding: **structure helps both**.

### What "structured" means

A **structured interview** has four properties, all of which you'll recognise in big-tech loops:

| Property | What it means | What you see as a candidate |
|---|---|---|
| **Job analysis first** | Questions derive from what the role actually requires | Senior loops add design and leadership; staff loops add more design and scope |
| **Same questions per role** | Candidates for the same role get comparable questions (from a bank, or of the same type and difficulty) | Recognisable problem families: "design a rate limiter", "tell me about a conflict" |
| **A rubric with defined levels** | Each rating level is described in advance, ideally by observable behaviours | Interviewers write evidence against named dimensions |
| **Independent scoring** | Each interviewer scores before discussing | Feedback written within hours, before debrief or committee |

Google's re:Work guide describes the same thing in practical terms: the same questions for candidates for a role, scored on a consistent scale with defined standards. The US Office of Personnel Management's structured-interview guidance is the same recipe from the public sector.

### The evidence

- **Validity.** Schmidt and Hunter's 1998 review put structured interviews among the best predictors of job performance. Sackett and colleagues' 2022 re-analysis — correcting an over-adjustment for range restriction in the older work — **placed structured interviews first** among common selection methods (≈ .42), with unstructured interviews well behind (≈ .19).
- **Unstructured interviews persist anyway.** Dana, Dawes and Peterson (2013) showed interviewers *feel* they learn a lot from free-form interviews even when those interviews add nothing — or reduce accuracy — over other information. That's why companies **impose** structure: the people interviewing would drift away from it otherwise.
- **Holistic judgement is weaker than scoring dimensions first.** Kahneman's "mediating assessments" — scoring several predefined attributes independently, then forming an overall judgement — consistently beats one global impression. Big-tech feedback forms are built this way: dimensions first, recommendation last.

### What this means for you

Three practical consequences:

1. **There is a rubric, even if nobody shows it to you.** Your interviewer has a feedback form with named dimensions and a scale. The exact text is internal, but the dimensions are predictable (Part C).
2. **Questions are a means, not the end.** A rate-limiter problem is a vehicle to observe problem navigation, trade-off reasoning and depth. The "answer" scores far less than the *behaviours* the problem lets you display.
3. **Interviewers are constrained.** In well-run loops, interviewers can't reward you for being likeable or penalise you for an unconventional (but sound) answer without evidence. Use that: give them evidence.

**Interview-grade sentence:** *"Structured interviewing — questions derived from the job, comparable across candidates, scored on a defined rubric independently by each interviewer — is the best-validated selection method we have, so I assume every round I'm in is measuring a small set of named dimensions, and I make sure my answer produces evidence on each of them."*

---

## Concept 3 — Signal, evidence and the write-up

### The interviewer's job, from their side of the table

Picture the interviewer an hour after your round. They open a feedback form. It typically has:

- **A summary of the question** and how far you got.
- **One section per dimension** (named by that company's rubric), each with free text and often a rating.
- **An overall recommendation** on a fixed scale.
- Sometimes **a level recommendation** ("senior", "consider for staff", "consider down-level").
- At some companies, **verbatim evidence**: quotes, code, the diagram.

They have 20–40 minutes to write it, and other work waiting. Whatever you said that is **easy to quote, clearly tied to a dimension, and unambiguous** makes it into the form. Everything else becomes a vague sentence — or disappears.

### Signal versus evidence

- **Signal** is what the interviewer *perceives* in the moment: "this person thinks clearly about failure modes".
- **Evidence** is what they can *write down and defend*: "Unprompted, at ~25 min, identified that a worker crash after the provider accepts the SMS creates a duplicate; chose idempotency keyed on the event id and accepted at-least-once with user-visible de-duplication; explained why exactly-once isn't achievable across the provider boundary."

Decision bodies act on evidence. An interviewer who writes "seemed strong on distributed systems" will be asked by a committee "what specifically?" — and a vague positive is routinely **discounted**, while a specific negative ("could not explain what happens to in-flight messages on a lock timeout") is **taken at face value**.

### What makes evidence strong

| Strong evidence | Weak evidence |
|---|---|
| A **decision** with a **reason** and a **condition** ("Postgres, because we need multi-row transactions; I'd revisit if write QPS exceeds what one primary can take, ~10k/s here") | A list of options with no choice |
| A **number** that **changed the design** ("~2,300 writes/s at peak — one Redis primary handles that, so no sharding yet") | A number with no consequence |
| A **failure mode** named **unprompted**, with a policy | Failure handling only when asked, or "we retry" |
| A **self-correction** stated explicitly ("I said 301 earlier — I'd change that to 302 because we need every click for analytics") | Silently changing the diagram |
| A **personal action** in a story ("I wrote the migration plan and ran the dual-write for three weeks") | "We decided…", "the team built…" |
| A **quantified result** ("p99 from 1.8 s to 240 ms; on-call pages dropped from ~20 to 3 a month") | "It went well" |

### "Narrate the signal"

Many senior behaviours are **invisible unless spoken**: choosing what not to build, considering and rejecting an option, checking an assumption, noticing a risk. The interviewer can't score what happens in your head. So the core skill of this whole curriculum, applied in every round, is to **say the reasoning out loud in a form that's easy to write down**. Concept 21 turns that into a phrasebook.

**Interview-grade sentence:** *"Decisions are made from what interviewers write down, not from what they felt in the room — so I make my reasoning explicit and quotable: the decision, the reason, the number behind it and the condition under which I'd change my mind."*

---

## Concept 4 — The rating scale

### The common shapes

Most companies' interviewer recommendations fall into one of three shapes. Exact labels are internal and change; the shapes are stable and widely reported:

| Shape | Typical labels | Reported at (examples) |
|---|---|---|
| **4-point, no middle** | Strong No Hire · No Hire (Lean No) · Hire (Lean Hire) · Strong Hire | Common across the industry; Tech Interview Handbook's synthesis uses it; *How Google Works* describes an overall 1–4 score |
| **5-point with a neutral** | Strongly Not Inclined · Not Inclined · Neutral · Inclined · Strongly Inclined | Amazon, per interviewing.io's interviewer-sourced guide |
| **7-point** | Strong No Hire · No Hire · Leaning No Hire · On the Fence · Leaning Hire · Hire · Strong Hire | Google, per interviewing.io's interviewer-sourced guide |
| **Binary + confidence / level note** | Hire / No Hire, plus confidence or "consider at level X" | Meta coding and design rounds, per interviewing.io's guide |

### Why even scales and forced choices

A neutral option is where uncertain interviewers hide. Forcing a direction ("lean hire" vs "lean no hire") makes every interviewer commit, which makes their scores informative. Where a neutral exists, it is usually treated as **not enough to hire** — remember the precision bias (Concept 1).

### The bands that matter

From a candidate's perspective, collapse any scale to three bands:

```text
CLEAR YES     Strong Hire / Hire / Inclined+      → the round supports the packet
WEAK          Lean Hire / On the Fence / Neutral   → the round doesn't hurt much, but doesn't help either
CLEAR NO      Lean No and below                     → the round is a liability the packet must overcome
```

A packet with **no clear-no rounds and mostly clear-yes rounds** is a hire. A packet with **one well-documented strong no** is, at most companies, very hard to rescue — and at some (Amazon's Bar Raiser or hiring manager, Meta's behavioral at E6+ per interviewing.io) it is effectively fatal.

### Asymmetric weighting

Because of the precision bias:

- A **"strong no" counts for more** than a "strong hire" counts for. Several reports (and committee-member accounts) agree: **one well-documented No Hire can outweigh several vague Hires**.
- A **"strong hire" is rarely needed** to pass — consistent "hire" across rounds is the typical successful packet. Strong Hire votes mostly matter for **level** and for **rescuing** a packet with one weak round.
- **Low-confidence negatives** are discounted. Meta's reported confidence score exists precisely so a shaky "no" from an uncertain interviewer weighs less.

**Interview-grade sentence:** *"Interview scales are built to force a direction, usually without a neutral middle, and decision bodies weigh a well-documented 'no' more heavily than a vague 'yes' — so a consistent run of clear hires beats a mix of brilliant and weak rounds."*

---
# Part B — The decision pipeline

## Concept 5 — Anatomy of a loop

Every large company's process is a variant of the same pipeline. Know each stage's purpose, because each stage decides something different.

```text
1. Application / referral / sourcing   → "Is this profile plausibly at the target level?"
2. Recruiter screen (20–30 min)        → logistics, level, compensation band, motivation, red flags
3. Technical screen(s) (45–60 min)     → a cheap filter: can they code (and sometimes design) at the bar?
      (sometimes an online assessment before or instead)
4. The loop / onsite (4–6 rounds)      → the measurement: independent samples on several dimensions
5. Debrief and/or committee review     → hire / no-hire from the written record
6. Leveling decision                   → at which level? (often the same body, sometimes a later review)
7. Team match (centralised companies)  → which team? (can take weeks; can fail)
8. Offer, executive/comp approval      → numbers within the level's band
```

Three things to notice:

- **Screens are a filter, not a measurement.** They're scored on a pass/fail basis and usually by one interviewer. A borderline screen may lead to a second screen rather than a rejection.
- **The loop's composition encodes the rubric.** A senior SWE loop at a large company typically includes **1–3 coding**, **1–2 system design**, **1 behavioral** (often called something like "leadership", "Googleyness & leadership", "behavioral", or distributed across rounds as at Amazon), and sometimes a **domain/deep-technical** round. Staff loops shift weight from coding to design and behavioral; architect loops add document-review and stakeholder rounds (Module 2).
- **Hire, level and team are separable decisions.** At Google and Meta they are explicitly separated (committee → team match). At team-driven companies (Microsoft, Apple, Netflix, most enterprises) they collapse into one hiring-manager decision — which is why those loops include your future colleagues.

### Typical senior loop shapes (reported, 2025–2026)

| Company | Senior/staff loop shape (typical, varies) |
|---|---|
| Google | 4–5 rounds: coding ×2–3, system design ×1 (more at L6+), Googleyness & leadership ×1; hiring committee; team match |
| Amazon (SDE III) | Phone screen 60 min (half LPs, half coding/design); loop of five 55-min interviews mixing coding, system design and LPs; Bar Raiser; live debrief |
| Meta (E5/E6) | Coding ×2 (one now AI-enabled for many candidates), system or product design ×1–2, behavioral ×1; committee; team match |
| Microsoft | Team-dependent; 4–5 one-hour interviews mixing coding, design and behavioral, often with the hiring manager and sometimes a final senior "AA" interviewer |

**Interview-grade sentence:** *"A loop is a pipeline of separate decisions — screens filter, the onsite measures, a debrief or committee decides hire, a leveling step decides scope, and centralised companies match a team afterwards — and the mix of rounds tells you what the company's rubric weights for the level."*

---

## Concept 6 — Three decision architectures

How the written evidence turns into a decision varies, and the variation changes what you should optimise for. There are three main architectures.

### 1. Central committee (Google, Meta — and many companies that copied them)

```text
Interviewers ──write──► Packet (resume, notes, scores, code, referrals) ──► Committee of non-interviewers
                                                                                │
                                                                     hire / no-hire / more data
                                                                     + level
                                                                                │
                                                                       Team match ──► Offer
```

- **Who decides:** a committee of engineers and managers **who did not interview you**, reading the packet. At Google they reportedly seek **consensus**; at Meta, interviewing.io and a former Meta hiring-committee chair describe a committee decision followed by a review by engineering directors for hire and level.
- **What it optimises:** consistency across a huge, team-agnostic pipeline; resistance to a hiring manager's urgency.
- **Implications for you:** you are **"interviewing with a machine"** (interviewing.io's phrase). Charm doesn't transmit; **written evidence does**. Every round counts roughly equally; a single strong round rarely carries a weak one. Because the process is centralised and standardised, it's **predictable** — the rubrics are the most knowable in the industry.

### 2. Bar Raiser and live debrief (Amazon)

```text
Pre-brief (assign LPs to interviewers) ──► Loop ──► Written feedback ──► Live debrief
                                                                         (hiring manager + Bar Raiser + interviewers)
                                                                                │
                                                                    hire / no-hire (HM or BR can effectively veto)
```

- **Who decides:** the interviewers in a **live debrief**, led by a **Bar Raiser** — a specially trained interviewer from **outside the hiring team**, whose job is to protect the company-wide bar against local hiring pressure. Per interviewing.io's guide, the Bar Raiser and the hiring manager can each effectively veto, and interviewers tend to follow the Bar Raiser's view.
- **What it optimises:** a high, company-wide bar while letting teams run their own loops.
- **Implications for you:** **Leadership Principles are not a side topic**; a poor LP showing is close to an automatic no, while strong LPs can make a borderline technical result negotiable. Every interviewer is probing specific LPs, so you need **many distinct, data-rich stories** (Module 35). The Bar Raiser's round is often the most probing.

### 3. Hiring-manager-led (Microsoft, Apple, Netflix, most enterprises and mid-size companies)

```text
Loop with future colleagues (+ sometimes a final senior interviewer) ──► Feedback ──► Hiring manager decides
                                                                                     (with recruiter, sometimes a senior leader)
```

- **Who decides:** the **hiring manager**, informed by interviewer recommendations. At Microsoft, a final senior interviewer ("as appropriate", AA) is widely reported for many loops; candidates report the AA reads earlier feedback first. Processes and even scales vary by team (interviewing.io: Microsoft interviewers grade on different scales depending on the team).
- **What it optimises:** fit to a specific team's needs; speed.
- **Implications for you:** the loop is **less standardised** and more variable ("chaotic", in interviewing.io's scoring — Apple and Netflix highest, then Microsoft). Questions often reflect the **team's actual domain**, and practical/real-world rounds are more common. The upside: you're interviewing with humans who'll work with you, so **relevant experience and collaboration signal travel further**. At many such companies you can also interview with **several teams concurrently** — no single gate.

### Summary

| | Central committee | Bar Raiser debrief | Hiring-manager-led |
|---|---|---|---|
| Decides | Non-interviewers, from packet | Interviewers live, BR as guardian | Hiring manager |
| Standardisation | High | Medium | Low–medium, team-dependent |
| Biggest lever for you | Written, specific evidence in every round | LP stories with data; no weak LP round | Domain fit, practical depth, team rapport |
| Biggest risk | One documented weak round; down-level | A weak LP round; Bar Raiser's doubt | Unpredictable format; one sceptical senior voice |
| Retry | Usually a cooling-off period (months) | Other teams may re-interview sooner | Often other teams immediately |

**Interview-grade sentence** *(for staff "how would you set up hiring?" questions)*: *"There are three main decision architectures — a central committee of non-interviewers reading a packet, a live debrief guarded by an independent bar raiser, and a hiring-manager decision informed by the panel — and they trade consistency against team fit and speed; whichever you choose, the decision should rest on independently written evidence against a shared rubric."*

---

## Concept 7 — Leveling, the second decision

### Why leveling is separate

A "hire" decision answers *"should this person be here?"*. A leveling decision answers *"what scope will we trust them with, and what will we pay?"* — and it's the decision that most often surprises experienced candidates. **The down-level** — "we'd like to offer you the role at one level below" — is a routine outcome, and in centralised processes the committee can make it without re-interviewing.

### What a level means

Levels are defined in companies' career ladders by **scope** (how large a problem you own), **ambiguity** (how well-defined the work is when it reaches you), **influence** (how far beyond your team your decisions reach) and **time horizon**. Public career frameworks — Dropbox's, for example, organises expectations by scope, collaborative reach and levers of impact — make the progression explicit:

| | Senior | Staff | Senior staff / Principal |
|---|---|---|---|
| Scope | A system or major feature; a team's technical direction | Several systems or teams; a technical area | An org's architecture; company-wide problems |
| Ambiguity | Given a problem, defines the solution | Given an area, defines the problems | Given a business goal, defines the strategy |
| Influence | Own team, adjacent teams | Across teams, without authority | Across the org; with leadership |
| Time horizon | Quarters | A year or more | Multi-year |

### Level names across companies (approximate equivalents, widely reported)

| Band | Google | Meta | Amazon | Microsoft |
|---|---|---|---|---|
| Mid | L4 (SWE III) | E4 | L5 (SDE II) | 61–62 (SDE II) |
| **Senior** | **L5** | **E5** | **L6 (SDE III)** | **63–64** |
| **Staff** | **L6** | **E6** | L7 (Principal) begins here at Amazon | **65–67 (Principal)** |
| Senior staff / Principal | L7 | E7 | L7–L8 | 67+ / Partner 68+ |

Mappings are imperfect — Amazon's L7 Principal is generally considered a larger jump than Google's L6, and Microsoft's 65 is often compared with Google's L6. Treat the table as orientation; levels.fyi publishes crowdsourced mappings.

### How the level gets decided

Across companies, the pattern is consistent:

- **Coding rounds mostly decide hire/no-hire.** A strong coding performance is necessary at most levels but says little about *which* senior level you are.
- **System design rounds mostly decide level.** interviewing.io's Meta guide states it directly — design interviewers are asked whether you should be considered at a different level, and staff candidates must pass both design rounds. Hello Interview notes that at senior level design **carries a disproportionate weight**.
- **Behavioral rounds decide level too**, especially at staff+: the scope of your stories (one team vs several, a feature vs an architecture, a decision vs a strategy) is the main evidence of operating level. A former Meta hiring-committee member describes going **first to the behavioral interview** when reading a senior packet.
- **The candidate's history is a prior.** Your current title and scope set the expectation; the loop confirms or revises it.

### Down-level mechanics

A down-level happens when the evidence supports hiring but not at the target scope. Typical triggers:

| Trigger | What the write-up says |
|---|---|
| Design was correct but shallow | "Covered the basics, needed prompting for deep dives; senior-level design, not staff" |
| Design was driven by the interviewer | "Candidate needed significant steering; would want more independence at this level" |
| Stories at too small a scope | "Examples were team-level features; no evidence of cross-team influence" |
| "We" without "I" | "Unclear what candidate personally did versus the team" |
| Strong coding, average everything else | "Clear hire at L4; not enough design/leadership signal for L5" |

The defence is to **produce level-appropriate evidence on purpose** (Concept 22): drive the design, go deep unprompted, size your stories to the target level, and say what *you* decided.

**Interview-grade sentence:** *"Leveling is a separate decision from hiring: coding rounds mostly establish whether you clear the bar, while system design and behavioral rounds establish scope — so for a senior or staff target I treat design depth and the scope of my stories as the level evidence, not as nice-to-haves."*

---

## Concept 8 — How a packet is really read

A committee or debrief doesn't add up scores. It reads a **profile** and asks whether the evidence supports a confident decision at the level. Five rules describe how that reading works in practice.

### Rule 1 — The weakest documented round dominates

Because of the precision bias (Concept 1), a clear, well-evidenced "no" in any round raises a question the rest of the packet must answer. A packet of "hire, hire, hire, strong no" is not a 2.5 average; it is "what happened in round four, and does it reveal something the others missed?". Sometimes the answer is "a bad interviewer or bad problem" and the packet passes — but you need the other rounds to be **specific** enough to make that argument.

### Rule 2 — Evidence quality multiplies the score

A "strong hire" backed by quotes, decisions and numbers carries weight. A "strong hire" backed by "great candidate, very sharp" is discounted. Reported committee behaviour (Google in particular) is that **vague positives can be overridden; specific negatives rarely are**.

### Rule 3 — Consistency across rounds is itself evidence

Two rounds that independently observe the same strength (or weakness) are far more convincing than one. If your coding interviewer *and* your design interviewer both note "drove the conversation, stated trade-offs without prompting", that's a pattern. If only one does, it's an anecdote. Committees also notice **variance** — a candidate who is brilliant in one round and weak in another is a risk.

### Rule 4 — Knockouts override averages

Some observations end the discussion regardless of other rounds: dishonesty (including undisclosed AI use where it's forbidden), contempt for colleagues or users, taking credit for others' work, refusing to engage with feedback, a design that silently violates a stated hard requirement with no awareness (Concept 23).

### Rule 5 — Level is read from the top of the evidence, hire from the bottom

To decide **whether** to hire, the reader looks at the weakest round. To decide **at which level**, they look at what the strongest, most level-specific rounds (design, behavioral) demonstrated — and whether the rest is consistent with it.

### The packet as code (a mental model, not anyone's algorithm)

For an engineer, a small model makes the shape concrete. This is **not** how any company literally decides — humans read packets — but it captures the five rules better than an average does:

```csharp
// A mental model of packet reading. Not any company's algorithm.
public enum Rec { StrongNoHire = 1, NoHire = 2, Hire = 3, StrongHire = 4 }
public enum Level { Mid, Senior, Staff }

public sealed record RoundFeedback(
    string Round,                 // "coding-1", "design", "behavioral" ...
    bool DecidesLevel,            // design and behavioral rounds
    Rec Recommendation,
    int SpecificEvidence,         // quotes, decisions, numbers the write-up cites
    Level ObservedLevel,          // the level the interviewer saw evidence for
    bool Knockout = false);

public sealed record Decision(bool Hire, Level? Level, string Reason);

public static class PacketReader
{
    public static Decision Read(IReadOnlyList<RoundFeedback> packet, Level target)
    {
        if (packet.Any(r => r.Knockout))
            return new(false, null, "Knockout observed");                                  // Rule 4

        // Rule 2: vague feedback carries less weight than documented feedback.
        static double Weight(RoundFeedback r) => r.SpecificEvidence >= 3 ? 1.0 : 0.5;

        // Rule 1: the weakest well-documented round dominates the hire decision.
        var documentedNos = packet.Where(r => r.Recommendation <= Rec.NoHire && Weight(r) == 1.0).ToList();
        var documentedYes = packet.Count(r => r.Recommendation >= Rec.Hire && Weight(r) == 1.0);
        if (documentedNos.Any(r => r.Recommendation == Rec.StrongNoHire) || documentedNos.Count > 1)
            return new(false, null, "Documented strong concern");
        if (documentedNos.Count == 1 && documentedYes < packet.Count - 1)
            return new(false, null, "One documented no, not outweighed by specific positives"); // Rule 3

        // Rule 5: level comes from the level-deciding rounds, capped by their weakest observation.
        var levelRounds = packet.Where(r => r.DecidesLevel).ToList();
        Level offered = levelRounds.Count == 0 ? target : levelRounds.Min(r => r.ObservedLevel);
        if (offered > target) offered = target;   // up-levels happen, but rarely in the same loop

        return offered < target
            ? new(true, offered, $"Hire, down-levelled to {offered}: level rounds showed {offered} scope")
            : new(true, offered, "Hire at target level");
    }
}
```

Run any realistic packet through it and you see the lessons: a vague strong hire doesn't rescue a documented no; a strong coding round doesn't raise your level; one design round observed at "Senior" caps a staff packet at senior.

**Interview-grade sentence:** *"Packets aren't averaged — the weakest well-documented round drives the hire decision, the level-deciding rounds drive the level, specific evidence counts far more than enthusiasm, consistent observations across interviewers beat one-offs, and a handful of behaviours are knockouts regardless of everything else."*

---
# Part C — What each round type scores

## Concept 9 — What coding rounds score

### The convergent rubric

Tech Interview Handbook's synthesis of coding rubrics across large companies (written by an ex-Meta staff engineer) finds the same four dimensions everywhere, whatever the labels:

| Dimension | Basic signals | Advanced signals |
|---|---|---|
| **Communication** | Asks clarifying questions; explains approach, rationale and trade-offs; keeps talking while coding; organised and succinct | Interviewer never has to ask what you're doing |
| **Problem solving** | Understands the problem quickly; systematic approach; reaches an optimised solution; correct time/space complexity; no major hints | Several solutions with trade-offs; picks the right one for the context; time left for extensions |
| **Technical competency** | Turns the approach into working code with few bugs; clean, no syntax errors, sensible abstractions, readable style | Compares implementation approaches; strong command of language constructs and idioms |
| **Testing** | Tests typical cases; finds and handles corner cases; finds and fixes own bugs; verifies systematically (dry-runs state) | Testing is proactive, not prompted |

The **overall** recommendation is a judgement across dimensions, not a sum. A candidate who reaches the optimal algorithm silently, with untested code, is typically a lean no; a candidate who reaches a slightly suboptimal solution, communicates throughout, and tests it well is often a hire.

### What changes at senior level

Your DS&A is strong, so the algorithmic part is not where you'll be differentiated. At senior level the coding round's standard shifts from *"can you solve it"* to ***"would I want this code in production"*** (Module 36):

- **Correctness under edge cases** is expected, not a bonus.
- **Code quality** — naming, decomposition, types, error handling — is scored more heavily. Amazon's SDE III page lists the objectives explicitly: efficiency, reliability, robustness, portability, maintainability, readability — and states code must be **syntactically correct, no pseudocode**, judged on being **scalable, robust and well-tested**, with **edge cases checked and bad input validated**.
- **API and type design** matter: the signature you choose, what you return on failure, whether the type invites misuse.
- **Speed still matters** at speed-focused companies (Meta is consistently reported as the fastest-paced), but at senior level you buy speed through *clarity of plan*, not typing.

### The C# angle

C# is accepted at every large company. Two things make it score well or badly:

- **Fluency without the IDE.** Interviewers notice `using` directives, exact BCL names (`TryGetValue`, `PriorityQueue<TElement,TPriority>`, `CollectionsMarshal`), collection-expression syntax, and whether you know `Dictionary` iteration order isn't guaranteed. Shared editors (CoderPad and similar) give you little or no IntelliSense.
- **Idiom as signal — in proportion.** Records for value objects, pattern matching, `readonly` structs where they matter, `Span<T>` only where it actually earns its complexity. Over-engineering a 40-line problem with interfaces and DI is a *negative* signal ("premature abstraction").

**Interview-grade sentence:** *"Coding rounds score four things — communication, problem solving, technical competency and testing — and at senior level the bar shifts to production quality: correct on edge cases, readable, tested without being asked, with an API that's hard to misuse."*

---

## Concept 10 — What system design rounds score

### The convergent rubric

Meta names its four design focus areas explicitly; Hello Interview — founded by former Meta and Amazon interviewers — finds the same themes across companies:

| Dimension | What it asks | Common failure modes |
|---|---|---|
| **Problem navigation** | Can you take an ambiguous prompt, find what matters, scope it, prioritise, and move through it to a working system? | Too little requirements work; spending time on trivial parts; getting stuck; no complete design |
| **Solution design** | Can you compose a coherent architecture that meets the requirements, including scale and performance? | Missing core concepts; ignoring scale; "spaghetti" design |
| **Technical excellence** | Do you know current technologies and patterns, and apply them correctly and deeply when probed? | Outdated approaches; not knowing tools; knowing names but not behaviour |
| **Communication & collaboration** | Can you explain clearly, take input, handle pushback, and work *with* the interviewer? | Unclear explanations; defensiveness; getting lost in the weeds |

Hello Interview's guidance stresses that **problem navigation is often the most important** dimension and the one candidates most often fail — which is why Module 3's framework puts structure first.

### Level expectations

All levels must produce a working design. What separates them is **where the time goes**:

| | Mid-level | Senior | Staff |
|---|---|---|---|
| Basics (requirements, API, data model, HLD) | Covered well, takes most of the time | **Covered quickly**, leaving time for depth | Covered very quickly; easy parts explicitly skipped |
| Deep dives | One, often interviewer-led | Two, mostly **candidate-led**, with trade-offs | On the **crux** of the problem, identified early |
| Trade-offs | Named | Chosen, with reasons and numbers | Chosen decisively, with **reversal conditions** and cost |
| Complexity | Adds components to be safe | Justifies components | **Cuts complexity** — "does this even need to scale?" |
| Interviewer's role | Guide | Partner | **Peer** — candidate communicates at peer bandwidth |

The staff column draws on Hello Interview's staff-level guidance: match your communication to a staff interviewer (don't explain 101 concepts), identify the crux, ruthlessly cut complexity, show depth from real experience, and **make decisions rather than listing options** — a candidate who only presents options leaves the interviewer asking "which would *you* choose?", and that question is itself a negative signal.

### Why design decides level

A coding problem has a narrow range of good answers. A design problem has an enormous range — so it **reveals scope**: what you consider worth worrying about, how far ahead you look, which failures you anticipate, whether you notice cost and operability. That's exactly what distinguishes levels, so committees lean on design rounds for leveling (Concept 7).

### The .NET/Azure angle

Generic design rubrics reward "knowing current technologies". In a .NET/Azure-focused loop, **stack precision** is a strong differentiator and an easy place to be caught out:

- **Precise:** "Service Bus duplicate detection only covers producer resends within the detection window — it doesn't help with redelivery after a lock expires, so the consumer still needs idempotency."
- **Vague:** "Service Bus guarantees exactly-once."

Module 38 scores this as a separate sub-dimension (D2b). Every claim you make about Cosmos DB consistency, Azure Functions scaling, EF Core tracking or `HttpClient` lifetime is evidence — for or against.

**Interview-grade sentence:** *"Design rounds score problem navigation, solution design, technical excellence and communication; at senior level I aim to cover the basics fast and spend the time on candidate-led deep dives with stated trade-offs, and at staff level I go straight to the crux, cut complexity and make decisions with the conditions under which I'd reverse them."*

---

## Concept 11 — What behavioral rounds score

### The premise: past behaviour predicts future behaviour

Behavioral rounds are a structured-interview technique: instead of asking what you *would* do, they ask what you *did*, because past behaviour in similar situations is a better predictor. Amazon says so on its own SDE III page — it focuses on the *what* and *how* of your experiences and the *why* of your decisions, asks for metrics, and recommends the **STAR** structure.

### What's scored

Behavioral rubrics are **competency** rubrics. Each company names its competencies differently (Part D), but the underlying evidence is the same:

| What's scored | What strong evidence looks like |
|---|---|
| **Ownership** | Clear personal decisions and actions, distinguished from the team's; "I" where it was you |
| **Scope and complexity** | Problems at or above the target level: ambiguous, cross-team, high-stakes |
| **Judgement** | Options you considered and rejected, and why; how you decided under uncertainty |
| **Results** | Quantified outcome; impact on users, system, team or business; what happened afterwards |
| **Collaboration and conflict** | Disagreements handled with evidence and respect; influence without authority |
| **Growth and self-awareness** | A real failure owned; a specific lesson applied later |

### Level changes the bar more than anything else here

Module 34 develops this fully. The short version:

- **Senior** stories describe **what you delivered** — a significant project you led technically, with real trade-offs.
- **Staff/architect** stories explain **why it mattered and how it shaped the system and the organisation** — multiple teams, a technical direction, a decision that outlived the project.

Tech Interview Handbook's senior-candidate guidance and committee members' accounts agree: for senior packets, the behavioral round is where leveling evidence often lives.

### How behavioral rounds fail senior candidates

- **"We" stories** — interviewers can't attribute any decision to you.
- **Scope too small** — excellent stories at mid-level scope.
- **No conflict in the conflict story** — a disagreement that resolved itself.
- **No numbers** — especially damaging at data-driven companies (Amazon explicitly asks for metrics).
- **Blame** — the team, the manager, the other department.
- **Over-rehearsed** — collapses at the second "why".

**Interview-grade sentence:** *"Behavioral rounds are competency interviews built on the idea that past behaviour predicts future behaviour; they score ownership, scope, judgement, results, collaboration and self-awareness — and the scope of the stories is one of the strongest leveling signals in the packet."*

---

## Concept 12 — What deep technical (domain) rounds score

### What these rounds are

Many senior loops — especially at team-driven companies, platform teams, and stack-specific roles like ".NET backend" — include a round that is neither a puzzle nor a design: a **deep technical conversation** in the role's domain. Google's term for the underlying attribute, **role-related knowledge (RRK)**, describes it well.

Common forms:

- **Rapid probes** across the stack: GC, async/await, DI lifetimes, EF Core tracking, threading.
- **A debugging scenario:** "p99 doubled after a deploy, CPU is flat, thread count climbs — walk me through it."
- **Project deep dive:** "Pick a system you built; let's go as deep as we can."
- **Code review:** read a snippet, find the problems, prioritise them.

### What's scored

| Dimension | Strong evidence |
|---|---|
| **Precision** | Exact semantics, correct defaults and limits — no hand-waving |
| **Depth on demand** | Can go three or four "why"s deep on the things you claim to know |
| **Diagnostic reasoning** | Forms hypotheses, orders them by likelihood and cost to test, names the tools (`dotnet-counters`, `dotnet-trace`, dumps), narrows systematically |
| **Calibrated honesty** | Says "I don't know, but here's how I'd find out" rather than bluffing — a bluff caught is a knockout-class signal |
| **Experience** | Real incidents, real numbers, real trade-offs from your own work |

### The project deep dive is a trap and an opportunity

You choose the project, so you control the terrain — but the interviewer will drill until they find the edge of your knowledge. **Choose a project where you made the important decisions yourself**, and rehearse the second, third and fourth "why" for each decision. "The architect chose that" is the end of your evidence.

**Interview-grade sentence:** *"Deep technical rounds measure role-related knowledge — precise semantics, depth when probed, systematic diagnosis — and they reward calibrated honesty: 'I don't know, here's how I'd find out' scores; a confident wrong answer that gets caught is one of the most damaging things in a packet."*

---

## Concept 13 — What architect rounds score

Module 2 covers the architect loop's shape in detail; here is what its rounds score.

The defining difference: an IC design round asks *"can you design it?"*; an architect round asks ***"can you decide, document it, and get people to align behind it — given the real organisation, its constraints, and the system that already exists?"***

| Dimension | What it asks | Strong evidence |
|---|---|---|
| **Decision framing** | Do you identify the actual decision, its drivers, reversibility and who's affected? | "The decision is whether we split billing out now or after the EU launch; it's hard to reverse, and it affects the payments and finance teams." |
| **Options and trade-offs** | Credible options compared on consequences, cost and risk — with a recommendation | A recommendation *and* the condition that would reverse it |
| **Stakeholder reasoning** | Can you speak to engineering, product, finance, security and operations in their terms? | Different framing for different audiences; dissent engaged, not dismissed |
| **Migration and risk** | Can you get from here to there safely? | Strangler fig, dual-run, shadow traffic, rollback, measurable gates (Module 32) |
| **Cost and ownership** | Do you price the decision — money, on-call, skills, build vs buy? | Named cost drivers; who will own and run it (Module 33) |
| **Documentation** | Is the decision recorded so it survives you? | ADR-shaped reasoning; C4 diagrams at the right level (Modules 30–31) |
| **Defence under challenge** | Do you update on good arguments and hold on good evidence? | Visible concessions; firm positions with reasons |

The common failure mode, noted in this curriculum's orientation: **a technically sound design that ignores team size, delivery constraints, or cost**. In an architect loop that is a *no*, however elegant the boxes.

**Interview-grade sentence:** *"Architect rounds score decision quality rather than design cleverness — framing the real decision, comparing credible options with cost and risk, a migration path from the system that exists, the stakeholders' concerns in their own terms, and documentation that lets the decision survive personnel changes."*

---

## Concept 14 — What AI-enabled rounds add

Since 2025 the industry has split (Module 38 has the full landscape). Some companies **forbid** AI in interviews and have reintroduced **in-person rounds** to enforce it (Google's at-least-one in-person round; Amazon's guidance that unauthorised GenAI use can disqualify; Microsoft's code of conduct requiring candidates to demonstrate their own skills unless assistance is explicitly permitted). Others have built **AI-enabled rounds** where an assistant is part of the environment (Meta's AI-enabled coding round, with similar formats reported at LinkedIn, Shopify and Canva).

Where AI is part of the round, the rubric gains rows. Publicly described expectations converge:

| What's scored | Strong | Weak |
|---|---|---|
| **Decomposition** | You split the task and decide what to delegate | Pasting the whole prompt into the assistant |
| **Verification** | You read, run and test every generated line | Accepting code because it compiles |
| **Critical reading** | You catch and explain the assistant's mistakes | Defending code you didn't read |
| **Design ownership** | You choose structure and algorithms; the assistant fills in | The assistant's structure becomes yours |
| **Fundamentals** | You can explain complexity, invariants and trade-offs of what's on screen | Can't explain the code |
| **Communication** | You narrate what you're asking for and why | Silent prompting |

The rubric **lens** hasn't changed — problem solving, technical competency, testing, communication — but the evidence has. Using AI well is scored; *depending* on it is penalised. And the rule that matters most: **match the round's AI mode exactly** — ask the recruiter in writing if unsure.

**Interview-grade sentence:** *"AI-enabled rounds keep the classic dimensions but change the evidence — interviewers look for decomposition, verification and critical reading of generated code, with the design decisions staying mine — while no-AI rounds still exist in the same loops, so I confirm the rules for each round in advance."*

---
# Part D — Company dialects

A caution before the details: **internal rubrics are not public, and they change.** What follows combines each company's official candidate guidance with consistent, widely reported accounts from former interviewers (interviewing.io's interviewer-sourced guides, Hello Interview, Tech Interview Handbook, committee members' interviews). Use it as orientation and confirm the current loop with your recruiter (Module 39).

## Concept 15 — Google's dialect

### The attributes

Google has long described four attributes it hires for — stated by Laszlo Bock (former SVP People Operations) and in Schmidt and Rosenberg's *How Google Works*:

| Attribute | What it means | Where it's mostly observed |
|---|---|---|
| **General cognitive ability (GCA)** | How you think: problem solving, learning, structuring ambiguity — not grades or trivia | Coding and design rounds |
| **Role-related knowledge (RRK)** | The skills and experience the role needs | Coding, design, domain rounds |
| **Leadership** | *Emergent* leadership — stepping up regardless of title, and stepping back when someone else should lead | Behavioral round; design driving |
| **Googleyness** | Comfort with ambiguity, bias to action, collaboration, intellectual humility, conscientiousness | Behavioral round ("Googleyness & Leadership", G&L) |

For software engineers the loop usually maps onto these via round types — **coding** (GCA + RRK), **system design** (RRK + GCA, more rounds at L6+), and a **G&L / behavioral** round.

### The mechanics

- **Interviewers write detailed feedback** and a recommendation — reported as a **seven-point scale** from Strong No Hire to Strong Hire. Feedback is largely **asynchronous**; interviewers rarely debrief live.
- A **hiring committee** of engineers and managers **who did not interview you** reads the packet (resume, referral and recruiter notes, all interviewer feedback, your code) and seeks consensus. It decides **hire and level**, and can **down-level** on mixed signals.
- **Team matching** follows approval; both you and the team must opt in. Approval without a match is possible.
- At least **one in-person round** for engineering roles (announced 2025).
- Interviewers historically have **broad freedom in technical questions** (often making up their own), with behavioral questions more standardised.

### What it means for preparation

- Because a committee reads text, **evidence quality is everything** — quotable decisions, stated complexity, tested code.
- **"How you think" is the north star** (interviewing.io classes Google as a "how" company): a well-reasoned journey to a near-optimal solution can pass where a silent optimal one doesn't.
- **Every round counts.** You can't concentrate preparation on one round type and hope it carries the packet.

---

## Concept 16 — Amazon's dialect

### The 16 Leadership Principles

Amazon publishes them, says they are used every day, and evaluates candidates against them in every round. The full list: **Customer Obsession · Ownership · Invent and Simplify · Are Right, A Lot · Learn and Be Curious · Hire and Develop the Best · Insist on the Highest Standards · Think Big · Bias for Action · Frugality · Earn Trust · Dive Deep · Have Backbone; Disagree and Commit · Deliver Results · Strive to be Earth's Best Employer · Success and Scale Bring Broad Responsibility.**

For senior engineers, some LPs map naturally onto technical behaviour:

| LP | How it shows up in a technical round |
|---|---|
| Dive Deep | Going below the abstraction when probed; knowing your system's numbers |
| Are Right, A Lot | Trade-off judgement; seeking disconfirming evidence |
| Invent and Simplify | Simpler designs; removing components |
| Insist on the Highest Standards | Testing, edge cases, operational readiness |
| Frugality | Cost awareness in designs |
| Ownership | Operating what you build; long-term over short-term |
| Bias for Action | Distinguishing reversible ("two-way door") from irreversible decisions |
| Have Backbone; Disagree and Commit | Holding a position with evidence; committing after a decision |

### The mechanics

- **Pre-brief:** interviewers are **assigned LPs** to probe (Amazon's own SDE III page says different interviewers are assigned different competencies).
- **The loop:** for SDE III, five 55-minute interviews mixing coding, system design (at least one design question) and LP questions; each interviewer typically asks **two or three behavioral questions**.
- **The Bar Raiser:** a trained interviewer from **outside the hiring team**, present to keep the bar company-wide.
- **The debrief:** live; Bar Raiser and hiring manager carry most weight and can each effectively block; reported scale "Strongly Inclined … Strongly Not Inclined".
- **LPs can decide the outcome:** per interviewing.io, a poor LP showing is almost always a no-hire, while strong LPs can make a below-bar technical result discussable.

### What it means for preparation

- **A story bank tagged by LP** — enough distinct stories that two interviewers probing different LPs don't hear the same one (Module 35).
- **STAR with data.** Amazon calls itself data-driven and asks for metrics.
- **Working code over clever code.** Syntactically correct, tested, edge cases handled.
- Amazon is a **"what"** company in interviewing.io's terms — results matter; get to a working answer.

---

## Concept 17 — Meta's dialect

### The focus areas

- **Coding:** fast-paced; historically two problems in 45 minutes. Since late 2025 one coding round has been reported replaced, for many candidates, by a **60-minute AI-enabled coding round** in a realistic codebase.
- **Design:** **system design** (infrastructure/back-end) or **product architecture** (APIs, data flow, data model, client-server) — scored on **problem navigation, solution design, technical excellence, technical communication**.
- **Behavioral:** commonly reported to probe competencies such as resolving conflict, driving results, working through ambiguity, growth, and influence beyond your team.

### The mechanics (per interviewing.io's interviewer-sourced guide and a former hiring-committee chair)

- **Coding rounds decide hire/no-hire:** a binary recommendation plus a confidence signal; failing both coding problems is usually a no.
- **Design and behavioral decide level:** design interviewers say whether to consider another level; **staff candidates must pass both design rounds**; the behavioral round is **medium-to-low weight** generally but can **sink an E6+ packet**.
- **Committee review**, then **engineering directors** confirm hire and level; then **team matching**.
- **The most standardised loop** in big tech — coding questions come from a pre-approved bank, interviewer training is the most rigorous — which makes Meta's rubric the most knowable.

### What it means for preparation

- **Speed with communication** in coding — Meta is a "what" company; finish, and talk while you do.
- **Design is the level lever.** For E5/E6, invest most in design depth and driving.
- **Prepare for the AI-enabled round explicitly** if your loop has one (Module 38, Concept 21).

---

## Concept 18 — Microsoft's dialect

This is the company most directly relevant to a .NET/Azure career, so it's worth precision about what is official and what is reported.

### What Microsoft says officially

From its *How we hire* page:

- Hiring uses **structured interviews and a consistent framework** to evaluate **real skills against the role's requirements**.
- It looks for **respect, integrity, accountability and growth mindset**.
- It considers **how you'll learn, collaborate and contribute to impact across teams**.
- **Most interviews include 2–4 conversations** with potential teammates and cross-functional colleagues, each up to an hour; they may be by phone, on Teams, or in person.
- Prepare **specific examples from past experience**, **how you'd approach tasks in the role**, and **how your skills translate**; some roles ask you to **write code** or share work samples.
- **AI:** responsible use in **preparation** is encouraged; in assessments and interviews, candidates should demonstrate their own skills **without outside assistance unless explicitly permitted**.

### What is consistently reported

- **Team-dependent process.** Each team or org chooses round types, questions, sometimes even the rating scale and whether to use a written rubric. Interviewers have historically had little central training. This makes Microsoft loops **less predictable** but often **closer to the team's real work**.
- **Engineering loops** commonly run 4–5 hour-long interviews mixing coding, design and behavioral, often including the hiring manager.
- **The AA ("as appropriate") interview:** many loops end with a senior interviewer, often from outside the immediate team, who reads earlier feedback and makes an overall judgement on standard and fit; it is not universal.
- **Growth mindset** is a recurring explicit theme — Microsoft's culture language since the mid-2010s — and shows up in behavioral probes about learning from failure and changing your mind.
- **Levels:** senior is 63–64; principal 65–67; partner 68+.

### What it means for preparation

- **Research the team, not just the company.** Its product area, stack and recent work shape the questions (Module 39).
- **Expect domain depth.** For Azure or .NET teams, the deep technical round may go far into the platform you'd work on — be precise about what you know and honest about what you don't.
- **Show learning.** "What did you change your mind about?" and "what did you learn from that failure?" are natural questions in a growth-mindset culture.
- **Treat the hiring manager conversations as evaluated,** including informal ones after the loop.

---

## Concept 19 — Everyone else

Most senior .NET and architect roles are **not** at the big four. The good news: most companies borrow heavily from them, so the same grammar applies. The differences are in format.

| Company type | Typical format | What's weighted |
|---|---|---|
| **Large product companies** (fintech, SaaS, marketplaces) | Big-tech-like loops; often a hiring committee or bar-raiser variant; more practical coding (bug fixing, extending a codebase, API integration) | Production-quality code, design, ownership |
| **Mid-size / scale-ups** | 3–5 rounds; practical coding or a **take-home**; system design based on their domain; founder/CTO conversation | Shipping speed, pragmatism, breadth, ownership |
| **Enterprises (banks, insurers, retail, public sector)** | Competency-based behavioral interviews; technical discussion with a panel; sometimes a presentation; architect roles often include a **case study** | Stakeholder management, governance, risk, existing estate (.NET Framework → modern .NET, on-prem → Azure) |
| **Consultancies / system integrators** | Technical depth plus **client-facing** scenarios; certifications sometimes weighed | Communication with clients, breadth across stacks, estimation, delivery |
| **Startups** | Informal; a practical exercise; time with founders | Speed, autonomy, breadth, culture-add |

Three adjustments for these loops:

1. **Practical beats puzzle.** Expect "extend this API", "review this PR", "debug this service", or a take-home. The coding rubric's dimensions still apply; testing and code quality matter more.
2. **Competency frameworks are explicit in enterprises.** Many publish values or competency frameworks; behavioral interviews are often scored against them literally, sometimes by HR alongside engineers.
3. **Architect roles are more varied.** "Solution architect" at an enterprise may be closer to the architect track (decision, stakeholders, governance); at a product company it may be a staff IC. Module 2 shows how to tell which, and how to calibrate.

**Interview-grade sentence:** *"Outside big tech the grammar is the same — structured rounds, written feedback, a level bar — but formats are more practical and team-specific, enterprises often score explicit competency frameworks, and 'architect' can mean either a staff IC or a decision-and-governance role, so I find out which before I prepare."*

---

## Concept 20 — The six universal dimensions

Put the company dialects side by side and they collapse onto six dimensions. These are the ones the rest of the curriculum uses — Module 38 turns them into behaviourally anchored rubrics.

| # | Dimension | The question it answers | Typical evidence |
|---|---|---|---|
| **D1** | **Framing and scoping** | Did they understand the real problem and bound it? | Clarifying questions; stated assumptions; explicit out-of-scope |
| **D2** | **Technical depth and precision** | Is what they say correct, deep and specific — including the stack? | Depth reached under probing; precise semantics; no hand-waving |
| **D3** | **Trade-offs and judgement** | Do they choose well among real options and know when they'd choose differently? | Options compared; choice with reasons; numbers; reversal conditions |
| **D4** | **Driving and communication** | Did they run the round — structure, time, signposting, check-ins? | Phase announcements; time budget met; summaries; readable diagrams |
| **D5** | **Collaboration and coachability** | How do they handle input, hints and disagreement? | Evaluates suggestions; updates on evidence; holds positions on evidence |
| **D6** | **Level signal** | Is the scope, ownership and impact right for the target level? | Scope of stories; org/cost/migration awareness; impact framing |

### The translation table

| Company vocabulary | Maps mostly onto |
|---|---|
| **Coding (generic):** communication · problem solving · technical competency · testing | D4 · D1+D3 · D2 · D2 |
| **Design (generic / Meta):** problem navigation · solution design · technical excellence · communication & collaboration | D1+D4 · D3 · D2 · D4+D5 |
| **Google:** GCA · RRK · Leadership · Googleyness | D1+D3 · D2 · D6+D5 · D5 |
| **Amazon:** Dive Deep · Are Right, A Lot · Invent and Simplify · Ownership · Deliver Results · Earn Trust · Have Backbone; Disagree and Commit · Think Big | D2 · D3 · D3 · D6 · D6 · D5 · D5 · D6 |
| **Microsoft:** real skills against role requirements · growth mindset · collaboration · impact across teams | D2+D3 · D5 · D5 · D6 |
| **Architect loops:** decision framing · options/trade-offs · stakeholders · migration/risk · cost · documentation | D1 · D3 · D5+D6 · D6 · D6 · D4 |

### Why this matters

You don't need a different preparation for each company. You need **evidence on six dimensions**, plus **company-specific emphasis**: LP-tagged stories for Amazon, design depth for Meta leveling, evidence density for Google's committee, team-domain depth for Microsoft. And for a .NET loop, split D2 into **D2a general depth** and **D2b stack precision** — the second is where you differentiate.

**Interview-grade sentence:** *"Under different names, almost every loop scores six things — framing, technical depth, trade-off judgement, driving the conversation, collaboration, and level-appropriate scope — so I prepare against those six and add each company's emphasis on top, rather than preparing six different ways."*

---
# Part E — Using rubric literacy

## Concept 21 — Producing writeable evidence

Concept 3 established that decisions are made from write-ups. This concept is the practical toolkit: **sentences shaped so an interviewer can paste them into a dimension box.**

### The evidence sentence: decision · reason · condition

The most useful single sentence shape in senior interviews:

> ***"I'll do X, because Y; I'd change to Z if W."***

- *"I'll store links in Cosmos DB partitioned by short code, because every read is a point read by that key and we need ~50k reads/s globally; I'd move to Azure SQL if we needed ad-hoc analytics on the same store — but I'd rather stream to a separate analytical store."*
- *"I'll keep this as one service for now, because one team owns it and the boundaries aren't stable yet; I'd split notification delivery out if its deploy cadence or scaling profile diverges."*

It produces evidence on **D3** (choice and reason), **D2** (the facts in the reason) and **D6** (the condition shows you think about change), in one breath.

### Phrasebook by dimension

| Dimension | Sentences that produce evidence |
|---|---|
| **D1 Framing** | "Before designing, who are the users and what's the one thing this must never get wrong?" · "I'll assume X; tell me if that's wrong." · "I'm putting Y out of scope — it doesn't change the core design." |
| **D2 Depth** | "Concretely, what happens here is…" · "The default is X; the reason is Y; the case where that's wrong is Z." · "I'm not sure of the exact limit — I'd check, but the design holds if it's within 2× of my assumption." |
| **D3 Judgement** | "Two options: A and B. I'd pick A because… The cost is…" · "That's about 2,300 writes a second, so one primary is fine." · "This is a two-way door, so I'd decide quickly and measure." |
| **D4 Driving** | "Here's my plan for the next 40 minutes…" · "Requirements done — moving to the API." · "I'd like to go deep on X and Y; is there one you'd prefer?" · "Let me summarise where we are." |
| **D5 Collaboration** | "Good point — that breaks my assumption about X, so I'd change…" · "I see the concern. My reason for keeping it is…; what would make you prefer the alternative?" · "Let me check I understood your question." |
| **D6 Level** | "Who will run this on call?" · "The migration matters more than the end state here — here's how we'd get there without a freeze." · "In my last role I made the call to… — the trade-off was…" |

### Make silent work audible

Many strong behaviours are invisible by default. Narrate them briefly:

| Silent behaviour | Audible version (one sentence) |
|---|---|
| Considering and rejecting an option | "I considered a queue here, but the latency budget rules it out." |
| Checking an assumption | "That only works if writes are rare — you said ~100/s, so fine." |
| Noticing a risk | "The risk I'd flag is the cache stampede after a deploy; I'll come back to it." |
| Choosing not to build something | "No need for sharding at this scale — one node handles ~10× this." |
| Testing mentally | "Let me run the empty input and the single-element case through it." |
| Correcting yourself | "I'd change what I said earlier — …" |

### Don't over-narrate

There is an opposite failure: **narrating everything**, especially basics. At staff level, explaining what a load balancer is costs time and — in Hello Interview's staff-level guidance — makes it look as though you've just learned it. Narrate **decisions and reasoning**, not definitions.

**Interview-grade sentence:** *"I give interviewers sentences they can write down — the decision, the reason, and the condition under which I'd change it — and I narrate the decisions that would otherwise be invisible, like options I rejected and risks I noticed, without explaining basics the interviewer already knows."*

---

## Concept 22 — Level signals

Concept 7 explained that levels are decided by scope. Here is what **observable evidence** of each level looks like, dimension by dimension — the difference between a "hire at senior" and a "hire at staff" write-up.

| Dimension | Senior evidence | Staff / architect evidence |
|---|---|---|
| **D1 Framing** | Separates functional from non-functional; picks the numbers that shape the design | Asks who the users are, what exists today, what failure costs, who'll operate it; finds the crux early |
| **D2 Depth** | Correct and reasonably deep on chosen components | Depth *drawn from experience*; can say what broke in production and why |
| **D3 Judgement** | Compares options, chooses with reasons | Decides crisply, names the reversal condition, the cost, the ownership implications; **cuts** complexity |
| **D4 Driving** | Runs the round; time well used | Communicates at peer bandwidth; earmarks hard parts; skips easy ones explicitly |
| **D5 Collaboration** | Handles input and pushback well | Brings the interviewer into the decision; handles disagreement like a design review |
| **D6 Scope** | The component, well designed | The component **in the system and the organisation**: migration path, team boundaries, risk, cost, operability |

### In behavioral stories

| Senior story | Staff/architect story |
|---|---|
| "I led the redesign of our checkout service" | "I identified that three teams were building incompatible payment integrations and drove a shared approach across them" |
| Outcome for the system | Outcome for the system *and* the organisation (fewer incidents across teams, a standard others adopted) |
| Resolved a disagreement in my team | Resolved a disagreement between teams or with leadership, without authority |
| Made a hard technical decision | Made a decision that outlived the project, and recorded it so others could reverse it safely |

### Common ways strong candidates under-level themselves

1. **Waiting to be asked** for deep dives, failure modes, or cost.
2. **Presenting options instead of decisions.**
3. **Picking small stories** because they're easier to tell.
4. **Saying "we"** when they personally drove the decision.
5. **Never mentioning the organisation** — teams, ownership, migration, on-call — in a design.

**Interview-grade sentence:** *"Level shows up in scope: at senior I design the component well and drive the round; at staff I go to the crux, decide with reversal conditions and cost, place the design in the organisation — migration, ownership, operability — and tell stories whose impact crossed team boundaries."*

---

## Concept 23 — Red flags and knockouts

Some behaviours do disproportionate damage. Know them by name so you never produce them under pressure.

### Knockouts (a single instance can end the packet)

| Knockout | Why it's fatal |
|---|---|
| **Dishonesty** — inflated claims, bluffing that gets caught, undisclosed AI use where forbidden | Destroys trust in every other piece of evidence |
| **Taking credit for others' work** | Integrity, and makes all your stories suspect |
| **Contempt** — for colleagues, previous employers, users, the interviewer | Culture risk no skill compensates for |
| **Refusing to engage** with a hint, a follow-up or a changed requirement | Coachability is scored everywhere |
| **Silently violating a hard requirement** — a design that loses payments or OTPs with no awareness | Judgement failure on the thing that mattered most |
| **Can't explain your own code** (especially AI-generated) | Suggests the code isn't yours |

### Red flags (don't end the packet alone, but accumulate)

| Red flag | What the interviewer writes |
|---|---|
| Jumps straight to boxes or code | "Didn't clarify requirements; solved an assumed problem" |
| Long silences | "Hard to follow thought process" |
| Defensiveness under pushback | "Argued rather than engaged with the concern" |
| Name-dropping technologies without semantics | "Product soup; couldn't explain how X behaves under Y" |
| Over-engineering | "Added Kafka, Kubernetes and CQRS to a 100 QPS problem" |
| Blame in stories | "Attributes failures to others" |
| No numbers anywhere | "Couldn't quantify impact or scale" |
| Over-rehearsed answers that collapse at the second "why" | "Polished but shallow" |
| Running out of time with no plan | "Poor time management; never reached the deep dive" |

### Recovering

Most red flags are recoverable **within the round** if you name them: *"Let me step back — I jumped to the design; let me check two requirements first."* Self-correction is itself positive evidence (Concept 3). Module 38, Concept 24 has recovery scripts.

**Interview-grade sentence:** *"A few behaviours end a packet on their own — dishonesty, taking credit, contempt, refusing to engage, silently breaking a hard requirement, not being able to explain your own code — while red flags like silence, defensiveness and over-engineering accumulate; most red flags can be recovered in the moment by noticing and correcting them out loud."*

---

## Concept 24 — Reverse-engineering an unknown rubric

For most companies you'll interview with, nobody has published an interviewer-sourced guide. You can still predict the rubric with high accuracy from four sources.

### Source 1 — The job description

Job descriptions are usually written from the same competency model the rubric uses. Read them as a rubric:

```text
"Design and own scalable services on Azure"           → system design round; D2 stack precision; D6 ownership
"Mentor engineers and raise the technical bar"        → behavioral: mentoring, influence; D6
"Work with product and business stakeholders"         → behavioral/architect: stakeholder reasoning; D5
"Experience modernising legacy .NET Framework systems" → brownfield design or deep technical round (Module 32)
"Strong focus on quality and testing"                  → coding: testing dimension weighted up
```

### Source 2 — The company's values or principles page

If the company publishes values (Amazon's LPs, Microsoft's respect/integrity/accountability/growth mindset, most enterprises' "our values"), **behavioral interviews are scored against them**. Map two stories to each value.

### Source 3 — The recruiter

Recruiters want you to pass — a pass is their success metric too. Ask specific, legitimate questions (Appendix C has the full list):

- *"What rounds are in the loop, and what does each one focus on?"*
- *"Is there a system design round? Is it more infrastructure or product/API design?"*
- *"Who decides — a committee, a debrief, the hiring manager?"*
- *"What level is the role, and how is level determined?"*
- *"What's the policy on AI tools in each round? Is any round in person?"*
- *"Is coding in a shared editor, an IDE, or on a whiteboard? Which languages?"*
- *"Is there anything the team particularly values that I should know about?"*

### Source 4 — Candidate reports and the team's own output

Glassdoor-type reports, Blind threads, Hello Interview's community questions, and interviewing.io's guides for the large companies; for smaller companies, the team's **engineering blog, conference talks and open-source code** tell you their stack and what they think matters.

### Assemble the predicted rubric

```text
Round            Dimensions predicted            Emphasis for this company          Prep action
Coding (C#)      D1 D2 D3 D4                     testing + code quality (JD)        2 practical problems/day, blank editor
Design           D1 D2 D3 D4 D6                  Azure precision; brownfield (JD)   Module 37 problems + one migration problem
Behavioral       D5 D6                           mentoring, stakeholders (values)   8 stories mapped to 4 values
Hiring manager   D6 D5                           team domain                        read 5 blog posts; prepare 3 questions
```

Module 39 turns this into a full checklist.

**Interview-grade sentence:** *"When a company's rubric isn't public, I predict it: the job description is written from the same competency model, the values page tells me what the behavioral round scores, the recruiter will tell me the rounds and how decisions and levels are made, and the team's own blog and talks tell me what they care about technically."*

---
# Worked example — The same design round, written up two ways

**Setting.** A senior IC loop (target: senior, with a stretch to staff) at a large product company on Azure. Round: system design, 45 minutes. Prompt: *"Design a rate limiter for our public API."* The candidate is good in both versions below — the **knowledge is identical**. What differs is how much of it becomes evidence.

### Version A — knowledge without signal

What the candidate did (condensed):

```text
00:00  Repeats the prompt. Starts drawing: API gateway → rate limiter service → Redis.
06:00  "We could use token bucket or sliding window." Draws Redis.
12:00  Explains token bucket mechanics in detail (refill rate, capacity).
20:00  Interviewer: "How many requests are we talking about?" Candidate: "Probably a lot. Redis can handle it."
27:00  Interviewer: "What happens if Redis is down?" Candidate: "We could fail open or fail closed."
33:00  Interviewer: "Which would you choose?" Candidate: "Depends on the business."
38:00  Adds a second region on request. Explains Redis replication.
45:00  Time.
```

The interviewer's write-up:

```text
Problem navigation:  Did not clarify requirements (tenants, limits, scale) until prompted at ~20 min. LEAN NO
Solution design:     Standard design (gateway + Redis token bucket). Reasonable but generic.    LEAN HIRE
Technical excellence: Good knowledge of token bucket mechanics. Redis claims correct but surface. LEAN HIRE
Communication:       Clear explanations, but needed prompting for scale, failure, and choices.   LEAN NO
Overall:             LEAN NO HIRE. Knowledgeable, but interviewer drove the round; no decisions
                     on failure policy; no numbers. Not senior-level independence.
Level:               If hired, consider at mid-level.
```

### Version B — the same knowledge, made legible

```text
00:00  "Before I design: who are the clients — first-party apps, third-party developers, or both? Per-key,
       per-IP, or per-tenant limits? Roughly how many requests per second at peak? And what's worse for the
       business — letting a burst through, or rejecting a legitimate request?"
       → third-party developers; per API key; ~40k req/s peak; ~200k keys; occasional bursts; wrongly
         rejecting paying customers is worse than letting a burst through.
04:00  "So: per-key limits, read-heavy check on every request, and a fail-open bias. Out of scope: billing
       and quota management — I'll assume limits are configured elsewhere."
06:00  "40k checks a second, one round trip each. A single Redis primary does well over 100k simple
       ops/s, so one shard is enough to start — I'll note the hot-key risk for very large tenants."
09:00  "Algorithm: token bucket — bursts are expected and it allows them up to capacity. Sliding-window log
       is more exact but costs memory per request; not worth it here."
12:00  HLD drawn: gateway middleware → local check → Redis (Lua script for atomic refill+take) → config store.
       "Every arrow is a synchronous hop on the hot path, so the latency budget is the main constraint."
18:00  "Two deep dives I'd suggest: Redis failure, and the hot-tenant case. Which do you prefer first?"
19:00  Failure: "If Redis is unavailable we fail open — that's the business preference you gave me — but
       with a local in-memory bucket per gateway instance as a coarse backstop, so a total outage can't
       become unlimited traffic. Circuit breaker on the Redis call with a 5 ms timeout so we don't add
       latency while it's down. I'd alert on fail-open duration."
29:00  Hot tenant: "A tenant at 10k req/s means one key gets 10k Lua calls/s on one shard. I'd lease tokens
       in batches to each gateway instance — fewer round trips, slight over-admission bounded by batch
       size. The trade-off is accuracy for throughput; at this tenant size I'd take it."
36:00  .NET specifics: "In ASP.NET Core I'd use the built-in rate-limiting middleware for the local backstop,
       a custom partitioned limiter keyed by API key for the distributed check, and a single shared
       connection multiplexer to Redis. HybridCache for the limit configuration."
40:00  Wrap-up: "What breaks first at 10×: the single shard. The seam is the key-partitioning — move to a
       Redis cluster partitioned by key hash. What I'd want with more time: per-tenant observability."
45:00  Time.
```

The interviewer's write-up:

```text
Problem navigation:  Clarified users, limit granularity, scale and business preference in the first 4 min;
                     stated out-of-scope explicitly. Used the fail-open preference later. STRONG HIRE
Solution design:     Coherent design; every component justified; latency budget identified as the constraint.
                     Chose token bucket with a reason, rejected sliding log with a reason.            HIRE
Technical excellence: Atomic Lua refill/take; circuit breaker with timeout on the hot path; token leasing
                     for hot keys with bounded over-admission. Correct ASP.NET Core rate-limiting usage. STRONG HIRE
Communication:       Drove the round; offered deep-dive choices; numbers each tied to a decision
                     ("one shard is enough", "40k checks/s"). Clean wrap-up with next bottleneck.     STRONG HIRE
Overall:             HIRE (strong). Senior-level independence and depth.
Level:               Senior, solid. Some staff signal (crux identification, business-driven failure policy);
                     would want to see cross-team / organisational reasoning for staff.
```

### What changed

| Behaviour | Dimension it produced evidence for |
|---|---|
| Four requirement questions in the first minute, including the business consequence of each failure | D1, D3, D6 |
| "40k checks a second … one shard is enough" — a number producing a decision | D3, D2 |
| Algorithm chosen *and* the alternative rejected with a reason | D3 |
| Offered the interviewer a choice of deep dives | D4, D5 |
| Failure policy decided from the stated business preference, with a backstop | D3, D6 |
| Precise stack claims | D2b |
| Named the next bottleneck and the seam | D3, D6 |

Notice also what **would** have moved Version B to staff: one or two sentences of organisational reasoning — *who owns limit configuration, how tenants are told about limits (headers, docs), how a limit change rolls out safely, the cost of the Redis tier, who's paged when fail-open trips.* That's Concept 22 in action.

---

# Common interview questions with model answers

Two kinds of questions live here: **meta-questions about hiring** (asked of staff and architect candidates more often than people expect) and **moments in a round** where rubric literacy decides how you respond.

### Questions about hiring and evaluation

**Q1. "How do you interview engineers? What do you look for?"**
> "I start from the role: what will this person actually do in their first year — design services, review code, run incidents, mentor? Each round should sample one of those, with questions decided in advance and a rubric that describes what each rating looks like in observable terms. In design rounds I'm looking at how they navigate an ambiguous problem, whether their trade-offs are backed by numbers, how deep they go when I probe, and whether they drive the conversation. I write down specific evidence — quotes, decisions, timestamps — and score each dimension before I decide my overall recommendation, because overall impressions form fast and absorb later evidence."
*Key signal:* structure, job analysis, evidence, dimensions before verdict.

**Q2. "How would you decide between two candidates with similar scores?"**
> "I'd go back to the evidence rather than the scores — the scores are summaries. I'd look at which dimensions each was strong and weak in, and weigh those by what the role needs: for a platform role, depth and operational judgement; for a role leading a new team, scoping and influence. I'd also look at consistency across interviewers — two independent observations of the same strength are worth more than one enthusiastic one. And if it's still genuinely close, I'd want one more targeted conversation on the deciding dimension rather than guessing."
*Key signal:* evidence over scores, role-weighted dimensions, consistency, gathering more signal.

**Q3. "What's the difference between a senior and a staff engineer, in your view?"**
> "Scope and the kind of ambiguity they absorb. A senior engineer takes a well-defined problem and owns the solution end to end — design, delivery, operation. A staff engineer is given an area and defines the problems in it; their work crosses team boundaries, they make decisions that outlive projects, and their impact often comes through other people — standards, reviews, direction. In an interview I'd expect a staff candidate to go to the crux quickly, cut complexity, and place the design in the organisation: migration, ownership, cost, operability."
*Key signal:* a crisp scope-based definition, tied to observable interview behaviour.

**Q4. "How would you reduce bias in your team's hiring?"**
> "Mostly by adding structure: decide the dimensions and questions per role in advance, write behaviourally anchored rubrics, have interviewers submit written feedback independently before any discussion, and make the decision from evidence rather than 'culture fit' gut feel. I'd train interviewers and calibrate them on shared material, and I'd review outcomes — pass rates by stage, interviewer score distributions — to spot where the process drifts."
*Key signal:* structure as the main lever; measurement of the process itself.

**Q5. "Have you ever disagreed with a hiring decision? What did you do?"** *(behavioral; answer with a real story)*
> Structure: the decision and your evidence → how you raised it (in the debrief, with specifics, not adjectives) → what happened → what you learned about the process (e.g. "we lacked a rubric for X, so I wrote one").
*Key signal:* evidence-based disagreement, respect for the process, improving it afterwards — "Have Backbone; Disagree and Commit" in Amazon terms.

**Q6. "How would you design the interview loop for a senior .NET engineer on Azure?"**
> "Job analysis first. Then: a practical coding round in C# — correctness, testing, readability, with a realistic problem rather than a puzzle; a system design round shaped like our domain, scored on navigation, solution design, depth and communication, with Azure precision as a sub-dimension; a .NET depth round anchored on a debugging scenario — async, memory, EF Core, ASP.NET Core; and a behavioral round on ownership, conflict and mentoring against our values. Each round has a written rubric and probe ladders; feedback is independent; the AI policy for each round is explicit and told to candidates in advance; and we review the loop's outcomes at least yearly."
*Key signal:* rounds mapped to the job, rubrics, independence, explicit AI policy, feedback loop on the loop.

### Moments in a round

**Q7. The interviewer gives you a one-line prompt and stays silent.**
> Treat silence as an invitation to drive (D4) and scope (D1): *"Before designing, I'd like to understand a few things that will shape it — who uses it, the scale, and what we must never get wrong."* Ask 4–6 targeted questions, state assumptions for anything they won't answer, and announce your plan.
*Key signal:* driving, framing.

**Q8. You know two good options and can't decide.**
> Decide anyway, and make the condition explicit: *"I'll go with A because of X; if Y turns out to be true, B becomes better and here's what would change."* Presenting options without choosing reads as an absence of judgement — at staff level it often prompts the interviewer to ask "which would *you* choose?", which is itself a negative note.
*Key signal:* D3 judgement with reversal condition.

**Q9. The interviewer asks about something you don't know.**
> *"I don't know the exact behaviour there. My understanding is X; if that's wrong, the design changes like this. I'd verify it by Y."* Never bluff: a confident wrong answer that gets caught damages every other claim in the packet.
*Key signal:* calibrated honesty; D2 depth boundary handled well.

**Q10. In a behavioral round, the interviewer asks "what was *your* role?"**
> That question means your story was too "we". Answer precisely: *"I wrote the design doc, I chose the dual-write approach over a big-bang migration, and I ran the cut-over. The team built the consumers; Maria owned the data backfill."* Distinguishing your work from others' *increases* credibility.
*Key signal:* D6 ownership; honesty.

**Q11. You realise your design has a flaw at minute 30.**
> Say so: *"Let me correct something — with edge caching, disabled links would keep working past our one-minute requirement. I'd cache only hot links at the edge and purge on disable."* Self-correction is scored positively; silently patching the diagram isn't scored at all.
*Key signal:* D3, D5; evidence of judgement.

**Q12. The interviewer pushes back on a decision you believe is right.**
> Engage, then hold or update on evidence: *"I see the concern — it's simpler. I'm keeping separate queues because a priority flag doesn't help an OTP that's behind a million queued bulk messages. If campaigns were small I'd agree. Is there a constraint I'm missing?"*
*Key signal:* D5 coachability without capitulation.

**Q13. "Do you have any questions for me?"**
> Ask questions that show you understand how good teams decide: *"How are design decisions recorded and revisited here?"* · *"What would make someone in this role clearly successful at twelve months?"* · *"What's the hardest technical trade-off the team made recently?"* Interviewers sometimes note these in the write-up.
*Key signal:* curiosity about decision-making and success criteria.

**Q14. A recruiter tells you the committee approved you at one level below your target.**
> Ask what evidence drove the leveling and whether there's a path to review — some companies allow an additional round focused on the deciding dimension. Weigh the offer on its own merits, including the scope you'd actually have and the promotion path. Then use the feedback: it tells you which dimension under-produced evidence.
*Key signal (to yourself):* the down-level is information about your evidence, not only about you.

---

# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Mental model | "Do well and they'll see it" | "Produce evidence they can write down, on each dimension the round scores" |
| Rubric awareness | Prepares content only | Knows each round's dimensions and allocates minutes to them |
| Requirements | Jumps to boxes or code | Scopes in the first minutes; states assumptions and out-of-scope |
| Numbers | None, or numbers with no consequence | Few numbers, each producing a stated decision |
| Decisions | Lists options | Decides, with a reason and a reversal condition |
| Deep dives | Waits to be asked | Proposes them, where the difficulty is |
| Failure handling | Only when prompted; "we retry" | Unprompted failure policy per dependency |
| Stack claims | Product names; vague semantics | Precise behaviour, defaults and limits |
| Communication | Silent thinking, or explaining basics | Narrates decisions and rejected options; peer-level bandwidth |
| Pushback | Defends everything, or caves | Updates on good arguments, holds on evidence |
| Behavioral | "We", no numbers, small scope | "I", quantified results, scope at target level |
| Coding | Optimal but untested; clever | Correct, tested unprompted, readable, idiomatic in proportion |
| Honesty | Bluffs when unsure | "I don't know exactly; here's my assumption and how I'd check" |
| Level | Aims for "good" | Aims for the target level's evidence on purpose |
| Weak round | Hopes a strong round compensates | Knows the weakest documented round dominates; prepares the weakest |
| Company dialect | One generic preparation | Six dimensions + company emphasis (LPs, design depth, team domain) |
| AI | Assumes the rules | Confirms the AI mode and in-person format per round with the recruiter |
| Unknown company | Guesses | Predicts the rubric from JD, values, recruiter, team output |
| Hiring questions (staff) | "I go with my gut" | Structured interviewing, independent evidence, calibrated interviewers |

---

# Practice exercises

1. **Map your target loops.** For each company you're targeting, write the loop shape (rounds, decision architecture, AI mode, in-person rounds) from official guidance and recruiter answers. Mark what's confirmed vs reported.
2. **Translate the dialects.** Take one target company's values or competency list and map each item to D1–D6. Which dimensions does it emphasise? Which does it not mention?
3. **Read your JD as a rubric.** For one real job description, convert each responsibility into a predicted round and dimension (Concept 24). Build the four-column predicted rubric.
4. **Write two write-ups.** Record yourself doing a 45-minute design problem. Then write the interviewer's feedback twice: once as a sceptical interviewer who only writes what's explicit, once as a generous one. Where do they differ? Those are the places your signal was implicit.
5. **Evidence-sentence drill.** Take ten design decisions from Module 37's problems and say each aloud as "I'll do X because Y; I'd change to Z if W" in under 20 seconds.
6. **Silent-to-audible audit.** In one recording, list every decision you made silently (rejected options, assumptions, risks). Rewrite each as one narrated sentence.
7. **Level the story bank.** For each story in your bank (Module 35), label its scope: team, cross-team, org. If fewer than half are at your target level, find or reframe stories accordingly — honestly.
8. **Knockout check.** Review three of your past mock recordings for any knockout or red flag in Concept 23. Write the recovery sentence you'd use next time.
9. **Packet reading.** Use the `PacketReader` model in Concept 8. Construct three packets: one passes at senior, one is down-levelled, one fails on a single documented no despite two strong hires. Then argue, as a committee member, whether the model's answer is right in each case — where would a human reader disagree?
10. **Recruiter script.** Write the seven recruiter questions from Concept 24 (Appendix C) in your own words; ask them for your next real loop.
11. **Stack precision list.** For your next design mock, list every .NET/Azure claim you make; mark each ✓ precise, ~ vague, ✗ wrong; fix the ~ and ✗ items with Microsoft Learn references.
12. **Hiring-question answers.** Answer Q1, Q3 and Q6 aloud in three minutes each; record and score with Module 38's behavioral rubric.

---
# Free resources and learning material

All free to read or use unless marked *(book)*, *(paid)* or *(partly paid)*. Start with the ★ items. Company process facts were checked on October 8, 2026; internal rubrics are not public, so third-party guides are labelled as reported.

### Official company guidance (primary sources)
- ★ [Amazon — SDE III / Sr. SDE interview prep](https://www.amazon.jobs/content/en/how-we-hire/sde-iii-interview-prep) — phone screen and loop structure, coding and system-design objectives, LP-based behavioral rounds, STAR.
- ★ [Amazon — Leadership Principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — all 16, with short videos.
- [Amazon — How we hire](https://www.amazon.jobs/content/en/how-we-hire) — the general process.
- [About Amazon — What's it like to interview at Amazon?](https://www.aboutamazon.com/news/workplace/whats-it-like-to-interview-at-amazon) — describes the Bar Raiser as an objective third party, usually from another team.
- ★ [Microsoft — How we hire](https://careers.microsoft.com/v2/global/en/hiring-tips) — structured interviews and a consistent framework; values; 2–4 conversations; candidate code of conduct on AI.
- [Microsoft — Interview tips](https://careers.microsoft.com/v2/global/en/hiring-tips/interview-tips.html) — clarifying questions, explaining reasoning, STAR.
- [Google — Our hiring process](https://www.google.com/about/careers/applications/how-we-hire) and [interview prep](https://www.google.com/about/careers/applications/interview-tips).
- ★ [Meta — Preparing for your full loop interview](https://www.metacareers.com/swe-prep-onsite) — includes a downloadable full-loop guide.
- [Anthropic — Guidance on candidates' AI usage](https://www.anthropic.com/candidate-ai-guidance) — AI for preparation; not in live interviews unless indicated.

### Interviewer-sourced guides to how decisions are made
- ★ [interviewing.io — Ultimate guide to FAANG interviews for senior engineers](https://interviewing.io/guides/hiring-process) — the "chaos score", team-dependent vs centralised processes, interviewer training, question standardisation.
- ★ [interviewing.io — Google's process](https://interviewing.io/guides/hiring-process/google) — seven-point scale, hiring committee, leveling, team match.
- ★ [interviewing.io — Amazon's process](https://interviewing.io/guides/hiring-process/amazon) — pre-brief, Bar Raiser, live debrief, LP weighting, five-point scale.
- ★ [interviewing.io — Meta's process](https://interviewing.io/guides/hiring-process/meta-facebook) — coding decides hire, design decides level; staff must pass both design rounds.
- [interviewing.io — Microsoft's process](https://interviewing.io/guides/hiring-process/microsoft) — team-dependent processes, scales and rubrics.
- [The Developing Dev — Meta hiring lead on senior+ engineering hiring](https://www.developing.dev/p/meta-hiring-lead-on-behind-the-scenes) — a former hiring-committee chair on packets and leveling.
- [Jos Visser — On writing interview feedback](https://josvisser.substack.com/p/on-writing-interview-feedback) — a former big-tech interviewer on why committees hire on feedback, not scores.
- [Tech Interview Handbook — Interview formats at top companies](https://www.techinterviewhandbook.org/interview-formats-top-companies/).

### Rubrics you can read
- ★ [Tech Interview Handbook — Coding interview rubrics](https://www.techinterviewhandbook.org/coding-interview-rubrics/) — communication, problem solving, technical competency, testing, with four-band descriptions.
- ★ [Tech Interview Handbook — Behavioral interview rubrics](https://www.techinterviewhandbook.org/behavioral-interview-rubrics/).
- [Tech Interview Handbook — Behavioral preparation for senior candidates](https://www.techinterviewhandbook.org/behavioral-interview-senior-candidates/).
- ★ [Hello Interview — System Design in a Hurry: introduction](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction) — problem navigation, solution design, technical excellence, communication and collaboration; level expectations.
- ★ [Hello Interview — 5 keys to staff-level system design interviews](https://www.hellointerview.com/blog/staff-level-system-design) — peer bandwidth, the crux, cutting complexity, decisions over options.
- [Hello Interview — Meta system design vs product architecture](https://www.hellointerview.com/blog/meta-system-vs-product-design) — Meta's four focus areas applied to both formats.
- [Hello Interview — The Meta SWE interview](https://hellointerview.com/blog/the-meta-swe-interview).
- [Prepfully — Meta system design and product architecture interview guide](https://prepfully.com/interview-guides/meta-swe-design-interview) — reported focus-area evaluation criteria.
- [Exponent — System design interview rubric](https://www.tryexponent.com/courses/system-design-interviews/system-design-interview-rubric) — five rubric signals from "very weak" to "very strong" *(partly paid)*.
- [Hello Interview — Delivery framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery) — the design-round structure that produces navigation evidence.

### Leveling and career frameworks (what "senior" and "staff" mean)
- ★ [StaffEng — Staff archetypes](https://staffeng.com/guides/staff-archetypes) — tech lead, architect, solver, right hand.
- [StaffEng — guides](https://staffeng.com/guides/) — what staff-plus scope looks like in practice.
- ★ [Dropbox Engineering Career Framework](https://dropbox.github.io/dbx-career-framework/) — IC1–IC7 expectations by scope, collaborative reach and levers of impact.
- [Levels.fyi](https://www.levels.fyi/) — crowdsourced level mappings and compensation across companies.
- [Levels.fyi — How to ace Microsoft's hiring process](https://www.levels.fyi/blog/ace-microsoft-hiring.html).

### Research: why structured interviews and rubrics exist
- ★ [Sackett, Zhang, Berry & Lievens (2022) — Revisiting meta-analytic estimates of validity in personnel selection](https://doi.org/10.1037/apl0000994).
- [SIOP — Is cognitive ability the best predictor of job performance? New research says think again](https://www.siop.org/tip-article/is-cognitive-ability-the-best-predictor-of-job-performance) — readable summary.
- [Schmidt & Hunter (1998) — The validity and utility of selection methods in personnel psychology](https://doi.org/10.1037/0033-2909.124.2.262).
- [Levashina, Hartwell, Morgeson & Campion (2014) — The structured employment interview](https://doi.org/10.1111/peps.12052).
- [Dana, Dawes & Peterson (2013) — Belief in the unstructured interview: the persistence of an illusion](https://journal.sjdm.org/12/121130a/jdm121130a.pdf).
- ★ [Kahneman, Lovallo & Sibony — A structured approach to strategic decisions (MIT Sloan Management Review)](https://sloanreview.mit.edu/article/a-structured-approach-to-strategic-decisions/) — mediating assessments.
- ★ [Google re:Work — Use structured interviewing](https://rework.withgoogle.com/en/guides/hiring-use-structured-interviewing) and [A guide to structured interviewing](https://rework.withgoogle.com/intl/en/guides/a-guide-to-structured-interviewing-for-better-hiring-practices).
- [US OPM — Structured interviews](https://www.opm.gov/policy-data-oversight/assessment-and-selection/structured-interviews/) and its [Structured Interview Guide (PDF)](https://www.opm.gov/policy-data-oversight/assessment-and-selection/structured-interviews/guide.pdf).
- [Behaviorally anchored rating scales — Wikipedia](https://en.wikipedia.org/wiki/Behaviorally_anchored_rating_scales) and [Precision and recall — Wikipedia](https://en.wikipedia.org/wiki/Precision_and_recall) — the measurement vocabulary behind Concept 1.

### Research and data on technical interviews specifically
- ★ [interviewing.io — Technical interview performance is kind of arbitrary. Here's the data](https://interviewing.io/blog/technical-interview-performance-is-kind-of-arbitrary-heres-the-data).
- [interviewing.io — After a lot more data, it really is kind of arbitrary](https://interviewing.io/blog/after-a-lot-more-data-technical-interview-performance-really-is-kind-of-arbitrary).
- [interviewing.io — People can't gauge their own interview performance](https://interviewing.io/blog/people-cant-gauge-their-own-interview-performance-and-that-makes-them-harder-to-hire).
- [Behroozi, Shirolkar, Barik & Parnin (2020) — Does stress impact technical interview performance?](https://doi.org/10.1145/3368089.3409712).

### AI in interviews, and the return of in-person rounds
- ★ [Hello Interview — Meta's AI-enabled coding interview: how to prepare](https://www.hellointerview.com/blog/meta-ai-enabled-coding).
- [Hello Interview — LinkedIn's AI-enabled coding interview](https://www.hellointerview.com/blog/linkedin-ai-enabled-coding) and [Shopify's AI coding interview](https://www.hellointerview.com/blog/shopify-ai-enabled-coding).
- [Canva Engineering — Yes, you can use AI in our interviews](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/).
- [CNBC via NBC New York — AI cheating in tech interviews; Pichai suggests in-person rounds](https://www.nbcnewyork.com/news/business/money-report/meet-the-21-year-old-helping-coders-use-ai-to-cheat-in-google-and-other-tech-job-interviews/6178911/).
- [CXO Digital Pulse — Google reinstates in-person interviews](https://www.cxodigitalpulse.com/google-reinstates-in-person-job-interviews-amid-rising-ai-cheating-concerns/) — Pichai's "at least one round in person".
- [GeekWire — Is it cheating? AI use during job interviews sparks debate](https://www.geekwire.com/2025/is-it-cheating-ai-use-during-job-interviews-sparks-debate-over-whether-to-restrict-emerging-tools/) — includes Amazon's statement.

### Architect-round context
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/) — reference architectures and the vocabulary architect interviewers use.
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/) — the five pillars as a review lens.
- [Architectural Katas](https://www.architecturalkatas.com/) — practice problems for decision-and-stakeholder rounds.
- [.NET application architecture guides](https://learn.microsoft.com/dotnet/architecture/) — Microsoft's official architecture e-books.

### .NET/Azure references used in this module's examples (stack precision, D2b)
- [Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/rate-limit) — limiter algorithms and partitioned limiters.
- [HybridCache library in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid).
- [Azure Service Bus — duplicate detection](https://learn.microsoft.com/azure/service-bus-messaging/duplicate-detection) — the scope that's often misstated.
- [Azure Cosmos DB — consistency levels](https://learn.microsoft.com/azure/cosmos-db/consistency-levels).
- [dotnet-counters](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-counters) and [dotnet-trace](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-trace) — the tools a deep technical round expects you to name.

### Books
- *(book)* Laszlo Bock — *Work Rules!* — Google's hiring attributes and structured-interview story from the inside.
- *(book)* Eric Schmidt & Jonathan Rosenberg — *How Google Works* — the four attributes and the 1–4 scale.
- *(book)* Gayle Laakmann McDowell, Aline Lerner, Mike Mroczka & Nil Mamano — *Beyond Cracking the Coding Interview* — how interviews are scored, with interviewing.io's data (nine chapters free on interviewing.io).
- *(book)* Daniel Kahneman, Olivier Sibony & Cass Sunstein — *Noise* — why judgements vary and how structure reduces it.
- *(book)* Geoff Smart & Randy Street — *Who* — a practitioner's structured-interviewing method.
- *(book)* Will Larson — *Staff Engineer: Leadership Beyond the Management Track* — staff scope.
- *(book)* Tanya Reilly — *The Staff Engineer's Path*.
- *(book)* Gregor Hohpe — *The Software Architect Elevator* — what architects are actually evaluated on in organisations.

### Modules to read next
- **Module 2** — Senior IC vs Architect: the two loop shapes and how this module's weights change between them.
- **Module 3** — the 7-step design framework that produces problem-navigation and driving evidence.
- **Modules 34–35** — STAR calibrated to level; the story bank.
- **Module 38** — the six dimensions as full behaviourally anchored rubrics you can self-score against.
- **Module 39** — the company research checklist built from Part D and Concept 24.

---

# Quick-recall sheet

**One sentence.** A loop is a measurement pipeline: rounds sample predefined dimensions, interviewers write evidence, a decision body reads it against a level bar and returns hire/no-hire *and* a level — produce legible, level-appropriate evidence on every dimension.

**Why rubrics.** Noisy instrument + asymmetric costs (bad hire ≫ missed hire) → optimise precision; ties go to "no"; several independent samples.

**Structured interviews.** Job analysis · same questions per role · defined rating levels · independent scoring. Best-validated method (Sackett 2022: ≈ .42 vs ≈ .19 unstructured). Dimensions before verdict (mediating assessments).

**Write-up.** If it isn't written, it didn't happen. Strong evidence = decision + reason + number + condition; failure mode unprompted; self-correction; "I"; quantified result.

**Scales.** 4-point no middle · 5-point (Amazon, reported) · 7-point (Google, reported) · binary + confidence/level (Meta, reported). Bands: clear yes / weak / clear no. A documented "no" outweighs vague "yes"es.

**Pipeline.** Recruiter screen → technical screen → loop (4–6) → debrief/committee → level → team match → offer.

**Architectures.** Central committee (Google, Meta) · Bar Raiser + live debrief (Amazon) · hiring-manager-led (Microsoft, Apple, Netflix, most others).

**Leveling.** Coding ≈ hire/no-hire; design + behavioral ≈ level. Down-levels from shallow/led design, small-scope or "we" stories. Senior: Google L5 · Meta E5 · Amazon L6 (SDE III) · Microsoft 63–64. Staff: L6 · E6 · (Amazon L7 Principal) · Microsoft 65–67.

**Packet reading.** Weakest documented round dominates hire · evidence quality multiplies · consistency is evidence · knockouts override · level from the top of the level rounds.

**Coding rubric.** Communication · problem solving · technical competency · testing. Senior: production quality, edge cases, tests unprompted, readable C# (Amazon: syntactically correct, no pseudocode).

**Design rubric.** Problem navigation · solution design · technical excellence · communication & collaboration. Senior: basics fast, candidate-led deep dives. Staff: crux, cut complexity, decide with reversal conditions, peer bandwidth.

**Behavioral.** Past behaviour → ownership, scope, judgement, results, collaboration, growth. Scope = level signal.

**Deep technical.** Precision, depth on demand, diagnosis, calibrated honesty.

**Architect.** Decision framing · options with cost/risk · stakeholders · migration · ownership · documentation · defence under challenge.

**AI-enabled.** Decomposition · verification · critical reading · ownership · fundamentals · narration. Confirm each round's AI mode.

**Dialects.** Google: GCA · RRK · Leadership · Googleyness; committee; team match; ≥ 1 in-person. Amazon: 16 LPs, Bar Raiser, STAR with data. Meta: four design focus areas; design decides level; AI-enabled coding round. Microsoft: structured + consistent framework; respect, integrity, accountability, growth mindset; team-dependent; AA.

**Six dimensions.** D1 framing · D2 depth (D2b .NET/Azure precision) · D3 trade-offs/judgement · D4 driving/communication · D5 collaboration/coachability · D6 level signal.

**Evidence sentence.** "I'll do X, because Y; I'd change to Z if W." Narrate decisions, rejected options, risks, self-corrections — not definitions.

**Knockouts.** Dishonesty · taking credit · contempt · refusing to engage · silently violating a hard requirement · can't explain own code.

**Unknown rubric.** JD → rounds/dimensions · values page → behavioral · recruiter → rounds, decision, level, AI, medium · team blog → domain.

---

# Appendix A — Company rubric translation card

```text
                 D1 Framing     D2 Depth          D3 Judgement        D4 Driving       D5 Collaboration      D6 Level
Generic coding   clarifying     tech competency   problem solving     communication    (communication)       (code quality)
                                + testing
Generic design   problem nav.   tech excellence   solution design     communication    collaboration         depth of dives
Google           GCA            RRK               GCA                 —                Googleyness           Leadership
Amazon           Customer Obs.  Dive Deep         Are Right; Invent   Deliver Results  Earn Trust; Backbone  Ownership; Think Big
                                                  & Simplify; Frugal                                         Hire & Develop
Meta             problem nav.   tech excellence   solution design     tech comm.       (behavioral)          design + behavioral
Microsoft        role reqs      real skills       real skills         —                growth mindset;       impact across teams
                                                                                       collaboration
Architect loops  decision       platform depth    options/cost/risk   documentation    stakeholders          migration; ownership
                 framing
```

# Appendix B — What the interviewer's form probably looks like

Use this to rehearse: after a mock, fill it in *as the interviewer would*, using only what you said explicitly.

```text
CANDIDATE ________  ROUND ________  LEVEL TARGET ________  DATE ________
QUESTION ASKED: ______________________   HOW FAR THEY GOT: ______________________

DIMENSION 1 ________   rating: SNH / NH / H / SH
  evidence (quotes, timestamps): _____________________________________________
DIMENSION 2 ________   rating: SNH / NH / H / SH
  evidence: ___________________________________________________________________
DIMENSION 3 ________   rating: SNH / NH / H / SH
  evidence: ___________________________________________________________________
DIMENSION 4 ________   rating: SNH / NH / H / SH
  evidence: ___________________________________________________________________

HINTS GIVEN: ______________________   CONCERNS / RED FLAGS: ______________________
LEVEL OBSERVATION:  below target / at target / above target — because ______________
OVERALL RECOMMENDATION:  Strong No Hire / No Hire / Hire / Strong Hire
SUMMARY FOR THE COMMITTEE (3–5 sentences): __________________________________________
```

# Appendix C — Recruiter questions checklist

```text
LOOP          □ Which rounds, in what order, how long each?  □ What does each round focus on?
              □ System design: infrastructure or product/API?  □ Is there a domain/.NET depth round?
              □ For architect roles: a document review, case study or presentation?
MEDIUM        □ Shared editor, IDE, or whiteboard?  □ Language choice (C# OK)?  □ Diagram tool?
RULES         □ AI policy per round (none / permitted / expected)?  □ Any round in person?
DECISION      □ Who decides — committee, debrief, hiring manager?  □ How is level determined?
              □ Is the role at a fixed level or a range?  □ Is team matching after approval?
EMPHASIS      □ Anything the team particularly values?  □ Which values/principles are assessed?
LOGISTICS     □ Practice environment available (e.g. for AI-enabled rounds)?  □ Time to decision?
              □ Can I reschedule if I need more preparation time?
```

# Appendix D — Evidence phrasebook (one card)

```text
FRAME     "Before designing — who uses it, how much, and what must it never get wrong?"
          "I'll assume X; correct me if that's wrong."   "Out of scope: Y — it doesn't change the core."
NUMBER    "That's about N per second — so [decision]."
DECIDE    "I'll do X because Y; I'd change to Z if W."
REJECT    "I considered A; it doesn't fit because B."
RISK      "The risk I'd flag is R; I'll come back to it."
DRIVE     "Plan for the next 40 minutes: …"  "Moving to the deep dives — X or Y first?"  "To summarise: …"
FAIL      "If D is down: users see U, we do P, and we alert on M."
CORRECT   "Let me correct something I said earlier — …"
UNSURE    "I don't know exactly; my assumption is A; if it's wrong, the design changes like this."
PUSHBACK  "Good point — that changes X."  /  "I see the concern; I'm keeping it because …; what would change your mind?"
STORY     "I decided …"  "The team built …; my part was …"  "The result was N, measured by M."
LEVEL     "Who runs this on call?"  "How do we get there from today's system without a freeze?"  "What does it cost?"
```

---

*Next: **Module 2 — Senior IC vs. Architect: calibrating your prep to your actual target**: the two evaluation shapes side by side, how the weight of each of the six dimensions shifts between them, how to tell from a job description and a recruiter call which loop you're actually in, and how to split your preparation time accordingly.*
