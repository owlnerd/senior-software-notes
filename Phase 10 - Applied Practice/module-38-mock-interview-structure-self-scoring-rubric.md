# Module 38 — Mock Interview Structure & a Self-Scoring Rubric
*Phase 10: Applied Practice · Senior/Architect Interview Prep for .NET & C#*

> **State of the world verified on October 8, 2026.** The method in this module is stable — it rests on decades of research into structured interviews and skill acquisition. What moves is the practice landscape and the rules about AI. The facts that calibrate the module:
>
> - **The research base.** Sackett, Zhang, Berry and Lievens (2022) re-ran the classic meta-analyses with corrected range-restriction adjustments: **structured interviews came out as the strongest single predictor of job performance (operational validity ≈ .42)**, with unstructured interviews far behind (≈ .19). The ordering survived the correction; the magnitudes shrank. This is why companies interview the way they do — same questions, a rubric, independent scoring — and why your mocks should copy that structure.
> - **Being watched changes performance.** Behroozi, Shirolkar, Barik and Parnin (ESEC/FSE 2020) ran a randomised trial: candidates solving a problem at a whiteboard **with an interviewer watching performed less than half as well** as those solving it in private, with higher measured stress and cognitive load. Your private practice overstates your interview performance; mocks exist to close that gap.
> - **You cannot grade yourself from memory.** interviewing.io's published data shows a **weak relationship between how candidates rated their own performance and how interviewers rated it**, and candidates **underestimated** themselves more often than they overestimated. Their data also shows **large interview-to-interview volatility** for the same person — one interview is a noisy measurement.
> - **Pramp now runs on Exponent.** Since **July 2024** all new Pramp sessions are hosted on **Exponent Practice**; peer mocks are still free with a **monthly allowance of free credits** (reported as five per month), with paid tiers above that. **interviewing.io** is mostly paid (anonymous mocks with senior engineers from large companies), but its **library of mock-interview replays** is free to watch. **Hello Interview** offers paid expert mocks and AI-guided practice for **system design, low-level design, AI-enabled coding and behavioral** rounds (one guided problem free). **Google's Interview Warmup** is a free, generic AI practice tool.
> - **The AI rules split the industry.** **Amazon** tells candidates not to use GenAI tools during interviews unless explicitly permitted, and recruiters can disqualify candidates who do. **Anthropic's** published candidate guidance **encourages AI for preparation and practice** but allows **no AI in live interviews or take-homes unless told otherwise**. **Google** reintroduced **at least one in-person round** for engineering candidates (announced 2025, citing AI-assisted cheating). In the other direction, **Meta** runs an **AI-enabled coding round** (one extended, multi-stage problem with an AI assistant in the environment, alongside a classic no-AI coding round), **Canva** replaced its CS-fundamentals screen with an **AI-assisted coding** interview where AI use is expected, and **LinkedIn** has been reported to run AI-enabled coding rounds too.
> - **Consequence for your practice:** every mock now has an **AI mode** that must match the real round — *no AI*, *AI permitted*, or *AI expected* — and you must ask the recruiter which applies. Using an AI **as your mock interviewer** is legitimate preparation; using one **during** a real interview that forbids it is disqualifying.
>
> Platforms, prices and policies change; the measurement discipline doesn't.

## Orientation

Here is the sentence to carry through the whole module: **a mock interview is a measurement instrument and a training stimulus at the same time — it is only useful if it reproduces the real round's pressures, scores observable evidence against a fixed rubric rather than an impression, is repeated often enough to average out the noise, and ends with exactly one targeted fix that the next mock checks.**

The curriculum entry reads: *Mock interview structure & a self-scoring rubric.* Module 37 ended with an eight-row self-scoring rubric for system design and the promise that this module would extend it into a full mock protocol for every round type in the curriculum.

**Why this module exists.** Modules 1–37 built knowledge. Interviews don't score knowledge; they score **performance of knowledge, under observation, against a clock, in a conversation you don't control**. Those are different skills, and the second one only improves through practice that resembles the real thing. Three facts make the case:

1. **Knowledge doesn't transfer to performance automatically.** You can know every trade-off in Module 37 and still spend 18 minutes on requirements, forget to say a number, or freeze when the interviewer says "what if Redis is down?". These are performance failures, not knowledge failures — and they are invisible to private study.
2. **Interviews are noisy.** A single round's outcome depends on the interviewer, the problem, your nerves and the first five minutes. You can't control the noise, but you can **shift your mean and shrink your variance** — and Concept 5 shows that shrinking variance is worth as much as raising the mean.
3. **Self-assessment is unreliable without structure.** "It went pretty well" is not data. A rubric, a recording and evidence quotes turn a feeling into a measurement you can improve against.

How this connects to earlier modules:

- **Module 1** (what companies score) and **Module 2** (IC vs architect loops) — the rubrics here are concrete, self-administrable versions of those scoring models.
- **Modules 3–5** (the design method, requirements, estimation) — the system-design rubric scores exactly those behaviours.
- **Modules 30–33** (design documents, ADRs, brownfield, cost) — the architect-round mocks in Part D are built on them.
- **Modules 34–35** (STAR calibrated to seniority, your story bank) — the behavioral mock and rubric draw directly on your story bank.
- **Module 36** (coding rounds) — the coding rubric extends Module 36's "would I want this code in production" standard.
- **Module 37** (worked design problems) — the default system-design problem set for mocks, and the origin of Appendix E's rubric, now expanded.
- **Module 39** (company/role research) — tells you which mock types and which AI mode to prioritise for a specific loop.

Why it matters in interviews:

1. **Mocks are the highest-leverage preparation activity after real interviews** — because they are the only practice that includes being watched, being interrupted and being timed.
2. **Senior and architect rounds are conversational** — the interviewer pushes back, changes requirements and asks "why?" — and you can't rehearse a conversation alone.
3. **Architect loops add round types most candidates never practise** — defending a design document, a brownfield redesign, a stakeholder trade-off conversation, a round with the engineers who'd implement your decision. If you've never done one, the first one you do is the real one.
4. **You'll be asked about hiring itself.** Staff and architect candidates are often asked how they'd interview, calibrate a hiring bar or give feedback. Running mocks properly — as interviewer *and* candidate — gives you real answers.

This module has six jobs:

1. **Explain why mocks work and what they must simulate** — the learning science, fidelity, self-assessment bias, and the statistics of noisy measurements (Part A).
2. **Define the structure of one mock session** — briefing, running, debriefing, recording and review (Part B).
3. **Build the rubric system** — design principles, the universal dimensions, round-specific behaviourally anchored rubrics, the scoring procedure, calibration and the error log (Part C).
4. **Design the mock programme** — the catalogue of mock types for every round in this curriculum, full-loop simulations, a schedule, AI interviewers, and AI-enabled and in-person rounds (Part D).
5. **Turn results into decisions** — readiness gates, plateaus, and performing on the day (Part E).
6. **Give you the tools** — a worked mock cycle, interviewer briefs, full rubrics, a debrief script, a mock log, AI interviewer and grader prompts, a six-week plan and a readiness checklist (appendices).

Seven framings to carry through:

1. **A mock is a measurement — design it like one.** Fixed rubric, observable evidence, independent scorers, known noise. Otherwise it's a chat.
2. **Fidelity is about pressure, not content.** The problem matters less than the clock, the observer, the interruptions and the ambiguity. Practise being watched.
3. **Score behaviour, not impressions.** Every score needs a quote or a timestamp behind it. "Seemed strong" is not evidence; "stated QPS and concluded one Redis shard was enough at 08:40" is.
4. **Score dimensions before the verdict.** Rate each dimension independently, *then* decide the overall recommendation — the way structured interviews and hiring committees do.
5. **One mock is one noisy sample.** Judge yourself on the rolling median of several, and treat consistency as a goal in its own right.
6. **Every mock ends with one fix.** Not five. One specific, observable behaviour, written down, and checked first in the next mock.
7. **Match the real round's rules.** Same length, same medium (whiteboard, shared editor, diagram tool), same AI mode, same level of interviewer pushback.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What a mock is for | Two jobs — measurement and training — that pull the design in different directions |
| 2 | Fidelity | Reproduce the pressures: clock, observer, interruption, ambiguity, stakes |
| 3 | The learning science | Deliberate practice, retrieval, spacing, interleaving, desirable difficulty |
| 4 | Why your impression is wrong | Weak self-rating accuracy, halo effect, first impressions — fix with structure |
| 5 | Noise and sample size | One mock is one sample; variance reduction is worth as much as mean |
| 6 | Anatomy of a session | Brief → run → silent scoring → debrief → review → log → one fix |
| 7 | Briefing the interviewer | Level, style, problem, hidden twist, probe ladder, interruption script |
| 8 | Running a mock as interviewer | Probe, don't rescue; timestamp; write verbatim |
| 9 | The debrief | Score independently first; SBI feedback; evidence; one fix |
| 10 | Recording and review | Record, transcribe, review in passes, compute a few metrics |
| 11 | Rubric design principles | Behavioural anchors, 4 points, independent dimensions, level-calibrated |
| 12 | The universal dimensions | Six dimensions every round scores — and how companies name them |
| 13 | Round-specific rubrics | System design, coding, behavioral, architect rounds |
| 14 | The scoring procedure | Evidence → dimension scores → knockouts → verdict |
| 15 | Calibration | Anchor your scale on reference recordings and partner agreement |
| 16 | The error log and fix taxonomy | Classify every miss; each class has a different remedy |
| 17 | The mock catalogue | One mock type per real round in this curriculum |
| 18 | Full-loop simulation | Rehearse the day: sequence, fatigue, recovery |
| 19 | The schedule | Baseline → targeted drills → full fidelity → taper |
| 20 | AI as interviewer and grader | Useful, available, lenient by default — separate the grader and force evidence |
| 21 | AI-enabled and in-person rounds | Practise each AI mode; practise the whiteboard again |
| 22 | Readiness gates | Rolling median, floor, consistency, per round type |
| 23 | Plateaus and regression | Diagnose by error class; change the stimulus |
| 24 | Performing on the day | Exposure, reappraisal, reset sentences, one round at a time |

---
# Part A — Why mocks work, and what they must simulate

## Concept 1 — What a mock interview is for

Start from first principles. A mock interview can do two different jobs:

| | **Measurement** | **Training** |
|---|---|---|
| Question it answers | "How would I score in a real round today?" | "How do I get better at one specific behaviour?" |
| Needs | Realism: unknown problem, real timing, a stranger, no help | Repetition and feedback on a narrow target |
| Interviewer behaviour | Neutral; never rescues; scores silently | Can pause, rewind, coach, replay a segment |
| Output | A score with evidence | A behaviour that's measurably better |
| How often | Every week or two | Several times a week |
| Analogy | A load test in a production-like environment | A focused benchmark of one hot path |

They conflict. A mock that stops to coach is a poor measurement (you'd never get that help in the real round), and a strict measurement mock is a poor teacher (you get one data point and twenty minutes of debrief). **Most people run hybrids by accident and get neither** — the partner gives hints when you stall, so the "score" is inflated, and the feedback is a vague list of everything, so nothing improves.

The fix is to **name the mode before every session**:

- **Full mock (measurement):** real format, real timing, no help, scored with the rubric, recorded. Debrief only after scoring.
- **Drill (training):** a 10–25 minute segment targeting one behaviour — "the first five minutes of requirements", "the estimation step out loud", "the 'Redis is down' follow-up", "a conflict story in under three minutes". Interrupt, rewind and repeat freely.
- **Hybrid with a rule:** run as a full mock, but the interviewer may give *one* scripted hint at a pre-agreed moment, and the score records that a hint was used (exactly as a real interviewer would note it).

### What a mock is not

- **Not a quiz on content.** If you discover a knowledge gap in a mock, that's useful data — but the remedy is study, not more mocks. Mocks are expensive; don't use them to learn material.
- **Not a confidence exercise.** A mock that only makes you feel good has failed as a measurement. Expect your first few scores to be lower than you think they "should" be — that is the gap between private practice and observed performance (Concept 2), and closing it is the point.
- **Not a performance to impress your partner.** If you're practising with a peer, the pull is to show off what you know. Resist it — perform as you would for the real interviewer, including the parts you're unsure about.

**Interview-grade sentence** *(if asked how you prepared or how you'd train interviewers)*: *"I treat a mock as a measurement or as a drill, never both at once — full mocks reproduce the real round and get scored against a fixed rubric with evidence, while short drills target one behaviour and can be paused and repeated — because a mock that coaches mid-session inflates the score and a strict mock teaches too little to change anything."*

---

## Concept 2 — Fidelity: what a mock must reproduce

**Fidelity** is how closely the practice reproduces the conditions of the real event. The intuitive mistake is to think fidelity means "the same questions". It doesn't — the content is the least important part. What degrades interview performance is a set of **pressures**, and a mock is high-fidelity to the extent it reproduces them.

### The five pressures

| Pressure | What it does to you | How the mock reproduces it |
|---|---|---|
| **Observation** | Splits attention between the problem and how you're coming across; raises cognitive load | A real person watching — camera on, preferably someone you don't know well |
| **The clock** | Forces trade-offs between depth and coverage; panic when behind | A visible timer; hard stop; the same length as the real round (45 or 60 min) |
| **Ambiguity** | You must ask, scope and decide without full information | Under-specified prompts; the interviewer answers only what's asked |
| **Interruption** | Breaks your plan; tests whether you can re-plan aloud | Scripted pushback and requirement changes at set times |
| **Stakes** | Raises arousal; triggers rumination after a mistake | Partner is a stranger or senior; the score is written down; recordings are kept |

The Behroozi et al. result (verified above) is the key evidence: the *same* problem, solved with an observer present, roughly halves performance in a lab population, with measurably higher stress and load. Two consequences follow:

1. **Private practice systematically overstates your interview performance.** Solving a design alone, at your desk, with time to think silently, measures something real — but not the thing the interview measures.
2. **The only reliable treatment is graded exposure to observation.** You don't become calm in front of an observer by reading about it; you become calm by being observed many times at increasing stakes. This is the standard logic of exposure: repeated, controlled contact with the stressor reduces the response.

### The fidelity ladder

Climb it over the preparation period. Each level adds a pressure.

| Level | Setup | Pressures present | Use for |
|---|---|---|---|
| **L0** | Alone, silent, untimed | None | Learning the material (not a mock) |
| **L1** | Alone, **out loud**, timed, recorded | Clock; mild self-observation via recording | Early drills; first full-length runs |
| **L2** | AI interviewer (voice), timed, recorded | Clock; ambiguity; interruption (if prompted) | Volume, any time of day |
| **L3** | Peer you know, camera on, timed | + Human observation | First human mocks |
| **L4** | Peer or engineer you **don't** know (platform, community), real format | + Social stakes | Most full mocks in the middle phase |
| **L5** | Senior/staff engineer or ex-interviewer at a target-class company, **unknown problem**, strict scoring, real tooling | All five | The last 2–4 measurement mocks before a real loop |

Real interviews themselves are the top of the ladder. If you have several target companies, **schedule a less-preferred company's loop first** — it's the highest-fidelity mock available. (Ethically fine as long as you'd genuinely consider the offer.)

### Medium fidelity

Practise in the medium you'll actually use. It matters more than people expect:

| Real round uses | Practise with | Why |
|---|---|---|
| Shared code editor (CoderPad, CodeSignal, HackerRank) | A plain editor with no IntelliSense, or the platform's own sandbox | C# without IntelliSense exposes what you've outsourced to tooling: `using` directives, exact API names (`TryGetValue`, `CollectionsMarshal.AsSpan`), `Span<T>` syntax |
| Online whiteboard (Excalidraw, Miro, the company's tool) | Excalidraw or tldraw, with a 45-minute clock | Drawing speed and layout are skills; a messy diagram costs minutes |
| Physical whiteboard (in-person rounds — Google again requires at least one) | A real whiteboard or large paper, standing | Writing code by hand, erasing, leaving space — very different from typing |
| Google Doc / plain doc (some design and architect rounds) | A shared doc | Writing structure under time pressure |
| AI-enabled environment | The same class of tool, at the same permission level (Concept 21) | Using AI well under time pressure is its own skill |

**Interview-grade sentence:** *"Fidelity is about reproducing the pressures, not the questions — being watched, the clock, ambiguity, interruptions and stakes — because the research shows that being observed alone can halve problem-solving performance; so I climb from timed solo runs to AI and peer mocks to strangers and senior interviewers, in the same medium the real round uses."*

---

## Concept 3 — The learning science behind a mock programme

You don't need psychology to run mocks, but five well-replicated findings explain *why* the structure in Parts B–D looks the way it does — and, more usefully, they tell you what to do when progress stalls.

### 1. Deliberate practice

Ericsson, Krampe and Tesch-Römer (1993) distinguished **deliberate practice** from mere repetition. Improvement comes from practice that has:

- **a specific, narrow goal** ("state the conclusion of every number", not "do better at system design");
- **full concentration** at the edge of current ability — tasks that are hard enough to fail sometimes;
- **immediate, informative feedback** about what went wrong;
- **repetition with refinement** — doing the same narrow thing again, adjusted.

Mapping: **drills** (Concept 1) are deliberate practice; **the one-fix rule** (Concept 9) is the "specific goal"; **the rubric with evidence** is the "informative feedback". Ten full mocks with vague feedback produce less improvement than five full mocks plus twenty targeted drills.

### 2. Retrieval practice (the testing effect)

Roediger and Karpicke (2006) showed that **retrieving information from memory strengthens it far more than re-reading it**, especially at delayed tests. A mock is a retrieval event: you must pull the latency numbers, the Cosmos DB consistency levels or your conflict story out of memory without prompts. That's why mocks also improve *knowledge* recall — and why re-reading Module 37 the night before is far less useful than answering three of its problems aloud.

### 3. Spacing

Practice distributed over time beats the same amount massed together (Cepeda et al., 2006, meta-analysis). For mocks: **two or three sessions a week for six weeks beats a dozen in the final week**. Spacing also gives the one-fix loop time to work — you need days between the diagnosis and the re-test.

### 4. Interleaving

Mixing problem types within practice (A, C, B, A, D…) rather than blocking them (A, A, A, B, B, B) makes practice feel harder and produces better transfer, because you practise **choosing** the approach, not just executing it. For mocks: don't do the five Module 37 problems in order; rotate system design, coding, behavioral and architect sessions, and vary the problem within each.

### 5. Desirable difficulties

Bjork's umbrella term for the above: conditions that **slow apparent progress during practice but improve long-term performance** — spacing, interleaving, retrieval, variation. The corollary is uncomfortable: **if your mocks feel smooth and your scores keep rising, you may be practising too easily** (familiar problems, friendly partner, no interruptions). Concept 23 uses this to diagnose plateaus.

### What this means for the programme

| Finding | Programme rule |
|---|---|
| Deliberate practice | Every full mock produces one fix; drills target that fix between mocks |
| Retrieval | Answer aloud from memory; no notes in full mocks |
| Spacing | 2–4 sessions a week over 4–8 weeks, not a final-week cram |
| Interleaving | Rotate round types and problems; never the same problem twice in a row |
| Desirable difficulty | Increase pressure deliberately (fidelity ladder); prefer unfamiliar problems for measurement |

**Interview-grade sentence** *(useful when asked about learning, mentoring or how you prepared)*: *"I structured preparation around what the learning research supports — deliberate practice on one narrow behaviour at a time with immediate feedback, retrieving answers aloud instead of re-reading, spacing sessions over weeks and interleaving problem types — and I treated practice that felt too easy as a sign I needed to raise the difficulty."*

---

## Concept 4 — Why you can't trust your own impression

After a mock, you have an impression: "that went well", "I bombed it". The impression is the least reliable data you'll produce.

### The evidence

- **Self-ratings track interviewer ratings only weakly.** interviewing.io compared candidates' self-assessments with their interviewers' ratings over hundreds, then about a thousand, interviews and found a weak fit — and, notably, more candidates **underestimated** their performance than overestimated it. Senior candidates are not immune; experience gives you better content, not better self-observation.
- **First impressions are quick and sticky.** Kahneman, Lovallo and Sibony (2019), drawing on the interview literature, describe how evaluators form a coherent impression early and then update it slowly, interpreting later evidence to fit. You do this to *yourself*: a shaky first five minutes colours how you remember the strong deep dive, and a confident start hides a weak wrap-up.
- **The halo effect.** One strong dimension (fluent communication) inflates ratings of unrelated ones (technical depth). For self-raters, the strongest halo is **how the session felt emotionally** — calm sessions get rated as good sessions.
- **Recall is reconstructive.** You remember the gist, not the timeline. Candidates routinely "remember" giving the estimation numbers they planned to give — the recording shows they skipped them.

### The fix: mediating assessments

Kahneman's original structured-interview work (Israeli Army, 1956), described in the same article, replaced holistic ratings with **separate scores on a few predefined attributes**, made independently, with the overall judgement **delayed until the attribute scores were complete**. The averaged attribute scores predicted outcomes better than the holistic rating — and a final intuitive judgement made *after* the structured scores added value on top.

That's the procedure your self-scoring must follow (Concept 14):

1. **Predefine the dimensions** (the rubric, Part C) — before the mock, not after.
2. **Score each dimension from evidence only** — a quote, a timestamp, an artefact — ideally from the recording, not memory.
3. **Score dimensions independently** — don't let "communication was great" leak into "trade-offs".
4. **Only then form the overall verdict** — and allow intuition at that step, informed by the profile.

### Two practical rules

- **Score from the recording, within 24 hours.** Memory-based self-scores are systematically biased (direction varies by person — learn yours by comparing with a partner's scores, Concept 15).
- **Write the evidence column first.** For each dimension, write two or three observations with timestamps, *then* choose the number. If you can't find evidence for a 4, it isn't a 4.

**Interview-grade sentence** *(for "how do you evaluate candidates?" in staff/architect loops)*: *"I don't trust holistic impressions — mine or anyone's — because they form in the first minutes and absorb later evidence; I score a few predefined dimensions independently from specific observations, and only then make the overall call, which is the structured approach the selection research supports."*

---

## Concept 5 — Noise, sample size and why consistency matters

### One mock is one sample

interviewing.io's data shows that the same engineer's performance varies a lot from one interview to the next — enough that a single outcome is a weak signal of ability. That has two implications: **don't over-react to one mock**, and **don't treat one good mock as proof of readiness**.

Treat your score on a round type as a random variable: a true level *μ* plus noise with standard deviation *σ* (problem fit, interviewer, nerves, sleep). Use a simple model with the 1–4 rubric scale:

```text
Standard error of your mean after n mocks:   SE = σ / √n
With σ ≈ 0.6 points:   n = 1 → 0.60    n = 4 → 0.30    n = 9 → 0.20
```

So telling a true 2.8 from a true 3.2 — the difference between "lean no hire" and "lean hire" territory — needs **at least 4–6 scored mocks per round type**. Fewer than that and you're reading noise. Use the **rolling median of the last 3–5** as your readiness signal (robust to one outlier in either direction).

### Why variance matters as much as the mean

A real loop has several rounds — say five — and hiring committees weigh every one; a single strong "no hire" round is often enough to sink a packet. Model it crudely: you pass a round if that round's score is at least 2.5 (lean hire or better), and you want a "clean" loop with no sub-bar round.

```text
Case A: μ = 3.2, σ = 0.6   P(one round < 2.5) = Φ((2.5 − 3.2)/0.6) = Φ(−1.17) ≈ 12%
        P(5 rounds all ≥ 2.5) = 0.88^5 ≈ 0.52

Case B: raise the mean   μ = 3.5, σ = 0.6   Φ(−1.67) ≈ 4.8%   → 0.952^5 ≈ 0.78

Case C: cut the variance μ = 3.2, σ = 0.4   Φ(−1.75) ≈ 4.0%   → 0.960^5 ≈ 0.82
```

**Cutting σ from 0.6 to 0.4 is worth about as much as raising your mean by 0.3 points.** The model is crude — real committees aren't strict "all must pass" gates, and round scores are correlated — but the direction is robust: **consistency is a target in its own right**. A candidate who reliably scores 3 beats one who alternates between 4 and 2.

### Where variance comes from — and how mocks reduce it

| Source of variance | Mock-programme remedy |
|---|---|
| Unfamiliar problem shape | Interleave many problems; practise the *method*, not answers (Module 37's toolbox) |
| Slow or chaotic start | Drill the first five minutes until they're automatic |
| Time collapse (deep dive at minute 35) | Timestamp every mock; drill the time budget |
| Recovery after a mistake | Practise interruptions and deliberate "you're wrong" pushback (Concept 7) |
| Nerves | Graded exposure (Concept 2); a pre-round routine (Concept 24) |
| Interviewer style mismatch | Mock with several partners of different styles |
| Fatigue late in a loop | Full-loop simulations (Concept 18) |

A **routine** — the same opening moves, the same structure, the same check-ins — is the main variance-reduction tool. It converts parts of the round from improvisation into execution, freeing attention for the parts that genuinely need thought.

**Interview-grade sentence:** *"A single interview is a noisy sample, so I judged readiness on the rolling median of several scored mocks per round type, and I worked on consistency as much as peak performance — in a five-round loop, cutting the variance of my round scores matters about as much as raising their average."*

---
# Part B — The structure of one mock session

## Concept 6 — Anatomy of a mock session

A full mock is not 45 minutes long. It's about two hours spread across two days, and **most of the value is created after the interview part ends**.

```text
T−24h   Brief         Interviewer receives the brief (Appendix A): level, round type, problem, twist, probes
T−10m   Setup         Tools open, recording on, timer visible, AI mode stated, candidate gets nothing
T+0     Run           45 or 60 minutes, exactly like the real round — no pausing, no coaching
T+45    Silent score  5–10 min: interviewer scores the rubric with evidence; candidate self-scores separately
T+55    Debrief       15–20 min: compare scores, interviewer gives SBI feedback, agree ONE fix
T+75    Swap          (peer mocks) roles reverse — you interview them, which teaches the rubric fastest
≤T+24h  Review        30–45 min: candidate reviews the recording in passes (Concept 10), re-scores from tape
≤T+24h  Log           Mock log entry (Appendix D): scores, evidence, error classes, the one fix
Between Drill         2–3 short drills on the fix
Next    Check         The next mock's brief lists the fix; the interviewer checks it first
```

### Why each step exists

- **Brief in advance.** An unprepared interviewer can't probe well and tends to rescue. A brief with a problem *and* a probe ladder turns a peer into a competent interviewer (Concept 7).
- **Silent scoring before talking.** If the debrief starts with conversation, the candidate's self-justification ("I was going to say the numbers…") anchors the interviewer's scores. Independent scores first, then comparison — the same rule hiring committees use to avoid groupthink.
- **Swap roles.** Interviewing someone else with the rubric in hand is the fastest way to internalise what each anchor *looks like*. You'll notice in others, within minutes, the habits you can't see in yourself.
- **Review from the recording.** Your memory is biased (Concept 4). The recording isn't.
- **Log and one fix.** Without a log you can't compute a rolling median or see recurring error classes. Without *one* fix, the debrief becomes a list of ten things, none of which change.

### Session hygiene

| Rule | Why |
|---|---|
| Same length as the real round, hard stop | Time management is scored; a mock that runs over hides the problem |
| Camera on for both | Observation pressure; you also practise looking up from the diagram |
| No notes, no cheat sheets | Retrieval practice; real rounds don't allow them |
| State the AI mode aloud at the start | Prevents drift; matches Concept 21 |
| Interviewer keeps timestamps | Feedback becomes "at 22:10 you…", not "at some point" |
| Agree in advance what's recorded and where it's stored | Consent and privacy — recordings show faces and voices |

**Interview-grade sentence:** *"A mock that's worth doing is a two-day loop, not a 45-minute call: a written brief for the interviewer, the round at real length with no coaching, independent silent scoring before any discussion, a short debrief that ends in one agreed fix, and my own review of the recording within a day, logged so I can see trends."*

---

## Concept 7 — Briefing the interviewer

The biggest quality difference between mocks is the interviewer, and the biggest difference between interviewers is **preparation**. A peer who has read a one-page brief for ten minutes interviews better than a senior engineer improvising.

### What a brief contains

1. **Target level and track** — "senior IC, Azure/.NET product company" or "solution architect, enterprise". It changes which anchors apply (Concept 11).
2. **Round type and format** — system design 45 min on Excalidraw; coding 45 min in a plain editor, C#; behavioral 45 min; architect design-doc defence 60 min.
3. **The problem** — and, for design rounds, the **hidden twist**: a requirement the interviewer reveals only if asked, or introduces at a set time ("it's for an internal tool with 10k links", "the downstream allows only 20 concurrent calls", "the existing system is a monolith on SQL Server").
4. **The probe ladder** — questions in order of depth for each likely deep dive (below).
5. **The interruption script** — two or three scripted interventions at set times.
6. **The AI mode** — none, permitted, expected.
7. **What to write down** — timestamps of phase changes, numbers stated, verbatim quotes of key claims, every hint given.
8. **The candidate's current fix** — what to check first, from the last mock's log.

### The probe ladder

Senior rounds are scored on depth, and depth is revealed by successive "why"s. Give the interviewer a ladder per topic — each rung goes one level deeper:

```text
Rung 1  WHAT     "What would you use for the link store?"
Rung 2  WHY      "Why that over Azure SQL?"
Rung 3  NUMBERS  "How many RU/s at peak? Will it fit one partition's limits?"
Rung 4  FAILURE  "What happens when that region is down? When the cache is cold?"
Rung 5  CHANGE   "Traffic goes 10×. What breaks first?"
Rung 6  PRECISION (stack) "How exactly does HybridCache behave across 30 instances?"
```

A candidate who reaches rung 5–6 with sound answers is showing senior depth; one who stalls at rung 2 ("it's more scalable") isn't. The interviewer's job is to climb until the candidate stalls, then note where.

### Interviewer styles to rotate

Real interviewers differ. Rotate styles across mocks so none surprises you:

| Style | Behaviour | What it trains |
|---|---|---|
| **Neutral** | Answers questions, says little, lets you drive | Driving the round; filling silence productively |
| **Probing** | Climbs the ladder hard on every choice | Depth and precision |
| **Challenging** | Disagrees ("I don't think that will scale"), sometimes wrongly | Holding a position with evidence; updating gracefully when wrong |
| **Time-pressuring** | "We have ten minutes — what's most important?" | Prioritisation; fast summaries |
| **Collaborative** | Thinks with you, offers ideas | Using input without being led; crediting and evaluating suggestions |
| **Silent / distracted** | Minimal reactions, looks away, types | Not depending on approval cues |

### The interruption script

Example for a 45-minute system design mock:

```text
~12:00  Requirement change:  "Actually, 20% of traffic is from one tenant."
~25:00  Pushback:            "I'm not convinced the cache helps here. Convince me."
~35:00  Time pressure:       "We have ten minutes left. What would you want me to remember?"
```

The candidate doesn't see the script. The debrief checks how each was handled (Concept 9).

### Where to find interviewers

| Source | Cost | Fidelity | Notes |
|---|---|---|---|
| Colleague or friend (senior engineer) | Free | L3 | Good probing if briefed; social stakes low |
| Peer platforms (Pramp on Exponent Practice) | Free credits monthly | L3–L4 | Strangers; quality varies — brief them anyway via chat |
| Engineering communities (local .NET user groups, Discord/Slack interview-prep communities) | Free | L4 | Swap mocks with people at your level |
| Paid expert mocks (interviewing.io, Hello Interview, Exponent coaches) | Paid | L5 | Worth it for the last 1–3 measurement mocks |
| AI interviewer | Free–low | L2 | Volume and availability; lenient unless constrained (Concept 20) |

**Interview-grade sentence** *(for staff/architect "how would you train interviewers?" questions)*: *"Interviewer quality comes from preparation more than seniority — I'd give every interviewer a brief with the level, the problem and its hidden twist, a probe ladder that climbs from 'what' to 'why' to numbers, failure and change, a short interruption script and a list of what to write down, and I'd rotate interviewer styles so candidates aren't calibrated to just one."*

---

## Concept 8 — Running a mock as the interviewer

You'll spend roughly half your peer-mock time as the interviewer. Done well, it's training for you too: you learn the rubric by applying it, and you see the habits you share with your partner.

### Seven rules

1. **Don't rescue.** When the candidate stalls, wait. Count to ten silently. Then ask a neutral question ("what are you thinking about?"). Only give a hint if the brief allows one — and log it. Real interviewers give hints and *score them*; your mock must too.
2. **Answer only what's asked.** If the candidate doesn't ask about read/write ratio, don't volunteer it. Not asking is evidence.
3. **Climb the ladder.** On every major choice, go one rung deeper than feels polite. You're helping by finding the edge.
4. **Keep the clock honest.** Signal the time once, at the same point the real round would (often around 10 minutes before the end), and stop on time.
5. **Write verbatim.** "Said 'it's more scalable' without numbers at 17:30" is useful; "vague on scaling" isn't. Two to four quotes per dimension.
6. **Timestamp phase changes.** Requirements done, estimation done, high-level done, deep dive 1 start, deep dive 2 start, wrap-up. These timestamps are the single most diagnostic data in a design mock.
7. **Stay in role until the end.** No "great point!" mid-round; no reassurance. Neutral warmth is fine ("OK, go on").

### Note-taking template (system design)

```text
TIMELINE   req __:__  est __:__  api __:__  data __:__  HLD __:__  DD1 __:__ (topic ____)  DD2 __:__  wrap __:__
NUMBERS    stated: _______________________________   conclusions drawn from them: __________________
QUOTES     framing: "…" (mm:ss)   trade-off: "…" (mm:ss)   failure: "…" (mm:ss)   .NET claim: "…" (mm:ss)
LADDER     topic → highest rung reached: ____ / ____ / ____
HINTS      (mm:ss) what was given: ___________
SCRIPT     req change handled? ____   pushback handled? ____   time pressure handled? ____
```

**Interview-grade sentence:** *"When I interview — in a mock or for real — I don't rescue a candidate who stalls, I answer only what's asked, I go one level deeper on each decision until I find the edge, and I write down timestamps and verbatim quotes, because scores without evidence can't be calibrated or defended in a debrief."*

---

## Concept 9 — The debrief

### The order matters

```text
1. Silent, independent scoring (both)           — 5–10 min, before any discussion
2. Candidate goes first: one thing that went well, one thing that didn't (60 seconds)
3. Interviewer reads the timeline and numbers  — facts, no judgement
4. Compare scores dimension by dimension       — discuss every gap of ≥ 1 point, with evidence
5. Interviewer gives 2–3 pieces of SBI feedback
6. Agree ONE fix: a specific, observable behaviour for the next mock
7. Interviewer states the overall verdict they would have written (strong/lean hire/no hire)
```

Candidate-first (step 2) surfaces your self-perception before it can be anchored by the interviewer's view — it's how you learn your personal bias over time (Concept 15).

### SBI feedback

The Center for Creative Leadership's **Situation–Behaviour–Impact** model is the simplest structure that keeps feedback specific and non-personal:

- **Situation:** *"At 21:40, when I asked about the cache being cold after a deploy…"*
- **Behaviour:** *"…you described cache-aside again rather than answering what happens to the database in the first minute…"*
- **Impact:** *"…so I couldn't tell whether you'd considered a stampede, and I'd have marked failure handling as 'only when asked'."*

CCL's extension (**SBII**) adds **Intent** — ask what the candidate was trying to do ("what were you aiming for there?"). In mocks that question often reveals the real error class: "I knew about stampedes but thought I'd run out of time" is a time-management error, not a knowledge gap (Concept 16).

### Why "one fix"

The feedback-intervention literature (Kluger & DeNisi, 1996, a large meta-analysis) found that feedback improves performance on average but **makes it worse in a substantial minority of cases** — particularly when it directs attention to the person ("you're not senior enough") rather than the task, or when it's diffuse. Narrow, task-focused, actionable feedback is the safe kind. Hence:

- **Good fix:** *"State a conclusion after every number — 'so one Redis shard is enough'."* Observable, checkable next time.
- **Good fix:** *"Have the high-level diagram complete by minute 22, even if rough."*
- **Bad fix:** *"Be more senior."* *"Communicate better."* *"Know more about Cosmos DB."* (the last is a study item, not a mock fix)

The next mock's brief starts with the fix, and the interviewer marks it **fixed / partially / not fixed** before anything else.

**Interview-grade sentence:** *"I run a debrief in a fixed order — independent scores first, the candidate's own view next, then facts from the timeline, then feedback in situation-behaviour-impact form — and it ends with one specific, observable fix that the next session checks first, because diffuse or personal feedback often doesn't help and sometimes hurts."*

---

## Concept 10 — Recording and reviewing

### Record everything (with consent)

Video plus the diagram or editor, both participants' audio. Tools: the meeting platform's recorder, or **OBS Studio** for full control (it can record your microphone on a separate audio track, which makes transcripts cleaner). Agree in advance where recordings live and when they're deleted.

### Transcribe locally

**OpenAI's Whisper** (open source) or **whisper.cpp** transcribe an hour of audio on a laptop and emit JSON with timestamped segments. A local transcript means the recording never leaves your machine — worth doing when a partner's face and voice are on it.

### Review in passes

Watching a recording "in general" is unpleasant and low-yield. Do focused passes, each looking for one thing:

| Pass | Speed | Look for |
|---|---|---|
| **1. Timeline** | 2× | Phase-change timestamps; when the high-level design was complete; time per deep dive |
| **2. Numbers** | Transcript search | Every number stated; whether each produced a stated conclusion |
| **3. Claims** | Transcript | Every technical claim — especially .NET/Azure ones. Is each *precise and correct*? ("HybridCache coalesces per process" ✓; "Cosmos is strongly consistent" ✗ — it depends on the account's level) |
| **4. Delivery** | 1×, sampled | Silences > 20 s without narration, filler density, monologues > 3 min without a check-in, looking away from the camera |
| **5. Interruptions** | 1× around script times | How you handled the requirement change, the pushback, the time call |

Then **re-score from the tape** and compare with your memory score. The difference is your bias (Concept 15).

### A few metrics worth computing

Interviewers don't count filler words, so don't obsess over delivery metrics — but a few numbers trend usefully across mocks:

| Metric | Why it matters | Rough target |
|---|---|---|
| Time to complete high-level design (45-min SD) | Deep dives carry the weight | ≤ 22 min |
| Longest silence without narration | Silence reads as stuck | < 20–30 s |
| Numbers stated with a conclusion / numbers stated | "Every number produces a decision" | ≥ 80% |
| Questions you asked in the first 5 min | Requirements discipline | 4–8 |
| Ladder rung reached on the main deep dive | Depth | 5+ |
| Hints used | Independence | 0–1 |

A short C# tool can pull several of these from Whisper's JSON output — a 15-minute exercise, and the kind of thing you'll actually reuse:

```csharp
// dotnet run -- mock.json "per second|QPS" "high level|high-level" "deep dive"
using System.Text.Json;
using System.Text.RegularExpressions;

if (args.Length == 0) { Console.WriteLine("usage: <whisper.json> [marker regex...]"); return; }

var transcript = JsonSerializer.Deserialize<Transcript>(
        File.ReadAllText(args[0]), new JsonSerializerOptions(JsonSerializerDefaults.Web))
    ?? throw new InvalidOperationException("Empty transcript.");

var segments = transcript.Segments.OrderBy(s => s.Start).ToList();
double minutes = (segments[^1].End - segments[0].Start) / 60.0;

var fillers = new Regex(@"\b(um+|uh+|you know|basically|kind of|sort of)\b",
                        RegexOptions.IgnoreCase | RegexOptions.CultureInvariant);
int fillerCount = segments.Sum(s => fillers.Count(s.Text));
int words = segments.Sum(s => s.Text.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length);

var longestGap = segments.Zip(segments.Skip(1), (a, b) => (At: a.End, Gap: b.Start - a.End))
                         .MaxBy(g => g.Gap);

Console.WriteLine($"Duration {minutes:F1} min · {words / minutes:F0} words/min · fillers {fillerCount / minutes:F1}/min");
Console.WriteLine($"Longest silence {longestGap.Gap:F0} s at {Clock(longestGap.At)}");

foreach (var marker in args.Skip(1))
{
    var rx = new Regex(marker, RegexOptions.IgnoreCase | RegexOptions.CultureInvariant);
    var first = segments.FirstOrDefault(s => rx.IsMatch(s.Text));
    Console.WriteLine(first is null ? $"'{marker}': never said" : $"'{marker}': first at {Clock(first.Start)}");
}

static string Clock(double seconds) => $"{(int)seconds / 60:D2}:{(int)seconds % 60:D2}";

public sealed record Segment(double Start, double End, string Text);
public sealed record Transcript(List<Segment> Segments);
```

Notes worth knowing: `JsonSerializerDefaults.Web` gives case-insensitive, camel-case binding, which matches Whisper's `segments`/`start`/`end`/`text` keys; `Regex.Count` exists from .NET 7; gaps measure silence across *both* speakers unless you transcribe only your own track. Markers like "high level" are proxies — the interviewer's timestamps are the authoritative timeline.

**Interview-grade sentence:** *"I review mock recordings in focused passes — timeline, numbers, technical claims, delivery and how I handled interruptions — re-score from the tape rather than memory, and track a handful of metrics like time to a complete high-level design and how many of my numbers produced a stated conclusion."*

---
# Part C — The rubric system

## Concept 11 — Rubric design principles

A rubric is a measuring instrument; build it the way you'd build any instrument — with defined units, known resolution and checks on reliability.

### Principle 1 — Behavioural anchors, not adjectives

A scale labelled "poor / fair / good / excellent" invites impressions. A **behaviourally anchored rating scale (BARS)** describes, for each level, **what the candidate observably did**. The US Office of Personnel Management's structured-interview guidance, and Google's re:Work guide, both build interviews around rubrics with defined, job-related standards for each rating level.

| Adjective scale (bad) | Behavioural anchor (good) |
|---|---|
| Estimation: "good" | "Stated peak QPS, storage and one hot-key figure; each produced a stated decision (e.g. 'fits one node', 'needs partitioning')" |
| Communication: "clear" | "Announced each phase; checked in at least twice; summarised before deep dives; no unnarrated silence > 30 s" |

### Principle 2 — Four points, no middle

Use **4 = strong hire, 3 = lean hire, 2 = lean no hire, 1 = strong no hire** (as in Module 37's Appendix E). An even number removes the "neutral" escape and forces a direction, which is exactly what real interviewers must do in their write-up. If you need more resolution later, add half-points — but anchor only the four whole points.

### Principle 3 — Independent dimensions

Each dimension should measure one thing, observable separately. "Communication and technical depth" as one row is two rows. Overlap causes double-counting and halo (Concept 4).

### Principle 4 — Evidence required

Every score has an evidence cell: quotes and timestamps. **No evidence → no score above 2.** This single rule removes most self-assessment inflation.

### Principle 5 — Level-calibrated anchors

The same behaviour can be a 4 at mid-level and a 2 at staff. Write anchors **for your target level**, and keep a note of how they shift:

| Dimension | Senior (IC) "3" looks like | Staff/architect "3" looks like |
|---|---|---|
| Requirements | Separates functional/non-functional; picks the numbers that shape the design | …and asks who the users are, what exists today, what failure costs, who'll operate it |
| Trade-offs | Compares two or three options with reasons and a choice | …and states when the choice would reverse, its cost and ownership implications |
| Scope | Designs the component well | Places it in the system and the organisation: migration path, team boundaries, risk |

### Principle 6 — A few knockouts

Some behaviours sink a round regardless of other scores. Name them in advance so you score them consistently (Concept 14): for example, *a design that loses money or double-charges with no awareness*, *code that doesn't run with no attempt to test it*, *a behavioral story where the candidate takes credit for others' work or blames the team throughout*, *dismissing a stakeholder concern without engaging*.

### Principle 7 — Stable over time

Don't edit the anchors after every mock. Version the rubric; change it deliberately (e.g. after calibration, Concept 15), and note which version scored which mock. Otherwise your trend line measures rubric drift.

**Interview-grade sentence:** *"A good rubric has behaviourally anchored levels rather than adjectives, an even four-point scale so every score commits to a direction, dimensions that measure one thing each, an evidence requirement for every score, anchors calibrated to the target level, a short list of knockouts, and versioning so scores stay comparable over time."*

---

## Concept 12 — The universal dimensions

Every round in a senior or architect loop — design, coding, behavioral, architect — scores some version of the same six things. Learn them once; each round-specific rubric (Concept 13) is these six, specialised.

| # | Dimension | The question it answers | Typical evidence |
|---|---|---|---|
| **D1** | **Framing and scoping** | Did they understand the real problem and bound it? | Clarifying questions; stated assumptions; explicit out-of-scope list |
| **D2** | **Technical depth and precision** | Is what they say correct, deep and specific — including the stack? | Ladder rung reached; precise semantic claims; no hand-waving |
| **D3** | **Trade-offs and judgement** | Do they choose well among real options, and know when they'd choose differently? | Options compared; choice with reasons; reversal conditions; numbers |
| **D4** | **Driving and communication** | Did they run the round — structure, time, signposting, check-ins? | Phase announcements; time budget met; summaries; diagrams that read |
| **D5** | **Collaboration and coachability** | How do they handle input, hints and disagreement? | Evaluates suggestions; updates on evidence; holds a position on evidence; credits |
| **D6** | **Level signal** | Is the scope, ownership and impact appropriate for the target level? | Org/cost/migration awareness (architect); impact framing; reasoning beyond the component |

### How companies name the same things

Companies publish different vocabularies; they map onto these six closely. (Exact internal rubrics aren't public and change; treat the mapping as orientation, not inside information.)

| Company vocabulary | Maps mostly to |
|---|---|
| **Google** — historically described as general cognitive ability, role-related knowledge, leadership, and "Googleyness"; scored with structured rubrics (re:Work describes the same questions and the same scale for candidates for one role) | D1+D3 · D2 · D5+D6 · D5 |
| **Amazon** — Leadership Principles assessed in every round, with a Bar Raiser from outside the team; behavioral answers expected in STAR form | D6 and D5 dominate behavioral rounds; "Dive Deep" ≈ D2; "Are Right, A Lot" ≈ D3; "Ownership", "Deliver Results" ≈ D6 |
| **Meta** (commonly reported for design) — problem navigation, solution design, technical excellence, technical communication | D1 · D3 · D2 · D4 |
| **Architect loops generally** — decision quality, stakeholder alignment, documentation, risk and migration | D3 · D5+D6 · D4 · D6 |

The point of the mapping: you don't need a different preparation per company — you need the six dimensions, plus company-specific emphasis from Module 39's research (for example, mapping every behavioral story to Leadership Principles for Amazon).

### The .NET/Azure precision sub-dimension

For a .NET-focused loop, split D2 into **D2a general technical depth** and **D2b stack precision** when scoring design and coding mocks. Stack precision is where you differentiate (Module 37, Concept 4) and where vague claims are easiest to catch. Evidence for D2b is a list of every .NET/Azure claim you made, each marked ✓ precise, ~ vague, ✗ wrong.

**Interview-grade sentence:** *"Under different names, most loops score six things — framing, technical depth, trade-off judgement, driving the conversation, collaboration and level-appropriate scope — so I prepared against those six with round-specific anchors, and for .NET roles I scored stack precision separately because that's where generic answers are easiest to spot."*

---

## Concept 13 — Round-specific rubrics

Below are the dimensions and the "3" and "4" anchors for each round type. The complete four-level anchors are in **Appendix B**.

### System design (45–60 min) — extends Module 37, Appendix E

| Dimension | 3 — lean hire | 4 — strong hire |
|---|---|---|
| Requirements (D1) | Functional + key NFRs; scope stated | Asked purpose and the design-shaping NFRs; found the twist; stated what's out of scope |
| Estimation (D3) | Correct numbers, most used | Few numbers, each producing a stated decision |
| High-level design (D2/D4) | Standard design, mostly justified; done by ~25 min | Every component justified; data flow and consistency per arrow; done by ~22 min |
| Deep dives (D2) | One solid deep dive | Two deep dives where the difficulty lives; alternatives, choice, failure modes; ladder rung 5+ |
| Failure handling (D3) | Covered when asked | Each dependency's failure and policy stated unprompted |
| Stack precision (D2b) | Correct products with reasons | Exact semantics and limits; default → why → when not |
| Pivots / wrap-up (D3/D6) | Reasonable pivot | Seams named; trade-off with a number; next scaling step |
| Driving (D4) | Clear, mostly self-directed | Drove the round; offered deep-dive choices; checked in; no time collapse |

### Coding (45 min, C#) — extends Module 36

| Dimension | 3 — lean hire | 4 — strong hire |
|---|---|---|
| Problem understanding (D1) | Clarified inputs and outputs; one or two edge cases | Clarified contract, constraints and edge cases before coding; restated with an example |
| Approach (D3) | Reasonable approach with complexity stated | Compared approaches with complexity; chose deliberately; mentioned what would change at scale |
| Code quality (D2) | Readable; sensible names; some decomposition | Idiomatic modern C# (records, pattern matching, `Span<T>` only where it earns it); small functions; clear types; no premature cleverness |
| Correctness and testing (D2) | Runs on the main case; fixes bugs when found | Walked through tests unprompted, including edge cases; found own bugs; explained invariants |
| Communication (D4) | Explained while coding, mostly | Narrated intent before code; summarised; kept the interviewer oriented |
| Speed (D4) | Finished the core problem | Finished with time for an extension or test discussion |
| AI use (AI-enabled rounds only) | Used the assistant for boilerplate; reviewed output | Decomposed the task for the assistant; verified every suggestion; caught and fixed its mistakes; stayed in control of design |

### Behavioral (45 min) — extends Modules 34–35

Score **each story** on these dimensions, then the round on the median plus coverage.

| Dimension | 3 — lean hire | 4 — strong hire |
|---|---|---|
| Fit to the question (D1) | Story answers the question asked | Story answers the question *and* the competency behind it; chose the right story from the bank |
| Ownership and "I" (D6) | Clear personal role | Clear personal decisions and actions, distinguished from the team's; scope at or above target level |
| Complexity and stakes (D6) | Real problem with constraints | Ambiguous, cross-team or high-stakes problem; trade-offs and risk explicit |
| Actions and reasoning (D3) | Actions described | Actions *and* why — options considered, what was rejected and why |
| Result (D3) | Outcome stated | Outcome quantified; impact on system/team/business; what happened afterwards |
| Reflection (D5) | Some learning | Specific learning applied later; honest about own mistakes |
| Delivery (D4) | 2–4 min; structured | ~2–3 min core; tight; handles follow-ups with depth (the second and third "why") |

### Architect rounds (60 min) — extends Modules 30–33

Four common formats; one rubric with format-specific emphasis.

| Dimension | 3 — lean hire | 4 — strong hire |
|---|---|---|
| Decision framing (D1) | States the decision and constraints | Frames the decision, its drivers, its reversibility and who's affected |
| Options and trade-offs (D3) | Two options compared | Credible options with consequences, cost and risk; recommendation with reversal condition |
| Stakeholders (D5/D6) | Acknowledges stakeholders | Speaks to each audience's concern (engineering, product, finance, ops) in their language; handles dissent |
| Migration and risk (D6) | Mentions phasing | Risk-first sequencing; strangler/dual-run/shadow; rollback; measurable gates |
| Cost and ownership (D6) | Mentions cost | Names cost drivers, ownership, on-call burden, build-vs-buy rationale |
| Documentation (D4) | Clear explanation | Decision recorded (ADR shape); C4-level diagrams appropriate to audience |
| Defence under challenge (D5) | Answers challenges | Concedes valid points, defends others with evidence, adjusts the design visibly |

**Interview-grade sentence:** *"Each round type uses the same six dimensions with its own anchors — design rounds weight deep dives and failure handling, coding rounds weight correctness and testing over cleverness, behavioral rounds weight ownership and quantified results, and architect rounds weight decision framing, stakeholders and migration risk."*

---

## Concept 14 — The scoring procedure

The order of operations is what makes a score trustworthy. Follow it every time — as interviewer and when you self-score from the recording.

```text
1. EVIDENCE    For each dimension, write 2–4 observations with timestamps. Facts only.
2. SCORE       Choose the anchor that best matches the evidence. No evidence → max 2.
3. KNOCKOUTS   Check the knockout list. A knockout caps the overall verdict at 2.
4. HINTS       Note hints used and where; a hint on the core of the problem costs a level on that dimension.
5. VERDICT     Only now: overall recommendation (4/3/2/1) — NOT the average.
6. JUSTIFY     One paragraph a hiring committee could read: strongest evidence for and against.
7. FIX         The one behaviour to change next time.
```

### Why the verdict is not the average

Averages hide the pattern a committee cares about. Consider two profiles on a design round:

```text
Candidate A:  Req 3  Est 3  HLD 3  Deep 3  Fail 3  .NET 3  Pivot 3  Drive 3   → mean 3.0
Candidate B:  Req 4  Est 4  HLD 4  Deep 2  Fail 2  .NET 4  Pivot 3  Drive 1   → mean 3.0
```

A is a solid lean hire. B is probably *not*: the deep dives — where senior signal lives — were weak, and "drive 1" usually means the interviewer had to run the round. The verdict step lets you weigh dimensions by their importance for the round and level, the way Kahneman's mediating-assessments approach allows informed intuition *after* the structured scores.

**Default weights** to keep in mind (not to compute mechanically):

| Round | Heaviest dimensions |
|---|---|
| System design | Deep dives, trade-offs, failure handling |
| Coding | Correctness/testing, approach, code quality |
| Behavioral | Ownership/level, actions and reasoning, result |
| Architect | Decision framing, trade-offs, stakeholders, migration |

### The committee test

Finish with the question real interviewers effectively answer in their write-up: ***"Would I argue for this candidate in a hiring committee, at this level, using only the evidence I wrote down?"*** If you'd need to say "trust me, they were good", the evidence is thin and the score should drop.

**Interview-grade sentence:** *"I score from evidence to dimensions to verdict in that order — observations with timestamps, then each dimension against its anchors, then knockouts and hints, and only then the overall recommendation, which weighs the dimensions that matter most for the round rather than averaging them — and I check it by asking whether I could defend it to a committee with only what I wrote down."*

---

## Concept 15 — Calibration

An uncalibrated scale drifts: your "3" this week isn't your "3" next month, and it isn't your partner's "3". Calibration makes scores comparable — across time, across scorers, and against the real bar.

### Three calibration checks

**1. Against reference recordings (external anchor).** Watch two or three published mock interviews — interviewing.io's free replay library, Hello Interview's and Exponent's YouTube mocks — and score them with your rubric *before* reading or hearing the published feedback or outcome. Then compare. If you consistently score higher than the experts' verdicts, your scale is lenient.

**2. Against a partner (inter-rater agreement).** Both of you score the same recording independently, then compute:

```text
exact agreement   = dimensions with identical scores / total dimensions
within-one        = dimensions differing by ≤ 1 / total dimensions
Targets after 2–3 rounds of calibration: exact ≥ 60%, within-one ≥ 90%
```

(If you want a chance-corrected statistic, Cohen's weighted kappa is the standard one — but simple agreement is enough for two people.) Discuss every 2-point gap until you agree what the anchor means; write the resolution into the rubric as an example. That's how anchors get sharp.

**3. Against yourself (test–retest).** Re-score an old recording two or more weeks later without looking at the original scores. Differences show **scale drift** (your standards moved) — useful to know before comparing old and new scores.

### Know your personal bias

Track, per dimension, **self-score minus partner score** across mocks. After five or six mocks you'll see a pattern — for example "I rate my communication +0.8 above my partners and my deep dives −0.5 below". That's your personal correction: apply it when you can only self-score (AI mocks, solo runs). The interviewing.io data suggests underestimation is common, so don't assume your bias is upward.

### Calibrating to the real bar

After real interviews, log what you can: the outcome, any feedback the recruiter shares, and your own immediate post-round self-score. Over several real rounds you learn how your mock scores map to real outcomes — the most valuable calibration of all, and the reason to keep logging after you've started interviewing for real.

**Interview-grade sentence** *(for "how do you calibrate interviewers?")*: *"I calibrate scorers three ways: score published reference interviews before seeing their verdicts, have pairs score the same recording independently and resolve every two-point gap by sharpening the anchor, and re-score old recordings later to catch drift — and I track each scorer's systematic bias per dimension so their scores can be read correctly."*

---

## Concept 16 — The error log and the fix taxonomy

Every miss in a mock belongs to one of a few **classes**, and each class needs a different remedy. Misclassifying is the most common reason people plateau: they respond to every miss with "study more", when most senior-level misses aren't knowledge gaps at all.

| Class | What it looks like | Diagnostic question | Remedy |
|---|---|---|---|
| **K — Knowledge gap** | Didn't know it (e.g. Service Bus duplicate detection's scope) | Could you explain it now, given time and no pressure? → *No* | Study the module; make a flash card; add a drill |
| **R — Retrieval failure** | Knew it, couldn't produce it under pressure | Could you explain it now? → *Yes, easily* | Retrieval drills: answer aloud from memory, spaced |
| **S — Structure / time** | Right content, wrong order or too late (deep dive at minute 35) | Was the content there, just mistimed? | Drill the time budget; checkpoint clock glances; scripted transitions |
| **C — Communication** | Did it, but the interviewer couldn't see it (silent reasoning, unlabelled diagram) | Did the reasoning happen without being said? | Narration drills; "conclusion first" sentences; diagram labelling |
| **J — Judgement** | Made a defensible-sounding but poor choice (2PC across a PSP; 301 redirects) | Would an expert disagree with the choice, not the explanation? | Study the trade-off; rehearse the decision sentence (default → why → when not) |
| **P — Pressure** | Froze, rushed or rambled; performance well below private practice | Would you have done it fine alone? | Exposure (fidelity ladder); pre-round routine; recovery script (Concept 24) |
| **L — Level** | Correct but scoped too low for the target (no migration, cost, org thinking for an architect round) | Did you answer as a senior engineer when they wanted a staff/architect? | Add the architect layer (Modules 30–33); level-specific drills |

### Writing log entries

Each entry: **what happened (with timestamp) → class → remedy → drill**. For example:

```text
22:40  Asked "what if the PSP times out?" — said "we retry".        Class J (and K?)  
       Check: could explain unknown-outcome state now? Yes, after a moment → J + R
       Remedy: rehearse Module 37 Q11 aloud ×3 this week; add to deep-dive menu card
       Drill: 5-min "PSP timed out" follow-up, partner asks 3 variants
```

### Patterns that matter

After four or five mocks, count classes across the log. The distribution tells you what kind of preparation you need next:

| Dominant class | It means | Shift your time toward |
|---|---|---|
| K | Content gaps remain | Study; fewer full mocks for now |
| R | Content is there but fragile | Short retrieval drills, daily |
| S / C | Performance mechanics | Drills on structure and narration; record more |
| J | Shaky trade-off reasoning | Decision-sentence drills; expert mocks for feedback |
| P | Pressure | More, higher-fidelity mocks; strangers; routine |
| L | Under-levelling | Architect-layer practice; level-specific rubrics |

**Interview-grade sentence:** *"I classified every miss — knowledge, retrieval, structure, communication, judgement, pressure or level — because they need different remedies, and the distribution across my log told me whether to study, drill retrieval, practise mechanics or raise the pressure; most of my senior-level misses turned out not to be knowledge gaps at all."*

---
# Part D — The mock programme

## Concept 17 — The mock catalogue: one mock type per real round

The curriculum's orientation listed the rounds in each loop. For senior/staff IC: a coding screen, one or two system design rounds, a deep technical round, behavioral. For architect: a design-document review, a brownfield/incremental-redesign exercise, a round with the engineers who'd implement the decision, and a stakeholder trade-off conversation. **Each needs its own mock type** — they test different things, and skill in one doesn't transfer automatically.

| Mock type | Source modules | Length | Medium | Interviewer | Problem bank |
|---|---|---|---|---|---|
| **System design** | 3–13, 26–29, 37 | 45–60 | Excalidraw / tldraw | Senior+ engineer | Module 37's five; plus chat/messaging, news feed, file storage, job scheduler, typeahead, metrics pipeline, booking/ticketing |
| **Coding (no AI)** | 36 | 45 | Plain editor, C# | Any engineer | LRU cache, token bucket, ledger with money type, interval merging, log parsing, an iterator/stream API, a thread-safe producer–consumer with `Channel<T>` |
| **Coding (AI-enabled)** | 36 + Concept 21 | 45–60 | Editor + assistant | Any engineer | Extend a small existing codebase in stages: add a feature, add tests, fix a seeded bug, handle a new requirement |
| **.NET deep technical** | 14–19, 25 | 45 | Conversation + snippets | .NET-experienced engineer | Rapid probes (GC, async, DI lifetimes, EF Core tracking) then one deep thread: "this service's p99 doubled after a deploy — walk me through it" |
| **Behavioral** | 34–35 | 45 | Conversation | Ideally a manager | Your story bank × question list; for Amazon, Leadership Principles; senior/staff follow-up ladders |
| **Design-document defence** | 30–31 | 60 | A doc you wrote (2–4 pages) | Two reviewers if possible | Your own design doc for one Module 37 problem, or a real past design (anonymised) |
| **Brownfield / incremental redesign** | 32 | 60 | Diagram + conversation | Senior+ engineer | "Here's our monolith on SQL Server and App Service; we need X — how do we get there without a rewrite?" |
| **Implementer round** | 30–33 | 45 | Conversation | Two engineers playing the team | "You've decided on event-driven integration. We're the team. Convince us, and answer our concerns." |
| **Stakeholder trade-off** | 33 | 45 | Conversation | Someone playing a product/finance lead | "Product wants the feature in six weeks; your design needs twelve. Walk me through options." |

### Writing good problems for the architect mocks

Architect mocks need more preparation from the interviewer, because the "problem" is a situation, not a prompt. A good brief contains:

- **The current state** — a short C4 context and container description (Module 31): "a .NET Framework 4.8 monolith on IIS, one SQL Server, nightly batch integration with an ERP".
- **The pressure** — a business driver with a date and a number: "expand to three EU markets by Q3; peak orders 5× today".
- **Constraints** — team size and skills, budget, compliance, a dependency you can't change.
- **Planted conflicts** — the implementer who'll push back on event-driven complexity; the finance lead who wants cost cut by 30%; the security architect who won't accept a new data store without a threat model.
- **What a strong answer would include** — written by the interviewer *before* the mock, so scoring is anchored.

### The implementer round deserves special attention

It's the round most candidates never practise and the one where architects most often fail: a technically sound decision delivered as a lecture, to engineers who will have to build and run it. What's scored:

- **Listening before defending** — asking the team what worries them.
- **Separating concerns that change the decision from those that change the plan** — "that's a real risk; it doesn't change the choice, but it changes the order we build it in".
- **Concessions** — visibly adjusting the design when a concern is right.
- **Ownership transfer** — leaving the team with decisions they can make themselves (ADRs, guardrails), not dependence on the architect.

**Interview-grade sentence:** *"I mocked every round type in the target loop separately — system design, coding with and without AI, a .NET deep-technical round, behavioral, and for architect loops a design-document defence, a brownfield redesign, a round with the engineers who'd implement the decision and a stakeholder trade-off conversation — because skill in one round doesn't transfer automatically to the others."*

---

## Concept 18 — Full-loop simulation

A real onsite (or virtual onsite) is four to six rounds in a day. Two things appear only at loop scale: **fatigue** and **carry-over** — a bad round 2 affecting round 3.

### Run one or two full-loop rehearsals

About one to two weeks before the real loop:

```text
09:30  Coding (no AI)                     45 min + 5 min silent scoring
10:30  System design                      60 min + 5
11:45  Behavioral                         45 min + 5
12:30  Lunch (keep it short, as on the day)
13:15  .NET deep technical / or architect round   45–60 min + 5
14:30  Second design or implementer round  45 min + 5
15:30  Batched debrief for all rounds     45–60 min
```

Rules:

- **Different interviewers for different rounds**, if at all possible — even two people alternating helps.
- **No debriefing between rounds.** On the real day you won't get feedback between rounds; practise moving on without it.
- **Sequence the hardest round late** at least once, to see fatigue effects on your weakest type.
- **Score each round independently**, then look at the loop profile: which rounds were under the bar, and did any one round drag the next one down?

### What you learn

- Your **energy curve** — many people's design rounds degrade after lunch; you can plan food, caffeine, breaks and a pre-round routine around it (Concept 24).
- Your **carry-over pattern** — whether you recover from a bad round. If the round after a weak one is also weak, that's a pressure-class problem (Concept 16), and the remedy is a reset routine, not more content.
- **Logistics** — camera, lighting, internet failover, a whiteboard that actually erases, water, a quiet room. Boring, and responsible for a surprising number of bad days.

**Interview-grade sentence:** *"Before a real loop I ran one or two full-day simulations with different interviewers and no feedback between rounds, which exposed fatigue and whether a weak round dragged down the next one — things a single mock can't show."*

---

## Concept 19 — The schedule

### Six-week default

| Week | Phase | Sessions (per week) | Fidelity | Goal |
|---|---|---|---|---|
| 1 | **Baseline** | 1 full mock per round type you'll face (3–5) | L3–L4 | Establish starting scores and error classes; calibrate with partners |
| 2 | **Targeted** | 2 full mocks + 4–6 drills | L2–L4 | Fix the top error class |
| 3 | **Targeted** | 2 full mocks + 4–6 drills | L3–L4 | Second error class; rotate round types |
| 4 | **Pressure** | 3 full mocks (strangers) + 3 drills | L4–L5 | Push fidelity up; harder problems; challenging interviewers |
| 5 | **Loop** | 1 full-loop simulation + 2 full mocks | L4–L5 | Fatigue and carry-over; real logistics |
| 6 | **Taper** | 1–2 light mocks + light drills; rest before the loop | L3–L4 | Maintain sharpness; protect sleep and confidence |

That's roughly **15–20 full mocks plus 20–30 drills** — about 5–6 scored samples per main round type, which matches the sample-size reasoning in Concept 5.

### Compressed two-week plan (when an interview is already scheduled)

```text
Days 1–2   Baseline: one design, one coding, one behavioral mock — scored and logged
Days 3–7   Daily: one drill on the top error class + one full mock rotating round types
Day 8      Full-loop simulation (abbreviated: 3–4 rounds)
Days 9–12  Two higher-fidelity mocks (strangers/experts) + retrieval drills on weak content
Days 13–14 Light: one short behavioral run-through, logistics check, rest
```

### Allocation across round types

Weight by **round count in the target loop × your current gap**:

```text
priority(round type) = rounds_in_loop × (target 3.0 − current rolling median, floor 0) + 0.5
```

The `+0.5` keeps every round type in rotation (interleaving, and to avoid regressions). Recompute weekly from the log.

### Drills — the 10-to-25-minute workhorses

| Drill | Duration | Targets |
|---|---|---|
| **First five minutes** — requirements and scoping on a random problem, then stop | 7 min | D1; the variance from slow starts |
| **Numbers to decisions** — estimation aloud, every number followed by "so…" | 8 min | Estimation conclusions |
| **Deep-dive ladder** — partner climbs the ladder on one component for 15 min | 15 min | D2 depth; precision |
| **"It's down"** — partner names a dependency; you give the failure policy | 5 × 2 min | Failure handling |
| **Decision sentence** — partner names a choice; you give default → why → when not in 30 s | 10 × 1 min | Judgement and precision |
| **Story in three minutes** + two follow-up "why"s | 3 × 6 min | Behavioral delivery and depth |
| **Concede or defend** — partner makes five claims about your design, some right, some wrong | 10 min | Collaboration; holding positions on evidence |
| **Blank-editor C#** — implement a small type without IntelliSense, then compile | 15 min | Stack fluency without tooling |
| **The 60-second summary** — summarise your design for an executive | 3 × 1 min | Architect communication |

**Interview-grade sentence:** *"I planned roughly six weeks — a scored baseline for every round type, two weeks of targeted drills on the top error classes, then rising fidelity with strangers and experts, a full-loop simulation and a taper — which gave me five or six scored samples per round type, enough to see a real trend rather than noise."*

---

## Concept 20 — AI as interviewer and as grader

An AI model is a genuinely useful mock interviewer: always available, endlessly patient, able to play any interviewer style, willing to climb a probe ladder for an hour, and able to produce a transcript. Anthropic's candidate guidance explicitly encourages using Claude to practise answers and prepare — that's preparation, not cheating. The limits are just as real, and a mock programme should use AI where it's strong and humans where they are.

### Strengths and weaknesses

| AI is good at | AI is weak at |
|---|---|
| Volume — a drill at 06:00 or 23:00 | **Leniency** — models tend to be agreeable and score generously unless constrained |
| Playing a specified style (probing, challenging, time-pressuring) | Real observation pressure — you know it's not a person (fidelity ~L2) |
| Climbing a probe ladder consistently | Seeing your diagram unless you share it (paste an image, or describe it in Mermaid) |
| Producing a transcript and timestamps | Natural interruption timing, especially by text |
| Generating unfamiliar variants of a problem | Knowing a specific company's current bar |
| Rapid-fire retrieval drills on modules 6–29 | Judging *delivery* — tone, presence, silence on video |

### The two-session pattern: interviewer and grader separated

The most important design choice: **don't let the interviewer grade itself.** An interviewer that has been chatting with you for 45 minutes is primed to be generous. Instead:

1. **Session 1 — interviewer.** A prompt that defines role, level, problem, hidden twist, probe ladder, interruption script, and strict rules: never volunteer information, never give solutions, ask one question at a time, keep answers short, track time. (Template in **Appendix E**.)
2. **Session 2 — grader**, in a fresh conversation. Give it the rubric (Appendix B), the transcript, and grading rules: quote evidence for every score, cap at 2 without evidence, list .NET/Azure claims with ✓/~/✗, give one fix. Ask it to grade **as a strict interviewer at the target level**.

Then **calibrate the AI grader like any human scorer** (Concept 15): grade two or three of your human-scored mocks with it and compare. If it's consistently +0.5 high, apply that correction or tighten the prompt.

### Voice beats text

Use voice mode where available: speaking forces real-time composition, exposes filler and silence, and is far closer to the real round than typing — where you can edit before sending. Text mode is fine for drills on content (retrieval of facts), not for full mocks.

### Privacy and policy

- **Anonymise** story-bank material before pasting it anywhere: company names, customer names, confidential figures. Replace with neutral labels ("a payments provider", "~40% of revenue").
- **Never use AI during a real interview** unless the company explicitly permits it for that round. Amazon's guidance says candidates using GenAI without permission can be disqualified; Anthropic's says no AI in live interviews or take-homes unless indicated. When in doubt, ask the recruiter in writing.

**Interview-grade sentence:** *"I used an AI as a mock interviewer for volume and as a strict grader in a separate session — given the rubric and the transcript, required to quote evidence for every score and capped at 'lean no hire' without it — and I calibrated it against my human-scored mocks, because an interviewer grading its own session is reliably too generous."*

---

## Concept 21 — AI-enabled rounds and in-person rounds

The industry is moving in two directions at once, and your mock programme needs to cover both.

### AI-enabled coding rounds

Where they exist (Meta's AI-enabled round, Canva's AI-assisted coding, others reported), the format typically changes as well as the tooling: **one longer, multi-stage problem** in a realistic codebase rather than two isolated puzzles, with an assistant available in the environment. Public descriptions from candidates and companies converge on what's scored:

| What interviewers look for | What it looks like | What fails |
|---|---|---|
| **Problem decomposition** | You break the task into steps and decide which parts to delegate | Pasting the whole prompt into the assistant |
| **Verification** | You read every generated line, run it, test edge cases | Accepting output that compiles |
| **Critical reading** | You catch the assistant's mistakes (off-by-one, wrong API, missing null handling) and say so | Defending AI code you didn't read |
| **Design ownership** | You choose the structure; the assistant fills in | The assistant's structure becomes yours by default |
| **Fundamentals still visible** | You can explain complexity, invariants, trade-offs | Can't explain the code that's on screen |
| **Communication** | You narrate what you're asking the assistant and why | Silent prompting |

Practice protocol for AI-enabled mocks:

1. Get the **practice environment** if the recruiter offers one (Meta reportedly provides a link) — learn its panel, model choices and how code is run.
2. Practise on **multi-stage problems in an existing codebase**: a small ASP.NET Core API with tests; stages like "add an endpoint", "add validation", "fix the failing test", "handle a new requirement".
3. **Seed bugs into the assistant's output deliberately** in drills (ask a second AI to produce subtly wrong code) and practise catching them aloud.
4. Score with the coding rubric plus the **AI-use row** (Concept 13).

### No-AI rounds still exist — usually in the same loop

Meta reportedly pairs the AI-enabled round with a classic no-AI coding round; Amazon and many others forbid AI entirely. **Practise both modes**, and state the mode at the start of every coding mock. A practical risk: after weeks of AI-assisted daily work, blank-editor fluency degrades. The "blank-editor C#" drill (Concept 19) is the counter.

### In-person and whiteboard rounds are back

Google's move to at least one in-person round, and similar moves by other large employers, mean the whiteboard is back in some loops. It's a different medium:

- **Handwriting code is slow** — practise writing compact C#, leaving space for insertions, and naming things short but clear.
- **Diagrams need planning** — reserve the left third of the board for requirements and numbers, the centre for the design, and the right for deep-dive details.
- **Physical presence** — standing, facing the interviewer while talking, not the board.

Do at least one or two **physical whiteboard mocks** if any round may be in person.

**Interview-grade sentence:** *"I practised coding in each AI mode the loop uses — no AI, AI permitted and AI expected — because AI-enabled rounds score decomposition, verification and critical reading of generated code rather than raw typing, while no-AI rounds still test blank-editor fluency; and I did whiteboard mocks in person since several companies have brought back at least one in-person round."*

---
# Part E — Turning results into decisions

## Concept 22 — Readiness gates

"Am I ready?" deserves the same treatment as any release decision: explicit gates, measured from data, decided in advance.

### Per round type

| Gate | Criterion (senior IC target) | Why |
|---|---|---|
| **Level** | Rolling median of the last 3–5 full mocks ≥ **3.0** | Lean hire or better is the bar |
| **Floor** | No dimension scored **1** in the last 3 mocks; no knockouts | Committees react to weaknesses, not averages |
| **Consistency** | Range of the last 4 scores ≤ **1.0** | Concept 5: variance matters |
| **Fidelity** | At least **2** of the last 4 mocks at L4–L5 | Scores from friendly mocks overstate |
| **Fix closure** | The top recurring error class hasn't appeared in the last 2 mocks | Habits, not one-offs |
| **Independence** | Hints used ≤ 1 per mock | Hints cost levels in real rounds |

For a **staff/architect** target, raise the level gate to a median of **3.5** on the architect-specific rounds, and add: *at least one mock per architect round type scored 4 on decision framing or stakeholders*.

### The loop-level decision

```text
GO          All round types in the loop pass their gates.
GO-WITH-RISK One round type misses by a little (e.g. median 2.8, no 1s); the loop has only one such round.
EXTEND      Two or more round types miss, or any round type has a recent knockout → ask the recruiter for 1–2 weeks.
RESCHEDULE  The dominant error class is K (knowledge) on core material → study first; mocks can't fix it fast.
```

Recruiters usually accept a reasonable request to move an interview by one or two weeks — and it's far cheaper than a failed loop, which often comes with a cooling-off period of six to twelve months before you can reapply (it varies by company).

**Interview-grade sentence:** *"I treated readiness as a release decision with gates set in advance — a rolling median at or above lean hire per round type, no dimension at the bottom of the scale recently, a narrow range across the last few mocks, some high-fidelity samples, and the top recurring error gone — and I moved interview dates when the gates weren't met rather than hoping."*

---

## Concept 23 — Plateaus and regression

Progress is rarely linear. A typical curve: a quick rise over the first five or six mocks (mechanics fixed), then a plateau, sometimes a dip when fidelity increases.

### Diagnose before you react

| Symptom | Likely cause | Response |
|---|---|---|
| Scores flat, mocks feel easy | Practising below the edge — familiar problems, friendly partner | Raise difficulty: new problems, challenging style, strangers (desirable difficulty) |
| Scores flat, the same error class recurs | The fix isn't being drilled between mocks | More drills, fewer full mocks; make the fix smaller and more observable |
| Scores dropped after moving to strangers/experts | Fidelity jump — expected | Keep going; compare only like-for-like fidelity levels |
| Scores dropped across the board | Fatigue, overtraining, life load | Rest 3–5 days; resume with drills |
| Scores vary wildly | High variance — no routine | Fix the opening routine and time checkpoints first (Concept 5) |
| Good scores but real rounds go badly | Mocks miscalibrated (lenient scorers, wrong format, wrong level) | Recalibrate (Concept 15); add L5 mocks; recheck format with Module 39 research |
| One round type lags far behind | Under-practised or a deeper gap | Reallocate (Concept 19 priority formula); consider an expert mock for diagnosis |

### Change the stimulus

When stuck, change **one** thing deliberately: a new interviewer style, a new problem family, a stricter time budget (40 minutes for a 45-minute problem), a different medium, or a higher-stakes partner. Plateaus usually mean the practice has become comfortable.

**Interview-grade sentence:** *"When my scores plateaued I diagnosed from the log before reacting — a recurring error class meant I wasn't drilling the fix, flat scores with easy-feeling mocks meant I needed harder conditions, and a drop after moving to stricter interviewers was expected and compared only like-for-like."*

---

## Concept 24 — Performing on the day

Everything above raises your mean and narrows your variance. The last piece is the day itself — managing arousal and recovering from the inevitable bad moment.

### Reappraise rather than suppress

Two well-cited studies are worth knowing because they change what you should *tell yourself*:

- **Reappraising arousal as useful** — Jamieson and colleagues (2010) told some students before a GRE-style test that physiological arousal can improve performance; they scored higher on the math section than a control group.
- **"I am excited" beats "I am calm"** — Brooks (2014) found that reframing pre-performance anxiety as excitement improved performance on public speaking, maths and singing tasks compared with trying to calm down.

Practical meaning: a racing heart before the round is not a sign that something is wrong. Don't fight it; label it as energy for the task. Rehearse that sentence in mocks so it's automatic.

A third finding is useful for the night before: **expressive writing** — Ramirez and Beilock (2011) found that ten minutes writing about one's worries before a high-stakes exam improved performance for anxious students, plausibly by freeing working memory. If you tend to ruminate, write it down and close the notebook.

### The pre-round routine (practise it in every mock)

```text
T−15 min  Logistics check: camera, mic, editor/whiteboard, water, phone silent
T−10 min  Read your one-page card: opening moves, time budget, the deep-dive menu for likely problems
T−5  min  Two slow breaths; one sentence of reappraisal ("this energy is for the problem")
T−0       Open with the same first move every time: restate the problem, ask the first scoping question
```

The routine is a variance-reduction device: the first five minutes become execution, not improvisation.

### Recovery scripts

Every round has a bad moment. What's scored is the recovery. Have sentences ready — and practise them in mocks via the interruption script:

| Moment | Recovery sentence |
|---|---|
| Blank on a fact | *"I don't remember the exact figure — I'll assume roughly X and note that the design depends on it; if it's off by 10× here's what changes."* |
| Realise a mistake | *"Let me correct something — earlier I said a 301; I'd change that to a 302 because we need every click for analytics."* (Correcting yourself is a positive signal.) |
| Behind time | *"I'm conscious of time — let me finish the high-level flow in two minutes so we can spend the rest on the hardest part."* |
| Interviewer pushes back and you think they're right | *"That's a good point — it breaks my assumption about X. I'd change Y to Z."* |
| Interviewer pushes back and you think they're wrong | *"I see the concern. My reason for keeping it is X, because of the number we derived earlier — what would make you prefer the alternative?"* |
| Totally stuck | *"Let me step back and state what I know: … The part I'm unsure about is …; one way forward is …"* |

### Between rounds

- **Don't post-mortem a round during the loop.** You can't change it, and interview-to-interview volatility means your impression of how it went is unreliable anyway (Concept 4).
- **Reset ritual** — a short walk, water, the reappraisal sentence, read the card for the next round type.
- **After the loop**, write down what you remember for your log *that evening* — then stop analysing.

**Interview-grade sentence:** *"On the day, I relied on routines I'd practised in every mock — a fixed opening, a time budget, prepared recovery sentences for blanks, mistakes and pushback — and I treated nerves as energy rather than trying to suppress them, which the reappraisal research supports; between rounds I deliberately didn't replay the previous one."*

---
# Worked example — One full mock cycle, end to end

**Candidate:** you, preparing for a senior .NET/Azure IC loop. **Mock #7**, week 3. **Round type:** system design, 45 minutes, Excalidraw, **no AI**. **Fidelity:** L4 (an engineer from a .NET community you hadn't met). **Current fix from mock #6:** *"State a conclusion after every number."*

### The brief (sent the day before)

```text
Level: senior IC, product company, .NET/Azure stack. Round: system design, 45 min, Excalidraw.
Problem: "Design a notification system that sends push, SMS, email and in-app notifications." (Module 37, Part D)
Hidden twist (reveal only if asked about traffic mix): marketing campaigns to 20M users happen weekly;
    OTP codes must arrive within 10 s.
Probe ladder (deep dive likely on delivery guarantees):
    1 What queue?  2 Why Service Bus over Event Hubs?  3 What happens if the worker crashes after the SMS
    provider accepted?  4 Can you guarantee exactly once?  5 What does the user see on a duplicate?
    6 How does Service Bus duplicate detection help here — exactly?
Interruption script: ~12:00 "Marketing wants 50M pushes in an hour next month."
                     ~26:00 "I don't think separate queues are worth it — one queue with priorities is simpler."
                     ~36:00 "Ten minutes left — what should I remember?"
Check first: the candidate's fix — conclusions after numbers.
AI mode: none.
```

### What happened — the interviewer's timeline and notes

```text
00:00–06:30  Requirements. Asked channels, preferences, latency for OTP vs marketing, delivery guarantee,
             compliance (consent/unsubscribe). Did NOT ask about campaign volume → twist missed until 12:00.
06:30–10:10  Estimation. "50M users, ~20M transactional/day — about 230 a second, peaks maybe 2,300."
             → no conclusion stated. "75M tokens at ~300 bytes is about 22 GB" → "so tokens fit in one
             Cosmos container easily, partition by user." (conclusion ✓)
10:10–12:00  API: POST /notifications with idempotency key from producer event id; 202 Accepted. ✓
12:00        Script: 50M in an hour. Candidate: "That's ~14,000 a second — six times the transactional
             peak — so OTPs can't share a queue with campaigns." (excellent, conclusion ✓)
12:30–21:40  High-level: API → planner → per-channel queues → workers → providers → receipts. Drew priority
             lanes after the twist. Mentioned Notification Hubs vs direct APNs/FCM v1. HLD complete 21:40. ✓
21:40–33:30  Deep dive: delivery guarantees. Claim–send–record; deterministic delivery id; residual crash
             window named unprompted: "duplicate for OTPs, I'd rather that than a lost code; collapse id on push".
             Ladder rung 6: "Service Bus duplicate detection only covers producer resends within the window —
             not redelivery after a lock expiry, so the worker still needs the claim." ✓✓
26:00        Script pushback (single queue + priorities). Candidate: "Priority on a shared queue doesn't
             help when a million bulk messages are already ahead of the OTP — I'd keep separate lanes;
             the cost is one more queue and worker pool." Held position with reason. ✓
33:30–36:00  Second deep dive started: provider failure. Retries with backoff, DLQ. Did not mention invalid-
             token cleanup or provider rate limits. Then said "abandon the message so it retries" — no
             mention that abandon has no backoff (~ imprecise).
36:00        Script: ten minutes left. Summary was 2.5 minutes, a bit long, but covered lanes, effectively-once,
             and the window. Did not name the next scaling bottleneck (provider limits).
39:00–45:00  Candidate questions.
Hints: none.
```

### Independent scores

| Dimension | Interviewer | Self (memory) | Self (from tape) | Evidence (interviewer) |
|---|---|---|---|---|
| Requirements | 3 | 3 | 3 | Good NFRs; missed campaign volume (twist found only via script, 12:00) |
| Estimation | 3 | 4 | 3 | 22 GB → conclusion ✓; 230/s and 2,300/s → no conclusion (08:20); 14k/s → excellent conclusion (12:00) |
| High-level design | 4 | 3 | 4 | Complete 21:40; every box justified; lanes added on new info |
| Deep dives | 3 | 3 | 3 | DD1 strong (rung 6); DD2 shallow and late (33:30) |
| Failure handling | 3 | 3 | 3 | Crash window unprompted ✓; provider limits, invalid tokens absent |
| .NET/Azure precision | 3 | 4 | 3 | Duplicate-detection scope precise ✓✓; "abandon so it retries" ~ (no backoff); Notification Hubs/FCM v1 ✓ |
| Pivots / wrap-up | 3 | 3 | 3 | Handled 50M pivot with number; no next bottleneck named |
| Driving | 3 | 2 | 3 | Phases announced; summary long; one check-in only (21:40) |
| **Verdict** | **3 — lean hire** | 3 | 3 | Strong core deep dive; second deep dive and wrap-up thin |

Two observations a calibrated reviewer would make:

- **Memory vs tape.** The candidate overrated estimation from memory (they *remembered* giving conclusions for every number) and underrated driving (they *felt* rushed). Both corrections are typical (Concept 4) and go into the personal-bias log.
- **The fix from mock #6** — "conclusions after numbers" — is *partially* fixed: two of three key numbers had conclusions. Progress, not done.

### The debrief (condensed)

> **Candidate first:** "Went well: the duplicate-detection point. Didn't: I ran out of time on providers."
>
> **SBI 1:** *"At 08:20, after 'peaks maybe 2,300 a second', you moved on to the API. I didn't learn what the number meant for the design until the campaign question at 12:00 — and if I hadn't asked, you might never have connected it to priority lanes."* **Intent?** — "I knew campaigns were bigger, I was saving it." → Class **C/S** (said it late), not K.
>
> **SBI 2:** *"From 21:40 to 33:30 you spent twelve minutes on delivery guarantees — it was the strongest part of the interview — but it left three minutes for provider failure, where you didn't get to rate limits or token cleanup."* → Class **S**.
>
> **SBI 3:** *"At 26:00 when I pushed for a single queue, you held your position with a concrete reason and named the cost. That's exactly what I'd write down as strong."*
>
> **The one fix (agreed):** *"At the end of high-level design, say out loud: 'I'll spend about eight minutes on X and eight on Y' — and glance at the clock at the switch."*

### The candidate's review and log entry (same evening)

Metrics from the transcript tool: HLD complete 21:40 ✓; longest silence 34 s at 15:10 (drawing without narrating); numbers with conclusions 2/3; questions in first five minutes: 6.

```text
#7  2026-10-21  SD  notifications  L4  no-AI  45m  interviewer: community peer (probing)
Scores (int/self-tape): Req 3/3 Est 3/3 HLD 4/4 Deep 3/3 Fail 3/3 NET 3/3 Pivot 3/3 Drive 3/3  Verdict 3/3
Memory-vs-tape bias: Est +1, Drive −1
Errors:  08:20 number without conclusion ........ C (fix #6 partially closed)
         21:40–33:30 deep-dive time split ........ S
         34:30 "abandon so it retries" ........... K/R → check: knew it once reminded → R
         15:10 34 s silence while drawing ......... C
Fix #7:  Announce the deep-dive time split at the end of HLD; clock glance at the switch.
Drills:  3× "deep-dive split" (15 min each, partner calls time); 1× retrieval: Service Bus abandon/backoff,
         invalid tokens, provider limits (Module 37 Concept 22) aloud.
Next mock brief: check fix #7 first; then fix #6 (conclusions after numbers).
```

**What made this a useful mock:** not the score — a 3 is fine at week 3 — but that it produced a classified error list, a precise personal-bias correction, and one fix that the next brief will check first.

---
# Common interview questions with model answers

Two kinds of questions live here. **Questions about hiring, feedback and learning** — asked of staff and architect candidates more often than people expect, and answered far better by someone who has actually run structured mocks. And **in-round moments** — situations every mock should rehearse, with the response that scores well.

### Questions about hiring, feedback and improvement

**Q1. "How do you interview candidates? What makes a good interview?"**
> "I treat an interview as a measurement, so I want it structured: a job-relevant question decided in advance, a rubric with behaviourally anchored levels for each dimension, and independent scoring with evidence before any discussion. During the round I don't rescue candidates, I answer only what's asked, and I go one level deeper on each decision until I find the edge — what, why, numbers, failure, change. I write timestamps and verbatim quotes, so my write-up is defensible. Afterwards I score each dimension first and only then the overall recommendation — that's the structured approach the selection research supports, and it's much less noisy than a gut feel."
*Key signal:* structure, evidence, dimensions-before-verdict — with reasons.

**Q2. "How would you design the interview loop for a senior .NET engineer on your team?"**
> "Start from the job: what does a senior engineer here actually do in their first year — design services, review code, own incidents, mentor? Each round should sample one of those. I'd do a practical coding round in C# focused on correctness, testing and readability rather than puzzles; a system-design round on a problem shaped like our domain; a .NET depth round — async, memory, EF Core, ASP.NET Core — anchored on a debugging scenario; and a behavioral round on ownership, conflict and mentoring. Each round has one owner, a written rubric and probe ladders, and interviewers submit scores independently before the debrief. I'd decide our AI policy per round explicitly and tell candidates in advance. And I'd review the loop's outcomes — offer rates, interviewer agreement, how hires perform — at least yearly."
*Key signal:* job analysis first, a rubric per round, independence, explicit AI policy, a feedback loop on the loop.

**Q3. "How do you keep interviewers calibrated?"**
> "Shadowing and reverse-shadowing for new interviewers; periodic calibration sessions where everyone scores the same recorded or written-up interview independently and we discuss every two-point disagreement until the anchor is clearer — then we update the rubric's examples. I'd also look at each interviewer's score distribution over time: someone who rates everyone a 3 isn't adding signal, and someone consistently harsher than the panel needs their scores read with that in mind."
*Key signal:* concrete mechanisms — independent scoring of common material, anchor refinement, distribution monitoring.

**Q4. "Tell me about feedback you received that changed how you work."** *(behavioral — answer with a real work story; the structure is what matters)*
> Situation and the specific feedback (what was observed, not a label) → what you initially thought → what you did differently, concretely → evidence it worked (a measurable change) → how you now apply it or pass it on.
*Key signal:* specificity, absence of defensiveness, a measurable change. A preparation-related example is acceptable as a *second* story but weaker than a work one.

**Q5. "How do you give critical feedback to a senior engineer?"**
> "Privately, soon after the event, and about a specific situation and behaviour rather than a trait — 'in Tuesday's design review, when the payments team raised the retry concern, you moved on without answering, and they left believing we hadn't considered it' — then I ask what they intended, because there's often a reason I can't see. I aim for one thing they can change, and I follow up. With senior people I also make clear what I'm *not* saying — it's about the behaviour, not their judgement overall."
*Key signal:* SBI(I), timeliness, one actionable point, follow-up.

**Q6. "Should candidates be allowed to use AI in coding interviews?"**
> "It depends on what the round is trying to measure, and companies have reasonably landed in different places. The case for allowing it is fidelity — engineers use assistants daily, so a round with one measures the job more directly, and it makes covert use pointless; that's the reasoning companies like Canva and Meta have given for AI-enabled rounds. The case against is that it can mask fundamentals and makes some questions trivial — which is why others forbid it and some have reintroduced in-person rounds. My view for a team I'm hiring for is to be explicit per round: at least one round that shows fundamentals without assistance, and one that shows how someone decomposes, verifies and owns code with an assistant — and tell candidates in advance which is which."
*Key signal:* weighing both sides, tying the decision to what each round measures, explicitness.

**Q7. "How do you get better at a skill that's hard to practise?"**
> "I separate measuring from practising. I find a way to get honest measurement under realistic conditions — for interviews that meant recorded mocks with people I didn't know, scored against a fixed rubric — and between measurements I do short, focused drills on one behaviour at a time, with immediate feedback. I space the practice out, I mix problem types, and I classify my mistakes because a knowledge gap, a retrieval failure and a time-management failure need different fixes."
*Key signal:* a method — measurement, deliberate practice, error classification.

### In-round moments

**Q8. The interviewer asks: "Would you like a hint?"**
> "Yes, thank you" — when you've genuinely been stuck for a couple of minutes. Then **use the hint visibly** and move: *"That helps — so if the counter lives in one shard, the hot tenant problem is exactly that one shard. Then token leasing makes sense…"* Refusing a hint you need costs more than taking it; taking one and not using it costs most.
*Key signal:* coachability — taking input and building on it.

**Q9. Ten minutes left; you haven't started the second deep dive.**
> *"We've got about ten minutes. I'd like to use six on the failure handling for the provider calls, since that's where this design could actually lose notifications, and keep the last few for anything you'd like to cover. Does that work?"* Then do a compressed version: the policy per failure, one number, one .NET mechanism.
*Key signal:* prioritising aloud and offering the interviewer control.

**Q10. The interviewer disagrees and you believe they're wrong.**
> *"I see why that's attractive — it's simpler. My reason for keeping separate lanes is the backlog: with a million bulk messages already queued, a priority flag doesn't move the OTP to the front of a shared queue. If campaigns were small, I'd agree with you. Is there a constraint I'm missing that makes one queue important?"*
*Key signal:* holding a position with evidence, naming the condition under which you'd change your mind, inviting new information.

**Q11. At minute 30 you realise there's a flaw in your design.**
> *"Let me flag something — the redirect path I drew caches at the edge, which means disabled links would keep working past our one-minute requirement. I'd change that: edge caching only for links detected as hot, with a purge on disable."* Then continue.
*Key signal:* self-correction. Interviewers score finding your own bug positively; hoping they don't notice is a gamble you lose.

**Q12. Start of an AI-enabled round.**
> Ask what's permitted and what they want to see: *"Just to confirm — I can use the assistant for any part, including generating tests? And would you like me to narrate what I'm asking it?"* Then decompose before prompting, read every generated line, and say what you're checking.
*Key signal:* clarifying the rules, owning the design, verification out loud.

**Q13. "What questions do you have for me?"**
> Two or three prepared, specific questions that show judgement: *"What's the hardest architectural decision the team made in the last year, and would you make it the same way again?"* *"How are design decisions recorded and revisited here?"* *"What would make someone in this role clearly successful at twelve months?"* (Module 39 builds these per company.)
*Key signal:* curiosity about how the team decides and what success means. Practise this in mocks too — it's often the last impression.

**Q14. Behavioral follow-up: "What would you do differently?"**
> A specific change, not "communicate more": *"I'd have written the migration risks into the design doc before the review instead of presenting them verbally — two teams only learned about the dual-write window in the incident review."* Then what you've done since.
*Key signal:* a concrete, owned change and evidence of learning applied.

**Q15. "How do you know your design is good enough?"**
> *"When it meets the requirements we stated with numbers to spare, when I can say how each component fails and what we do about it, and when the next likely change — ten times the traffic, a second region — has a seam rather than a rewrite. Past that, more design is speculation; I'd ship, measure, and record the decision with the condition that would reopen it."*
*Key signal:* explicit, testable stopping criteria; reversibility.

**Q16. "How did you prepare for this interview?"** *(sometimes asked, especially by curious interviewers)*
> Honest and specific: *"I reviewed the fundamentals, then ran recorded mock interviews with engineers I didn't know, scored against rubrics — mostly to get used to designing out loud under time pressure, which is different from designing at my desk."* No need to apologise for preparing; preparation is a signal of seriousness.
*Key signal:* honesty; a method rather than cramming.

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Mock purpose | Every session a coaching-chat hybrid | Named mode: full mock (measurement) or drill (training) |
| Fidelity | Practising alone, silently, untimed | Out loud, timed, observed, climbing the fidelity ladder |
| Partners | Always the same friendly colleague | Rotating strangers and interviewer styles |
| Medium | IDE with IntelliSense for all coding practice | The real round's medium — plain editor, whiteboard, the AI mode in use |
| Problems | Repeating known problems until they feel smooth | Interleaved, unfamiliar problems for measurement mocks |
| Briefing | Partner improvises the problem on the call | Written brief with twist, probe ladder and interruption script |
| As interviewer | Rescues, reassures, volunteers information | Waits, answers only what's asked, climbs the ladder, writes verbatim |
| Scoring | One overall impression | Dimensions first from evidence, then the verdict |
| Evidence | "Seemed good at trade-offs" | Quotes and timestamps per dimension; no evidence → max 2 |
| Self-assessment | Trusting memory | Re-scoring from the recording; known personal bias |
| Verdict | Averaging dimension scores | Weighted judgement; knockouts; the committee test |
| Calibration | Never compared with anyone | Reference recordings; partner agreement; test–retest |
| Debrief | Ten pieces of feedback, conversation first | Independent scores first; SBI; one fix |
| Fixes | "Be more senior", "know more Cosmos" | One observable behaviour, checked first next time |
| Errors | Every miss → study more | Classified K/R/S/C/J/P/L with different remedies |
| Statistics | Judging readiness from one great mock | Rolling median; variance as a target |
| Schedule | A dozen mocks in the final week | Spaced over 4–8 weeks; drills between mocks |
| Round coverage | Only system design and coding | Every real round type, including implementer and stakeholder rounds |
| Loop | Never rehearsed a full day | One or two full-loop simulations; fatigue and carry-over known |
| AI interviewer | AI interviews and grades itself, generously | Separate grader session; evidence required; calibrated against humans |
| AI policy | Assumes the rules | Asks the recruiter per round; never uses AI live without explicit permission |
| AI-enabled rounds | Pastes the whole problem into the assistant | Decomposes, verifies, reads critically, owns the design |
| Readiness | "I feel ready" | Gates set in advance; extend dates when gates fail |
| Plateaus | More of the same | Diagnose from the log; change one stimulus |
| On the day | Suppressing nerves; replaying the last round | Reappraisal; routine; recovery sentences; one round at a time |
| Hiring questions (staff) | "I go with my gut" | Structured interviewing, calibration, explicit AI policy, loop review |

---
# Practice exercises

1. **Baseline.** Run one scored, recorded full mock for each round type in your target loop within a week. Log them with Appendix D. Don't try to do well; try to measure.
2. **Write your rubric.** Copy Appendix B, then rewrite every "3" and "4" anchor for your *exact* target level and track (senior IC vs solution architect). Version it as v1.
3. **Score a reference recording.** Pick two mocks from interviewing.io's replay library (one design, one coding) and one Hello Interview or Exponent mock video. Score each with your rubric *before* reading or hearing the published verdict; compare and note where your scale is lenient or harsh.
4. **Memory vs tape.** After your next mock, self-score from memory immediately, then from the recording the next day. Compute the per-dimension difference. Repeat for three mocks and write down your personal bias.
5. **Partner calibration.** With a partner, score the same recording independently; compute exact and within-one agreement; resolve every two-point gap by adding an example to the rubric. Repeat until within-one agreement ≥ 90%.
6. **Brief writing.** Write interviewer briefs (Appendix A) for three Module 37 problems, each with a hidden twist, a six-rung probe ladder for the most likely deep dive, and a three-step interruption script.
7. **Be the interviewer.** Interview a peer with one of your briefs. Afterwards, list three habits you saw in them that you suspect you share. Check your next recording for them.
8. **Transcript tool.** Build the C# transcript tool from Concept 10; run it on three recordings; add one metric of your own (e.g. longest monologue without the interviewer speaking, using a separate track for each speaker).
9. **Error classification.** Take your last three mock logs and classify every miss K/R/S/C/J/P/L. Compute the distribution and decide what next week's time goes to.
10. **Drill design.** For your top error class, design a 10-minute drill that would fix it, run it five times over a week, and measure the behaviour in the next full mock.
11. **AI interviewer + grader.** Run a voice mock with the Appendix E interviewer prompt, then grade the transcript in a fresh session with the grader prompt. Compare the AI grade with a human score of the same recording, and adjust the grader prompt until they're within one point on every dimension.
12. **AI-enabled coding.** Create a small ASP.NET Core minimal-API project with five tests. Have a partner define four sequential stages of change. Complete them with an assistant in 50 minutes, narrating; score with the AI-use row.
13. **Blank-editor C#.** Without IntelliSense, implement a thread-safe LRU cache and a token-bucket limiter with `TimeProvider`; then compile and fix. Record the number of compile errors; repeat weekly until zero.
14. **Architect mocks.** Run one design-document defence (with your Module 30 doc), one brownfield redesign and one implementer round. Score with the architect rubric. Which dimension is weakest?
15. **Full-loop simulation.** Run a four-to-five-round day with at least two interviewers and no feedback between rounds. Plot round scores in order; look for fatigue and carry-over.
16. **Readiness.** Implement the readiness checker from Appendix D over your log; decide GO / GO-WITH-RISK / EXTEND for your next real loop and write down why.
17. **Hiring questions.** Answer Q1–Q3 aloud in three minutes each, as if in a staff/architect behavioral round; record and score them with the behavioral rubric.

---
# Free resources and learning material

All free to read or use unless marked *(book)*, *(paid)* or *(partly paid)*. Start with the ★ items. Platform and policy facts were checked on October 8, 2026.

### Research: why structured interviews and rubrics work
- ★ [Sackett, Zhang, Berry & Lievens (2022) — Revisiting meta-analytic estimates of validity in personnel selection](https://doi.org/10.1037/apl0000994) — structured interviews ranked first after corrected range-restriction adjustments.
- [SIOP — Is cognitive ability the best predictor of job performance? New research says think again](https://www.siop.org/tip-article/is-cognitive-ability-the-best-predictor-of-job-performance) — readable summary of the 2022 paper.
- [Schmidt & Hunter (1998) — The validity and utility of selection methods in personnel psychology](https://doi.org/10.1037/0033-2909.124.2.262) — the classic review the 2022 paper revisits.
- [McDaniel, Whetzel, Schmidt & Maurer (1994) — The validity of employment interviews](https://doi.org/10.1037/0021-9010.79.4.599).
- [Levashina, Hartwell, Morgeson & Campion (2014) — The structured employment interview: narrative and quantitative review](https://doi.org/10.1111/peps.12052).
- [Dana, Dawes & Peterson (2013) — Belief in the unstructured interview: the persistence of an illusion](https://journal.sjdm.org/12/121130a/jdm121130a.pdf).
- ★ [Kahneman, Lovallo & Sibony — A structured approach to strategic decisions (MIT Sloan Management Review)](https://sloanreview.mit.edu/article/a-structured-approach-to-strategic-decisions/) — mediating assessments; why dimension scores come before the verdict.
- ★ [Google re:Work — Use structured interviewing](https://rework.withgoogle.com/en/guides/hiring-use-structured-interviewing) and [A guide to structured interviewing](https://rework.withgoogle.com/intl/en/guides/a-guide-to-structured-interviewing-for-better-hiring-practices).
- ★ [US OPM — Structured interviews](https://www.opm.gov/policy-data-oversight/assessment-and-selection/structured-interviews/), its [Structured Interview Guide (PDF)](https://www.opm.gov/policy-data-oversight/assessment-and-selection/structured-interviews/guide.pdf) and an [example question with a rating scale (PDF)](https://www.opm.gov/policy-data-oversight/assessment-and-selection/examples/structured-interview-example.pdf).
- [Behaviorally anchored rating scales — Wikipedia](https://en.wikipedia.org/wiki/Behaviorally_anchored_rating_scales), [Halo effect — Wikipedia](https://en.wikipedia.org/wiki/Halo_effect), [Cohen's kappa — Wikipedia](https://en.wikipedia.org/wiki/Cohen%27s_kappa).

### Research: technical interviews specifically
- ★ [Behroozi, Shirolkar, Barik & Parnin (2020) — Does stress impact technical interview performance?](https://doi.org/10.1145/3368089.3409712) and the [NC State summary](https://www.sciencedaily.com/releases/2020/07/200714101228.htm).
- ★ [interviewing.io — Technical interview performance is kind of arbitrary. Here's the data](https://interviewing.io/blog/technical-interview-performance-is-kind-of-arbitrary-heres-the-data) and [After a lot more data, it really is kind of arbitrary](https://interviewing.io/blog/after-a-lot-more-data-technical-interview-performance-really-is-kind-of-arbitrary).
- ★ [interviewing.io — People can't gauge their own interview performance](https://interviewing.io/blog/people-cant-gauge-their-own-interview-performance-and-that-makes-them-harder-to-hire) and [People are still bad at gauging their own interview performance](https://interviewing.io/blog/own-interview-performance).
- [interviewing.io — We analyzed 100K technical interviews](https://blog.interviewing.io/we-analyzed-100k-technical-interviews-to-see-where-the-best-performers-work-here-are-the-results/) — includes their claim about mock practice and pass rates.

### Learning science
- ★ [Ericsson, Krampe & Tesch-Römer (1993) — The role of deliberate practice in the acquisition of expert performance](https://doi.org/10.1037/0033-295X.100.3.363).
- ★ [Roediger & Karpicke (2006) — Test-enhanced learning](https://doi.org/10.1111/j.1467-9280.2006.01693.x).
- [Cepeda et al. (2006) — Distributed practice in verbal recall tasks: a review](https://doi.org/10.1037/0033-2909.132.3.354).
- [Dunlosky et al. (2013) — Improving students' learning with effective learning techniques](https://doi.org/10.1177/1529100612453266) — which study techniques actually work.
- [Bjork Learning and Forgetting Lab — research on desirable difficulties](https://bjorklab.psych.ucla.edu/research/).

### Feedback and debriefs
- ★ [Center for Creative Leadership — Situation-Behaviour-Impact (SBI) and SBII](https://www.ccl.org/articles/leading-effectively-articles/closing-the-gap-between-intent-vs-impact-sbii/).
- [Kluger & DeNisi (1996) — The effects of feedback interventions on performance](https://doi.org/10.1037/0033-2909.119.2.254) — why diffuse or person-focused feedback can backfire.

### Performance under pressure
- [Jamieson, Mendes, Blackstock & Schmader (2010) — Turning the knots in your stomach into bows](https://doi.org/10.1016/j.jesp.2009.08.015) — arousal reappraisal.
- [Brooks (2014) — Get excited: reappraising pre-performance anxiety as excitement](https://doi.org/10.1037/a0035325).
- [Ramirez & Beilock (2011) — Writing about testing worries boosts exam performance](https://doi.org/10.1126/science.1199427).

### Mock interview platforms and practice tools
- ★ [Pramp](https://www.pramp.com/) — free peer mocks, now on [Exponent Practice](https://www.tryexponent.com/practice) *(free monthly credits; partly paid)*.
- ★ [interviewing.io mock interview replays](https://interviewing.io/mocks) — free recordings for calibration; live mocks *(paid)*.
- [Hello Interview Guided Practice](https://www.hellointerview.com/practice) — system design, low-level design, AI-enabled coding, behavioral *(one problem free; partly paid)*.
- [Google Interview Warmup](https://grow.google/interview-warmup) — free, generic AI practice with transcripts.
- ★ [Tech Interview Handbook — Mock interviews](https://www.techinterviewhandbook.org/mock-interviews/), [Coding interview rubrics](https://www.techinterviewhandbook.org/coding-interview-rubrics/), [Behavioral interview rubrics](https://www.techinterviewhandbook.org/behavioral-interview-rubrics/), [Preparing as a senior candidate](https://www.techinterviewhandbook.org/behavioral-interview-senior-candidates/) and [Interview formats at top companies](https://www.techinterviewhandbook.org/interview-formats-top-companies/).
- [Hello Interview — Delivery framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery) — the design-round structure your mocks rehearse.
- [The System Design Primer — GitHub](https://github.com/donnemartin/system-design-primer) — a large bank of practice problems.

### Reference recordings for calibration
- ★ [interviewing.io — mock replays](https://interviewing.io/mocks).
- [Hello Interview — YouTube](https://www.youtube.com/@hello_interview) — mock walkthroughs with commentary.
- [Exponent — YouTube](https://www.youtube.com/@tryexponent) — design, coding and behavioral mocks.

### Architect-round practice
- ★ [Architectural Katas](https://www.architecturalkatas.com/) — Ted Neward's team design exercises; see [the rules](https://www.architecturalkatas.com/rules.html) and [run a kata](https://www.architecturalkatas.com/kata.html). Excellent source of brownfield and stakeholder-round problems.
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/) — reference architectures to base architect mock problems on.
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/) — the review lens an architect interviewer often uses.
- [StaffEng — guides](https://staffeng.com/guides/) — what staff-plus scope looks like, for calibrating level anchors.

### Company interview guidance (for matching format and emphasis)
- ★ [Amazon — How we hire](https://www.amazon.jobs/content/en/how-we-hire), [SDE III interview prep](https://amazon.jobs/content/en/how-we-hire/sde-iii-interview-prep) and [What's it like to interview at Amazon?](https://www.aboutamazon.com/news/workplace/whats-it-like-to-interview-at-amazon) — Leadership Principles, STAR, Bar Raisers.
- [Google — Hiring process](https://www.google.com/about/careers/applications/how-we-hire) and [interview prep](https://www.google.com/about/careers/applications/interview-tips).
- [Meta — Preparing for your full loop interview](https://www.metacareers.com/swe-prep-onsite).

### AI in interviews: policies and formats
- ★ [Anthropic — Guidance on candidates' AI usage](https://www.anthropic.com/candidate-ai-guidance) — AI encouraged for preparation; none in live interviews unless indicated.
- ★ [Canva Engineering — Yes, you can use AI in our interviews](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/).
- ★ [Hello Interview — Meta's AI-enabled coding interview: how to prepare](https://www.hellointerview.com/blog/meta-ai-enabled-coding).
- [GeekWire — Is it cheating? AI use during job interviews sparks debate](https://www.geekwire.com/2025/is-it-cheating-ai-use-during-job-interviews-sparks-debate-over-whether-to-restrict-emerging-tools/) — includes Amazon's statement on unauthorised tools.
- [The Register — Canva requires AI coding assistants in interviews](https://www.theregister.com/2025/06/11/canva_coding_assistant_job_interviews/).
- [Fortune — Anthropic changes its candidate AI policy](https://fortune.com/2025/07/21/billion-dollar-giant-anthropic-ai-ban-hiring-policy-change-job-seekers-interview-process).

### Tools for running and reviewing mocks
- [Excalidraw](https://excalidraw.com/), [tldraw](https://www.tldraw.com/) and [diagrams.net](https://app.diagrams.net/) — free whiteboards.
- [OBS Studio](https://obsproject.com/) — recording with separate audio tracks.
- [OpenAI Whisper](https://github.com/openai/whisper) and [whisper.cpp](https://github.com/ggml-org/whisper.cpp) — local transcription with timestamps.
- [.NET Fiddle](https://dotnetfiddle.net/) and [SharpLab](https://sharplab.io/) — run C# in a browser without IntelliSense crutches.
- [System.Text.Json overview](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview) and [Regex.Count](https://learn.microsoft.com/dotnet/api/system.text.regularexpressions.regex.count) — used by the transcript tool.

### Books
- *(book)* Daniel Kahneman, Olivier Sibony & Cass Sunstein — *Noise* — why judgements vary and how structure reduces it.
- *(book)* Daniel Kahneman — *Thinking, Fast and Slow* — first impressions, halo, structured judgement.
- *(book)* Anders Ericsson & Robert Pool — *Peak* — deliberate practice for a general audience.
- *(book)* Brown, Roediger & McDaniel — *Make It Stick* — retrieval, spacing, interleaving.
- *(book)* Laszlo Bock — *Work Rules!* — Google's structured-interviewing story.
- *(book)* Tanya Reilly — *The Staff Engineer's Path* — staff-level scope for calibrating level anchors.
- *(book)* Gayle Laakmann McDowell, Aline Lerner, Mike Mroczka & Nil Mamano — *Beyond Cracking the Coding Interview* — includes interviewing.io's data on how interviews are scored.

### Previous modules to revisit
- Modules 1–2 (what's scored; IC vs architect loops) — the basis of the dimensions.
- Modules 3–5 (method, requirements, estimation) — the system-design rubric's rows.
- Modules 30–33 (design docs, ADRs, brownfield, cost) — the architect mocks.
- Modules 34–35 (STAR, story bank) — the behavioral mocks.
- Module 36 (coding rounds) and Module 37 (worked problems) — the coding rubric and the design problem bank.
- Module 39 (company/role research) — which mocks and which AI mode to prioritise per company.

---
# Quick-recall sheet

**One sentence.** A mock is a measurement and a training stimulus: reproduce the real pressures, score observable evidence against a fixed rubric, repeat enough to beat the noise, and leave with one fix the next mock checks.

**Two modes.** Full mock = measurement (no help, scored, recorded). Drill = training (10–25 min, one behaviour, pause and repeat).

**Five pressures.** Observation · clock · ambiguity · interruption · stakes. Being watched alone can halve performance (Behroozi et al. 2020).

**Fidelity ladder.** L1 solo aloud timed recorded → L2 AI voice → L3 known peer → L4 stranger → L5 senior/expert, unknown problem → real (less-preferred) loops.

**Learning science.** Deliberate practice (narrow goal, feedback, edge of ability) · retrieval · spacing · interleaving · desirable difficulty.

**Self-assessment.** Weak and biased; underestimation common. Score from the tape; dimensions before verdict (mediating assessments).

**Statistics.** SE = σ/√n; σ ≈ 0.6 → need 4–6 mocks per round type. Five-round loop: μ 3.2/σ 0.6 → ~52% clean; σ 0.4 → ~82%; μ 3.5/σ 0.6 → ~78%. Consistency ≈ +0.3 mean.

**Session.** Brief (T−24h) → setup → run (real length) → silent independent scores → debrief (candidate first, SBI, one fix) → swap → review tape ≤24h → log → drills → next mock checks the fix.

**Brief.** Level · round · problem · hidden twist · probe ladder (what → why → numbers → failure → change → stack precision) · interruption script · AI mode · what to write · fix to check.

**Interviewer rules.** Don't rescue · answer only what's asked · climb the ladder · honest clock · verbatim quotes · timestamp phases · stay in role.

**Rubric principles.** Behavioural anchors · 4 points (4 strong hire … 1 strong no hire) · independent dimensions · evidence required (none → max 2) · level-calibrated · knockouts · versioned.

**Six dimensions.** D1 framing · D2 depth (+ D2b .NET/Azure precision) · D3 trade-offs/judgement · D4 driving/communication · D5 collaboration/coachability · D6 level signal.

**Scoring.** Evidence → dimension scores → knockouts → hints → verdict (weighted, not averaged) → committee paragraph → one fix.

**Calibration.** Reference recordings before verdicts · partner agreement (exact ≥ 60%, within-one ≥ 90%) · test–retest · personal bias per dimension · map to real outcomes.

**Error classes.** K knowledge → study · R retrieval → aloud drills · S structure/time → time drills · C communication → narration · J judgement → decision sentences · P pressure → exposure + routine · L level → architect layer.

**Catalogue.** SD · coding no-AI · coding AI-enabled · .NET deep technical · behavioral · design-doc defence · brownfield · implementer round · stakeholder trade-off.

**Schedule.** Baseline → targeted (2 wks) → pressure → full loop → taper. ~15–20 full mocks + 20–30 drills. priority = rounds × gap + 0.5.

**AI.** Interviewer and grader in separate sessions; grader quotes evidence, capped at 2 without it; calibrate vs humans; voice > text; anonymise stories; never in a live round unless permitted (Amazon: disqualification risk; Anthropic: none unless indicated).

**AI-enabled rounds.** Multi-stage problem; scored on decomposition, verification, critical reading, design ownership, fundamentals, narration. Practise no-AI and whiteboard too (Google: ≥ 1 in-person round).

**Readiness gates.** Median ≥ 3.0 (architect rounds 3.5) · no 1s in last 3 · range ≤ 1.0 · ≥ 2 high-fidelity · top error gone · hints ≤ 1 → GO / GO-WITH-RISK / EXTEND / RESCHEDULE.

**On the day.** Routine (logistics, card, breath, reappraisal, fixed first move) · "I'm excited" > "calm down" · recovery sentences · no between-round post-mortems.

---
# Appendix A — Interviewer brief templates

Send one of these to your interviewer at least a day before. Keep it to one page.

### A1. System design

```text
CANDIDATE LEVEL / TRACK:  senior IC | staff | solution architect   ·  stack: .NET / Azure
ROUND:  system design · 45 | 60 min · Excalidraw | whiteboard · AI mode: none
PROBLEM (read aloud exactly):  "_______________________________________________"
ANSWER ONLY IF ASKED:  scale ______  read/write ______  latency ______  consistency ______  existing systems ______
HIDDEN TWIST (reveal if asked, else at ~12:00):  _________________________________
LIKELY DEEP DIVES:  1 ____________  2 ____________  3 ____________
PROBE LADDER for deep dive 1:
   1 what? ____  2 why over ____?  3 numbers: ____  4 failure: ____  5 10× / change: ____  6 .NET precision: ____
INTERRUPTION SCRIPT:  ~12:00 requirement change ____  ~26:00 pushback ____  ~36:00 "ten minutes left"
WRITE DOWN:  phase timestamps · numbers + conclusions · verbatim claims · ladder rung reached · hints
CHECK FIRST:  candidate's current fix: ___________________________________
DON'T:  rescue, reassure, volunteer information, run over time
```

### A2. Coding (C#)

```text
ROUND:  coding · 45 min · plain editor (no IntelliSense) | CoderPad | whiteboard · AI mode: none | permitted | expected
PROBLEM:  ____________________  (core + one extension for strong candidates)
CONSTRAINTS TO REVEAL IF ASKED:  input size ____  duplicates? ____  ordering? ____  thread safety? ____
EDGE CASES YOU EXPECT THEM TO RAISE:  ____ ____ ____
FOLLOW-UPS:  complexity? · how would you test it? · what changes if called concurrently? · API design changes?
AI-ENABLED ONLY:  stages: 1 ____ 2 ____ 3 ____ 4 ____ ; seeded bug: ____
WRITE DOWN:  time to first code · bugs found by candidate vs by you · tests discussed · hints
```

### A3. .NET deep technical

```text
ROUND:  45 min conversation + snippets
RAPID PROBES (2–3 min each, pick 5):  GC generations & LOH · Server vs Workstation GC · async state machine & SynchronizationContext ·
   ThreadPool starvation symptoms · DI lifetimes & captive dependency · EF Core change tracking & AsNoTracking · N+1 ·
   IOptions vs IOptionsSnapshot vs IOptionsMonitor · HttpClient/IHttpClientFactory · Span<T> limits (no async, no heap)
DEEP THREAD (20 min):  "After Tuesday's deploy, p99 latency of an ASP.NET Core API doubled; CPU is flat; thread count climbs."
   Reveal on request: sync-over-async in a new middleware (.Result on an HTTP call) · dotnet-counters shows ThreadPool queue length rising
WRITE DOWN:  hypotheses in order · tools named (dotnet-counters, dotnet-trace, dumps) · precision of claims
```

### A4. Behavioral

```text
ROUND:  45 min · 3–4 questions with follow-ups · target level ____ · company emphasis (e.g. Amazon LPs): ____
QUESTIONS:  1 conflict / technical disagreement  2 failure  3 leading through ambiguity  4 influence without authority
FOLLOW-UP LADDER (use 2–3 per story):  What exactly did YOU do? · What options did you reject, and why? · What was the
   measurable result? · What would you do differently? · What did the other person think was happening?
WRITE DOWN:  story chosen · "I" vs "we" · numbers · reflection · length
```

### A5. Architect rounds

```text
FORMAT:  design-doc defence | brownfield redesign | implementer round | stakeholder trade-off · 45–60 min
CURRENT STATE (give in writing):  C4 context + containers: ________________________________
BUSINESS PRESSURE:  driver ____ date ____ number ____
CONSTRAINTS:  team size/skills ____ budget ____ compliance ____ fixed dependency ____
PLANTED CONFLICTS (play these roles):  implementer concern ____ · finance/product concern ____ · security/ops concern ____
WHAT A STRONG ANSWER INCLUDES (write BEFORE the mock):  ____ ____ ____ ____
WRITE DOWN:  options offered · reversal conditions · how each conflict was handled (concede / defend with evidence / ignore)
```

---
# Appendix B — Full rubrics (behaviourally anchored)

Score 4 = strong hire · 3 = lean hire · 2 = lean no hire · 1 = strong no hire. **No evidence → max 2.** Anchors are written for a **senior IC**; for staff/architect, use the level adjustments in Concept 11.

### B1. System design

| Dimension | 4 | 3 | 2 | 1 |
|---|---|---|---|---|
| Requirements | Asked purpose and design-shaping NFRs; found the twist; stated scope and out-of-scope | Functional + key NFRs; scope stated | Features only; NFRs when prompted | Jumped to boxes |
| Estimation | Few numbers, each producing a stated decision | Correct numbers, most used | Numbers wrong, irrelevant or unused | None |
| High-level design | Every component justified; data flow and consistency per arrow; done ≤ ~22 min | Standard design, mostly justified; done ≤ ~25 min | Boxes without reasons; late (> 28 min) | Incoherent or incomplete |
| Deep dives | Two, where the difficulty lives; alternatives, choice, failure modes; rung 5+ | One solid deep dive (rung 4+) | Shallow everywhere (rung ≤ 2) | None |
| Failure handling | Each dependency's failure and policy stated unprompted | Covered when asked, correctly | Vague ("we retry") | Ignored |
| .NET/Azure precision | Exact semantics and limits; default → why → when not | Correct products with reasons | Product soup or several imprecise claims | Wrong claims |
| Pivots / wrap-up | Seams named; trade-off with a number; next scaling step | Reasonable pivot | Rewrites the design | Unable |
| Driving | Drove the round; offered deep-dive choices; checked in; no time collapse | Clear, mostly self-directed | Needed steering; time problems | Lost or silent |

**Knockouts:** a design that silently violates a stated hard requirement (loses money, loses OTPs) with no awareness; refusing to engage with a follow-up.

### B2. Coding (C#)

| Dimension | 4 | 3 | 2 | 1 |
|---|---|---|---|---|
| Problem understanding | Clarified contract, constraints, edge cases before coding; restated with an example | Clarified inputs/outputs; some edge cases | Started coding with misunderstandings, corrected later | Solved a different problem |
| Approach | Compared approaches with complexity; deliberate choice; scale note | Reasonable approach; complexity stated | Approach works but complexity unclear or poor without awareness | No workable approach |
| Code quality | Idiomatic modern C#; small functions; clear types and names; no premature cleverness | Readable; sensible names; some decomposition | Hard to follow; one big method; poor names | Unreadable |
| Correctness & testing | Walked through tests unprompted incl. edge cases; found own bugs; stated invariants | Main case works; fixes bugs when found | Bugs found by interviewer; little testing | Doesn't work; no testing |
| Communication | Narrated intent before code; summarised | Explained mostly | Long silences; explained only when asked | Silent |
| Speed | Core done with time for extension/tests | Core done | Core partly done | Little progress |
| AI use *(AI rounds)* | Decomposed for the assistant; verified everything; caught its mistakes aloud; owned design | Used for boilerplate; reviewed output | Accepted output with light review | Pasted the problem; couldn't explain the code |

**Knockouts:** can't explain code on screen; dishonesty about AI use.

### B3. Behavioral (score each story, then the round)

| Dimension | 4 | 3 | 2 | 1 |
|---|---|---|---|---|
| Fit to question | Answers the question and its underlying competency; best story chosen | Answers the question | Partially relevant | Off-topic |
| Ownership ("I") | Clear personal decisions distinct from the team's; scope at/above level | Clear personal role | Mostly "we"; role unclear | Others' work presented as own, or none |
| Complexity & stakes | Ambiguous, cross-team or high-stakes; risk explicit | Real problem with constraints | Routine task | Trivial |
| Actions & reasoning | Options considered and rejected with reasons | Actions described | Actions vague | None |
| Result | Quantified; impact on system/team/business; aftermath | Outcome stated | Outcome unclear | No result |
| Reflection | Specific learning, later applied; owns mistakes | Some learning | Generic ("communication matters") | Blames others |
| Delivery | ~2–3 min core; tight; handles 2nd and 3rd "why" | 2–4 min; structured | Rambling or too short | Unstructured |

**Knockouts:** blaming others throughout; taking credit for others' work; contempt for colleagues or users.

### B4. Architect rounds

| Dimension | 4 | 3 | 2 | 1 |
|---|---|---|---|---|
| Decision framing | Decision, drivers, reversibility and affected parties explicit | Decision and constraints stated | Jumps to a solution | Wrong problem |
| Options & trade-offs | Credible options with consequences, cost, risk; recommendation + reversal condition | Two options compared | One option, justified loosely | No options |
| Stakeholders | Each audience addressed in its own terms; dissent handled | Stakeholders acknowledged | Engineering-only view | Dismissive |
| Migration & risk | Risk-first sequencing; strangler/dual-run/shadow; rollback; measurable gates | Phasing mentioned | Big-bang plan | None |
| Cost & ownership | Cost drivers, ownership, on-call, build-vs-buy | Cost mentioned | Cost ignored until asked | Ignored |
| Documentation | ADR-shaped decision; C4 diagrams right for the audience | Clear explanation | Hard to follow | None |
| Defence under challenge | Concedes valid points, defends others with evidence, adjusts visibly | Answers challenges | Defensive or caves on everything | Hostile |

**Knockouts:** dismissing a stakeholder or implementer concern without engaging; a plan that requires a full stop-the-world rewrite with no awareness of risk.

---
# Appendix C — Debrief script (20 minutes)

```text
0:00  Both: silent independent scoring with evidence (already done before this clock starts)
0:00  Candidate: "One thing that went well, one that didn't." (60 s, interviewer listens)
1:00  Interviewer: the timeline — phase timestamps, numbers stated, hints. Facts only.
3:00  Compare scores row by row. For any gap ≥ 1: each gives their evidence. Agree, or record both.
9:00  Interviewer: 2–3 SBI points.  Situation (time) → Behaviour (what was said/done) → Impact (on the score)
      → Intent: "What were you aiming for there?"
14:00 Agree ONE fix. Test: is it observable? Can the next interviewer check it in the first 15 minutes?
16:00 Interviewer: "Here's the verdict I'd have written, and the one paragraph a committee would read."
18:00 Candidate: "What would you have needed to see for one level higher?"
20:00 Swap roles (peer mocks) or end.
```

---
# Appendix D — Mock log template and readiness checker

### D1. Per-mock entry (Markdown)

```text
#__  date ____  round ____  problem ____  fidelity L_  AI mode ____  length __m  interviewer: ____ (style ____)
Fix checked from last mock: ____ → fixed | partial | not fixed
Scores (interviewer / self-memory / self-tape):
   D1 _/_/_  D2 _/_/_  D2b _/_/_  D3 _/_/_  D4 _/_/_  D5 _/_/_  D6 _/_/_   Verdict _/_/_
Timeline: HLD complete __:__ · DD1 __:__–__:__ · DD2 __:__–__:__ · hints: __
Metrics: longest silence __s · numbers with conclusions _/_ · questions in first 5 min __
Errors (time · what · class K/R/S/C/J/P/L):
   __:__ ____________________ _
   __:__ ____________________ _
Fix for next mock: ____________________________________________
Drills before next mock: ________________________________________
```

### D2. CSV for trend analysis

```text
id,date,roundType,fidelity,verdict,minDimension,hints
1,2026-10-13,systemDesign,3,2.5,2,1
2,2026-10-14,coding,3,3,2,0
```

### D3. Readiness checker (C#)

Applies the Concept 22 gates per round type — another small tool worth writing yourself:

```csharp
// dotnet run -- mocks.csv
using System.Globalization;

var mocks = File.ReadLines(args[0]).Skip(1)
    .Where(line => !string.IsNullOrWhiteSpace(line))
    .Select(line => line.Split(','))
    .Select(f => new Mock(
        int.Parse(f[0], CultureInfo.InvariantCulture),
        DateOnly.Parse(f[1], CultureInfo.InvariantCulture),
        f[2],
        int.Parse(f[3], CultureInfo.InvariantCulture),
        double.Parse(f[4], CultureInfo.InvariantCulture),
        int.Parse(f[5], CultureInfo.InvariantCulture),
        int.Parse(f[6], CultureInfo.InvariantCulture)))
    .ToList();

foreach (var round in mocks.GroupBy(m => m.RoundType).OrderBy(g => g.Key))
{
    var recent = round.OrderBy(m => m.Date).ThenBy(m => m.Id).TakeLast(4).ToList();
    if (recent.Count < 4) { Console.WriteLine($"{round.Key,-14} only {recent.Count} mocks — keep sampling"); continue; }

    var last3 = recent.TakeLast(3).ToList();
    (string Gate, bool Pass)[] gates =
    [
        ("median>=3.0",   Median(last3.Select(m => m.Verdict)) >= 3.0),
        ("no 1s",         last3.All(m => m.MinDimension > 1)),
        ("range<=1.0",    recent.Max(m => m.Verdict) - recent.Min(m => m.Verdict) <= 1.0),
        (">=2 high-fid",  recent.Count(m => m.Fidelity >= 4) >= 2),
        ("hints<=1",      recent.All(m => m.Hints <= 1)),
    ];

    string verdict = gates.All(g => g.Pass) ? "PASS" : "NOT YET";
    Console.WriteLine($"{round.Key,-14} {verdict,-8} " + string.Join("  ", gates.Select(g => $"{(g.Pass ? "+" : "-")}{g.Gate}")));
}

static double Median(IEnumerable<double> values)
{
    var sorted = values.Order().ToArray();
    if (sorted.Length == 0) return double.NaN;
    int mid = sorted.Length / 2;
    return sorted.Length % 2 == 1 ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2;
}

public sealed record Mock(int Id, DateOnly Date, string RoundType, int Fidelity, double Verdict, int MinDimension, int Hints);
```

The "top recurring error is gone" gate is a judgement from the error log; leave it human.

---
# Appendix E — AI interviewer and grader prompts

### E1. Interviewer (voice mode preferred)

```text
You are a strict but fair interviewer at a large technology company running a {45}-minute {system design}
interview for a {senior software engineer, .NET/Azure}. Behave exactly like a real interviewer.

Problem (read it aloud, nothing more): "{Design a notification system that sends push, SMS, email and in-app notifications.}"

Rules:
- Ask one question at a time. Keep your turns short (1–3 sentences).
- Never propose solutions, never teach, never give hints unless I am silent or stuck for more than two minutes;
  if you give a hint, say "hint" explicitly.
- Answer clarifying questions only with the facts below. If I don't ask, don't volunteer them.
  Facts: {50M users; ~20M transactional notifications/day; weekly campaigns to 20M users; OTP within 10 s;
  at-least-once acceptable but duplicates visible to users are bad; Azure; team of 6 .NET engineers}.
- At about 12 minutes say: "{Marketing wants 50 million pushes in an hour next month.}"
- At about 26 minutes push back on my main design choice, even if it is reasonable, and see whether I hold
  or update my position with reasons.
- On my deep dive, keep asking deeper questions in this order until I stop giving precise answers:
  what → why that over an alternative → the numbers → what happens when it fails → what changes at 10× →
  exact semantics of the .NET/Azure component I named.
- Track time. At about 36 minutes say "We have about ten minutes left." End at 45 minutes.
- Do not praise or reassure during the interview. Do not score or give feedback. When time is up, say
  "That's time, thank you" and stop.
```

### E2. Grader (a new conversation; paste the rubric and transcript)

```text
You are a calibrated hiring-committee reviewer. Grade the interview transcript below for a {senior .NET/Azure
software engineer} using the rubric provided. You did not conduct this interview.

Procedure:
1. For each rubric dimension, list 2–4 pieces of evidence as direct quotes with timestamps from the transcript.
2. Score each dimension 1–4 using the anchors. If you cannot quote evidence for a level, you may not award it.
   Without evidence the maximum is 2. Be strict: grade as the most demanding interviewer at this level would.
3. List every .NET or Azure technical claim the candidate made and mark it ✓ precise, ~ vague, ✗ wrong,
   with a one-line correction for ~ and ✗.
4. Identify knockouts, if any.
5. Give the overall verdict (4 strong hire, 3 lean hire, 2 lean no hire, 1 strong no hire). Do not average —
   weigh the dimensions that matter most for this round type, and explain in one paragraph a committee would read.
6. Classify each miss as K (knowledge), R (retrieval), S (structure/time), C (communication), J (judgement),
   P (pressure) or L (level).
7. Propose exactly ONE fix: a specific, observable behaviour for the next mock.
Do not soften conclusions. Do not add encouragement.

RUBRIC:
{paste Appendix B table}

TRANSCRIPT:
{paste transcript with timestamps}
```

### E3. Drill partner (text is fine)

```text
Run a {10}-minute drill on {failure handling}. Name one dependency at a time from my design ({Azure Service Bus,
Azure Managed Redis, the SMS provider, Cosmos DB}). I have 60 seconds to state: what fails, what the user sees,
the policy, and the .NET mechanism. After each answer, say only "next" — save all feedback for the end, then
list my imprecise or wrong claims with corrections.
```

---
# Appendix F — Six-week calendar (default)

```text
WEEK 1  Baseline     Mon SD mock (L3) · Tue coding mock (L3) · Wed behavioral mock (L3) · Thu architect mock if relevant
                     Fri calibration: score 2 reference recordings; partner agreement on 1 · Weekend: write rubric v1
WEEK 2  Targeted     Mon drill ×2 · Tue full mock (top gap) · Wed drill ×2 · Thu AI mock + AI grader · Fri full mock (rotate)
WEEK 3  Targeted     Same shape; new error class; interleave round types; one blank-editor C# drill
WEEK 4  Pressure     Mon full mock (stranger, L4) · Tue drill · Wed full mock (challenging style) · Thu drill
                     Fri expert mock (L5) · Weekend: review, recalibrate personal bias
WEEK 5  Loop         Tue full-loop simulation (4–5 rounds) · Thu batched review · Fri one targeted full mock
WEEK 6  Taper        Mon light behavioral run-through · Wed one short SD mock (L4) · Thu logistics + card · Fri rest
                     Real loop following week (or a lower-preference company's loop first)
```

---
# Appendix G — Self-review checklist (recording)

```text
TIMELINE   □ requirements ≤ 5–7 min  □ estimation conclusions  □ HLD complete ≤ 22 min  □ two deep dives  □ wrap-up
NUMBERS    □ each number followed by "so…"  □ the twist found by asking
CLAIMS     □ every .NET/Azure claim marked ✓ / ~ / ✗  □ corrections written
DELIVERY   □ no unnarrated silence > 30 s  □ check-ins ≥ 2  □ no monologue > 3 min  □ summary ≤ 90 s
INTERRUPT  □ requirement change absorbed with a number  □ pushback: held or updated with reasons  □ time call handled
SELF       □ re-scored from tape  □ memory-vs-tape differences logged  □ errors classified  □ one fix written
```

---
# Appendix H — Readiness and loop-day checklists

### H1. Readiness gates (per round type)

```text
□ Rolling median (last 3–5) ≥ 3.0  (architect rounds ≥ 3.5 for architect roles)
□ No dimension scored 1 in the last 3; no knockouts
□ Range of last 4 verdicts ≤ 1.0
□ ≥ 2 of the last 4 at fidelity L4–L5
□ Top recurring error class absent in the last 2
□ Hints ≤ 1 per mock
□ AI mode practised exactly as the real round (confirmed with the recruiter)
DECISION:  GO | GO-WITH-RISK | EXTEND (ask for 1–2 weeks) | RESCHEDULE (study first)
```

### H2. Loop day

```text
NIGHT BEFORE  □ 10 min writing down worries, then close the notebook  □ card printed  □ sleep
MORNING       □ camera/mic/lighting/internet failover  □ whiteboard markers or tools logged in  □ water, food plan
EACH ROUND    □ T−10 card  □ breath + reappraisal sentence  □ fixed first move  □ time budget  □ recovery sentences ready
BETWEEN       □ no replay of the last round  □ walk, water  □ read the next round's card
AFTER         □ log what you remember the same evening (scores from memory, flagged as memory)  □ then stop analysing
```

---

*Next: **Module 39 — Company/role research checklist**: how to find out what a specific company's loop contains (round types, AI policy, in-person rounds, levelling), map its values and rubric vocabulary to the six dimensions, tailor your story bank and mock catalogue to it, and prepare the questions you'll ask your interviewers.*
