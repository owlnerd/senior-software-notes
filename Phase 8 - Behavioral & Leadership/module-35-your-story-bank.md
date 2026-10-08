# Module 35 — Your Story Bank
*Phase 8: Behavioral & Leadership · Senior/Architect Interview Prep for .NET & C#*

> **State of practice verified on October 8, 2026.** The method in this module — a small set of rich, true stories indexed against competencies — is stable and has been taught by career centres and interview coaches for decades. What matters for calibration is how many behavioral questions you will actually face, what the rubrics are called, and what the current practice tools and norms are:
>
> - **Volume.** Amazon's own preparation pages say each interviewer **typically asks two or three behavioral questions** tied to the Leadership Principles, and its SDE III phone screen spends about half its hour on them. A five-interview loop plus a phone screen therefore means roughly **12–18 behavioral questions** — the main reason a bank of one or two "great stories" fails.
> - **Amazon** still publishes **16 Leadership Principles** and assigns each interviewer a subset; the Bar Raiser and debrief compare notes across interviewers, so repeating one story across the loop is visible.
> - **Meta** runs a **dedicated 45-minute behavioral round**, scored on its own rubric, usually led by a senior engineer from outside the team you'd join. Commonly reported signal areas: **resolving conflict, growing continuously, embracing ambiguity, driving results, communicating effectively**. Prep sources note that different questions ("a conflict with a coworker", "a disagreement with your manager", "giving hard feedback") are often probing the *same* signal from different entry points.
> - **Google**'s Googleyness & Leadership round mixes **behavioral and hypothetical** questions; its leadership attribute explicitly includes **emergent leadership** — leading without a title, and knowing when to step back.
> - **Microsoft**'s careers site says it looks for **respect, integrity, accountability and growth mindset**, and encourages candidates to use AI tools responsibly during *preparation*, as long as the result reflects their true capabilities. Behavioral questions are typically woven into every round rather than isolated.
> - **Practitioner guidance on bank size** ranges widely — from 4–6 stories (some role-specific courses) through 5–10 and 8–12 (several 2025–26 guides) to 12–15 (a 2026 senior data-architect playbook). The point every source agrees on is structural: **each story should cover several themes, and each theme should be covered by more than one story.** This module derives a specific answer — **6–8 core stories plus 2–4 reserves** — from first principles in Concept 3.
> - **Practice tools.** Pramp's free peer mock interviews have run on **Exponent Practice** since July 2024; 2026 reviews report a free allowance of about **five peer sessions a month**. AI mock interviewers are widely available; the 2026 norm from Module 34 still holds — **AI for preparation, never for live answers** unless an interviewer explicitly permits it.
> - **Learning science** is unchanged and decisive for the practice plan: in Roediger and Karpicke's 2006 experiments, students who practised recall retained about **61%** of a passage after a week versus **40%** for those who reread it; Dunlosky and colleagues' 2013 review rated **practice testing** and **distributed practice** the two highest-utility learning techniques out of ten.
>
> Company formats are "verified on this date"; the method is stable.

## Orientation

Here is the sentence to carry through the whole module: **a story bank is an index, not an anthology — six to eight rich, true, level-appropriate stories, each sliced into "moments", tagged against the competencies you will be scored on, cross-checked so every competency has at least two strong options, and rehearsed by retrieval rather than rereading — so that any behavioral question becomes a lookup: decode the competency, select the story and moment, tell it.**

The curriculum entry reads: *Your story bank — mapping real experience to the 6–8 stories that cover most behavioral questions (conflict, failure, leading through ambiguity, technical disagreement, mentoring, influence without authority).* Module 34 ended by promising exactly this: Module 34 was about the **shape and level** of each story (STAR as a data format, the six seniority axes, the follow-up ladder); this module is about **which** stories, **how many**, **how they're indexed**, and **how you practise them** so they hold up under twelve to eighteen questions across a loop.

A note on tone. Most of this module is **preparation discipline**, not something you say to an interviewer. Each concept therefore ends with a **carry-away sentence** — the one idea to retain — rather than Module 34's "interview-grade sentence". Where a concept *is* something you might say aloud (to a recruiter, a coach, or an interviewer asking how you prepared), the carry-away sentence is phrased so you can.

How this connects to earlier modules:

- **Module 34** (STAR calibrated to seniority) — every story in your bank must meet Module 34's standard: headline first, actions with reasons, measured results, a changed mechanism, and readiness for the follow-up ladder. Module 34's Appendix A story card is extended here with a *moments index*.
- **Modules 1–2** (rubrics, IC vs architect evaluation) — the competency list your bank is indexed against is a rubric. This module turns rubrics into a coverage matrix.
- **Modules 30–33** (design documents, ADRs and C4, brownfield migration, cost and debt) — these are the *substance* of the architect-flavoured stories in your bank: the design you defended, the decision record that outlived you, the migration you sequenced, the business case you won.
- **Modules 13, 25, 28** (reliability, Polly, observability) — your failure and incident stories live here, told as blameless post-mortems on yourself.
- **Module 38** (mock interviews and self-scoring) — the practice plan in Part F feeds directly into it.

Why it matters in interviews:

1. **The question space is huge; the answer space must be small.** Lists of "top behavioral questions" run to hundreds. You cannot prepare hundreds of answers, and you shouldn't — interviewers ask many phrasings of perhaps a dozen competencies. Preparing *for competencies* rather than *for questions* is the only approach that scales.
2. **Retrieval under pressure is the real bottleneck.** Most experienced engineers *have* the stories. What fails in the room is finding the right one in five seconds, at the right level, without repeating the one you told the previous interviewer. A bank is a pre-computed index that removes that search from the critical path.
3. **Loops are read as a whole.** Interviewers write independent feedback, but debriefs and committees read it together. A loop where every interviewer heard the same migration story suggests a narrow career; a loop where the stories contradict each other on numbers suggests embellishment. Both are avoidable with a bank.
4. **Level is set by the *best* evidence you provide.** A bank lets you put your highest-scope stories in front of the competencies that matter most for the target level, rather than whatever comes to mind first.

This module has six jobs:

1. **Explain why a bank works** — the coverage problem, the data model (stories, moments, tags), the arithmetic behind "6–8", the canonical competency set, and how to decode question phrasings into competencies.
2. **Show how to mine your experience** — excavating episodes, rebuilding facts from evidence, scoring candidates, finding hub stories, balancing the portfolio, and treating client-delivery and founder work as first-class material.
3. **Define each core story type** — flagship project, conflict, technical disagreement, failure, ambiguity, influence without authority, mentoring, feedback and growth, plus the supporting cast — with what each must contain at senior and staff level.
4. **Build the coverage matrix** — the matrix as a data structure, mappings to Amazon's Leadership Principles, Google's attributes, Meta's signal areas and enterprise frameworks, gap analysis, and loop allocation.
5. **Make every story level-appropriate and probe-proof** — the level audit, the story pre-mortem, the fact sheet, truthfulness rules, and adapting a story to a question it wasn't built for.
6. **Give you a practice plan grounded in learning science** — retrieval, spacing, interleaving, a four-week schedule, mocks, the day-of protocol and long-term maintenance.

Seven framings to carry through:

1. **Index on competencies, not questions.** Questions are phrasings; competencies are what gets scored.
2. **A story is a container; a moment is the answer.** One rich project contains several answerable moments — a decision, a disagreement, a setback, a person you grew.
3. **Coverage needs redundancy.** Every important competency needs at least two strong stories, because you'll use one and need a fresh one for the next interviewer.
4. **Fewer, deeper stories beat more, thinner ones.** Each story must survive four levels of probing. You can't keep twenty stories that deep.
5. **The bank is a portfolio.** Balance success and failure, technical and human, peer and upward, recent and long-horizon — so the loop sees range.
6. **True or nothing.** Probes are designed to find embellishment, and interviewers compare notes. A modest true story beats an impressive composite.
7. **Practise retrieval, not recitation.** Being asked a random question and finding the story is the skill; rereading story cards isn't practice.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | The coverage problem | Hundreds of questions, a dozen competencies — prepare for the competencies |
| 2 | Stories, moments and tags | The data model: a story holds several moments; each moment is tagged with competencies |
| 3 | How many stories | 6–8 core + 2–4 reserve, derived from loop volume, redundancy and depth limits |
| 4 | The canonical competency set | Six core families plus a supporting cast cover almost every question |
| 5 | Decoding questions | Map the phrasing to the competency before you choose the story |
| 6 | The raw inventory | Excavate 25–40 episodes before you choose any |
| 7 | Evidence sources | Rebuild facts from artefacts, not memory |
| 8 | Screening | Score candidates on level, centrality, depth, richness, verifiability, recency |
| 9 | Hub and spoke stories | Two or three hub stories carry most of the weight |
| 10 | Portfolio balance | Diversity constraints the bank must satisfy as a whole |
| 11 | Client-delivery and founder careers | First-class raw material, told at the right altitude |
| 12 | The flagship project story | Your highest-scope story, ready for a 45-minute deep dive |
| 13 | Conflict | People disagree; no villains; relationship afterwards |
| 14 | Technical disagreement | Substance over people; options, evidence, decision, commitment |
| 15 | Failure | Right-sized, owned, mechanism-level, with evidence of change |
| 16 | Leading through ambiguity | What you were given vs what you created |
| 17 | Influence without authority | Why they followed someone who couldn't make them |
| 18 | Mentoring and growing others | A person, before and after, and what you specifically did |
| 19 | Feedback, being wrong, growth | You updated — visibly — and it stuck |
| 20 | The supporting cast | Deadline, customer, ownership beyond scope, saying no, simplifying, learning fast, cost |
| 21 | The matrix as a data structure | Stories × competencies, strength 0–3, with constraints |
| 22 | Amazon's Leadership Principles | Two stories per principle; the ones people forget |
| 23 | Google's attributes | Emergent leadership, Googleyness, hypotheticals anchored in real stories |
| 24 | Meta's signal areas | Five signals, many entry points; conflict weighs heavily |
| 25 | Microsoft, enterprises, EA, startups | Growth mindset, collaboration, stakeholders, governance, ownership |
| 26 | Gap analysis | Find thin columns; fill them honestly — re-mine, re-angle, or accept |
| 27 | Loop allocation | Plan primaries and backups; never repeat with one interviewer |
| 28 | The level audit | Run every story through the six axes at the target level |
| 29 | The story pre-mortem | Attack your own story like a sceptical Bar Raiser |
| 30 | The fact sheet and consistency | One canonical set of numbers, names and dates per story |
| 31 | Truthfulness | No composites, no borrowed credit, no inflated numbers — and why |
| 32 | Adapting on the fly | The 70% question, bridging, the honest near-miss |
| 33 | How memory for stories works | Retrieval, spacing, interleaving, desirable difficulty |
| 34 | The four-week plan | Inventory → cards → spoken drills → mocks |
| 35 | Mocks | Peers, AI interviewers, recordings — run them like experiments |
| 36 | Day-of protocol | Index card, tracking sheet, warm-up, after-action review |
| 37 | Maintaining the bank | Brag document, quarterly refresh, retire and replace |

---
# Part A — Why a story bank works

## Concept 1 — The coverage problem: many questions, few competencies

Start with an observation that every list of "top behavioral interview questions" makes obvious if you read it carefully: **the questions are many, but what they test is few.**

Take five real prompts:

1. *"Tell me about a time you had a conflict with a co-worker."*
2. *"Tell me about a disagreement with your manager."*
3. *"Describe a time you had to give someone difficult feedback."*
4. *"Tell me about someone you found difficult to work with."*
5. *"Have you ever had to push back on a senior stakeholder?"*

Five phrasings; one underlying competency — **handling interpersonal disagreement productively** — with variations in *direction* (peer, upward, downward, cross-functional) and *flavour* (conflict, feedback, pushback). An interviewer assigned "resolving conflict" (Meta) or "Have Backbone; Disagree and Commit" (Amazon) may ask any of them.

**Think of it as a many-to-few mapping.** If *Q* is the set of possible questions (practically unbounded — every interviewer rephrases) and *C* is the set of competencies on the rubric (roughly 10–16 depending on the company), there is a function *f: Q → C* (sometimes *Q → C × C* — a question that tests two things). Your preparation should be **indexed on the codomain**, not the domain:

| Approach | What you prepare | Scales? | Typical failure |
|---|---|---|---|
| **Question-indexed** | An answer per question from a list of 50–100 | No — the list is never complete, and answers blur | A phrasing you didn't see; or the same story told badly six ways |
| **Competency-indexed** | Stories tagged by the competencies they evidence | Yes — new phrasings map to existing competencies | Requires a decoding step (Concept 5) — which is learnable |

**Why competency count is small — from first principles.** A rubric is a model of what predicts success in the job (Module 34, Concept 2: structured interviews derive questions from job analysis). Job analyses for software engineering roles keep finding the same behaviors: delivering results, handling ambiguity, collaborating and resolving conflict, influencing, learning, raising quality, developing others, and judgment about trade-offs. Different companies name and weight them differently, but the underlying set is small and overlapping — which is why Concept 4 can give you a single canonical set that maps onto Amazon, Google, Meta, Microsoft and enterprise frameworks.

**What this changes in preparation:**

- **Stop collecting questions; start collecting competencies.** A question list is useful only as *test data* for your bank (Appendix D), not as the thing you prepare.
- **Expect the same competency several times in one loop**, phrased differently, by different interviewers. That's not bad luck; it's how a loop gets independent evidence on the competencies that matter most. You'll need *different stories* for each occurrence (Concept 3).
- **Expect two-competency questions.** "Tell me about a time you disagreed with your manager and turned out to be wrong" tests conflict *and* being wrong. Your story must contain both moments.

**Carry-away sentence:** *"I prepare for competencies, not questions: behavioral questions come in hundreds of phrasings but test roughly a dozen competencies, so I index my stories by what they evidence — conflict, ambiguity, failure, influence and so on — and treat question lists as test data for that index rather than as answers to memorise."*

---

## Concept 2 — Stories, moments and tags: the data model

Here is the idea that turns six to eight stories into coverage for dozens of questions: **a story is not an answer. A story is a container of answers.**

**Definitions:**

- **Story** — a bounded real episode from your career: a project, an incident, a programme, a relationship over time. It has a situation, stakes, a cast, a timeline and an ending. *"Moving the ordering system off the legacy platform, 2023–24."*
- **Moment** — a slice of a story that, told with its own STAR shape, evidences one competency. *"Week 6: the reporting team refused to change their schema; how I resolved it."* A moment reuses the story's situation (so you only explain context once in your head) but has its own task, actions, result and reflection.
- **Tag** — a competency label attached to a moment, with a **strength** (how good the evidence is for that competency at your target level).

As an engineer, read it as a schema:

```text
Story (1) ──< Moment (n) >── Tag (m) ── Competency
  id, title, period,           id, story_id, title,      moment_id, competency,
  context, scope markers,      headline, task, actions,  strength (0–3)
  canonical numbers            result, reflection
```

**Why moments, not whole stories, are the unit of answer.** Interviewers ask about a competency. A whole project told end to end evidences many competencies *weakly* — a little conflict, a little ambiguity, a little delivery — and takes eight minutes. A moment evidences one competency *strongly* in two to three minutes, with the rest of the story available as context and as material for follow-ups. A good answer is: *the story's context in two sentences → the moment told fully → the outcome of the whole story as part of the result.*

**An example: one migration story, six moments.**

| Moment | Competency it answers | The question it fits |
|---|---|---|
| Choosing strangler fig over a rewrite | Technical judgment / design defense | "Tell me about an important technical decision" |
| The reporting team refusing schema changes | Conflict / influence without authority | "A time you needed something from a team that didn't want to give it" |
| Discovering two months in that the cutover plan was wrong | Being wrong / adapting | "A time you changed course" |
| Handing the order module to a mid-level engineer | Mentoring / delegation | "Tell me about someone you helped grow" |
| Negotiating the go-live criteria with the business | Stakeholder management / saying no | "A time you pushed back on a deadline" |
| Shutting down the old system and the licence saving | Delivering results / frugality | "Your most impactful project" |

Worked example 2 tells all six. The point here is structural: **a rich story with five or six moments is worth more to your bank than five thin stories with one moment each**, because the context only needs to be learned once, the facts are consistent by construction, and follow-ups in one direction can draw on the rest of the story.

**Two cautions about reuse:**

1. **Don't use two moments of the same story with the same interviewer.** They'll see it as a narrow career. Across a loop, reuse is acceptable in moderation (Concept 27) — but each interviewer should hear different stories.
2. **Moments must be genuinely separable.** A "mentoring moment" where the mentoring was one sentence ("and I also helped a junior") isn't a moment; it's a mention. Each moment must stand up to the follow-up ladder on its own (Module 34, Concept 14).

**Tags have strengths.** Not every moment is equally good evidence. Use a 0–3 scale for each tag:

| Strength | Meaning |
|---|---|
| **0** | Doesn't evidence this competency |
| **1** | Touches it, but thin or below target level |
| **2** | Solid evidence at target level |
| **3** | Strong, distinctive, at or above target level, probe-proof |

Concept 21 builds the coverage matrix from these tags.

**Carry-away sentence:** *"I model my preparation as stories containing moments: a story is a bounded real episode, a moment is a slice of it that answers one competency with its own task, actions and result, and each moment is tagged with the competencies it evidences and how strongly — which is how six to eight rich stories cover dozens of questions without my telling the same thing twice."*

---

## Concept 3 — How many stories: deriving "6–8 core plus 2–4 reserve"

Practitioner advice ranges from four stories to fifteen. Rather than pick a number on authority, derive it from three constraints.

### Constraint 1 — Volume: how many behavioral questions will you face?

Use Amazon as the upper bound, since it's the most behavioral-heavy major loop and publishes its format:

```text
Phone screen (SDE III):  ~30 min of LP questions      ≈ 2–4 behavioral questions
On-site loop:             5 interviewers × 2–3 each    ≈ 10–15 behavioral questions
Total                                                  ≈ 12–19 behavioral questions
```

Other loops are lighter but not trivial: Meta's dedicated 45-minute behavioral round typically covers four to six questions plus follow-ups; Google's Googleyness & Leadership round several; hiring-manager rounds and Microsoft-style loops weave one or two behavioral questions into most rounds. **Plan for 10–15 behavioral questions in a full senior or staff loop.**

### Constraint 2 — Redundancy: you need more than one answer per competency

Three reasons:

1. **The same competency is asked more than once** in a loop — conflict and ownership are asked by several interviewers at many companies.
2. **You should not tell the same story twice to the same interviewer**, and should limit repetition across the loop, because the debrief reads feedback together.
3. **A story sometimes doesn't fit** the exact phrasing — you need a second option.

So: **each core competency needs at least two stories with strength ≥ 2.** With six core competency families (Concept 4), that's twelve competency-slots to fill with strong evidence.

### Constraint 3 — Depth: you can't keep many stories probe-proof

Every story in your bank must survive the follow-up ladder: attribution, alternatives, measurement, the other side's view, reflection — and three or four levels of "dive deep" on one technical detail (Module 34, Concept 14). That requires knowing the facts cold: numbers, dates, who said what, what you considered and rejected. **Practitioners consistently find that people can maintain roughly six to ten stories at that depth**; beyond that, details blur and stories begin to contaminate each other ("was it 40% or 60%? was that the billing project or the pricing one?").

### Putting it together

If each core story has **three to five usable moments** (Concept 2), and the bank needs **~12–16 strong competency-slots** (six core families × two, plus the supporting cast), then:

```text
slots needed ÷ strong moments per story  ≈  14 ÷ 2.5  ≈  6 stories (lower bound)
+ redundancy for repetition limits and imperfect fit   →  7–8
+ a few single-purpose reserves for niche principles   →  2–4 reserves
```

(The "2.5" is deliberately conservative: a story may have five moments, but typically only two or three are *strong* at your target level.)

**Result: six to eight core stories, each rich (three to five moments, at least two of them strong), plus two to four reserve stories** — shorter, single-purpose stories for competencies the core doesn't cover well (often Frugality, Customer Obsession, learning something fast, or a second failure). That's the number in the curriculum entry, and it matches the structural consensus across sources: *several themes per story, several stories per theme.*

**When to deviate:**

- **Fewer (5–6)** if your career is concentrated in two or three very rich projects — but then each must yield five strong moments, and you must manage repetition carefully.
- **More (9–10 core)** for Amazon loops at staff level or for principal/EA roles where the competency list is long — but only if you can keep each one deep.
- **Never fewer than two failure-capable stories** and **never fewer than two conflict-capable stories**: these are the most frequently asked families and the ones where a weak answer costs the most.

**Carry-away sentence:** *"I keep six to eight core stories plus two to four reserves, because a full loop asks ten to fifteen behavioral questions, each core competency needs at least two strong options so I never repeat myself to an interviewer, and that's about as many stories as I can keep probe-proof — so each core story has to be rich enough to yield three to five moments."*

---

## Concept 4 — The canonical competency set

To index stories you need a fixed set of competencies. Company frameworks use different names, but they overlap heavily. The set below is the one this curriculum uses; Part D maps it onto each company's rubric.

### The six core families (from the curriculum entry)

| # | Family | What's really being tested | Typical phrasings |
|---|---|---|---|
| **C1** | **Conflict** (interpersonal) | Can you disagree with *people* — peers, managers, stakeholders — productively, fairly, and keep the relationship? | "conflict with a co-worker", "disagreement with your manager", "difficult person", "difficult feedback you gave" |
| **C2** | **Failure** | Do you own mistakes, understand their mechanism, and change something structural? | "a time you failed", "biggest mistake", "a project that went wrong", "something you'd do differently" |
| **C3** | **Leading through ambiguity** | Can you make progress — and create structure — when the problem or goal isn't defined? | "incomplete information", "unclear requirements", "no clear owner", "had to figure out what to do" |
| **C4** | **Technical disagreement** | Can you resolve disagreements about *substance* — designs, technologies, approaches — on evidence, and commit? | "disagreed on a technical decision", "defended a design", "convinced the team to change approach" |
| **C5** | **Mentoring / growing others** | Do you multiply others — specifically, with a before and after? | "helped someone grow", "mentored", "onboarded", "raised the team's bar" |
| **C6** | **Influence without authority** | Can you change what people do when you can't tell them to? | "drove a change across teams", "convinced people you didn't manage", "got buy-in" |

Notice the deliberate split between **C1 (conflict about people and priorities)** and **C4 (disagreement about technical substance)**. Many candidates use one story for both and score poorly on one of them. Concept 14 explains why they are evaluated differently.

### The supporting cast (asked frequently; often covered by moments of core stories)

| # | Competency | Typical phrasings |
|---|---|---|
| **S1** | **Flagship / most significant project** | "most complex project", "proudest achievement", "biggest impact", "walk me through a project" |
| **S2** | **Delivering under pressure** | "tight deadline", "competing priorities", "had to cut scope" |
| **S3** | **Customer / user focus** | "went above and beyond for a customer", "used customer feedback", "understood user needs" |
| **S4** | **Ownership beyond scope** | "something outside your job description", "noticed a problem nobody owned" |
| **S5** | **Feedback and being wrong** | "critical feedback you received", "changed your mind", "were wrong" |
| **S6** | **Prioritisation / saying no** | "said no to a stakeholder", "had to choose between two important things" |
| **S7** | **Simplification / innovation** | "simplified something complex", "invented a new approach" |
| **S8** | **Raising the bar / quality** | "insisted on high standards", "pushed back on cutting corners" |
| **S9** | **Learning fast** | "learned a new technology quickly", "stepped into an unfamiliar domain" |
| **S10** | **Cost / frugality** | "did more with less", "reduced cost", "worked under resource constraints" |
| **S11** | **Dive deep** | "the hardest bug", "a time data contradicted intuition", "got to the root cause" |
| **S12** | **Motivation and fit** | "why this company", "why leave", "what do you want next" — not STAR stories, but your bank's headlines feed them |

S5 (feedback and being wrong) is close enough to "growing continuously" (Meta) and "growth mindset" (Microsoft) that this module treats it as a **near-core** family in Part C (Concept 19). S1 also gets its own concept (Concept 12) because at staff level the flagship story is often an entire round.

**The rule of thumb:** a bank that covers the six core families twice each, and the supporting cast at least once each, covers the overwhelming majority of what senior and staff loops ask.

**Carry-away sentence:** *"I index my bank against six core families — conflict, failure, ambiguity, technical disagreement, mentoring and influence without authority — plus a supporting cast of flagship project, delivering under pressure, customer focus, ownership beyond scope, feedback and being wrong, saying no, simplifying, raising the bar, learning fast, cost and dive-deep; I keep conflict about people separate from disagreement about technical substance because they're scored differently."*

---

## Concept 5 — Decoding questions: phrasing → competency → story

With a competency-indexed bank, answering becomes a three-step lookup:

```text
question ──decode──▶ competency (+ direction, level, polarity) ──select──▶ story + moment ──tell──▶ answer
```

The **decode** step is where most mis-answers originate: a candidate hears "a time you had to deliver with incomplete information", decodes it as *delivery*, and tells a deadline story — when the interviewer was assigned *ambiguity* and wanted to hear how you created structure.

**Decode four attributes, not one:**

| Attribute | Question to ask yourself | Example |
|---|---|---|
| **Competency** | What is being scored? | "incomplete information" → ambiguity (C3), possibly bias for action |
| **Direction** | With whom — peer, manager (upward), report/junior (downward), other team, customer, executive? | "disagreed with your manager" → conflict, *upward* |
| **Polarity** | Do they want a success, a failure, or a mixed outcome? | "a time you were wrong" → negative polarity; a success story won't do |
| **Level cue** | Does the phrasing hint at scope? | "across teams", "organization", "strategy" → staff-level scope expected |

**Keyword heuristics** (Appendix D has a full decoder bank):

| If you hear… | Think… |
|---|---|
| "disagree", "conflict", "difficult person", "push back on someone" | C1 conflict — check direction |
| "disagree on an approach / design / technology", "defend a decision" | C4 technical disagreement |
| "fail", "mistake", "went wrong", "regret", "do differently" | C2 failure — negative polarity |
| "unclear", "incomplete", "ambiguous", "no owner", "figure out" | C3 ambiguity |
| "convince", "buy-in", "influence", "didn't report to you", "across teams" | C6 influence |
| "mentor", "grow", "coach", "onboard", "develop" | C5 mentoring |
| "feedback you received", "changed your mind", "criticism" | S5 feedback / being wrong |
| "deadline", "pressure", "competing priorities" | S2 delivery |
| "customer", "user", "client" | S3 customer focus |
| "outside your role", "nobody asked", "noticed" | S4 ownership |
| "proudest", "most complex", "biggest impact", "walk me through" | S1 flagship |

**Compound questions.** Some questions encode two competencies: *"Tell me about a time you disagreed with your team and later realised you were wrong."* Decode both, and choose a story where both moments are real. If no story has both, pick the one with the stronger *rarer* moment (here, being wrong) and be honest about the other.

**When decoding is uncertain, ask.** A short clarifying question is entirely acceptable and is itself a senior signal: *"Do you want a disagreement about a technical approach, or more about a working relationship?"* Google's own preparation guidance says rephrasing or asking for clarity is fine. Interviewers would rather redirect you in five seconds than get a well-told answer to the wrong competency.

**Hypothetical questions** ("What would you do if two senior engineers on your team disagreed?") decode the same way. At senior level, answer the hypothetical briefly and then anchor it in a real moment from your bank: *"Here's how I'd approach it — and here's a time something close to that happened."* (Concept 23.)

**Carry-away sentence:** *"Before choosing a story I decode the question into four things — the competency being scored, the direction of the relationship, whether they want a success or a failure, and any level cue like 'across teams' — and if I'm unsure which competency they mean, I ask a one-line clarifying question rather than answering the wrong one well."*

---
# Part B — Mining your experience

## Concept 6 — The raw inventory: excavate before you select

The most common story-bank mistake is **selecting too early**: the candidate thinks of their three most memorable projects, writes them up, and stops. Memorable is not the same as best. Vivid memories are biased towards recent events, dramatic events and events with strong emotion — and against the quiet, high-leverage work (a design review that prevented a bad migration, a standard that eleven teams adopted) that often carries the strongest staff-level evidence.

So split the work into two phases, as you would any search problem: **generate broadly, then filter.** This concept is the generate phase.

**Target: 25–40 raw episodes** before you choose anything. An episode is any bounded event or period where something happened that you could describe in a sentence. Quantity matters here because the filter (Concept 8) will discard most of them.

**Method 1 — The timeline sweep.** Go role by role (or client by client, or product phase by product phase). For each, write the years and then list everything that happened:

```text
Role / engagement:     ___________________   (years: ____–____)
Systems I owned or shaped:
Launches / migrations / releases:
Incidents (mine, my team's, ones I helped with):
Decisions I made or influenced:
Disagreements (technical and personal):
People I helped / who helped me:
Things I'd do differently:
Things I started that outlived me:
Numbers I remember:
```

**Method 2 — Prompt-driven recall.** Memory is cue-dependent: you recall episodes far better when given specific prompts than when asked "what did you do?". Run through these prompts for each role:

| Prompt family | Prompts |
|---|---|
| **Firsts and lasts** | The first big thing you shipped there; the last thing you did before leaving; the first time you led something |
| **Pain** | The worst week; the incident you remember; the deadline you nearly missed; the project that was cancelled |
| **Friction** | The person you found hardest to work with; the decision you argued against; the time you escalated; the time someone escalated against you |
| **Surprise** | The time data contradicted everyone's intuition; the bug that wasn't where anyone thought; the requirement that turned out to be wrong |
| **Leverage** | Something you built that others used; a practice you introduced; a person who now does something they couldn't before |
| **Restraint** | Something you deliberately didn't build; a rewrite you argued against; a feature you talked a stakeholder out of |
| **Being wrong** | A view you held strongly and abandoned; feedback that stung and was right |
| **Money and customers** | A cost you cut; a customer you saved; revenue you protected or enabled; a client you won or kept |
| **Outside the job** | Things nobody asked you to do; gaps you filled because no one else would |

**Method 3 — Artefact-driven recall** (Concept 7 covers it fully): your git history, design documents, post-mortems, calendar and old messages will surface episodes you've forgotten entirely.

**Write each episode as one line**, with a title and a hook:

```text
E07  2023 Q2  Orders → Service Bus migration      — ERP downtime stopped losing orders; reporting team pushback
E08  2023 Q3  EF Core pooling incident             — my DbContext change exhausted connections at peak (my fault)
E09  2023 Q4  Onboarded two contractors            — wrote starter tasks; time-to-first-PR halved
E10  2024 Q1  Argued against Cosmos DB for catalog — lost the first round; spike changed the decision
```

**Don't judge yet.** Episodes that seem small ("I fixed the flaky test suite") sometimes turn out to be the best ownership-beyond-scope story you have. The filter comes next.

**Carry-away sentence:** *"I build my bank in two phases, like any search: first I excavate twenty-five to forty raw episodes role by role, using timeline sweeps, specific prompts — worst week, hardest person, decision I argued against, something I deliberately didn't build — and old artefacts, writing each as a one-line title and hook; only then do I filter, because the most memorable stories aren't necessarily the strongest ones."*

---

## Concept 7 — Evidence sources: rebuild the facts from artefacts, not memory

Memory of past projects is not just incomplete; it's **reconstructive**. Each time you retell a story your memory of it is partly rebuilt, and details drift — numbers round up, timelines compress, your own role grows. That's normal human cognition, not dishonesty. But in an interview it is a liability: probes test details, and two interviewers who hear different numbers for the same story will notice in the debrief.

So for every story that makes the shortlist, **rebuild the facts from artefacts** before you rehearse it.

**Where the evidence lives:**

| Source | What it gives you | Notes |
|---|---|---|
| **Git history, PRs, code review threads** | Dates, your actual contributions, who reviewed what, the size of changes | `git log --author`, PR descriptions; your review comments reveal decisions you've forgotten |
| **Design documents, RFCs, ADRs** | Options considered, the reasons, who objected and why | The single best source for the *Action* section (Modules 30–31) |
| **Post-mortems / incident reports** | Timelines, impact numbers, root causes, action items | The backbone of failure and incident stories (Module 28) |
| **Dashboards and metrics** | Baselines and results: latency, error rates, cost, throughput | Screenshots you saved; old dashboards may still exist |
| **Tickets / project trackers** | Scope, dates, cycle times, how many people were involved | Good for delivery and deadline stories |
| **Calendar** | Who you met, how often, when — reconstructs influence campaigns | Shows the pre-wiring and working sessions you've forgotten |
| **Chat and email** | The disagreement, the escalation, the "thank you" from a stakeholder | Quote-worthy moments; the other side's actual words |
| **Performance reviews and promotion packets** | What your manager credited you with, in their words | Good for calibrating level claims — if your manager didn't see it as staff work, an interviewer may not either |
| **Release notes, demo decks, status reports** | What shipped, when, and what was claimed at the time | Contemporary claims are usually more accurate than today's memory |
| **Statements of work, client reports** (client delivery) | Scope, deliverables, client feedback, follow-on contracts | The evidence of shaping requirements and of client satisfaction |
| **Your brag document** | Everything, if you kept one | If you didn't, start now (Concept 37) |
| **Former colleagues** | Their memory of your role; numbers you can't find | A short message — "do you remember the p99 before the fix?" — is entirely normal |

**Confidentiality still applies.** You are reconstructing facts *for yourself*. You don't need to take documents from a former employer (and shouldn't), and you will anonymise in the telling (Module 34, Concept 34). Often you only need to *confirm* a number or a date, not to keep the document.

**What to extract per story** — this becomes the story's **fact sheet** (Concept 30):

- Dates and duration; your role and title at the time.
- Scope markers: people, teams, systems, load, money.
- Baselines and results with how they were measured.
- The options considered and the reasons for the decision.
- Who disagreed, and their best argument in their own words if you can find it.
- What happened afterwards — six months, a year, two years later.

**When you can't recover a number**, decide your honest approximation *now*, write it on the fact sheet, and use it consistently: *"roughly a 60% reduction — I don't have the exact figure."* Module 34, Concept 27 covers honest quantification.

**Carry-away sentence:** *"Because memory of old projects is reconstructive and numbers drift with every retelling, I rebuild each shortlisted story from artefacts — commits and PRs, design docs and ADRs, post-mortems, dashboards, tickets, calendar, reviews, and a quick message to former colleagues — and write one canonical fact sheet per story, with an honest approximation decided in advance wherever I can't recover the exact figure."*

---

## Concept 8 — Screening: scoring candidate stories

Now filter the 25–40 episodes down to a shortlist of 10–14, from which the 6–8 core stories and 2–4 reserves will be chosen after the coverage matrix (Part D).

**Seven screening criteria:**

| Criterion | Question | Why it matters |
|---|---|---|
| **Level** | Does this episode show scope, ambiguity, influence and impact at my *target* level? | Junior-level stories don't get senior offers, however well told (Module 34, Concept 26) |
| **Centrality** | Was I central to it — did I make the key decisions or do the key work? | Interviewers probe attribution; peripheral involvement collapses under "what did *you* do?" |
| **Depth** | Can I go three or four levels deep on at least one technical or human detail? | The follow-up ladder is where level is confirmed |
| **Richness** | How many separable moments does it contain? | Rich stories cover more competencies with less memorisation (Concept 9) |
| **Verifiability** | Do I have (or can I rebuild) numbers, dates and specifics? | Specifics make stories credible; vagueness makes true stories sound invented |
| **Recency** | Within the last 3–5 years — or clearly still representative? | Old stories invite "what have you done lately?" |
| **Tension** | Was there a real obstacle — a disagreement, a constraint, a setback, a risk? | Frictionless stories carry little evidence; behavior is revealed under difficulty |

**Score each 0–3 and weight by target level.** A simple weighted sum works; the weights below reflect what matters most at senior/staff level:

| Criterion | Weight (senior) | Weight (staff/architect) |
|---|---|---|
| Level | 3 | 4 |
| Centrality | 3 | 3 |
| Depth | 2 | 2 |
| Richness | 2 | 2 |
| Verifiability | 2 | 2 |
| Recency | 1 | 1 |
| Tension | 2 | 2 |

```text
score(episode) = Σ weight(c) × rating(c)     max = 3 × Σ weights   (senior: 45, staff: 48)
```

**Read the score as a guide, not a verdict.** Two adjustments matter more than arithmetic:

1. **Hard floors.** Any episode with **Centrality ≤ 1** or **Level ≤ 1 (for core stories)** is out, however high it scores elsewhere. A brilliant project you watched is not your story.
2. **Uniqueness bonus.** Keep at least one episode that only *you* could tell — an unusual domain, an unusual constraint, a distinctive decision. Interviewers hear dozens of "we migrated to microservices" stories; they remember the one about repair technicians on a factory floor.

**Recency trade-offs.** An older story can still be your best evidence — particularly for long time horizons ("two years later, it's still the standard"). Use it, and say why: *"This is from five years ago, but it's the best example of a decision whose consequences I saw play out over years."*

**Output of this step:** a ranked shortlist of 10–14 episodes, each with its score, its likely moments, and a note of which criteria are weak.

**Carry-away sentence:** *"I screen raw episodes on seven criteria — level, my centrality, depth I can go to, number of moments, verifiable specifics, recency and real tension — weighted towards level and centrality for staff roles, with hard floors so nothing I was peripheral to makes the bank, and I keep at least one story only I could tell."*

---

## Concept 9 — Richness: hub stories and spoke stories

Shortlisted stories are not equal. In almost every strong bank, **two or three stories carry most of the weight**. Call them **hub stories**; the rest are **spoke stories**.

**A hub story** is a long, high-scope episode — usually a multi-quarter project or programme — with five or more separable moments, at least three of them strong at the target level. Typical hubs: the migration you led, the platform you built, the programme you rescued, the product you launched. Your **flagship story** (Concept 12) is always a hub.

**A spoke story** is shorter and more focused — an incident, a disagreement, a mentoring relationship — usually with one to three moments. Spokes give you **independence from the hubs**: they let you answer a conflict question with a story the interviewer hasn't heard pieces of already.

**The seven ingredients of richness.** A story is rich to the extent it contains these — score them while screening:

1. **Stakes** — something important for customers, money or the organization.
2. **A hard decision** — real options, a trade-off, a cost accepted.
3. **A person who disagreed** — with a fair, strong argument.
4. **A setback** — something went wrong or turned out differently.
5. **A measurable result** — baseline and outcome.
6. **A lasting mechanism** — something that kept working after you.
7. **A lesson that changed you** — a practice you still use.

A story with all seven is a hub candidate. A story with three or four is a good spoke.

**Why hubs are efficient.** The cost of a story in your bank is mostly fixed: learning its context, its cast, its numbers and its timeline. The value is roughly proportional to the number of strong moments. Hubs have the best value-to-cost ratio — the same reason you'd normalise data rather than duplicate it.

**Why you can't rely on hubs alone.** Three risks:

- **Narrowness.** If every answer in a loop comes from the same project, the debrief concludes your experience is narrow — or that one project is the only time you operated at this level.
- **Contamination.** Telling many moments from one story makes it easy to slip into re-telling the context each time, burning minutes.
- **Single points of failure.** If an interviewer probes deeply into one moment and finds a weakness, every other moment from that story is weakened too.

**Shape of a healthy bank:**

```text
Hubs (2–3):    flagship programme · second large project · possibly a long-running responsibility
Spokes (4–5):  a failure · a conflict · a mentoring relationship · a technical disagreement · a feedback story
Reserves (2–4): frugality · a customer moment · learning something fast · a second failure
```

**Carry-away sentence:** *"In my bank two or three hub stories — long, high-scope projects with five or more moments, containing stakes, a hard decision, a fair opponent, a setback, a measured result, a lasting mechanism and a lesson — carry most of the weight, and four or five focused spoke stories give me independence from them, so that no interviewer hears the whole loop come from one project."*

---

## Concept 10 — Portfolio balance: the diversity constraints

A bank isn't judged story by story; across a loop it's judged **as a portfolio**. Interviewers write independently, but the debrief reads the packet together and forms a picture: *what kind of engineer is this, across what range of situations?* Balance the bank against these constraints — the same way you'd check a test suite for coverage across code paths, not just a count of tests.

| Dimension | Balance to aim for | Why |
|---|---|---|
| **Polarity** | At least 2 failure/negative stories; at least 1 mixed outcome | All-success banks read as either low-risk work or selective memory |
| **Substance** | Technical-heavy and people-heavy stories in roughly equal number | Only technical → doubts about reach; only people → doubts about depth (Module 34, Concept 33) |
| **Direction of relationships** | Peer, upward (manager/leadership), downward (junior/mentee), cross-team, external (customer/client/vendor) | Conflict and influence questions specify direction; you need one of each |
| **Scale** | At least two stories at the *target* level; others may be smaller but must be at least target − 1 | Level is set by your best evidence; the rest shows range |
| **Time horizon** | At least one story whose consequences you saw over a year or more | Staff-level evidence requires "since then" (Module 34, Concept 20) |
| **Context** | Stories from more than one employer, client or product, where possible | Shows the behavior generalises, not one lucky environment |
| **Role shape** | Stories matching the target archetype (tech lead, architect, solver, right hand — Module 34, Concept 25), plus at least one from another | Matching the role; avoiding one-dimensionality |
| **Emotional range** | A story where you were frustrated, one where you were wrong, one where you were proud | Authenticity; questions like "what frustrates you?" exist |
| **Domain** | Avoid every story being the same technology | Five Cosmos DB stories make you a Cosmos DB specialist in the reader's mind |

**Check it with a quick tally** once the matrix is built:

```text
Polarity:   success 5 · mixed 2 · failure 2           ✔
Substance:  technical 4 · people 3 · both 2           ✔
Direction:  peer 3 · upward 2 · downward 2 · cross-team 3 · external 1   ✔ (external thin)
Horizon:    ≥1 year consequences: 2 stories           ✔
Context:    3 employers/clients                        ✔
```

A thin dimension doesn't always mean a new story — often an existing story has an unused moment in that direction (Concept 26).

**Carry-away sentence:** *"I check my bank as a portfolio, not story by story: at least two failures and a mixed outcome, technical and people stories in balance, every relationship direction from peer to upward to downward to cross-team to external, at least two stories at target level, at least one whose consequences I saw over a year or more, and more than one context — because the debrief reads the whole loop and forms a picture of my range."*

---

## Concept 11 — Client-delivery and founder careers as raw material

Two career shapes need specific mining techniques, because their best evidence is often hidden by how the work is usually described: **client delivery** (outsourcing, consultancies, agencies, contracting) and **founding or building your own product**.

### Client delivery: the hidden material

Engineers who delivered projects for clients through a services company often under-mine their experience because the work is framed — on CVs and in their own heads — as "delivered project X for client Y". Module 34, Concept 34 described the assumptions product-company interviewers may bring (narrow scope, executing specs, no long-term ownership) and how to counter them. For *mining*, the important point is that **client work is unusually rich in exactly the competencies architects are scored on**:

| Client-delivery situation | Competency it evidences | Where the moment usually hides |
|---|---|---|
| Pre-sales / estimation / discovery | Ambiguity, scoping, customer focus | The workshop where the client's request turned out to be a symptom |
| Shaping or pushing back on requirements | Influence without authority, saying no | The change request you talked the client out of — or into |
| Working with the client's own teams and vendors | Cross-team influence, conflict | The client architect who disagreed with your design |
| Fixed-price or fixed-date constraints | Delivering under pressure, frugality, trade-offs | The scope you cut and how you negotiated it |
| Joining a new domain every 6–18 months | Learning fast, dive deep | The domain model you had to learn in two weeks |
| Handover to the client's team | Mechanisms, durability, mentoring | The runbooks, ADRs and training that let them own it |
| Several clients in parallel | Prioritisation, ownership | The week two clients needed you at once |
| Follow-on contracts | Results, customer trust | The renewal or extension that followed your work |

**Three mining tips specific to client work:**

1. **Mine the boundaries, not just the build.** The richest moments are where your team met the client's people: discovery workshops, design reviews with their architects, steering meetings, acceptance negotiations, handover.
2. **Find out what happened after you left.** Is the system still in use? Did the client extend the engagement? A short message to a former colleague often answers it — and converts "I left before the consequences" into a durability result.
3. **Count your own scope honestly.** On many client projects you were the technical owner for that client — design, delivery, production, support. That is end-to-end ownership, and you should say so.

### Founder or own-product work: the hidden risks and material

Building your own product gives **total scope and ownership** — every decision was yours — and real material for ambiguity, prioritisation, trade-offs under tight resources, and making the business case (Module 33). Interviewers probe two things (Module 34, Concept 34): **real constraints and real users**, and **whether you can operate inside an organization**.

Mining questions for founder work:

- What did you decide *not* to build, and why? (Restraint and prioritisation.)
- What architectural decision did you make early that you'd defend or reverse today? (Judgment, time horizon.)
- What did you learn from users, testers, investors or partners that changed your plan? (Customer focus, being wrong.)
- What would you have done differently with a team of five? (Level — Module 34's "twice the scope" probe.)
- Where did you have to persuade someone you couldn't control — an investor, a partner, a contractor, an early customer? (Influence without authority.)

**Honesty about scale is non-negotiable.** Be exact about users, customers and revenue — including when the number is zero or the product is pre-launch. A well-reasoned architecture with no users yet is a legitimate *design and judgment* story; it is not a *scale* or *operating in production* story, and should not be told as one.

**Balance it.** If your recent years were spent on your own product, pair founder stories with at least two or three stories that involve **other teams, managers or clients**, so the loop sees you working inside an organization. Worked example 7 builds a starter inventory for exactly this kind of career.

**Carry-away sentence:** *"Client-delivery work is unusually rich in architect competencies — discovery under ambiguity, shaping requirements without authority, working with the client's own teams, delivering under fixed constraints, learning domains fast and handing over durable systems — so I mine the boundaries where my team met the client's, and I find out what happened after I left; for own-product work I mine decisions and restraint, I'm exact about users and revenue even when they're small, and I pair it with stories of working inside organizations."*

---
# Part C — The core story types

Each concept in this part follows the same structure: **what is really tested**, the **ingredients** the story must contain, how it changes from **senior to staff/architect**, the **red flags** that sink it, and the **framework labels** it maps to (fully developed in Part D). Read them as *specifications* for stories in your bank — acceptance criteria you test each candidate story against.

## Concept 12 — The flagship project story

**What is really tested.** "Tell me about your most significant / complex / impactful project" is the closest a behavioral round gets to a system-design round about your career. At staff and architect level it is frequently **an entire 45–60-minute round** (a project deep-dive, sometimes with a presentation — Module 34, Concepts 13 and 36). The interviewer wants to see the **highest level at which you have operated**, and whether you can explain a complex system and its human context clearly.

**Ingredients:**

1. **Why it mattered** — business stakes, in numbers (the "so what" chain, Module 34, Concept 28).
2. **The architecture at the right altitude** — a C4 container view you can draw in two minutes (Module 31).
3. **Two or three hard decisions** with options, criteria, costs and reversibility.
4. **The hardest part** — almost always people, data or sequencing, not the code.
5. **How you sequenced risk** — first slice, point of no return, rollback (Module 32).
6. **Results in layers** — technical, customer, business, system-shaping.
7. **What you'd change** — at the level of decision-making, not just execution.

**Senior → staff.**

| | Senior flagship | Staff / architect flagship |
|---|---|---|
| Scope | A system end to end; 1–3 teams | Several systems or a technical area; several teams; multi-quarter |
| Your role | Technical owner and lead | Framed the programme, set direction, aligned teams, owned the key decisions |
| Hardest part | A technical problem and coordinating dependencies | Aligning people with different incentives; sequencing; data ownership |
| Result | Shipped, measured outcome, healthier system | Business impact plus "since then": the platform, standard or capability that persisted |

**Red flags.** A tour of the technology stack; no decisions with alternatives; "we" throughout; a result that stops at "it went live"; unable to draw it.

**Prepare it in layers** (it will be probed for 30+ minutes): a 3-minute overview → the architecture → the hardest decision → the people → the results → what you'd change. Each layer should be openable by the interviewer's next question. Rehearse drawing the C4 container diagram from memory.

**Maps to:** Amazon — Deliver Results, Think Big, Ownership, Dive Deep; Google — role-related knowledge, leadership; Meta — driving results, communicating effectively; enterprise — delivery and stakeholder management.

**Carry-away sentence:** *"My flagship story is my highest-scope project prepared as a forty-five-minute deep dive in layers — why it mattered in numbers, an architecture I can draw from memory, two or three decisions with real alternatives, the hardest part, which is usually people or data, how I sequenced risk, layered results and what I'd decide differently — because at staff level it's often an entire round."*

---

## Concept 13 — Conflict (interpersonal)

**What is really tested.** Whether you can **disagree with people** — peers, managers, stakeholders — **productively and fairly**, resolve the disagreement on substance, and **preserve or improve the relationship**. Meta's "resolving conflict" signal carries particular weight; Amazon covers it under "Have Backbone; Disagree and Commit" and "Earn Trust".

**Conflict is about people and priorities; it is not the same as technical disagreement** (Concept 14). In conflict stories the interesting part is the *relationship*: competing goals, different incentives, a breakdown in communication, a clash of working styles, a priority dispute between a PM and engineering. The resolution is interpersonal as much as substantive.

**Ingredients:**

1. **A real disagreement with a real person** — about priorities, approach, scope, behavior or ownership.
2. **Their position stated at its strongest** — why a reasonable person in their place would hold it.
3. **What you did first** — usually a private conversation to understand, before any escalation.
4. **The resolution mechanism** — data, a compromise, a decision-maker, a written comparison, disagree-and-commit.
5. **The outcome** — for the work *and* for the relationship.
6. **What you'd do differently** — usually earlier, or with a different first move.

**Direction matters — have stories for several:**

| Direction | Typical story | What's being checked |
|---|---|---|
| **Peer** | Another tech lead wanted a different approach for a shared component | Collaboration, fairness |
| **Upward** | Your manager or a director proposed something you thought was wrong | Backbone *and* respect; disagree and commit |
| **Cross-functional** | A PM pushed a date that you believed was unsafe | Business empathy; negotiating trade-offs |
| **Downward** | A team member's behavior or performance needed addressing | Giving hard feedback with care |
| **External** | A client, vendor or partner team with different incentives | Influence without authority; professionalism |

**Senior → staff.** At senior level, a conflict between you and one peer over a design or a priority is appropriate. At staff level, expect conflicts **between teams or with senior leaders**, about **direction** — and stories where you *mediated* a conflict between others, not just resolved your own. A code-style dispute is not a senior story (Module 34, Concept 32).

**Red flags.** A villain ("he was just difficult"); a conflict you "won" with no compromise; avoiding the conflict until it resolved itself; escalation as the first move; no mention of the relationship afterwards; a conflict so mild it isn't one.

**Probes to prepare:** *What did they think? What would they say about you? What did you change your mind about? How is the relationship now? What if they had refused?*

**Maps to:** Meta — resolving conflict; Amazon — Have Backbone; Disagree and Commit, Earn Trust; Google — Googleyness (collaboration, respect); Microsoft — respect, collaboration.

**Carry-away sentence:** *"My conflict stories are about people and priorities, not code: a real disagreement with a named role, their position at its strongest, the private conversation I started with, the mechanism that resolved it, the outcome for both the work and the relationship — and I have them in several directions, peer, upward, cross-functional, downward and external, because questions specify direction and staff loops expect conflicts between teams and with senior leaders."*

---

## Concept 14 — Technical disagreement

**What is really tested.** Whether you can resolve disagreements about **technical substance** — architecture, technology choices, approaches, standards — through **evidence and reasoning**, reach a decision, and **commit** to it, whichever way it went. It is the behavioral twin of Module 30 (writing and defending a design document).

**How it differs from conflict (Concept 13):**

| | Conflict | Technical disagreement |
|---|---|---|
| Centre of gravity | The relationship, incentives, priorities | The options, criteria and evidence |
| Typical resolution | Understanding, compromise, escalation, commitment | Spike, benchmark, prototype, written comparison, decision forum, ADR |
| Key signal | Empathy, fairness, backbone | Judgment, intellectual honesty, decision quality |
| Failure mode | Villains, avoidance | Being right as the point; "I proved them wrong" |

A good technical-disagreement story *can* contain interpersonal tension — but if your story's interesting part is the person, it's a conflict story; tag it so.

**Ingredients:**

1. **The decision and why it mattered** (cost of getting it wrong; reversibility).
2. **Both positions with their real merits** — not a straw man.
3. **How you made the disagreement tractable** — turned opinions into criteria and criteria into evidence: a spike, a benchmark, a prototype, a requirements check, a written comparison.
4. **Who decided and how** — you, a forum, an architect, the team.
5. **The outcome — including when you lost.** A story where you lost the argument, committed fully, and made the chosen approach work is among the strongest you can tell.
6. **What was recorded** — the ADR, with revisit conditions (Module 31).

**Senior → staff.** Senior: a disagreement within your team or with a peer about a design. Staff/architect: a disagreement that affected several teams, or a recurring disagreement you turned into **a decision framework** for the organization (Module 34, Worked example 2 — the event-sourcing guide).

**Three useful shapes:**

- **"I was right, and here's how I made it about evidence"** — fine, but weakest unless you changed something because of the other side's point.
- **"We were both partly right"** — the synthesis took their strongest point. Usually the best shape.
- **"I was wrong / I lost, and I committed"** — the strongest for "Disagree and Commit" and for intellectual honesty.

**Red flags.** Technology tribalism ("they wanted MongoDB, which is obviously wrong"); no evidence, just authority or persistence; "and then they agreed I was right"; no record of the decision; the disagreement was trivial (tabs vs spaces, naming).

**Probes to prepare:** *What was the strongest argument for their option? What would have changed your mind? What did the spike actually measure? Was the decision reversible? Looking back, who was right?*

**Maps to:** Amazon — Are Right, A Lot; Have Backbone; Disagree and Commit; Dive Deep; Google — role-related knowledge, intellectual humility; Meta — resolving conflict, communicating effectively; architect roles — decision quality.

**Carry-away sentence:** *"I keep technical-disagreement stories separate from conflict stories: they're about substance, so I show both options with their real merits, how I turned opinions into criteria and evidence with a spike, benchmark or written comparison, who decided and how, what we recorded — and ideally a case where we were both partly right, or where I lost the argument and committed fully, because that's stronger evidence of judgment than being right."*

---

## Concept 15 — Failure

**What is really tested.** Ownership, self-awareness, understanding of **mechanism**, and **structural** change. Failure questions are among the most frequently asked at every level, and among the most revealing: candidates who blame others, choose trivial failures, or can't name a lesson are quickly marked down. Module 34, Concept 32 gave the structure (blameless post-mortem on yourself); this concept is about **choosing** the failures for your bank.

**Keep at least two failure-capable stories**, of different kinds:

| Kind | Example | What it shows |
|---|---|---|
| **A technical decision you got wrong** | A default you set caused a retry storm; a schema decision forced a migration | Judgment; mechanism-level understanding |
| **An execution or estimation failure** | A project you led slipped by a quarter | Planning, communication, early warning |
| **A people or alignment failure** | You didn't bring a team along and the change was rejected or reverted | Influence; self-awareness about relationships |
| **A failure to act** | You saw a risk and didn't raise it strongly enough | Ownership; backbone |

The people/alignment failure is the most valuable at staff level, because staff work *is* alignment — and the most commonly missing from banks.

**Right-sizing.** The failure must be **big enough to matter** (real cost: users, money, time, trust) and **owned enough to be yours** (your decision or omission). Too small reads as evasive; catastrophic negligence reads as a judgment red flag. The sweet spot: a reasonable decision under the information you had, with a real consequence, that taught you something you now do differently.

**Ingredients** (Module 34, Concept 32): what happened → **my part, stated plainly** → why it made sense at the time → what I did when it went wrong → the mechanism I changed → **evidence the change worked**.

**Red flags.** "My biggest failure is that I care too much"; a failure that was really someone else's; a failure with no consequence; a lesson that's a platitude ("communicate more"); excessive self-flagellation; a failure so recent you're still upset about it.

**Probes to prepare:** *What exactly was your part? What did you tell your manager, and when? What would have caught it earlier? What do you do now? Has the new practice ever caught something?*

**Maps to:** Amazon — Earn Trust (vocally self-critical), Ownership, Learn and Be Curious; Meta — growing continuously; Google — intellectual humility; Microsoft — growth mindset, accountability.

**Carry-away sentence:** *"I keep at least two failure stories of different kinds — ideally one technical decision I got wrong and one people or alignment failure, which is the most valuable at staff level — sized so the cost was real and the decision was mine, and told as a blameless post-mortem on myself ending with the mechanism I changed and evidence that it has since caught something."*

---

## Concept 16 — Leading through ambiguity

**What is really tested.** What you do when **the problem, the goal or the path isn't defined** — whether you wait for clarity or **create** it, and whether you can make sound progress under uncertainty. Module 34, Concept 17 called ambiguity "the axis most strongly correlated with level" because it measures **what you were given versus what you created**.

**Three kinds of ambiguity — your bank should have at least two:**

| Kind | What's unclear | Example |
|---|---|---|
| **Problem ambiguity** | What the real problem is | "Make reporting better" turned out to mean "month-end close takes four days" |
| **Solution ambiguity** | How to solve a known problem; no precedent | First event-driven integration in the company; no reference architecture |
| **Ownership ambiguity** | Who should act | Recurring incidents in a shared database nobody owned |

A useful lens is the **Cynefin** framework's distinction between *complicated* problems (analysable by experts — investigate, then decide) and *complex* ones (cause and effect only visible in retrospect — probe with safe-to-fail experiments, sense, respond). Good ambiguity stories show you **chose the right mode**: analysing when analysis was possible, experimenting when it wasn't.

**Ingredients:**

1. **What exactly was unclear** — specifically; "requirements were vague" is not enough (Module 34, Concept 17's manufactured-ambiguity trap).
2. **How you reduced uncertainty cheaply** — shadowing users, a spike, a prototype, data, a time-boxed trial.
3. **How you created structure** — a problem statement, a decision record, explicit assumptions with review triggers, a phased plan.
4. **Decisions made before full information** — which you deferred, which you made early because they were reversible.
5. **A course correction** — what new information changed and how you responded.
6. **The result** — including whether the reframing was right.

**Senior → staff.** Senior: given a goal ("reduce checkout latency"), supplied decomposition and solution amid uncertainty. Staff: given a **symptom or nothing**, found and framed the problem, and got others to agree it was the right one *before* proposing a solution (Module 34, Concept 17, rung 4).

**Red flags.** "I asked the PM and they told me" (that's a question, not ambiguity); paralysis presented as diligence; recklessness presented as bias for action; no course correction (suggests you weren't really uncertain).

**Probes to prepare:** *What did you know and not know at the start? What did you assume? How did you decide when you had enough information? What would you have done if the spike had failed?*

**Maps to:** Meta — embracing ambiguity; Amazon — Bias for Action, Ownership, Dive Deep, Think Big; Google — Googleyness (comfort with ambiguity), general cognitive ability; Microsoft — growth mindset.

**Carry-away sentence:** *"My ambiguity stories say exactly what was unclear — the problem, the solution or the ownership — then how I reduced uncertainty cheaply, how I created structure with a problem statement or explicit assumptions and review triggers, which decisions I made early because they were reversible, and the course correction when new information arrived; at staff level the strongest version starts from a symptom and ends with agreement that I'd found the right problem."*

---

## Concept 17 — Influence without authority

**What is really tested.** Whether you can **change what other people do when you can't make them** — the defining skill of staff engineers and architects, who rarely manage the people whose work they need to change. Module 34, Concepts 18 and 30 gave the means of influence and the mechanics; this concept specifies the *story*.

**Ingredients:**

1. **A change you needed** from people outside your authority — other teams, a manager's priorities, a client, executives.
2. **Why they had no obligation** — they had their own goals, roadmap, constraints.
3. **How you framed it in their interest** — not "this helps me" but "this helps you".
4. **The named mechanisms** — problem-first document, data, prototype, RFC, pre-wiring, co-authors, sponsor, paved road, pilot (Module 34, Concept 30).
5. **The resistance** and how you handled it — including **what you changed** in your proposal.
6. **When persuasion ran out** — transparent escalation, exception with a sunset date, or disagree-and-commit.
7. **How it stuck** — defaults, ownership transferred, standards.
8. **The result**, honestly partial if it was ("nine of eleven teams").

**Reach — vary it across the bank:**

| Reach | Example |
|---|---|
| **Adjacent team** | Getting another team to change their API for you |
| **Many teams** | A standard, a library, a practice adopted across an org |
| **Upward** | Changing your manager's or director's priorities; getting funding (Module 33) |
| **External** | A client's architects, a vendor, a partner company |

**Senior → staff.** Senior: influence across one or two adjacent teams, using expertise and data. Staff: influence across many teams *and* upward, using a combination of mechanisms, ending with a durable change in how the organization works.

**Red flags.** "I convinced them" with no mechanism; influence that was really authority ("as tech lead I decided"); unanimous instant agreement (implausible or trivial); escalation as the main tool; no durability.

**Probes to prepare:** *Why did they do it — they didn't report to you? What was the real objection? What did you give up? What happened with the team that said no? Is it still in place?*

**Maps to:** Amazon — Earn Trust, Think Big, Have Backbone, Deliver Results; Google — leadership (emergent); Meta — driving results, communicating effectively; enterprise/EA — stakeholder management.

**Carry-away sentence:** *"My influence-without-authority stories show a change I needed from people with no obligation to make it, framed in their interest, the named mechanisms I used — data, a problem-first document, a prototype, pre-wiring, co-authors, a sponsor, making it the default — the real objection and what I changed because of it, what I did when persuasion ran out, how it stuck, and an honestly partial result; and I vary the reach across adjacent teams, many teams, upward and external."*

---

## Concept 18 — Mentoring and growing others

**What is really tested.** Whether you **multiply** other engineers — Module 34, Concept 21's leverage axis. At senior level it's expected; at staff level it's how scope is achieved. Interviewers are not checking whether you're kind; they are checking whether people around you **become more capable because of you**.

**The structure that scores: a person, a before, an after, and what you specifically did.**

1. **A specific person** (anonymised: "a mid-level engineer on my team", "a new hire from a front-end background").
2. **What they couldn't do before** — concretely: lead a design, own on-call, present to leadership, review others' code.
3. **What you did — specifically** — not "I was available for questions". Examples: paired on the first design; asked questions instead of giving answers; delegated a real decision and let it go their way when reversible; gave timely, specific feedback; created a visible opportunity (presenting at the architecture forum); sponsored them — mentioned their work where decisions are made.
4. **The cost you accepted** — "it took three weeks longer than if I'd done it".
5. **What they can do now** — "she owns that area; design reviews no longer wait for me"; promotion if it happened (but don't claim credit for it).
6. **What you learned about mentoring.**

**Mentoring vs sponsorship.** Lara Hogan's distinction is useful in interviews: **mentoring** shares advice and knowledge; **sponsorship** puts the person forward for opportunities and visibility — and doesn't require managerial power. Staff-level growth stories often include sponsorship: you gave someone the high-visibility piece of work, or put their name forward.

**Shapes worth having:**

- **One-to-one growth** — a single person over months.
- **Raising a team** — onboarding guides, review practices, pairing rotations that changed a whole team's capability, with a metric (time to first PR, review turnaround).
- **Growing a leader** — helping someone become a tech lead; at staff/principal level, developing other senior engineers.
- **Mentoring that didn't work** — a powerful failure-adjacent story if you learned something real about how people learn.

**Senior → staff.** Senior: mentored one or two engineers in your team. Staff: grew someone into owning an area; created mechanisms that grow many people; developed other leaders.

**Red flags.** No specific person; no before/after; "I answered questions"; taking credit for someone's promotion; mentoring presented as a chore; only ever mentoring juniors at staff level.

**Probes to prepare:** *What specifically did you do differently for this person? How did you know it was working? What did you let them get wrong? What would they say you did?*

**Maps to:** Amazon — Hire and Develop the Best, Strive to be Earth's Best Employer; Google — leadership; Meta — growing continuously (others' and yours); Microsoft — growth mindset, "model, coach, care"-style manager expectations; staff archetypes — tech lead.

**Carry-away sentence:** *"My growing-others stories are about a specific person with a before and an after: what they couldn't do, what I specifically did — questions instead of answers, a real decision delegated and allowed to go their way, feedback, a visible opportunity I sponsored them into — the cost I accepted in speed, and what they own now; and at staff level I also have a story of a mechanism that grew a whole team, or of developing another leader."*

---

## Concept 19 — Feedback, being wrong and growth

**What is really tested.** Whether you **update** — from feedback, from data, from colleagues — visibly and durably. Meta calls it **growing continuously**; Microsoft's stated value is **growth mindset**; Amazon's **Learn and Be Curious** and **Are Right, A Lot** (which explicitly includes seeking to disconfirm your beliefs) cover it; Google calls part of it **intellectual humility**.

**Three distinct stories — try to have each:**

| Story | Prompt | Shape |
|---|---|---|
| **Critical feedback received** | "The most difficult feedback you've received" | Feedback → initial reaction (honest) → how you checked it → what you changed → evidence it changed |
| **Being wrong** | "A time you changed your mind" | Strong view → evidence or colleague showed otherwise → you changed publicly → outcome better → practice to find out earlier |
| **Deliberate growth** | "How have you grown in the last two years?" / "a skill you developed" | Gap you identified → plan → practice → evidence |

**The feedback story is the one candidates most often fumble**, by choosing feedback that's secretly a compliment ("I was told I work too hard") or feedback they dismissed. Choose feedback that was **genuinely uncomfortable and genuinely right** — about how you communicate, how you run reviews, how you delegate, how you handle disagreement. Module 34's Q10 model answer (coming across as dismissive in design reviews) is the shape.

**Ingredients for all three:** a specific trigger; your honest first reaction (a brief admission of defensiveness is credible); how you verified it (asked for examples, asked others, looked at data); the specific behavior you changed; evidence of the change (the next review cycle, the colleague's behavior, a metric).

**Senior → staff.** Senior: changed your own working practice. Staff: changed your view on a technical direction or an organizational approach — and changed the organization's practice so others find out they're wrong earlier (pre-mortems, revisit dates on ADRs, incident-pattern reviews).

**Red flags.** Humble-brag feedback; feedback you disagreed with and didn't act on (unless the story is explicitly about respectful disagreement); "I was wrong" about something trivial; no evidence of lasting change.

**Maps to:** Meta — growing continuously; Microsoft — growth mindset; Amazon — Learn and Be Curious, Are Right A Lot, Earn Trust; Google — intellectual humility, Googleyness.

**Carry-away sentence:** *"I keep three growth stories — feedback that was genuinely uncomfortable and right, a time I held a strong view and changed my mind publicly, and a skill I deliberately developed — each with my honest first reaction, how I verified it, the specific behavior I changed and evidence it stuck; at staff level the best version also changes how the organization discovers it's wrong earlier."*

---

## Concept 20 — The supporting cast

These competencies are asked often enough that your bank must cover them, but usually **as moments of core stories** rather than dedicated stories. For each, here is what's tested and the minimum the moment must contain.

| Competency | What's tested | Minimum content of the moment | Often found in |
|---|---|---|---|
| **S2 Delivering under pressure** | Trade-offs under constraint; scope negotiation over heroics | The constraint and why it was real; what you cut, deferred or borrowed and why (Module 33's deliberate debt with a repayment trigger); what you protected; the debt repaid afterwards | Flagship; migration; client fixed-date projects |
| **S3 Customer / user focus** | Working backwards from users; going beyond the ticket | A specific user or customer; what you learned by observing or listening; what you changed because of it; the effect on them | Ambiguity stories; client discovery |
| **S4 Ownership beyond scope** | Noticing and acting on what nobody owned | The gap; why it wasn't your job; what you did; making it someone's job afterwards (mechanism) | Glue work (Module 34, Concept 33); incident follow-ups |
| **S6 Prioritisation / saying no** | Cost-aware refusal with an alternative | The request and the legitimate need behind it; the cost in *their* units; the alternative offered; who decided; relationship afterwards | Stakeholder conflict; client scope; founder roadmap |
| **S7 Simplification / innovation** | Reducing complexity; novel approaches with judgment | What was complex and what it cost; the simpler design; what you removed; evidence it was better | Brownfield work; consolidation (Module 34 Q13) |
| **S8 Raising the bar** | Insisting on quality with judgment about where it matters | The standard; the pressure to lower it; how you held it without blocking delivery; the mechanism | Testing, security reviews, design reviews |
| **S9 Learning fast** | Rapid competence in an unfamiliar domain or technology | What you didn't know; how you learned (sources, people, experiments); how quickly you were productive; what you got wrong early | Client domain changes; new platforms |
| **S10 Cost / frugality** | Doing more with less; cost as an engineering constraint | Baseline cost; the change; the saving with method; what you didn't sacrifice (Module 33) | Cloud cost work; licence retirement; build-vs-buy |
| **S11 Dive deep** | Getting to root cause; data over opinion | The symptom; the misleading first hypothesis; the data that pointed elsewhere; three levels of detail you can go to | Performance and incident stories (ThreadPool, EF Core, retry storms) |
| **S12 Motivation and fit** | Coherent career narrative; genuine interest | Not STAR — but your bank's headlines become evidence: "the work I want more of is what I did in X and Y" | All of them |

**Reserves.** When the core can't supply a strong moment for one of these — commonly **S10 cost**, **S3 customer** or **S9 learning fast** — create a short reserve story for it. Reserves can be simpler (one moment, two minutes) but must meet the same truthfulness and probe-proofing standard.

**Two that deserve dedicated preparation for Amazon specifically:** *Frugality* and *Success and Scale Bring Broad Responsibility*. Few engineering stories naturally contain them, and Amazon interviewers may be assigned them. Concept 22 shows how to find them.

**Carry-away sentence:** *"I cover the supporting cast — delivering under pressure, customer focus, ownership beyond scope, saying no, simplifying, raising the bar, learning fast, cost and dive-deep — mostly as moments inside my core stories, each with the minimum content that makes it evidence, and I add short reserve stories for whatever the core can't cover well, typically cost, customer focus or learning something fast."*

---
# Part D — The coverage matrix

## Concept 21 — The matrix as a data structure

The **coverage matrix** is the core artefact of your bank: a table whose **rows are stories** (or, more precisely, moments grouped by story), whose **columns are competencies**, and whose **cells are evidence strengths** (0–3, Concept 2).

```text
                 C1   C2   C3   C4   C5   C6 | S1   S2   S3   S4   S5   S6 ...
               Conf Fail Ambg Tech Ment Infl|Flag Dlvr Cust Ownr Fdbk SayN
S1 Migration     2    1    2    3    2    3 |  3    2    1    1    1    2
S2 Pool outage   0    3    1    1    0    0 |  0    1    1    2    2    0
S3 Lending       1    0    3    1    0    2 |  2    1    3    1    0    1
...
```

**Constraints the matrix must satisfy** — think of them as invariants you check, like assertions in a test:

1. **Coverage.** Every core column (C1–C6) has **at least two cells ≥ 2**, in **different stories**.
2. **Supporting coverage.** Every supporting column has **at least one cell ≥ 2**.
3. **Level.** For each core column, **at least one** of those cells is a story at the *target* level.
4. **Polarity.** C2 (failure) and S5 (feedback/being wrong) cells come from stories with genuinely negative or mixed outcomes.
5. **No single point of failure.** No one story is the *only* strong (≥ 2) option for more than one core column.
6. **Load spread.** No story is tagged ≥ 2 in more than five columns that you plan to use it for in one loop — it will be overused.

**Why rows are stories and not moments.** You could make every moment a row. In practice, grouping by story makes constraint 5 (single point of failure) and loop allocation (Concept 27) easy to see. Put the moment's name in the cell if helpful: `3 (strangler decision)`.

**The algorithmic view.** Choosing which stories make the bank is a small instance of **set multicover**: the universe is the set of competencies, each candidate story "covers" the competencies where it scores ≥ 2, and you want the smallest family of stories that covers every core competency at least twice. Set cover is NP-hard in general, and the classic greedy algorithm — repeatedly pick the story covering the most still-uncovered requirements — gives a logarithmic approximation. With 10–14 candidates and ~20 columns, you don't need an algorithm at all, but the greedy intuition is exactly right:

```text
while some requirement is unmet:
    pick the candidate story that satisfies the most unmet (column, count) requirements,
        breaking ties by level score, then by portfolio balance (Concept 10)
    add it to the bank; decrement the requirements it satisfies
then: add reserves for any supporting column still uncovered
```

**Keep it as a living document** — a spreadsheet or a Markdown table — with a column per company framework you're targeting (Concepts 22–25 add these). Appendix C is a template.

**Carry-away sentence:** *"My bank's core artefact is a coverage matrix — stories as rows, competencies as columns, evidence strength zero to three in each cell — checked against invariants: every core competency has at least two strong cells in different stories, one of them at target level, failure cells come from real failures, and no story is the only strong option for more than one core competency; choosing stories is a small set-multicover problem, and greedy selection works fine."*

---

## Concept 22 — Amazon's Leadership Principles

Amazon is the loop where a story bank matters most: **many** behavioral questions, each interviewer assigned specific principles, deep probing, metrics expected, and a Bar Raiser reading across the loop. The common preparation target from Amazon coaches is **two stories per Leadership Principle** — achievable only through moments, not sixteen times two separate stories.

**Mapping the canonical set to the 16 Leadership Principles:**

| Leadership Principle | Primary families | What interviewers listen for | Commonly missing — how to find it |
|---|---|---|---|
| **Customer Obsession** | S3, S1 | Working backwards from a specific customer; trust | Engineers talk about systems, not customers. Find the moment you changed a design because of what a user or client actually did |
| **Ownership** | S4, C2, S1 | Acting beyond your role; long-term over short-term; "that's not my job" never said | Usually present — make the "nobody owned it" explicit |
| **Invent and Simplify** | S7, C4 | Simplification; new approaches; external ideas adapted | Consolidations, removed complexity, a novel approach to a constraint |
| **Are Right, A Lot** | C4, S5 | Good judgment; seeking diverse perspectives; disconfirming your own beliefs | Pair a good decision with a time you changed your mind |
| **Learn and Be Curious** | S9, S5 | Learning new domains; curiosity beyond the job | Client domain switches; a technology you learned deliberately |
| **Hire and Develop the Best** | C5 | Raising the bar in hiring; developing people | ICs often lack hiring stories — interviewing you did, an interview loop you improved, people you grew |
| **Insist on the Highest Standards** | S8 | Holding quality under pressure; fixing root causes | A standard you held when others wanted to cut corners — and where you didn't over-engineer |
| **Think Big** | S1, C6 | A bold direction; communicating a vision | Staff-level: the multi-year direction you proposed |
| **Bias for Action** | C3, S2 | Calculated risk-taking; reversible decisions made quickly | A reversible decision you made fast without full information |
| **Frugality** | S10 | Accomplishing more with less; constraints breed resourcefulness | Often missing. Cloud cost reductions, licence retirement, building with a tiny team, *not* building something |
| **Earn Trust** | C1, C2, S5 | Listening, candour, being vocally self-critical | A failure you disclosed before it was found; feedback you acted on |
| **Dive Deep** | S11 | Staying connected to details; data over anecdote | Performance and incident stories — and be able to go four levels deep |
| **Have Backbone; Disagree and Commit** | C1, C4 | Respectfully challenging decisions; committing fully once decided | Needs *both halves*: you disagreed, *and* you committed and made it work |
| **Deliver Results** | S1, S2 | Delivering the key inputs on time despite setbacks | Present in most banks; make the setback explicit |
| **Strive to be Earth's Best Employer** | C5, C1 | Creating a safer, more productive, more empathetic environment; growing people | Often missing. Look for: improving on-call load, psychological safety in reviews, removing toil, making a team more inclusive |
| **Success and Scale Bring Broad Responsibility** | S8, S4 | Considering second-order effects on customers, communities, the world | Often missing. Look for: security and privacy decisions, accessibility, reliability for vulnerable users, data protection, energy/cost efficiency, ethical concerns you raised |

**Three Amazon-specific preparation rules:**

1. **Metrics in every story.** Amazon's own pages tell candidates to include data. A story without a number will be probed until one appears or credibility drops.
2. **Expect "Dive Deep" ladders on any story** — not only Dive Deep questions. Every story in your Amazon bank must survive three or four levels of detail on at least one point.
3. **Prepare the forgotten principles explicitly.** Frugality, Hire and Develop the Best, Strive to be Earth's Best Employer, and Success and Scale Bring Broad Responsibility are the most common gaps in engineering banks. Mine for them deliberately (Concept 26); a short, true reserve is far better than stretching an unrelated story.

**Use the vocabulary lightly.** A phrase like "this is where I had to dive deep" helps the interviewer file evidence; "I demonstrated Customer Obsession by…" sounds recited (Module 34, Concept 5).

**Carry-away sentence:** *"For Amazon I aim for two stories per Leadership Principle by mapping moments rather than writing thirty-two stories, with metrics in every one and readiness for four-level dive-deep probes on any of them; I mine deliberately for the principles engineering banks usually lack — Frugality, Hire and Develop the Best, Earth's Best Employer, and Success and Scale Bring Broad Responsibility — and I make sure 'Disagree and Commit' stories contain both the disagreement and the commitment."*

---

## Concept 23 — Google's attributes

Google's structured-interviewing model evaluates four attributes; behavioral evidence appears mostly in two of them, often within a "Googleyness and Leadership" (G&L) round that mixes **behavioral** ("tell me about a time…") and **hypothetical** ("imagine that…") questions.

| Attribute | What it means | Bank coverage |
|---|---|---|
| **General cognitive ability (GCA)** | How you approach open-ended problems; structure, reasoning, use of data | Ambiguity stories (C3) and dive-deep moments (S11) show your reasoning process; hypotheticals test it directly |
| **Role-related knowledge (RRK)** | Technical depth for the role | Technical rounds, but also the depth you show in flagship and technical-disagreement stories |
| **Leadership** | Including **emergent** leadership — stepping up without a title, and **stepping back** when someone else should lead | Influence without authority (C6), ownership beyond scope (S4), mentoring (C5); *and* a story where you deliberately let someone else lead |
| **Googleyness** | Comfort with ambiguity, bias to action, collaboration, intellectual humility, conscientiousness, doing the right thing | Ambiguity (C3), conflict (C1), feedback and being wrong (S5), raising the bar (S8) |

**The "stepping back" story is the distinctive Google requirement.** Emergent leadership is defined in both directions: taking the lead when needed, *and* recognising when another person is better placed and supporting them. Most banks only contain the first. A mentoring story where you delegated the lead (Concept 18) or a project where you deliberately handed decision-making to a domain expert often serves.

**Hypotheticals anchored in the bank.** For a G&L hypothetical — *"Imagine your team strongly disagrees with a product decision. What do you do?"* — the strong answer is a structured approach followed by a real anchor: *"I'd first make sure I understood the reasoning behind the decision… then… — and this is close to something that happened when…"* Your bank supplies the anchor; tag moments that work as anchors for common hypotheticals (disagreement with a decision, an underperforming teammate, conflicting priorities, an ethical concern).

**The hiring committee reads the packet.** Google's decision is made by people who never met you, from written feedback. Crisp, transcribable evidence (Module 34, Concept 3) matters even more; so does consistency across rounds (Concept 30).

**Carry-away sentence:** *"For Google I map my bank to leadership — including emergent leadership in both directions, stepping up without a title and deliberately stepping back for someone better placed — and to Googleyness: ambiguity, collaboration, intellectual humility and doing the right thing; I tag moments that can anchor common hypotheticals, so I answer 'what would you do' with a structured approach and then a real example, in evidence a hiring committee can read."*

---

## Concept 24 — Meta's signal areas

Meta's dedicated behavioral round is scored on a small set of signal areas, commonly described as **resolving conflict, growing continuously, embracing ambiguity, driving results and communicating effectively** — sometimes with "proactivity" or "autonomy" listed alongside ambiguity, and "scope" assessed throughout. The interviewer is usually a senior engineer from outside the team you'd join, and moves briskly through several questions with follow-ups.

| Signal | Bank families | What makes it strong at E5/E6 |
|---|---|---|
| **Resolving conflict** | C1, C4 | Conflict with peers *and* cross-functional partners; structured resolution (written comparison, data, experiment); relationship afterwards; at E6, conflicts between teams |
| **Growing continuously** | S5, C2, C5 | Feedback acted on; being wrong; deliberate skill growth; growing others |
| **Embracing ambiguity** | C3, S4 | Proactivity: finding the problem; autonomy: making progress without direction |
| **Driving results** | S1, S2, C6 | Measurable outcomes; driving others' work; at E6, multi-team impact |
| **Communicating effectively** | C6, S1 | Shown through *how* you answer, and through stories of communicating across audiences |

**Two Meta-specific preparation notes:**

1. **Conflict carries extra weight**, and prep sources note that "conflict with a coworker", "disagreement with your manager" and "giving difficult feedback" often probe the *same* signal from different entry points. Have **three** conflict-capable moments in different directions, so the follow-up "tell me about another one" doesn't send you back to the same story.
2. **Scope is read across everything.** Practitioners who have chaired Meta hiring committees describe going straight to the behavioral feedback when levelling staff candidates. For E6 and above, make sure your conflict and ambiguity stories — not just your flagship — show multi-team scope.

**Carry-away sentence:** *"For Meta I map my bank to five signals — resolving conflict, growing continuously, embracing ambiguity, driving results and communicating effectively — with at least three conflict-capable moments in different directions because conflict is weighted heavily and asked from several angles, and for staff-level loops I make sure even my conflict and ambiguity stories carry multi-team scope, because that's where levelling evidence is read."*

---

## Concept 25 — Microsoft, enterprises, enterprise-architect roles and startups

**Microsoft.** Microsoft's careers site names **respect, integrity, accountability and growth mindset** as what it looks for, and behavioral questions are usually woven into each round rather than isolated. Map:

| Value | Bank families |
|---|---|
| **Growth mindset** | S5 feedback and being wrong, C2 failure, S9 learning fast, C5 growing others |
| **Respect** | C1 conflict (no villains), C5 mentoring, collaboration across groups |
| **Integrity** | S8 raising the bar, saying no for the right reasons, a time you raised an uncomfortable truth |
| **Accountability** | C2 failure (ownership), S4 ownership beyond scope, S1 delivery |

Expect customer-focus and cross-group collaboration questions too; Microsoft's scale means "working with another org" stories land well.

**Large enterprises (banks, insurers, manufacturers, public sector).** Often use explicit **competency frameworks** — sometimes with level descriptors per competency. The UK Civil Service's public *Success Profiles* framework is a good example of the genre: behaviours such as *seeing the big picture, changing and improving, making effective decisions, leadership, communicating and influencing, working together, developing self and others, managing a quality service, delivering at pace*, each with level-specific examples, and only a subset assessed for any given role. If a job advert lists competencies, **add a matrix column per competency** and check coverage exactly as for Amazon.

**Enterprise-architect roles.** Add columns for: **stakeholder management** (business leaders, risk, procurement), **governance that works** (standards people follow rather than route around), **portfolio and roadmap** (rationalisation, sequencing, funding), **business-capability thinking**, and **vendor management** (build-vs-buy, Module 33). Expect panels with business stakeholders: stories must work for a non-technical listener.

**Startups and scale-ups.** Fewer formal rubrics; the conversation with a founder or CTO looks for **end-to-end ownership, speed with judgment, building from zero, comfort with unglamorous work, and knowing when not to add process**. Founder or early-stage stories (Concept 11) fit naturally; so do client-delivery stories about fixed constraints.

**Consultancies hiring architects.** Client influence, ambiguous scoping, delivering under commercial constraints, handling a difficult client — client-delivery careers are a natural fit.

**Carry-away sentence:** *"For Microsoft I map my bank to respect, integrity, accountability and growth mindset, with cross-group collaboration stories; for enterprises with explicit competency frameworks I add a matrix column per listed competency, the way the UK Civil Service's Success Profiles lists behaviours with level descriptors; for enterprise-architect roles I add stakeholder management, governance that works, portfolio and vendor stories told for non-technical panels; and for startups I lead with end-to-end ownership and speed with judgment."*

---

## Concept 26 — Gap analysis: filling thin columns honestly

Once the matrix is filled, some columns will be thin — fewer than two strong cells for a core competency, or none for a supporting one. There are four ways to fill a gap, **in this order of preference**:

| Strategy | What it means | When it works |
|---|---|---|
| **1. Re-mine** | Go back to the raw inventory and artefacts for an episode you dismissed or forgot | Most gaps. The competency happened; you didn't remember it as a story |
| **2. Re-angle** | Find an unused moment inside an existing story | When a story you already have contains the behavior, but you've always told it from another angle |
| **3. Reserve** | Write a short, single-purpose story | Supporting competencies; Amazon's often-missing principles |
| **4. Accept and plan** | Acknowledge the gap; prepare the honest near-miss (Concept 32) and a hypothetical answer | When you genuinely haven't done it at the target level |

**Never fill a gap by invention, composite or borrowed credit** (Concept 31). A gap filled dishonestly is worse than a gap, because the probe ladder is designed to find it.

**Common gaps and where they usually hide:**

| Gap | Where to look |
|---|---|
| **People / alignment failure** | A change that was resisted or reverted; a decision made without the right people in the room; a handover that went badly |
| **Upward conflict** | A deadline you challenged; a technology mandate you questioned; a priority you got changed |
| **Mentoring at staff level** | Someone who took over an area from you; an onboarding programme; a review practice; a person you sponsored into visibility |
| **Customer focus** | Discovery workshops; support escalations you got involved in; time spent with real users |
| **Frugality** | Cost reductions; licences retired; deliberately not building; small-team constraints |
| **Earth's Best Employer** | On-call improvements; toil removal; making reviews safer; mentoring under-represented colleagues; improving a hiring process |
| **Broad responsibility** | Security/privacy decisions; accessibility; reliability for critical users; ethical concerns raised |
| **Stepping back (Google)** | Letting a domain expert or a mentee lead; deferring to someone with better context |

**Gaps are information.** If you genuinely lack staff-level influence stories, the matrix is telling you something about level — and Module 34, Concept 26's advice applies: target the level you can evidence, or close the gap in your current work before interviewing (Concept 37).

**Carry-away sentence:** *"When a column in my matrix is thin I fill it in order of preference — re-mine my inventory and artefacts, re-angle an existing story to an unused moment, write a short true reserve, or accept the gap and prepare an honest near-miss — never by invention or borrowed credit, and if I genuinely lack evidence at the target level I treat that as information about which level to target."*

---

## Concept 27 — Loop allocation: primaries, backups, no repeats

Before a specific loop, turn the bank into a **plan**: which story is your first choice for each competency, which is the backup, and how you'll avoid repeating yourself.

**What you usually know in advance:**

- **The company framework** (always).
- **The number and type of rounds** (ask the recruiter — Module 34, Concept 37).
- **Sometimes, the focus of each round** — e.g., "the hiring manager round will focus on leadership", "one round is a project deep-dive". Amazon recruiters sometimes indicate which principles to emphasise; don't count on per-interviewer assignments.

**The allocation rules:**

1. **Per interviewer: never repeat a story.** Not even a different moment of the same story — it reads as narrow.
2. **Per loop: limit each hub story to two or three uses**, each a different moment, ideally with interviewers whose focus differs.
3. **Reserve the flagship** for the deep-dive or the most senior interviewer, if you know which that is.
4. **Primaries and backups per competency**: for each core family, name a primary and a backup *from different stories*.
5. **Track as you go.** Between rounds, jot down which stories and moments you used with whom (on your own paper, between interviews — not during, where notes may be disallowed).

**The algorithmic view.** Loop allocation is an **assignment problem**: interviewers' likely competencies are one side of a bipartite graph, story-moments are the other, edges are weighted by evidence strength, and the constraint is that each interviewer gets distinct stories and each story is used a bounded number of times. In real time you won't run the Hungarian algorithm; the practical approximation is: **use your primary unless you've already used that story with this interviewer or it's hit its loop limit, then fall to the backup.** That works because you've pre-computed primaries and backups from different stories.

**A one-page allocation plan:**

```text
Competency         Primary (story · moment)            Backup (story · moment)
Conflict           S7 Launch vs threat model · PM      S1 Migration · reporting team
Failure            S2 Pool outage · my config          S4 ADR rollout · first attempt rejected
Ambiguity          S3 Lending · "faster decisions"     S1 Migration · cutover plan wrong
Tech disagreement  S5 Cosmos vs SQL · spike            S1 Migration · strangler vs rewrite
Mentoring          S6 Integration owner                S4 ADR rollout · review rotation
Influence          S4 ADR rollout · 5 teams            S1 Migration · business go-live criteria
Flagship           S1 Migration (deep-dive round)      S3 Lending
Feedback / wrong   S8 One-page docs                    S5 Cosmos vs SQL · I was half wrong
Frugality          R1 Azure cost −38%                  S1 Migration · licence retired
Customer           R2 Repair technicians               S3 Lending · underwriters
```

Worked example 6 applies this to an Amazon loop and a Meta loop.

**When the plan collides with reality.** An interviewer asks the conflict question you'd planned for the next round. Use the primary now, promote the backup to primary for later, and find a third option from the matrix if needed. That's why constraint 1 of the matrix (two strong options in *different* stories) exists.

**Carry-away sentence:** *"Before a loop I turn my bank into a one-page allocation plan — for each competency a primary and a backup from different stories, the flagship held for the deep-dive or most senior interviewer, no story repeated with the same interviewer and no hub used more than two or three times across the loop — and between rounds I note which stories I've used with whom, which is a practical greedy approximation to what is really an assignment problem."*

---
# Part E — Making every story level-appropriate and probe-proof

## Concept 28 — The level audit

Every story in the bank must pass a **level audit** against your target level before you rehearse it. The audit applies Module 34's six axes (Concepts 15–21) one at a time — and its output is either "passes", "re-tell it", or "demote it".

**The audit questions** (answer in writing on the story card):

| Axis | Audit question | Staff/architect pass condition |
|---|---|---|
| **Scope** | What was I responsible for — in people, teams, systems, money, time? | Several teams or a technical area; organizational boundaries crossed |
| **Ambiguity** | What was I given, and what did I create? | Given a symptom or a direction; I found or framed the problem |
| **Influence** | Whose behavior did I change, and by what means? | People outside my authority, by named mechanisms, including upward |
| **Impact** | Output → outcome → business → system-shaping: how far up does it go? | Business impact *and* a "since then" |
| **Horizon** | Over what period did my decisions play out? | A year or more; sequencing; second-order effects |
| **Leverage** | What could others do afterwards that they couldn't before? | Platforms, standards, practices, people grown |

**Three possible outcomes:**

1. **Passes at target level** — rehearse it as is.
2. **At level but told below it** — the staff-level actions happened but aren't in the telling (the "just meetings" problem, Module 34, Concept 26). **Re-tell**: move the framing, alignment and mechanism work into the Action and Result; re-headline it. This is the most common and most valuable fix.
3. **Genuinely below target level** — keep it only if it's the best story for a supporting competency, and tell it honestly at its level. Don't inflate it.

**The re-telling test.** If making a story staff-level requires adding actions you didn't take, it is not a staff story. Module 34, Exercise 4 put it exactly: keep the senior version.

**Audit the bank, not just stories.** After auditing each story, check the matrix: **are the strongest cells in the core columns at the target level?** It's acceptable for a mentoring story to be senior-level if your influence and flagship stories are clearly staff — but if *every* core column's best story is senior-level, that's the down-levelling signal Module 34 warned about.

**Carry-away sentence:** *"Before rehearsing, I audit every story against my target level on scope, ambiguity, influence, impact, horizon and leverage, with one of three outcomes — it passes, it's at level but told below it so I re-tell it with the framing and alignment work moved into the actions, or it's genuinely below level so I keep it only for a supporting competency and tell it honestly — and if raising a story's level would require actions I didn't take, I keep the lower-level version."*

---

## Concept 29 — The story pre-mortem: attack your own story

Module 30 introduced the **pre-mortem** for designs: imagine the project has failed and ask why. Apply the same technique to each story: **imagine a sceptical Bar Raiser has just concluded your story was inflated, borrowed or weak — what made them think so?**

**The pre-mortem questions:**

| Attack | Question | Defence to prepare |
|---|---|---|
| **Attribution** | "Was this really you, or your team, your manager, the senior engineer next to you?" | One sentence separating your part from others'; who else could have done it |
| **Counterfactual** | "Would this have happened anyway without you?" | What specifically would have gone differently; evidence |
| **Numbers** | "Where does that number come from? What was the baseline? What else changed?" | The fact sheet: method, timeframe, attribution caveats |
| **Alternatives** | "Why didn't you just do the obvious thing?" | The options you considered and why you rejected them |
| **The other side** | "What would the person who disagreed with you say about this?" | Their best argument, stated fairly; what you conceded |
| **Durability** | "Is it still there? What happened after you left?" | Who owns it now; evidence it persisted — or honesty that you don't know |
| **Depth** | "Take one technical detail and go four levels down." | The pre-chosen deep-dive detail, rehearsed |
| **Level** | "This sounds like a senior story. What made it staff-level?" | The framing, alignment and system-shaping actions |
| **Consistency** | "The previous interviewer heard a different number." | One canonical fact sheet (Concept 30) |
| **Weak spot** | "What part of this story are you least proud of?" | The weak spot you volunteer yourself (Module 34, Concept 14) |

**Run it with someone else.** You'll be too kind to your own stories. Give a peer — or an AI assistant set up as a demanding interviewer, which is permitted for *preparation* — the story and Appendix D of Module 34 (the probe bank), and ask them to find the weakest point. Then fix the story (more facts, a narrower claim, a volunteered caveat) or demote it.

**Fix by narrowing, not by adding.** The usual repair for an attackable claim is to **claim less, more precisely**: not "I led the migration" but "I designed the migration strategy and led the data-ownership work; another lead ran the front-end track." Narrower claims survive probing; broader ones collapse.

**Carry-away sentence:** *"For each story I run a pre-mortem as if a sceptical Bar Raiser had concluded it was inflated — attacking attribution, the counterfactual, the numbers, the alternatives, the other side's view, durability, depth, level and consistency — ideally with a peer or an AI mock interviewer, and I repair weak points by claiming less, more precisely, rather than by adding detail."*

---

## Concept 30 — The fact sheet and consistency across the loop

Interviewers write independent feedback; debriefs and hiring committees read it together. **Inconsistencies across a loop are visible** — and an inconsistency in numbers, dates, team sizes or your role reads as embellishment even when it's just memory drift.

**Each story gets one canonical fact sheet**, built from the evidence in Concept 7:

```text
FACT SHEET — S1 Ordering platform migration
Period:            Mar 2023 – Aug 2024 (17 months)
My role:           Lead architect (title: senior engineer, acting as architect)
Teams:             2 teams of 6; reporting team (4) and partner-export owner as dependents
System:            ASP.NET MVC 5 monolith on-prem → .NET 8 (later .NET 10) on Container Apps; YARP facade
Key numbers:       lead time for ordering changes ~6 weeks → ~4 days (median, from tracker)
                   data-centre exit: 17 months vs 18-month lease deadline
                   reporting licence retired: ~€40k/yr (approximate — finance figure not seen directly)
Decisions:         strangler over rewrite; CDC + daily reconciliation for reporting; named point of no return
Who disagreed:     reporting team lead (schema stability for regulatory reports — legitimate)
                   my manager initially preferred a big-bang cutover (date certainty)
Weak spot:         stalled at ~70% for two quarters on the reporting dependency
Since then:        (as of my last contact) still running; second team extracted pricing the same way
Deep-dive detail:  CDC → reconciliation: how mismatches were detected and resolved
Anonymisation:     "a European e-commerce retailer, ~300 employees"
```

**Rules:**

1. **One number per fact, everywhere.** Pick the honest figure (or range) once; use it in every telling. If you say "about four days" in round 2, don't say "under a week" in round 4 and "three days" in round 5.
2. **Write approximations as approximations** on the sheet — "~€40k/yr (approximate)" — so you remember to say so.
3. **Align with your CV and LinkedIn.** Interviewers read your CV before the round. If it says "led a team of 12" and the story says "I was one of four engineers", you have a credibility problem before you start. Fix whichever is wrong.
4. **Align with references.** At senior and staff level, reference checks and informal backchannels happen. Your story should be recognisable to the people who were there.
5. **Anonymisation is part of the fact sheet.** Decide once how you'll describe the organization and the systems, so you don't name a client in one round and anonymise in another.

**Carry-away sentence:** *"Every story has one canonical fact sheet — period, role, teams, system, key numbers with how they were measured, the decisions, who disagreed, the weak spot, what happened since, my deep-dive detail and how I anonymise it — so that I use the same honest figure in every round, my CV and references tell the same story, and nothing in the debrief looks like embellishment."*

---

## Concept 31 — Truthfulness: why composites, borrowed credit and inflated numbers fail

This concept is short because the rule is simple: **every story in your bank is true, yours, and told at the scale it happened.** But it's worth understanding *why*, beyond ethics, because the temptation is real when the matrix shows gaps.

**Three forms of untruth, and why each fails mechanically:**

| Form | What it is | Why it fails in the room |
|---|---|---|
| **Composite story** | Merging two or three real episodes into one better-shaped story | Probes on timeline, cast and numbers expose seams: "Wait — was this before or after the reorganisation?" You can't keep a composite consistent across follow-ups, let alone across a loop |
| **Borrowed credit** | Telling a colleague's decision or work as yours | Attribution probes ("why did *you* choose that?", "what did you consider?") require reasoning you didn't do. Depth collapses at the second level |
| **Inflated numbers or scope** | Rounding up; claiming the team's outcome as your scope | The baseline/method probe ("how did you measure that?") and the scope probe ("how many teams?") are routine; inflated answers wobble, and a wobble costs credibility for the whole loop |

**What *is* legitimate:**

- **Selection** — choosing which true story and which moment to tell.
- **Compression** — leaving out irrelevant detail, simplifying the cast ("the reporting team" rather than six names).
- **Reordering for the listener** — headline first, STAR order rather than chronology (Module 34, Concept 6).
- **Anonymisation** — describing organizations by type and scale.
- **Honest approximation** — ranges and proxies, labelled as such.

**Fabrication is also a hiring risk after you're hired.** Exaggerated claims set expectations you then have to meet; reference checks and later conversations can surface them. And the 2026 environment adds a new failure mode: **AI-generated stories**. Using AI to *structure and critique* your true stories is permitted preparation at most companies; using it to *invent* experience is not, and invented stories fail probes for the same reasons composites do.

**If you have a genuine gap**, the honest near-miss (Concept 32) and a well-reasoned hypothetical answer score better than any invented story — interviewers are trained to prefer an honest "I haven't had exactly that; the closest was…"

**Carry-away sentence:** *"Every story in my bank is true, mine, and told at the scale it happened: I select, compress, reorder, anonymise and approximate honestly, but I never merge episodes, borrow a colleague's decisions or inflate numbers — not only because it's wrong, but because composites break on timeline probes, borrowed credit collapses at the second 'why', and inflated numbers wobble under 'how did you measure that?', costing credibility for the whole loop."*

---

## Concept 32 — Adapting on the fly: the 70% question and the honest near-miss

No bank anticipates every question. Two skills cover the rest.

### The 70% question

Most questions will be **about 70% like** a moment you've prepared: the competency matches but the specifics differ — "a disagreement with a *product manager*" when your best conflict moment is with a tech lead; "a time you *missed* a deadline" when your story is about meeting one under pressure.

**Technique: bridge, then tell the moment, then close on the question.**

1. **Bridge** in one sentence, acknowledging the difference: *"The closest example I have is with a tech lead rather than a PM, but the disagreement was about priorities, which I think is what you're after."*
2. **Tell the moment** with emphasis shifted towards the asked-for aspect — in this case, the priority dispute rather than the technical details.
3. **Close on the question** in the result and reflection: *"…and what I took from it for working with product is…"*

**Re-weighting, not re-writing.** Because you've prepared facts and structure rather than scripts (Module 34, Concept 12), you can move emphasis — spend 60% of the action on the part the question cares about — without changing any fact.

### The honest near-miss

When no story fits even at 70%:

1. **Say so briefly** — *"I haven't been in exactly that situation."*
2. **Offer the closest real experience** and why it's relevant — *"The closest was…, which had the same tension between…"*
3. **Or, if nothing is close, answer as a hypothetical anchored in principles you've applied elsewhere** — *"Here's how I'd approach it, based on how I've handled…"*
4. **Let the interviewer choose** — *"Would that be useful, or would you prefer a different angle?"*

Interviewers are trained to prefer this to a fabricated perfect match, and it often scores *better* than a forced fit, because it shows self-awareness.

### When the interviewer redirects

Sometimes, two sentences into your headline, the interviewer says "actually, can you give me a different example?" — often because they've heard this story's context from a colleague, or because it's not the competency they need. **Don't defend the choice.** Go to your backup (Concept 27). Having a backup from a *different* story is what makes this painless.

**Carry-away sentence:** *"When a question is only seventy percent like a moment I've prepared, I bridge in one sentence, tell the moment with the emphasis shifted to what was asked, and close on the question itself — re-weighting facts rather than rewriting them; when nothing fits I say so, offer the closest real experience or a principled hypothetical, and let the interviewer choose, and if they redirect I go straight to my backup from a different story."*

---
# Part F — The practice plan

## Concept 33 — How memory for stories works: retrieval, spacing, interleaving

Most candidates practise their stories by **rereading** their notes. Learning research is unusually clear that this is close to the least effective method available. Understanding why shapes the whole practice plan.

**Retrieval practice (the testing effect).** Pulling information *out* of memory strengthens it far more than putting it in again. In Roediger and Karpicke's 2006 experiments, students who practised recalling a passage retained about 61% after a week; those who reread it retained about 40% — even though the rereaders felt more confident. Dunlosky and colleagues' 2013 review of ten common learning techniques rated **practice testing** as one of only two "high utility" techniques; rereading and highlighting rated low.

*For stories:* being asked a question and having to **find and tell** the story without looking is the practice. Reading the story card is not.

**Spacing (distributed practice).** The other high-utility technique in Dunlosky's review. Practice spread over days and weeks produces much more durable memory than the same amount of practice massed into one session. Retrieval tells you *what* practice should consist of; spacing tells you *when*.

*For stories:* twenty minutes a day for three weeks beats a six-hour weekend before the loop.

**Interleaving.** Mixing different types of problems in one session, rather than practising one type in blocks, feels harder and produces worse performance *during* practice — but better long-term learning, because each attempt requires choosing *which* approach applies.

*For stories:* practising "five conflict questions in a row" trains telling conflict stories. Practising a **random mix** of questions trains the real skill — **decoding the question and selecting the right story** (Concept 5). Shuffle the deck.

**Desirable difficulties.** Robert and Elizabeth Bjork's term for conditions that slow apparent progress but improve durable learning: retrieval, spacing, interleaving, variation. The practical implication: **if practice feels easy, it's probably not working.** Telling the same story in the same order to the same mirror becomes fluent quickly — and brittle under interruption.

**Variation (for stories specifically).** Tell each story in different lengths (two minutes, five minutes), starting from different points (from the result, from the conflict), and in answer to different questions. This builds the flexible, fact-based knowledge Module 34 (Concept 12) called "structure and facts, not sentences".

**Spaced-repetition software** (Anki is free) works well for the *decoding* skill: one card per question, front = the question, back = your primary and backup story-moments. Reviewing the deck trains instant lookup; telling the story aloud on a subset of cards trains delivery.

**Carry-away sentence:** *"I practise stories the way learning research says works: retrieval rather than rereading — being asked a random question and finding and telling the story without notes — spaced over weeks rather than crammed, interleaved across competencies so I practise the decoding and selection step, and varied in length and starting point; if practice feels easy, it's probably building fluency that won't survive interruption."*

---

## Concept 34 — The four-week plan

A plan for a candidate starting from nothing, with about **30–45 minutes on weekdays and 1–2 hours at weekends**. Compress to two weeks if you must; expand if you have time. Weeks 1–2 produce the bank; weeks 3–4 make it reliable under pressure.

### Week 1 — Excavate and screen

| Day | Activity | Output |
|---|---|---|
| 1–2 | Timeline sweep and prompt-driven recall for each role (Concept 6) | 25–40 one-line episodes |
| 3 | Artefact sweep: git, docs, post-mortems, calendar, reviews (Concept 7) | Episodes added; facts noted |
| 4 | Screen and score (Concept 8) | Ranked shortlist of 10–14 |
| 5 | Identify hubs and spokes; list candidate moments per story (Concept 9) | Moments list |
| Weekend | Build the coverage matrix; check invariants; gap analysis (Concepts 21, 26) | Matrix v1; gap list |

### Week 2 — Build the cards

| Day | Activity | Output |
|---|---|---|
| 1–3 | Story cards for the 6–8 core stories (Appendix B): headline, STAR-L, moments index | Draft cards |
| 4 | Fact sheets; recover missing numbers; message former colleagues (Concept 30) | Fact sheets |
| 5 | Level audit on each card (Concept 28) | Re-told or demoted stories |
| Weekend | Fill gaps: re-mine, re-angle, reserves (Concept 26); company-framework columns (Concepts 22–25); pre-mortem the two hubs (Concept 29) | Matrix v2; 2–4 reserves |

### Week 3 — Speak, record, vary

| Day | Activity | Output |
|---|---|---|
| Daily | **Spaced retrieval drill (15 min):** shuffle the question deck; for 5 random questions, decode, pick the story-moment, and tell it aloud in 2–3 minutes without notes | Decoding speed; fluency |
| Mon/Wed/Fri | **Record one story** at two lengths; score with Module 34's Appendix B rubric; fix one thing | Recordings; fixes |
| Tue/Thu | **Follow-up drill:** for one story, answer the five standard probes and three level probes aloud; go four levels deep on the chosen detail | Probe-proofing |
| Weekend | First **mock interview** (peer or AI interviewer) — 45 minutes, 4–5 questions, relentless follow-ups (Concept 35) | Mock notes; weak spots |

### Week 4 — Mocks and calibration

| Day | Activity | Output |
|---|---|---|
| Daily | Spaced retrieval drill continues — now mixing in hypotheticals and two-competency questions | Robust lookup |
| 2–3 sessions | Mocks calibrated to the target loop: Amazon (LP-heavy, deep probes), Meta (brisk, conflict-heavy), staff deep-dive (flagship, 45 min) | Calibrated delivery |
| 1 session | Flagship presentation rehearsal if the loop has one; draw the C4 container diagram from memory | Deep-dive readiness |
| End | Loop allocation plan (Concept 27); one-page index card (Concept 36) | Day-of materials |

**Maintenance after week 4:** keep the daily drill at 10 minutes until the loop; after each real interview, run the after-action review (Concept 36).

**If you only have one week:** day 1 inventory → day 2 screen and matrix → day 3 cards for 6 stories → day 4 fact sheets → days 5–7 daily retrieval drills plus two mocks. You'll have less depth, but the index will exist.

**Carry-away sentence:** *"My plan runs four weeks: week one excavates and screens episodes and builds the coverage matrix, week two writes story cards and fact sheets, audits level and fills gaps, week three is daily spaced retrieval drills on shuffled questions plus recorded tellings and follow-up drills, and week four is mocks calibrated to the target loop, ending with an allocation plan and a one-page index — and if I only have a week, I compress it but keep the index and the retrieval drills."*

---

## Concept 35 — Mocks: peers, AI interviewers and recordings

**Mocks are experiments, not performances.** Each should have a hypothesis ("my conflict stories are too long", "my flagship doesn't survive the data-ownership probe") and produce a measurement.

**Three kinds of mock, each with a different strength:**

| Kind | Strength | Weakness | How to get the most from it |
|---|---|---|---|
| **Peer mock** (colleague, friend, peer platform) | A real human reacting; realistic social pressure; you also practise *being* the interviewer | Quality varies; peers are often too polite | Give them Module 34's probe bank (Appendix D) and the rubric; ask them to interrupt and to probe at least three levels; swap roles — interviewing someone else teaches you what an interviewer needs |
| **AI mock interviewer** | Available any time; tireless follow-ups; can be configured as a sceptical Bar Raiser or a brisk Meta interviewer | No real social pressure; may be too generous unless instructed otherwise | Ask it to ask only follow-ups after your first answer; to flag "we" without "I", results without baselines and claims it finds implausible; and **not** to suggest content — the material stays yours |
| **Recording yourself** | Objective; shows filler, pacing, time to headline | No probes | Measure: seconds to headline, share of time on actions, numbers stated, "we" without "I" (Module 34, Appendix B) |

**Free and low-cost options in 2026:** peer mocks on **Exponent Practice** (Pramp's platform since July 2024; reports in 2026 cite about five free peer credits a month); colleagues and former colleagues; local meetups and online communities; AI assistants for drills and probing; your phone for recordings. Paid platforms with experienced interviewers (interviewing.io and similar) are worth considering for one or two final calibrated mocks for a specific company.

**Running a mock well:**

1. **Brief the interviewer**: target company, level, and which competencies to probe — or ask them to choose at random from Appendix D so you also practise decoding.
2. **Simulate format**: 45 minutes; camera on; no notes on screen (Module 34, Concept 36).
3. **Probe relentlessly**: at least half the time on follow-ups.
4. **Score immediately**: the interviewer scores you on the rubric; you score yourself; compare.
5. **Log one fix per story** and re-test it next session.

**What to look for in mock feedback:** answers that took more than 30 seconds to reach the action; stories that collapsed at the second probe; a competency where you hesitated choosing a story (a matrix gap); numbers you stated differently in two tellings (a fact-sheet problem); and level — *"would this answer get you the level you want?"*

**Carry-away sentence:** *"I treat mocks as experiments with a hypothesis and a measurement: peers for real social pressure and for practising the interviewer's side, an AI mock interviewer instructed to ask only follow-ups and flag implausible claims without supplying content, and recordings for objective timing — each run in the real format with at least half the time on probes, scored on the rubric immediately, with one logged fix per story."*

---

## Concept 36 — Day-of protocol and the after-action review

**The one-page index.** The night before, condense the bank onto one page — for your own review *before* the interview, not for use during it (many companies disallow notes, and reading from a screen is visible — Module 34, Concept 36):

```text
STORY                         HEADLINE (one line)                                  NUMBERS            MOMENTS
S1 Ordering migration         Strangled a monolith; DC exit in 17 mo; lead time 6w→4d   6w→4d · 17mo      strangler · reporting team · cutover wrong · mentee · go-live
S2 Connection-pool outage     My pooling change took checkout down 35 min at peak       35 min · 3×       my config · post-mortem · load-test gate
S3 Lending decisions          "Faster decisions" was really the manual queue            3d → 4h           shadowing · reframing · underwriters
...
PRIMARIES  Conflict S7 | Failure S2 | Ambiguity S3 | Tech S5 | Mentor S6 | Influence S4 | Flagship S1
```

**Before the loop:**

- **Warm up aloud** for 10 minutes: two random questions from the deck, told fully. It gets the stories into working memory and your voice into speaking mode.
- **Re-read the company framework** — the literal wording of the principles.
- **Prepare your tracking sheet**: a blank grid of interviewers × stories.

**Between rounds:**

- **Log what you used**: interviewer, questions, story and moment, anything you said that wasn't on the fact sheet.
- **Don't replay the previous round.** Note one thing to adjust; move on.
- **Adjust the allocation**: if your conflict primary is spent, promote the backup.

**After the loop — the after-action review (AAR).** Borrowed from military and incident practice, and essentially a blameless post-mortem on the interview:

1. **What questions were asked?** Write them down while you remember — they're the best test data for your deck.
2. **Which stories did I use, and how did they land?** Where did the interviewer probe hardest? Where did I hesitate?
3. **Where was I unprepared?** A competency with no ready story; a probe I couldn't answer.
4. **What will I change in the bank?** One concrete update per weakness.

Whatever the outcome, the AAR makes the next loop better — and if an offer arrives at a lower level than you targeted, it often shows exactly which competency lacked level-appropriate evidence.

**Carry-away sentence:** *"The night before I condense my bank to a one-page index of headlines, numbers, moments and primaries for my own review; on the day I warm up aloud with two random questions, log which stories I used with each interviewer between rounds and adjust my allocation, and afterwards I run an after-action review — the questions asked, how each story landed, where I was unprepared, and one change per weakness."*

---

## Concept 37 — Maintaining the bank

A story bank isn't a one-off interview artefact. Maintained, it becomes a **career instrument** — feeding promotion cases, performance reviews, CV updates and, eventually, the next interview search — at a fraction of the cost of rebuilding it.

**The brag document as the bank's write-ahead log.** Julia Evans' widely cited practice — a running document of what you did and why it mattered — is the input stream. Keep it lightweight:

```text
2026-10 · Pricing module refactor approved (business case: capacity, ~$300k/yr); checkpoint at week 6
        · Disagreed with the platform lead on retry defaults — spiked both; adopted a hybrid
        · Mentee presented the integration design at the architecture forum
```

Fifteen minutes every two weeks is enough. When you next build or refresh the bank, the brag document replaces most of the excavation and evidence-gathering of Concepts 6–7.

**A quarterly refresh** (30 minutes):

- **Add** one or two new episodes from the brag document as candidates.
- **Re-score** the matrix: has a new story overtaken an old one for a competency?
- **Retire** stories that have aged out (older than five years without a clear reason to keep them) or that no longer represent your level.
- **Update "since then"** for existing stories — durability evidence keeps accumulating after projects end.

**Use gaps as a career plan.** If the matrix has had a thin column for a year — no staff-level influence story, no upward conflict, no growing another leader — that is a statement about the work you've been doing, not just about your preparation. The most reliable way to fill a gap is to **do the work**: volunteer for the cross-team initiative, take the mentee, write the strategy document. A year later, it's a story.

**Keep it private and anonymised.** The bank contains candid assessments of colleagues and organizations. Store it privately; write it in anonymised form from the start, so it's interview-ready and never embarrassing if seen.

**Carry-away sentence:** *"I maintain my bank as a career instrument: a lightweight brag document every couple of weeks feeds it, a thirty-minute quarterly refresh adds new candidates, re-scores the matrix, retires stale stories and updates what's happened since, and any column that stays thin for a year becomes a deliberate goal for my actual work — because the reliable way to fill a gap is to do the thing."*

---
# Worked examples

All people, companies and figures below are **illustrative composites** — they show the *method*. Your bank must contain only your own true stories.

The worked examples follow one composite candidate: a senior .NET engineer with eleven years' experience across a services company (client projects) and two product companies, targeting **staff engineer / software architect** roles. The bank they end up with:

| ID | Story | Type | Period |
|---|---|---|---|
| **S1** | Ordering platform migration — strangler fig from an ASP.NET MVC 5 monolith to .NET on Container Apps | Hub · flagship | 2023–24 |
| **S2** | Connection-pool outage — my outbox-dispatch change exhausted the SQL pool at peak | Spoke · failure | 2023 |
| **S3** | Lending decisions — "faster loan decisions" turned out to be a manual underwriting queue | Hub · ambiguity | 2020–21 (client) |
| **S4** | ADRs and design review across five teams — rejected the first time | Spoke · influence + people failure | 2024–25 |
| **S5** | Cosmos DB vs Azure SQL for the multi-tenant catalog — I was half wrong | Spoke · technical disagreement | 2022 |
| **S6** | Growing a mid-level engineer into the integration owner | Spoke · mentoring | 2024 |
| **S7** | Launch date vs threat-model findings — negotiating with the PM | Spoke · conflict | 2025 |
| **S8** | "Your design docs are too long, and you dominate reviews" | Spoke · feedback | 2022 |
| **R1** | Azure spend cut by about 38% | Reserve · frugality | 2025 |
| **R2** | Shadowing repair technicians at an industrial manufacturer | Reserve · customer | 2019 (client) |
| **R3** | Productive on AKS in three weeks for a client | Reserve · learning fast | 2021 (client) |

---

## Worked example 1 — From raw inventory to a bank

### Step 1: excavate (excerpt of 31 episodes)

```text
E01 2015 Q3  First production release (client: logistics)      — shipped on time; nothing hard
E04 2018 Q2  On-call rota redesign (services co.)               — cut weekend pages; nobody owned it
E06 2019 Q1  Repair-workflow discovery (client: industrial)     — shadowed technicians; requirement was wrong
E09 2020 Q4  Lending pipeline (client: fintech)                 — "faster decisions" → manual queue
E11 2021 Q2  AKS crash course (client: fintech)                 — 3 weeks to productive; first deployment failed
E13 2022 Q1  Cosmos vs SQL argument                             — lost round 1; spike changed decision; I was half wrong
E14 2022 Q3  Feedback from manager: docs too long, dominate reviews
E16 2023 Q1  Strangler migration kick-off                       — argued down a rewrite
E17 2023 Q3  Outbox dispatcher incident                          — my change; 35 min checkout degradation
E18 2023 Q4  Reporting team schema dispute                       — legitimate regulatory constraint
E20 2024 Q1  Delegated integration module to mid-level engineer
E22 2024 Q3  Data-centre exit                                    — 17 months, lease deadline 18
E23 2024 Q4  ADR proposal rejected by tech leads                 — tried again with co-authors
E26 2025 Q1  PM vs threat-model findings                         — phased launch
E27 2025 Q2  Azure cost review                                   — ~38% reduction
E29 2025 Q3  Hiring loop redesign for backend roles              — structured rubric; interviewer training
E31 2025 Q4  Argued against a second message broker              — "no" with an alternative
…
```

### Step 2: screen (shortlist of 14, staff weights from Concept 8, max 48)

| Episode | Level | Centr. | Depth | Rich. | Verif. | Rec. | Tension | Score | Note |
|---|---|---|---|---|---|---|---|---|---|
| E16/E18/E22 Migration (merged as one story — same project) | 3 | 3 | 3 | 3 | 3 | 3 | 3 | **48** | Hub |
| E09 Lending | 3 | 3 | 2 | 3 | 2 | 1 | 3 | **42** | Hub; client work, older |
| E23 ADR rollout | 3 | 3 | 2 | 2 | 2 | 3 | 3 | **42** | Influence + people failure |
| E17 Outbox incident | 2 | 3 | 3 | 2 | 3 | 3 | 3 | **40** | Failure |
| E13 Cosmos vs SQL | 2 | 3 | 3 | 2 | 3 | 2 | 3 | **39** | Tech disagreement |
| E26 PM vs threat model | 2 | 3 | 2 | 2 | 2 | 3 | 3 | **37** | Conflict |
| E20 Mentee | 2 | 3 | 2 | 1 | 2 | 3 | 2 | **33** | Mentoring |
| E14 Feedback | 1 | 3 | 1 | 1 | 2 | 2 | 3 | **28** | Feedback — level is low but competency needs it |
| E29 Hiring loop | 2 | 3 | 1 | 1 | 2 | 3 | 1 | **28** | Candidate for Hire and Develop the Best |
| E27 Azure cost | 2 | 3 | 2 | 1 | 3 | 3 | 1 | **31** | Frugality reserve |
| E06 Technicians | 1 | 3 | 2 | 2 | 1 | 1 | 2 | **26** | Customer reserve; old |
| E11 AKS | 1 | 3 | 2 | 1 | 2 | 1 | 2 | **24** | Learning-fast reserve |
| E04 On-call rota | 1 | 3 | 1 | 1 | 1 | 0 | 2 | **20** | Ownership — and possibly Earth's Best Employer |
| E31 Second broker | 2 | 3 | 2 | 1 | 2 | 3 | 2 | **34** | Saying no — maybe a moment of S1? (No — different project) |

Note the merge on the first line: E16, E18 and E22 are **moments of one story**, not three stories (Concept 2). The feedback story (E14) scores low on level but is kept because S5 (feedback) needs coverage and level matters less for that competency.

### Step 3: select (greedy against the invariants)

1. **Migration** first: covers flagship, tech disagreement, influence, mentoring (weakly), saying no, delivery.
2. **Lending** next: ambiguity (3), customer (3), influence (2), a second flagship.
3. **Outbox incident**: failure (3), dive deep (3), ownership (2).
4. **ADR rollout**: influence (3), and a second failure-capable story (people failure, 2).
5. **Cosmos vs SQL**: second tech disagreement (3), being wrong (2).
6. **PM vs threat model**: conflict (3) — cross-functional; saying no (3).
7. **Mentee**: mentoring (3).
8. **Feedback**: feedback (3).

Eight core stories. Matrix check (Worked example 3) shows gaps in **upward conflict**, **cost**, **customer** (only one strong cell), **learning fast**, and Amazon's **Hire and Develop** and **Earth's Best Employer** → reserves R1–R3 and a re-mine of E29 and E04 (Worked example 8).

---

## Worked example 2 — One hub story, six moments

**Story S1 — Ordering platform migration.** Shared context (two sentences, used at the start of any moment): *"I was lead architect for the ordering platform of a European e-commerce retailer — an ASP.NET MVC 5 monolith in a data centre whose lease ended in eighteen months, two teams of six, and a shared SQL Server that the reporting team and a nightly partner export also depended on. The business needed two things: out of the data centre on time, and much faster change in ordering."*

**Moment A — Technical decision (C4): strangler fig over rewrite.**
> *"This is about arguing down a rewrite in favour of a strangler fig — we left the data centre a month early. The team's instinct was a clean rewrite in .NET 8; my manager liked it because it promised one cutover date. I put the options side by side: rewrite, lift-and-shift, or replatform plus strangle. The rewrite failed on the criterion that mattered most — eighteen months with nothing delivered and a parity target nobody could define. I proposed a YARP facade as a walking skeleton, with the first slice being the new marketplace capability the business wanted, not a migrated feature. The cost I accepted was running two systems for over a year, mitigated by a named point of no return with go/no-go criteria. We exited in seventeen months, and lead time for ordering changes went from about six weeks to about four days."*

**Moment B — Influence / conflict (C6, C1): the reporting team.**
> *"…The reporting team refused any schema change, and they were right to worry — their regulatory reports had to reconcile exactly. Instead of pushing, I asked what 'safe' would mean to them, and we agreed: their views stay stable, and we prove equality daily. We fed their views from change data capture with a daily reconciliation job and a dashboard they owned. It took two quarters longer than I wanted — that stall at about 70% is the part I'd change — but it never broke a report, and their lead later co-presented the approach to the architecture forum."*

**Moment C — Being wrong / adapting (S5, C3): the cutover plan.**
> *"…Two months in, a load test showed my cutover plan — moving order ownership per region — would double-write for weeks in the largest region. I'd been confident in it and had presented it. I told the steering group the plan was wrong, why, and proposed cutover by order type instead, which removed the long double-write window. It cost us three weeks. Since then I put a load test of the cutover itself in the plan before committing to a sequence."*

**Moment D — Mentoring (C5): the order module.**
> *"…I deliberately handed the order-module extraction to a mid-level engineer who'd never led a design. We met twice a week; I asked questions rather than giving answers, and I let one reversible decision go her way that I'd have made differently. It shipped about three weeks later than if I'd done it — which I'd agreed with her manager in advance — and she now owns ordering."*
> *(Note: S6 is a separate, deeper mentoring story; this moment is the backup.)*

**Moment E — Saying no / stakeholder (S6): go-live criteria.**
> *"…Six weeks before the planned switch, the commercial director wanted to bring it forward for a campaign. I said not like that — and showed the cost in his terms: switching before reconciliation had run clean for two weeks risked mis-billed orders in the campaign itself. I offered an alternative: run the campaign's new checkout features in the new system behind the facade, which delivered what he needed without moving the point of no return. He agreed; the campaign ran on the new features; the switch happened on the original date."*

**Moment F — Delivering results / frugality (S1, S10): the outcome.**
> *"…We exited the data centre in seventeen months against an eighteen-month lease; ordering lead time went from about six weeks to four days; and we retired the reporting licence, roughly €40k a year — I'm approximating, that's the figure finance quoted. Since then another team has extracted pricing the same way."*

**What to notice:**

- **The shared context is two sentences** and identical in every moment — learned once.
- **Each moment has its own headline, task, decision and result** — and could be probed on its own.
- **The numbers are identical** across moments (17 months, six weeks → four days, ~€40k approximate) — fact sheet discipline (Concept 30).
- **One story, six competencies** — but in any loop, this candidate would use at most two or three of these moments, with different interviewers (Concept 27).

---

## Worked example 3 — The filled coverage matrix and gap analysis

Strengths 0–3 at **staff** level. Core columns first.

| | C1 Conf | C2 Fail | C3 Ambg | C4 Tech | C5 Ment | C6 Infl | S1 Flag | S2 Dlvr | S3 Cust | S4 Ownr | S5 Fdbk | S6 SayN | S7 Simp | S8 Bar | S9 Lrn | S10 Cost | S11 Deep |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **S1 Migration** | 2 | 1 | 2 | 3 | 2 | 3 | 3 | 3 | 1 | 2 | 2 | 3 | 2 | 1 | 1 | 2 | 2 |
| **S2 Pool outage** | 0 | 3 | 1 | 1 | 0 | 0 | 0 | 1 | 1 | 2 | 2 | 0 | 0 | 2 | 0 | 0 | 3 |
| **S3 Lending** | 1 | 0 | 3 | 1 | 0 | 2 | 2 | 2 | 3 | 1 | 0 | 1 | 2 | 0 | 2 | 0 | 2 |
| **S4 ADR rollout** | 1 | 2 | 1 | 1 | 1 | 3 | 1 | 0 | 0 | 2 | 2 | 0 | 1 | 2 | 0 | 0 | 0 |
| **S5 Cosmos vs SQL** | 1 | 1 | 1 | 3 | 0 | 1 | 0 | 0 | 1 | 0 | 2 | 0 | 1 | 1 | 1 | 2 | 3 |
| **S6 Mentee** | 0 | 0 | 0 | 0 | 3 | 1 | 0 | 0 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 |
| **S7 PM vs threat model** | 3 | 0 | 1 | 1 | 0 | 2 | 0 | 2 | 2 | 1 | 0 | 3 | 0 | 3 | 0 | 0 | 1 |
| **S8 Feedback** | 1 | 1 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 3 | 0 | 1 | 0 | 0 | 0 | 0 |
| **R1 Azure cost** | 0 | 0 | 1 | 1 | 0 | 1 | 0 | 0 | 0 | 2 | 0 | 0 | 2 | 0 | 0 | 3 | 2 |
| **R2 Technicians** | 0 | 0 | 2 | 0 | 0 | 0 | 0 | 0 | 3 | 1 | 1 | 0 | 1 | 0 | 2 | 0 | 0 |
| **R3 AKS** | 0 | 1 | 1 | 0 | 0 | 0 | 0 | 1 | 0 | 0 | 1 | 0 | 0 | 0 | 3 | 0 | 1 |
| **Strong cells (≥2)** | **2** | **2** | **3** | **2** | **2** | **4** | 2 | 3 | 4 | 5 | 6 | 2 | 4 | 3 | 3 | 3 | 6 |

### Invariant checks

| Invariant | Result |
|---|---|
| 1. Each core column ≥ 2 strong cells in different stories | ✔ — but C1, C2, C4, C5 have exactly two: no slack |
| 2. Each supporting column ≥ 1 strong cell | ✔ |
| 3. One strong core cell at target (staff) level | C1 conflict: S7 is cross-functional with a PM — **senior-level**; S1's reporting-team moment is the only staff-level conflict ⚠ |
| 4. Failure cells from real failures | ✔ S2 (my config), S4 (first attempt rejected) |
| 5. No story the only strong option for > 1 core column | ✔ |
| 6. No story overloaded | ⚠ S1 is ≥ 2 in eleven columns — must cap its use per loop at three moments |

### Portfolio tally

```text
Polarity:   success 6 · mixed 3 (S1, S4, S5) · failure 2 (S2, S4)    ✔
Direction:  peer 3 · upward 1 (S1-E only) · downward 2 · cross-team 4 · external 2    ⚠ upward thin
Horizon:    ≥ 1 year: S1, S3, S4                                      ✔
Context:    services co. (S3, R2, R3) · product co. A (S5, S8) · product co. B (rest)    ✔
```

### Gaps and fixes

| Gap | Fix | Strategy |
|---|---|---|
| **Conflict has no slack, and only one staff-level option** | Re-mine 2024–25 for a disagreement with a director or between teams → found E31 (arguing against a second message broker with the head of platform) | Re-mine → new spoke S9 |
| **Upward direction thin** | E31 is upward; also re-angle S1 moment E (commercial director) as an upward conflict | Re-mine + re-angle |
| **Failure has no slack** | R3's first failed AKS deployment is small (strength 1). Re-mine for a people failure → S4 already; accept two, and tighten S4's failure moment | Accept + strengthen |
| **Mentoring has no slack** | E29 hiring-loop redesign gives a team-level growth mechanism (interviewer training) | Re-mine → reserve R4 |
| **Amazon: Earth's Best Employer** | E04 on-call rota redesign (cut weekend pages) — old (2018) but genuine | Reserve R5, used only for Amazon |
| **Google: stepping back** | S6 contains it — the candidate let the mentee present at the forum and own the decision | Re-angle |

---

## Worked example 4 — Amazon Leadership Principles mapping (two per principle)

| Leadership Principle | Primary (story · moment) | Backup |
|---|---|---|
| Customer Obsession | R2 Technicians · requirement was wrong | S3 Lending · underwriters' queue |
| Ownership | S2 Pool outage · I owned the post-mortem and the gate | R5 On-call rota · nobody owned it |
| Invent and Simplify | R1 Azure cost · consolidated three environments | S1 Migration · facade instead of rewrite |
| Are Right, A Lot | S5 Cosmos vs SQL · spike, and I was half wrong | S1 · cutover plan corrected |
| Learn and Be Curious | R3 AKS in three weeks | S3 Lending · learning credit decisioning |
| Hire and Develop the Best | S6 Mentee | R4 Hiring-loop redesign |
| Insist on the Highest Standards | S7 Threat-model findings | S2 · load-test gate for pool metrics |
| Think Big | S1 Migration · two-track strategy | S4 ADR rollout · org-wide decision practice |
| Bias for Action | S3 Lending · two-day shadowing before estimating | S2 · mitigation in minutes via config |
| Frugality | R1 Azure cost −38% | S1 · reporting licence retired |
| Earn Trust | S2 · disclosed my part within the hour | S8 Feedback |
| Dive Deep | S2 · connection held across an external await | S5 · RU cost of cross-partition queries |
| Have Backbone; Disagree and Commit | S9 Second broker (disagreed; partially lost; committed) | S5 (lost round one; committed to the spike outcome) |
| Deliver Results | S1 · DC exit in 17 months | S3 · decisions in hours |
| Strive to be Earth's Best Employer | R5 On-call rota | S6 · creating space for the mentee |
| Success and Scale Bring Broad Responsibility | S7 · didn't ship with an unmitigated data-exposure risk | S3 · fair, explainable lending pre-screen rules |

Note what this needs: fourteen stories (nine core including S9 from Worked example 8, five reserves) cover 32 slots — at the top of the range in Concept 3, which is what "when to deviate" anticipated for Amazon at staff level; R5 exists *only* for Amazon. And **no story is primary for more than three principles**, which keeps loop allocation feasible.

---

## Worked example 5 — Pre-mortem and fact sheet for S5 (Cosmos DB vs Azure SQL)

**The story in one paragraph.** For a multi-tenant product catalog, a principal engineer proposed Cosmos DB with tenant as partition key; the candidate argued for Azure SQL with elastic pools. The candidate lost the first discussion, proposed a two-day spike against the five real query patterns, and found: Cosmos was clearly better for per-tenant reads and global distribution; but two reporting-style queries were cross-partition and cost roughly 25× the estimated RUs. Outcome: Cosmos for the catalog with a hierarchical partition key, plus a change-feed projection into Azure SQL for the cross-tenant reporting queries. The candidate was *half wrong* — they'd underweighted the global-distribution requirement.

**Pre-mortem (selected attacks and defences):**

| Attack | Defence |
|---|---|
| *Attribution*: "Was the spike your idea, or the principal's?" | Mine — I proposed it at the end of the meeting I'd lost; he agreed to defer the decision two days. He wrote the Cosmos half of the spike; I wrote the SQL half and the query harness. |
| *Numbers*: "25× — measured how?" | RU charge reported per query in the spike against a 2-million-item synthetic dataset matching the tenant-size distribution we expected; I'd estimated from the capacity calculator. "Roughly 25×" — the exact figures varied by query between about 18× and 30×. |
| *The other side*: "What was his strongest argument?" | Global distribution — two regions with low-latency reads within a year, which I'd treated as speculative and he'd already confirmed with product. He was right about that. |
| *Level*: "Why is this more than a senior story?" | Honestly, mostly it is a senior story. The staff-level part is what came after: I turned the spike harness into a template we used for the next two data-store decisions, and the ADR's revisit condition was triggered a year later. If I need a staff-level technical disagreement, S1 moment A is stronger. |
| *Depth*: four levels | Why cross-partition queries cost more → fan-out to all physical partitions → RU charge proportional to partitions touched → why a hierarchical key (tenant, category) helped the per-tenant queries but not the cross-tenant ones → hence the change-feed projection. |
| *Durability* | The ADR had a revisit condition — "if cross-tenant reporting exceeds 10% of catalog queries". It was triggered after a year; the projection pattern held. |

**Repair from the pre-mortem:** the level attack showed this story is senior-level for C4. The candidate **re-tagged** it: C4 strength 2 (not 3) at staff level, S5 "being wrong" strength 2, and promoted S1 moment A to the primary staff-level technical disagreement. *Claim less, more precisely.*

---

## Worked example 6 — Loop allocation for two loops

### Amazon, SDE III (senior) loop — phone screen + five interviews

The recruiter said: one round with the hiring manager (leadership-heavy), one Bar Raiser, three technical rounds each with two or three LP questions.

```text
Round            Likely LPs (guess)                        Planned stories (primary → backup)
Phone screen     Ownership, Dive Deep                       S2 Pool outage → R1 Azure cost
R1 Coding        Deliver Results, Bias for Action           S3 Lending → S1·F outcome
R2 Sys design    Invent and Simplify, Think Big             S1·A strangler → R1 consolidation
R3 Coding        Learn and Be Curious, Insist on Standards  R3 AKS → S7 threat model
R4 Hiring mgr    Hire and Develop, Earn Trust, Backbone     S6 Mentee · S8 Feedback · S9 Second broker
R5 Bar Raiser    Customer Obsession, Are Right A Lot, any   R2 Technicians · S5 Cosmos (half wrong) · S4 ADR (people failure)

Limits: S1 at most 3 moments in the whole loop (A, F, + one spare); no story twice with one interviewer.
```

**During the loop:** R1 asked a failure question (not on the plan). The candidate used S2 (planned for the phone screen, already used there — but with a *different* interviewer, so acceptable), then promoted S4's people-failure moment to the failure primary for the rest of the day. R4 asked about influence without authority; S4 was now reserved for failure, so the candidate used S1·B (reporting team) — S1's second moment of the loop.

### Meta, E6 loop (one dedicated behavioral round)

Four to six questions, brisk, conflict-heavy. Planned:

```text
Conflict (×2 likely)  S9 Second broker (upward, staff-level)  → S1·B reporting team → S7 PM
Ambiguity             S3 Lending                              → S1·C cutover wrong
Growth                S8 Feedback                             → S5 half wrong
Driving results       S1·F outcome                            → S4 ADR rollout
Communication         S4 ADR rollout (writing, forums)        → S1·E commercial director
```

**Why three conflict options:** Concept 24 — Meta probes the conflict signal from several angles ("another time?", "with your manager?"), and E6 needs multi-team scope in the conflict story itself.

---

## Worked example 7 — A starter inventory for a client-delivery and own-product career

This example shows how to begin a bank for a career shaped like this: several years delivering projects for a range of clients through a software services company, followed by building an own multi-tenant product. It lists **only project types** and **prompts** — the facts, numbers and outcomes have to come from your own memory and artefacts. Clients are named by type rather than by name, which is also how you'll tell them in interviews.

| Engagement (described by type) | Moments to look for | Prompts to answer from memory and artefacts | Likely competencies |
|---|---|---|---|
| **Internal tooling for a game-technology company** (an internal operations tool; a contributor-licence-agreement check integrated with GitHub) | Working inside a large client's engineering culture; integrating with their workflows and security requirements; adoption by their engineers | Who were the users and how many? What did their engineers push back on? What did you change after first use? Is it still in use? | Customer focus (internal users), influence without authority, raising the bar, learning fast |
| **Product-analysis and repair-workflow platform for an industrial-equipment manufacturer** | Domain learning (repair processes); requirements that changed after seeing real users; integration with existing enterprise systems | What did you not understand about the domain at first? Did you see real technicians or analysts work? Which requirement turned out wrong? What did the client's own IT insist on? | Ambiguity, customer focus, learning fast, technical disagreement |
| **Micro-lending platform for a fintech** | Correctness and auditability of money flows; regulatory constraints; trade-offs between speed and risk | What was the hardest correctness problem? What failure would have cost money, and how did you prevent it? Who did you disagree with — risk, product, the client's CTO? | Raising the bar, failure (or near-miss), technical disagreement, broad responsibility |
| **Internal logistics tool for a heavy-lift and crane-services company** | Operational users in the field; reliability and offline/latency constraints; replacing spreadsheets or manual processes | What did the tool replace? What did field users actually need? What did you deliberately leave out? What changed for them afterwards? | Customer focus, simplification, saying no, delivering results |
| **E-commerce platform rebuild for a German custom-PC retailer** | Peak traffic; product configurators and pricing; migration from an older platform; a launch | What was the hardest launch moment? Was there an incident? What did you decide about migration vs rewrite? What happened to conversion or performance? | Flagship candidate, delivering under pressure, failure, dive deep |
| **Own multi-tenant assessment platform** (public consumer tenant plus private B2B tenants) | Architectural decisions with no one to defer to; tenant isolation and multi-tenancy design; prioritisation under very limited resources; preparing a business case for funding | What did you decide not to build? Which early decision would you defend, and which reverse? Who did you have to persuade — partners, contractors, testers, potential investors — and what changed your mind? What are the exact current users and revenue, including zero? | Ambiguity, saying no, technical judgment, business case (Module 33), being wrong |

**How to use it:**

1. **For each engagement, write five to ten one-line episodes** using the prompts (Concept 6). Expect thirty or more.
2. **For client engagements, mine the boundaries** — the discovery workshop, the client architect's objection, the acceptance negotiation, the handover (Concept 11) — and message a former colleague to learn what happened after the engagement ended.
3. **Score them** (Concept 8). Watch the *Level* criterion: client projects can be staff-level when you shaped the solution and influenced the client's people; they're senior-level when you implemented an agreed design well. Both are legitimate; tag them honestly.
4. **Look for the hub.** In careers like this the flagship is often either the largest client platform (where you can show technical ownership and client influence) or the own product (where you can show total ownership of decisions). Ideally you have one of each — and the own-product story is balanced by client stories that show you operating inside other organizations.
5. **Plan the narrative thread** for "tell me about yourself" and "why are you looking now": breadth across domains through client work → depth and full ownership through the own product → the kind of role you want next. Your bank's headlines are the evidence for that thread.

**Things to watch for in this career shape:**

- **"We delivered" framing.** Services-company language is team- and contract-centric. Convert it: *"I was the technical owner for that client — architecture, delivery and production support."*
- **Durability gaps.** You may not know what happened after handover. Find out where you can; otherwise say what you built to make it sustainable — runbooks, ADRs, training.
- **Founder scale honesty.** Tell the own-product story as a judgment and ownership story at its true scale. A clear account of decisions, trade-offs and what you learned from early users or testers is strong; implied scale that isn't there is a credibility risk.
- **Anonymisation consistency.** Decide once how each client is described ("a European industrial-equipment manufacturer") and put it on the fact sheet.

---

## Worked example 8 — Rescue: filling two gaps honestly

**Gap 1: no staff-level conflict story.** The composite candidate's best conflict story (S7, PM vs threat model) is cross-functional but senior-scope.

*Re-mine.* Prompts from Concept 6 — "the decision you argued against", "the time you escalated". The candidate remembered **E31**: the head of platform wanted to introduce Kafka alongside Service Bus for a new analytics stream. The candidate thought a second broker would double operational load for a modest benefit.

*Check it's real conflict, not just technical disagreement.* The interesting part turned out to be people: the head of platform had promised the analytics team a "streaming platform" and felt the candidate was undermining that commitment in front of the CTO. That makes it **C1 (conflict, upward)** with a C4 element.

*Build the moment.*
> *"This is about disagreeing with our head of platform — two levels above me — on adding a second message broker, and how we ended up with a compromise neither of us had proposed. She'd committed to the analytics team that they'd get a streaming platform; I thought running Kafka alongside Service Bus would roughly double our messaging on-call surface for a stream that Event Hubs with its Kafka-compatible endpoint could serve. I'd made the mistake of raising it first in a meeting with the CTO, which put her on the defensive — that's the part I'd do differently. I asked for a one-to-one, said that, and asked what she'd promised and why. The real requirement was replay and multiple independent consumers. We wrote a one-page comparison together; we chose Event Hubs with the Kafka endpoint, so the analytics team could use Kafka clients without us running a Kafka cluster. She presented it to the CTO as the platform team's decision, which it was. A year on it handles the analytics stream with no extra on-call rota, and she and I co-authored the messaging guidelines afterwards."*

Strength at staff level: C1 3, C4 2, S5 2 (raised it badly at first), S10 2. Gap filled — and a backbone-and-commit story for Amazon.

**Gap 2: mentoring has no slack.** Only S6 is strong.

*Re-mine* found **E29**: the candidate redesigned the backend hiring loop — a structured rubric, calibration sessions, and pairing new interviewers with experienced ones. *Is it mentoring?* Partly — it grows interviewers, and it's direct evidence for Amazon's Hire and Develop the Best. Tagged **C5 2, S8 2**, kept as reserve R4. The candidate also re-angled **S1 moment D** as a backup — weaker (strength 2), but a different story from S6.

**What the candidate did *not* do:** promote a two-week onboarding buddy assignment into a "growing a leader" story, or merge S6 with another mentee to make a better arc. Both would have failed the probes in Concept 29.

---
# Common interview questions with model answers

Questions 1–5 are **meta-questions** about preparation and stories themselves — they come up with recruiters, coaches, hiring managers and occasionally interviewers. Questions 6–16 show **how the bank is used** for real prompts: the decode, the selection, and the opening of the answer. Model answers use the composite bank from the worked examples.

**Q1. "How did you prepare for this interview?"** *(hiring manager, sometimes as small talk)*
> "For the behavioral side, I went back through the last five or six years — design docs, post-mortems, old dashboards — and picked the projects that best show how I work, then checked I could back up the numbers. For the technical side, I reviewed system design and your stack. I didn't script answers; I wanted to be able to talk about real work in depth."
*Why it works:* honest, shows rigour, signals that answers will be specific and true — and doesn't over-explain the machinery.

**Q2. "Can you give me a different example?"** *(redirect)*
> "Sure." — then go to the backup from a different story, with a fresh headline. *Never defend the first choice; never take a second moment of the same story.*

**Q3. "You mentioned this project earlier — do you have something from somewhere else?"** *(interviewer has noticed reuse)*
> "Yes — from my time on client projects: …" *A bank with context diversity (Concept 10) makes this painless.*

**Q4. "That sounds like a team effort. What would have gone differently without you?"** *(counterfactual probe — Concept 29)*
> "Two things, I think. The team would probably have gone with the rewrite — that was the default view, and I was the one who put the options side by side and argued for the strangler. And the reporting dependency: without the reconciliation approach I negotiated, I think we'd have stalled indefinitely, because that team had a legitimate veto. The implementation of the new ordering service was the team's work — that would have been fine without me."

**Q5. "Tell me about yourself."** *(not STAR, but the bank feeds it)*
> *Shape:* present role and level → two headlines from the bank as evidence of the kind of work you do → what you want next and why this role. *"I'm a senior .NET engineer working as the architect for an ordering platform. Two things I'm proudest of: leading a strangler-fig migration that got us out of a data centre a month early and cut lead time for changes from weeks to days, and getting five teams to adopt decision records after my first attempt was rejected. What I want next is more of that cross-team architecture work, which is what this role is."*
*Key signal:* 60–90 seconds; headlines, not stories; ends by connecting to the role.

**Q6. "Tell me about a time you had a conflict with a co-worker."**
> *Decode:* C1 conflict, direction peer, success or mixed polarity. *Select:* S1 moment B (reporting team) — staff-level, cross-team; backup S7.
> *Opening:* "This is about a team that had a legitimate veto over our migration and how we found a way forward that kept their regulatory reports safe — it cost us two quarters but never broke a report."

**Q7. "Tell me about a time you disagreed with your manager."**
> *Decode:* C1, **upward**. *Select:* S9 (head of platform) — or S1 moment A if the manager-rewrite angle is stronger.
> *Opening:* "This is about disagreeing with our head of platform on adding a second message broker — I raised it badly at first, and we ended up with an option neither of us had proposed."

**Q8. "Tell me about a decision you made with incomplete information."**
> *Decode:* C3 ambiguity (not delivery), plus judgment under uncertainty. *Select:* S3 Lending (problem ambiguity) — or S5 if they want a technical decision.
> *Opening:* "A fintech client asked us for 'faster loan decisions' — the real problem turned out to be a manual underwriting queue, and we got most decisions from about three days to a few hours."

**Q9. "Tell me about a time you were wrong."**
> *Decode:* S5 being wrong, negative/mixed polarity. *Select:* S1 moment C (cutover plan) — high stakes, public correction; backup S5 (half wrong).
> *Opening:* "This is about a cutover plan I'd designed and presented to the steering group, which a load test showed was wrong two months in."

**Q10. "Tell me about a time you failed."**
> *Decode:* C2 failure. *Select:* S2 (technical, mine) or S4 (people failure — stronger at staff level). For a staff loop, consider leading with S4.
> *Opening (S4):* "This is about the first time I tried to introduce decision records across five teams — the tech leads rejected it, and they were right to, given how I'd done it."

**Q11. "Describe a time you influenced people who didn't report to you."**
> *Decode:* C6. *Select:* S4 (second attempt, five teams) — or S1 moment E if S4 has already been used as failure.
> *Opening:* "This is about getting five teams to adopt architecture decision records on the second attempt, after the first was rejected — the difference was co-authors and making it cheaper than not doing it."

**Q12. "Tell me about someone you helped grow."**
> *Decode:* C5. *Select:* S6. *Opening:* "This is about a mid-level engineer who'd never led a design and now owns our integration platform — which also took me off the critical path for it."

**Q13. "Tell me about a time you had to choose between doing it right and doing it fast."** *(compound: S2 delivery + S8 raising the bar)*
> *Decode:* both. *Select:* S7 (launch vs threat model) — contains both the deadline and the standard.
> *Opening:* "This is about a launch where a threat model found a data-exposure risk two weeks before the date — we shipped on time with a phased scope, and the risky feature followed three weeks later."

**Q14. "What would you do if two senior engineers on your team strongly disagreed on an approach?"** *(hypothetical — Google style)*
> *Shape:* structured approach (understand both positions, turn opinions into criteria, cheapest evidence that would decide, who decides, record it, ensure both commit) → anchor: *"…which is close to what happened with the Cosmos DB versus SQL decision, except I was one of the two engineers."*

**Q15. "Tell me about a time you did more with less."** *(Amazon Frugality)*
> *Select:* R1. *Opening:* "This is about cutting our Azure spend by roughly 38% without touching production capacity — mostly by consolidating environments nobody had questioned."

**Q16. "Is there anything you'd like to tell me that we haven't covered?"** *(end of round)*
> Use it if a competency you consider important for the role hasn't come up and you have a strong story for it: *"One thing we didn't touch on is how I work with product when we disagree on priorities — would a two-minute example be useful?"* Keep it short; let the interviewer decline.

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Preparation unit | Prepares answers to a list of 50 questions | Prepares stories indexed by competency; uses question lists as test data |
| Story count | Two or three "great stories" for a 12-question loop | 6–8 core plus 2–4 reserves, every core competency covered twice |
| Story richness | Many thin stories, one moment each | Hub stories with several separable moments |
| Selection | Most memorable stories | Excavated 25–40 episodes, screened on level and centrality |
| Facts | From memory; numbers drift between rounds | Rebuilt from artefacts; one canonical fact sheet per story |
| Conflict vs disagreement | One story for both | Separate stories: people/priorities vs technical substance |
| Conflict direction | Only peer conflicts | Peer, upward, cross-functional, downward, external |
| Failure | One small failure | Two right-sized failures, one of them a people/alignment failure |
| Mentoring | "I answered juniors' questions" | A person, before and after, specific actions, sponsorship |
| Feedback | Humble-brag feedback | Uncomfortable, correct feedback with evidence of change |
| Company calibration | Same bank everywhere | Matrix columns per framework; deliberate mining for forgotten principles |
| Amazon | Stretches unrelated stories for Frugality or Earth's Best Employer | Short, true reserves for the commonly missing principles |
| Google | Only "stepping up" leadership | Emergent leadership in both directions; hypotheticals anchored in real stories |
| Meta | One conflict story | Three conflict-capable moments; multi-team scope at E6 |
| Gaps | Fills with composites or borrowed credit | Re-mines, re-angles, writes reserves, or prepares an honest near-miss |
| Level | Tells staff-level work as "I built X" | Level audit; re-tells framing and alignment work into the actions |
| Probes | Discovers weak points in the interview | Pre-mortem with a sceptical partner; claims narrowed to what survives |
| Repetition | Same story to several interviewers | Allocation plan with primaries and backups from different stories |
| Adapting | Forces a poor-fit story or invents one | Bridges the 70% question; offers an honest near-miss |
| Practice | Rereads cards; memorises scripts | Spaced, interleaved retrieval drills; varied lengths; recorded tellings |
| Mocks | Polite friend, no follow-ups | Briefed partner or AI interviewer asking only probes; scored on a rubric |
| Day of | Notes on screen | Pre-read index; tracking sheet between rounds; after-action review |
| Maintenance | Rebuilds from scratch every search | Brag document and quarterly refresh; gaps become career goals |

---
# Practice exercises

1. **Competency decoding.** Take the 30 questions on the Tech Interview Handbook's common-questions page and, for each, write the competency, direction, polarity and any level cue (Concept 5). Check yourself against Appendix D.
2. **Excavation.** Run the timeline sweep and prompt-driven recall (Concept 6) for every role or engagement. Target at least 25 one-line episodes. Do it over two sessions on different days — the second session will surface episodes the first missed.
3. **Artefact sweep.** For your top 10 episodes, find at least one artefact each (commit, doc, post-mortem, dashboard, ticket, message). Write down every number you recover and every number you can't.
4. **Screening.** Score your episodes on the seven criteria with the weights for your target level (Concept 8). Apply the hard floors. Produce a ranked shortlist of 10–14.
5. **Moments.** For your top three stories, list every separable moment and the competency each answers. Aim for at least four moments in your best story.
6. **The matrix.** Build the coverage matrix (Appendix C). Check all six invariants (Concept 21) and the portfolio tally (Concept 10). Write down every gap.
7. **Company columns.** Add columns for one target company's framework (Amazon LPs, Google attributes, Meta signals or a published competency list). Find two stories per item, or mark the gap.
8. **Gap filling.** For each gap, try the four strategies in order (Concept 26). Record which strategy filled it. If none did, write the honest near-miss.
9. **Level audit.** Run every core story through the six-axis audit (Concept 28) and classify it: passes, re-tell, or demote. Re-tell at least one story so the staff-level actions are in the telling.
10. **Pre-mortem.** Give your two hub stories to a peer or an AI mock interviewer with the attack list from Concept 29. Repair each weakness by narrowing a claim, adding a fact, or volunteering a caveat.
11. **Fact sheets.** Write a fact sheet for every core story (Concept 30). Compare it with your CV and LinkedIn; fix any mismatch.
12. **The 70% drill.** Ask a partner for ten questions that are deliberately *slightly off* from your prepared moments ("with a PM rather than an engineer", "a time you missed the deadline"). Bridge, re-weight, and close on the question (Concept 32).
13. **Spaced retrieval deck.** Build an Anki (or paper) deck: question on the front, primary and backup story-moments on the back. Review it daily for two weeks; tell five random cards aloud each day.
14. **Recorded variation.** Record one story three ways: two minutes headline-first, five minutes in layers, and starting from the conflict. Score each on Module 34's Appendix B rubric.
15. **Loop allocation.** For a real or hypothetical loop at a target company, write the one-page allocation plan (Concept 27) and the day-of index card (Concept 36). Then run a five-question mock and track which stories you used; check you never repeated one with the same interviewer.

---
# Free resources and learning material

All free to read online unless marked *(book)* or *(partly paid)*. Start with the ★ items. Company pages and policies were checked on October 8, 2026.

### Story banks and story selection
- ★ [Choosing responses strategically — Hello Interview](https://www.hellointerview.com/learn/behavioral/course/select-choosing-responses-strategically) *(partly paid)* — story catalog, and selection by scope, relevance, uniqueness and recency.
- ★ [How Behavioral Interviews Really Work — Hello Interview](https://www.hellointerview.com/blog/how-behavioral-interviews-really-work) — why level-appropriate stories matter and how down-levelling happens.
- ★ [Stop memorizing STAR — start selecting better stories — interviewing.io](https://interviewing.io/blog/stop-memorizing-star-for-behavioral-interviews-start-selecting-better-stories) — selection over structure at senior-plus.
- [How to Nail Big Tech Behavioral Interviews as a Senior Software Engineer — Engineering Leadership newsletter](https://newsletter.eng-leadership.com/p/how-to-nail-big-tech-behavioral-interviews) — a former hiring-committee chair on the story catalog and high-water-mark stories.
- [Building a Leadership Story Bank for Onsite Interview Week — techinterview.org](https://www.techinterview.org/post/3233474667/leadership-story-bank/) — themes per story, stories per theme, and a maintenance loop.
- [Story-bank builder prompt — techinterview.org](https://www.techinterview.org/interview_prompt_library/story-bank-builder/) — an AI interview prompt that mines your career and marks "needs a number" rather than inventing one; useful as permitted preparation.
- [Story selection — RocketBlocks](https://www.rocketblocks.me/behavioral-interviews/story-selection.php) — building and ranking a "story library".
- [Framing answers for multiple questions — RocketBlocks](https://www.rocketblocks.me/behavioral-interviews/framing-interview-answers.php) — using one story for several questions, and why not to repeat across a loop.
- [The Behavioral Interview Roadmap — ByteByteGo](https://bytebytego.com/courses/behavioral-interview/the-behavioral-interview-roadmap) *(partly paid)* — the "moments within a story" idea, with a timeline example.
- [Deeper dive into behavioral signal areas — The Behavioral (Substack)](https://thebehavioral.substack.com/p/deeper-dive-into-the-signal-areas) — signal areas including scope, from a former big-tech interviewer.

### Question banks (use as test data, not as answers)
- ★ [The 30 most common behavioral questions — Tech Interview Handbook](https://www.techinterviewhandbook.org/behavioral-interview-questions/) — plus company-specific lists.
- [Behavioral interview step-by-step — Tech Interview Handbook](https://www.techinterviewhandbook.org/behavioral-interview/).
- [Behavioral interview rubrics — Tech Interview Handbook](https://www.techinterviewhandbook.org/behavioral-interview-rubrics/).
- ★ [Behavioral interviews for senior candidates — Tech Interview Handbook](https://www.techinterviewhandbook.org/behavioral-interview-senior-candidates/).
- [Preparing a self introduction — Tech Interview Handbook](https://www.techinterviewhandbook.org/self-introduction/) — the bank's headlines feed it.
- [Preparing final questions to ask — Tech Interview Handbook](https://www.techinterviewhandbook.org/final-questions/).
- [The STAR method (with worksheet) — MIT CAPD](https://capd.mit.edu/resources/the-star-method-for-behavioral-interviews).

### Amazon
- ★ [Amazon Leadership Principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) — read every principle literally; they are the column headers.
- ★ [SDE III interview prep — Amazon](https://amazon.jobs/content/en/how-we-hire/sde-iii-interview-prep) — phone screen and loop format; STAR and metrics.
- [BIE interview prep — Amazon](https://amazon.jobs/content/en/how-we-hire/bie-interview-prep) — states that each interviewer typically asks two or three behavioral questions.
- [The interview loop and Bar Raisers — Amazon](https://amazon.jobs/content/en/how-we-hire/interview-loop).
- ★ [A Senior Engineer's Guide to the Amazon Leadership Principles Interview — interviewing.io](https://interviewing.io/guides/amazon-leadership-principles) — built from Amazon interviewers and mock data.
- [Amazon Leadership Principles questions — IGotAnOffer](https://igotanoffer.com/en/advice/amazon-leadership-principles) — example questions per principle.
- [Amazon behavioral interview questions — IGotAnOffer](https://igotanoffer.com/blogs/tech/amazon-behavioral-interview) — a 60+ question bank to test your matrix against.

### Google
- [How we hire — Google](https://www.google.com/about/careers/applications/how-we-hire) and [Interview tips — Google](https://www.google.com/about/careers/applications/interview-tips).
- ★ [A guide to structured interviewing — Google re:Work](https://rework.withgoogle.com/intl/en/guides/a-guide-to-structured-interviewing-for-better-hiring-practices) — attributes, behavioral and hypothetical questions, scoring.
- [Googleyness & Leadership interview questions — IGotAnOffer](https://igotanoffer.com/blogs/tech/googleyness-leadership-interview-questions).

### Meta
- ★ [The Meta SWE interview, explained by a former Meta interviewer — Hello Interview](https://www.hellointerview.com/blog/the-meta-swe-interview) — including the behavioral round and its signals.
- [Meta behavioral interview cheat sheet — techinterview.org](https://www.techinterview.org/post/3233477365/meta-behavioral-interview-cheat-sheet/) — signals, entry points and story prompts.

### Microsoft, enterprises and public competency frameworks
- ★ [Interview tips and how we hire — Microsoft Careers](https://careers.microsoft.com/us/en/interviewtips) — respect, integrity, accountability and growth mindset; responsible AI use in preparation.
- [How to ace Microsoft's hiring process — Levels.fyi](https://www.levels.fyi/blog/ace-microsoft-hiring.html).
- ★ [Success Profiles — UK Civil Service (GOV.UK)](https://www.gov.uk/government/publications/success-profiles) — a fully public competency framework with level descriptors.
- [Success Profiles: Civil Service behaviours — GOV.UK](https://www.gov.uk/government/publications/success-profiles/success-profiles-civil-service-behaviours) — see how behaviours are specified per grade; a model for enterprise competency columns.

### AI and preparation norms
- ★ [Guidance on candidates' AI usage — Anthropic](https://www.anthropic.com/candidate-ai-guidance) — AI for research and practice; live interviews are "all you" unless told otherwise.
- [Is it cheating? AI use during job interviews — GeekWire (2025)](https://www.geekwire.com/2025/is-it-cheating-ai-use-during-job-interviews-sparks-debate-over-whether-to-restrict-emerging-tools/).

### Levels and role shapes (for the level audit)
- ★ [Dropbox Engineering Career Framework](https://dropbox.github.io/dbx-career-framework/) — compare [IC4](https://dropbox.github.io/dbx-career-framework/ic4_software_engineer.html) and [IC5 Staff](https://dropbox.github.io/dbx-career-framework/ic5_staff_software_engineer.html).
- [progression.fyi](https://progression.fyi/) — many public career frameworks.
- [Levels.fyi](https://www.levels.fyi/) — cross-company level mapping.
- ★ [Staff archetypes — StaffEng](https://staffeng.com/guides/staff-archetypes/).
- [Interviewing for Staff-plus roles — StaffEng](https://staffeng.com/guides/interviewing-staff-plus-roles/).
- [Promotion packets — StaffEng](https://staffeng.com/guides/promo-packets/) — a promotion packet is a written story bank at staff scope.
- [Staff projects — StaffEng](https://staffeng.com/guides/staff-projects/) and [Work on what matters — StaffEng](https://staffeng.com/guides/work-on-what-matters/) — for using matrix gaps as a career plan.
- [StaffEng stories](https://staffeng.com/stories/) — how staff and principal engineers describe their own impact.

### Content for specific story types
- ★ [Postmortem culture — Google SRE Book](https://sre.google/sre-book/postmortem-culture/) — failure stories.
- [Blameless PostMortems and a Just Culture — John Allspaw (Etsy)](https://www.etsy.com/codeascraft/blameless-postmortems) — "why it made sense at the time".
- [Thomas–Kilmann conflict modes — Wikipedia](https://en.wikipedia.org/wiki/Thomas%E2%80%93Kilmann_Conflict_Mode_Instrument) — competing, collaborating, compromising, avoiding, accommodating: vocabulary for analysing your conflict stories.
- [2016 letter to shareholders — Jeff Bezos](https://www.aboutamazon.com/news/company-news/2016-letter-to-shareholders) — "disagree and commit" and reversible decisions in the original.
- [Cynefin framework — Wikipedia](https://en.wikipedia.org/wiki/Cynefin_framework) — complicated vs complex problems, for ambiguity stories.
- [Writing an engineering strategy — Will Larson](https://lethain.com/eng-strategies/) — the skeleton for strategy-level ambiguity and influence stories.
- [Staying aligned with authority — StaffEng](https://staffeng.com/guides/staying-aligned-with-authority/) and [Being visible — StaffEng](https://staffeng.com/guides/being-visible/) — influence stories.
- [Create space for others — StaffEng](https://staffeng.com/guides/create-space-for-others/) — delegation and "stepping back" stories.
- ★ [How to be a sponsor when you're a developer — Lara Hogan](https://larahogan.me/blog/how-be-sponsor-when-youre-developer) — mentoring vs sponsorship, with non-managerial examples.
- [The feedback equation — Lara Hogan](https://larahogan.me/blog/feedback-equation/) — structure for feedback stories.
- ★ [Being Glue — Tanya Reilly](https://noidea.dog/glue) — making invisible work visible as evidence.
- [The Architect Elevator — Gregor Hohpe](https://architectelevator.com/) — architect-level "why it mattered".
- [DORA — research and metrics](https://dora.dev/) — numbers for delivery stories.

### Rebuilding facts and maintaining the bank
- ★ [Get your work recognized: write a brag document — Julia Evans](https://jvns.ca/blog/brag-documents/) — the bank's input stream.

### Learning science for the practice plan
- ★ [Which study strategies make the grade? — Association for Psychological Science](https://www.psychologicalscience.org/news/releases/which-study-strategies-make-the-grade.html) — summary of Dunlosky et al. (2013): practice testing and distributed practice rated highest.
- [Roediger & Karpicke (2006), "The Power of Testing Memory" — notes by Andy Matuschak](https://notes.andymatuschak.org/zGfjkW1ociSmSUCLcpbhKjf) — the testing and spacing effects, annotated.
- [Retrieval Practice](https://www.retrievalpractice.org/) — practical guides to retrieval, spacing and interleaving.
- [The Learning Scientists — downloadable materials](https://www.learningscientists.org/downloadable-materials) — one-page summaries of six evidence-based strategies.
- [Bjork Learning and Forgetting Lab — UCLA](https://bjorklab.psych.ucla.edu/research/) — desirable difficulties.
- [Augmenting Long-term Memory — Michael Nielsen](http://augmentingcognition.com/ltm.html) — an engineer-friendly essay on using spaced repetition seriously.
- [Spaced repetition — Gwern Branwen](https://gwern.net/spaced-repetition) — a deep review of the research.
- [Anki](https://apps.ankiweb.net/) — free spaced-repetition software for the question deck.

### Practice and mocks
- ★ [Behavioral peer mock interviews — Exponent Practice](https://www.tryexponent.com/practice/behavioral-mock-interviews) — Pramp's platform since July 2024; a limited number of free peer sessions per month.
- [How to prepare for a mock interview — Exponent](https://www.tryexponent.com/blog/how-to-prepare-for-a-mock-interview/).
- [interviewing.io](https://interviewing.io/) *(paid mocks; free guides)* — for one or two calibrated final mocks.

### The algorithms behind the matrix (for fun, and for the analogy)
- [Set cover problem — Wikipedia](https://en.wikipedia.org/wiki/Set_cover_problem) — the greedy approximation behind story selection.
- [Hungarian algorithm — Wikipedia](https://en.wikipedia.org/wiki/Hungarian_algorithm) — the assignment problem behind loop allocation.

### Books
- *(book)* Tanya Reilly — *The Staff Engineer's Path* (O'Reilly).
- *(book)* Will Larson — *Staff Engineer: Leadership beyond the management track*.
- *(book)* Peter C. Brown, Henry L. Roediger III, Mark A. McDaniel — *Make It Stick: The Science of Successful Learning* — the practice plan's foundations.
- *(book)* Chip Heath & Dan Heath — *Made to Stick* — why concrete, specific stories are remembered.

### Previous modules to revisit
- Module 34 — STAR, the six seniority axes, the follow-up ladder, the probe bank (Appendix D) and the self-scoring rubric (Appendix B).
- Modules 30–33 — the substance of architect stories.
- Modules 13, 25, 28 — incident and failure content.

---
# Quick-recall sheet

**One sentence.** A story bank is an index, not an anthology: 6–8 rich, true, level-appropriate stories, sliced into moments, tagged by competency, cross-checked for coverage, practised by retrieval — so any question becomes decode → select → tell.

**Coverage problem.** Hundreds of phrasings → ~12 competencies. Index on competencies; question lists are test data.

**Data model.** Story (bounded episode, shared context) → Moments (one competency each, own STAR) → Tags (competency + strength 0–3).

**How many.** Loops ask 10–15+ behavioral questions; each core competency needs 2 strong options; ~6–10 stories is the depth limit → **6–8 core + 2–4 reserves**. Never fewer than two failure-capable and two conflict-capable.

**Canonical set.** Core: C1 conflict · C2 failure · C3 ambiguity · C4 technical disagreement · C5 mentoring · C6 influence. Supporting: flagship · delivery · customer · ownership · feedback/wrong · saying no · simplify · raise the bar · learn fast · cost · dive deep · motivation.

**Decode.** Competency · direction (peer/up/down/cross/external) · polarity · level cue. Unsure → ask one clarifying question.

**Excavate.** 25–40 one-line episodes: timeline sweep, prompts (worst week, hardest person, decision I argued against, thing I didn't build), artefacts. Don't judge yet.

**Evidence.** Memory is reconstructive. Rebuild from commits, docs/ADRs, post-mortems, dashboards, tickets, calendar, reviews, colleagues. Decide approximations in advance.

**Screen.** Level · centrality · depth · richness · verifiability · recency · tension. Hard floors on centrality and level. Keep one story only you could tell.

**Hubs and spokes.** 2–3 hubs (stakes, hard decision, fair opponent, setback, measured result, mechanism, lesson) + 4–5 spokes + reserves. Hubs efficient; don't rely on them alone.

**Portfolio.** ≥2 failures + a mixed outcome · technical/people balance · all relationship directions · ≥2 at target level · ≥1 with year-plus consequences · several contexts · domain variety.

**Client and founder careers.** Mine the boundaries with clients; find out what happened after; own-product: decisions and restraint, exact about users and revenue; pair with stories inside organizations.

**Story-type specs.**
- *Flagship:* layered 45-min deep dive; drawable architecture; 2–3 decisions; hardest part; layered results.
- *Conflict:* people/priorities; their best argument; private first; resolution; relationship after; several directions.
- *Technical disagreement:* substance; both options' merits; evidence (spike/benchmark); who decided; ADR; ideally both partly right, or lost and committed.
- *Failure:* right-sized; mine; post-mortem shape; mechanism; evidence. Include a people/alignment failure.
- *Ambiguity:* what exactly was unclear (problem/solution/ownership); cheap uncertainty reduction; structure; course correction.
- *Influence:* no obligation; their interest; named mechanisms; what I changed; when persuasion ran out; how it stuck.
- *Mentoring:* a person; before/after; specific actions; sponsorship; cost accepted.
- *Feedback/wrong:* uncomfortable and right; honest first reaction; verified; changed behavior; evidence.

**Matrix invariants.** Each core column ≥2 strong cells in different stories · each supporting column ≥1 · ≥1 core cell at target level · failure from real failures · no single point of failure · no overloaded story. Selection = small set multicover; greedy works.

**Amazon.** 2 per LP via moments; metrics everywhere; dive-deep anywhere; mine Frugality, Hire and Develop, Earth's Best Employer, Broad Responsibility; Disagree **and** Commit.
**Google.** Emergent leadership both ways (step up, step back); Googleyness; hypotheticals anchored in real moments; committee-readable.
**Meta.** Conflict · growth · ambiguity · results · communication; three conflict moments; E6 scope in every story.
**Microsoft / enterprise / EA / startups.** Respect, integrity, accountability, growth mindset; a column per listed competency; stakeholders, governance, portfolio; ownership and speed with judgment.

**Gaps.** Re-mine → re-angle → reserve → accept + near-miss. Never invent. Persistent gaps = level information.

**Allocation.** Primary + backup per competency from different stories; flagship for deep-dive; no repeats per interviewer; hubs ≤2–3 uses per loop; track between rounds.

**Level audit.** Six axes → passes / re-tell (move framing and alignment into actions) / demote. Never add actions you didn't take.

**Pre-mortem.** Attack attribution, counterfactual, numbers, alternatives, other side, durability, depth, level, consistency, weak spot. Repair by claiming less, more precisely.

**Fact sheet.** Period · role · teams · system · numbers + method · decisions · who disagreed · weak spot · since then · deep-dive detail · anonymisation. One number per fact everywhere; align with CV and references.

**Truth.** Select, compress, reorder, anonymise, approximate — never composite, borrow or inflate. AI to structure and critique, never to invent.

**Adapting.** 70% question: bridge → re-weighted moment → close on the question. No fit: say so → closest real → principled hypothetical → let them choose. Redirect: backup immediately.

**Practice science.** Retrieval > rereading (61% vs 40% at one week). Practice testing + distributed practice = highest utility. Interleave (shuffled questions train decoding). Vary length and starting point. Easy practice ≈ weak practice.

**Four weeks.** W1 excavate, screen, matrix · W2 cards, fact sheets, audit, gaps, pre-mortems · W3 daily shuffled retrieval, recordings, probe drills, first mock · W4 calibrated mocks, deep-dive rehearsal, allocation plan, index card.

**Mocks.** Experiments with a hypothesis. Peers (pressure; play interviewer too) · AI interviewer (follow-ups only; flag implausible; no content) · recordings (timing). Half the time on probes; score immediately; one fix per story.

**Day of.** One-page index (pre-read only) · warm up aloud · log stories per interviewer between rounds · adjust allocation · after-action review.

**Maintenance.** Brag doc every two weeks · quarterly refresh (add, re-score, retire, update "since then") · thin columns become career goals · private and anonymised.

---
# Appendix A — Episode inventory worksheet

```markdown
# Episode inventory — <role / engagement>, <years>

## Timeline sweep
- Systems I owned or shaped:
- Launches / migrations / releases:
- Incidents:
- Decisions I made or influenced:
- Disagreements (technical / personal):
- People I helped / who helped me:
- Things I'd do differently:
- Things that outlived me:
- Numbers I remember:

## Prompt-driven recall
- First big thing / last thing / first time I led:
- Worst week / incident / near-miss / cancelled project:
- Hardest person / decision I argued against / time I escalated / was escalated against:
- Data contradicted intuition / bug in the wrong place / requirement was wrong:
- Something others used / practice I introduced / person who grew:
- Something I deliberately didn't build / rewrite I argued against:
- View I abandoned / feedback that stung and was right:
- Cost cut / customer saved / revenue protected or enabled:
- Outside my job / gaps I filled:

## Episodes (one line each)
| ID | When | Title | Hook | Artefacts to check |
|----|------|-------|------|--------------------|
| E01 | | | | |
```

---
# Appendix B — Story card v2 (with moments index)

Extends Module 34's Appendix A story card. One per core story; reserves can use a shortened version.

```markdown
# Story <ID>: <short title>
Type: hub | spoke | reserve        Target level: senior | staff | principal
Period: <from–to>                  Context (anonymised): <org type and scale>
Shared context (2 sentences, identical every telling):

## Fact sheet  (canonical — see Concept 30)
- Role / title:
- Teams / people / systems / load / money:
- Key numbers (baseline → result, method, timeframe):   ~approximate where marked
- Decisions (options · chosen · why · cost):
- Who disagreed and their best argument:
- Weak spot I'll volunteer:
- Since then:
- Deep-dive detail (4 levels):

## Moments index
| Moment | Headline (one sentence) | Competencies (strength 0–3) | Direction | Polarity |
|--------|-------------------------|-----------------------------|-----------|----------|
| A | | C4:3, S7:2 | peer | success |
| B | | C1:2, C6:3 | cross-team | mixed |

## Per moment (repeat)
### Moment <X>
- Task (how it became mine; goal as outcome):
- Actions (diagnosis · decision with alternatives · people/obstacles · mechanism):
- Result (layered; closes to the goal):
- Reflection (changed mechanism + evidence):
- Probes ready: attribution · alternatives · measurement · other side · differently

## Level audit (Concept 28)
Scope · Ambiguity · Influence · Impact · Horizon · Leverage → passes | re-tell | demote

## Pre-mortem notes (Concept 29)
- Weakest attack and my repair:
```

---
# Appendix C — Coverage matrix template

```markdown
| Story \ Competency | C1 Conf | C2 Fail | C3 Ambg | C4 Tech | C5 Ment | C6 Infl | S1 Flag | S2 Dlvr | S3 Cust | S4 Ownr | S5 Fdbk | S6 SayN | S7 Simp | S8 Bar | S9 Lrn | S10 Cost | S11 Deep | <company cols…> |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| S1 | | | | | | | | | | | | | | | | | | |
| … | | | | | | | | | | | | | | | | | | |
| **Strong cells (≥2)** | | | | | | | | | | | | | | | | | | |

Invariants:
[ ] Each core column ≥ 2 strong cells, in different stories
[ ] Each supporting column ≥ 1 strong cell
[ ] Each core column has ≥ 1 strong cell at target level
[ ] Failure / feedback cells come from genuinely negative or mixed stories
[ ] No story is the only strong option for > 1 core column
[ ] No story planned for > 3 moments in one loop

Portfolio tally:
Polarity  success __ · mixed __ · failure __
Substance technical __ · people __ · both __
Direction peer __ · upward __ · downward __ · cross-team __ · external __
Horizon   ≥ 1 year consequences: __
Context   employers / clients / products: __
```

**Company column sets to paste in:**

- **Amazon:** CustObs · Ownership · Invent&Simplify · AreRight · Learn&Curious · Hire&Develop · HighestStd · ThinkBig · BiasAction · Frugality · EarnTrust · DiveDeep · Backbone&Commit · DeliverResults · EarthsBestEmployer · BroadResponsibility
- **Google:** GCA · RRK · Leadership(step up) · Leadership(step back) · Googleyness
- **Meta:** Conflict · Growth · Ambiguity · Results · Communication · (Scope)
- **Microsoft:** Respect · Integrity · Accountability · GrowthMindset · Collaboration · Customer
- **Enterprise / EA:** the competencies listed in the job advert, plus Stakeholders · Governance · Portfolio/Roadmap · Vendor

---
# Appendix D — Question decoder bank

Use as test data for the matrix and as the front of your spaced-repetition cards. *Dir* = direction; *Pol* = polarity (+ success, − failure, ± either).

| # | Question | Competency | Dir | Pol |
|---|---|---|---|---|
| 1 | Tell me about a time you had a conflict with a co-worker | C1 | peer | ± |
| 2 | Tell me about a disagreement with your manager | C1 | up | ± |
| 3 | Describe a time you had to give someone difficult feedback | C1 (+C5) | down | ± |
| 4 | Tell me about someone you found difficult to work with | C1 | any | ± |
| 5 | A time you pushed back on a senior stakeholder | C1 / S6 | up | ± |
| 6 | A time you disagreed with a product manager on priorities | C1 | cross | ± |
| 7 | Tell me about a time you resolved a conflict between others | C1 (staff) | cross | + |
| 8 | Tell me about a time you failed | C2 | — | − |
| 9 | Your biggest professional mistake | C2 | — | − |
| 10 | A project that didn't go as planned | C2 / S2 | — | − |
| 11 | A decision you regret | C2 / S5 | — | − |
| 12 | A time you missed a deadline | C2 / S2 | — | − |
| 13 | A time you had to work with incomplete information | C3 | — | ± |
| 14 | A time requirements were unclear | C3 | — | ± |
| 15 | A problem nobody owned that you took on | C3 / S4 | — | + |
| 16 | How do you approach a project with no clear direction? | C3 (hyp. + anchor) | — | ± |
| 17 | A time you had to change course midway | C3 / S5 | — | ± |
| 18 | Tell me about a technical decision you disagreed with | C4 | peer/up | ± |
| 19 | A time you defended a design under criticism | C4 | — | ± |
| 20 | A time you convinced your team to change technical approach | C4 / C6 | peer | + |
| 21 | A technical decision you made that turned out wrong | C4 / C2 | — | − |
| 22 | How do you decide between two technologies? | C4 (hyp. + anchor) | — | ± |
| 23 | Tell me about someone you mentored | C5 | down | + |
| 24 | How have you helped your team get better? | C5 / S8 | down | + |
| 25 | A time you onboarded someone | C5 | down | + |
| 26 | A time you helped someone who was struggling | C5 | down | ± |
| 27 | A time you let someone else lead | C5 / Google step-back | peer/down | + |
| 28 | A time you influenced a decision without authority | C6 | cross | + |
| 29 | A change you drove across teams | C6 | cross | + |
| 30 | How did you get buy-in for an unpopular idea? | C6 | any | ± |
| 31 | A time you changed your manager's mind | C6 / C1 | up | + |
| 32 | A time you needed something from a team that didn't want to give it | C6 / C1 | cross | ± |
| 33 | Your most significant project | S1 | — | + |
| 34 | The most technically complex thing you've done | S1 / S11 | — | + |
| 35 | Walk me through a project end to end | S1 | — | + |
| 36 | The project you're proudest of | S1 | — | + |
| 37 | A time you delivered under a tight deadline | S2 | — | + |
| 38 | How do you handle competing priorities? | S2 / S6 | — | ± |
| 39 | A time you had to cut scope | S2 / S6 | — | ± |
| 40 | A time you went above and beyond for a customer | S3 | external | + |
| 41 | A time customer feedback changed your plan | S3 / S5 | external | + |
| 42 | How do you understand what users actually need? | S3 (hyp. + anchor) | external | ± |
| 43 | Something you did outside your job description | S4 | — | + |
| 44 | A time you noticed a problem before anyone else | S4 / S11 | — | + |
| 45 | The most difficult feedback you've received | S5 | up | ± |
| 46 | A time you changed your mind | S5 | — | ± |
| 47 | A time you were wrong | S5 | — | − |
| 48 | How have you grown in the last two years? | S5 | — | + |
| 49 | A time you said no to a stakeholder | S6 | up/cross | ± |
| 50 | A time you had to choose between two important things | S6 | — | ± |
| 51 | A time you simplified something complex | S7 | — | + |
| 52 | Something you invented or a novel approach you took | S7 | — | + |
| 53 | A time you refused to lower the bar | S8 | any | + |
| 54 | A time you pushed back on cutting corners | S8 / C1 | up/cross | ± |
| 55 | A time you learned something new quickly | S9 | — | + |
| 56 | A time you joined an unfamiliar domain | S9 / C3 | — | ± |
| 57 | A time you did more with less | S10 | — | + |
| 58 | A time you reduced cost | S10 | — | + |
| 59 | The hardest bug you've fixed | S11 | — | + |
| 60 | A time data contradicted your intuition | S11 / S5 | — | ± |
| 61 | A time you disagreed and then committed | C1/C4 + Commit | up | ± |
| 62 | A time you made a decision quickly without full data | C3 / Bias for Action | — | ± |
| 63 | A time you thought big | S1 / C6 | — | + |
| 64 | A time you earned the trust of a sceptical group | C6 / C1 | cross | + |
| 65 | A time you improved the working environment for your team | Earth's Best Employer / S4 | down/peer | + |
| 66 | A time you considered the wider impact of a technical decision | Broad Responsibility / S8 | external | ± |
| 67 | A time you improved how your team hires | Hire & Develop / C5 | — | + |
| 68 | A time you raised an ethical or privacy concern | Broad Responsibility / S8 | up | ± |
| 69 | How do you use AI tools in your work? | S9 / S8 (Module 34, Q14) | — | ± |
| 70 | Imagine your team disagrees with a leadership decision. What do you do? | C1 / C6 (hyp. + anchor) | up | ± |
| 71 | Imagine a teammate is consistently underperforming. What do you do? | C1 / C5 (hyp. + anchor) | peer/down | ± |
| 72 | Tell me about yourself | Narrative (headlines) | — | + |
| 73 | Why are you leaving / why now? | Narrative (motivation) | — | + |
| 74 | Why this company / this role? | Narrative (fit) | — | + |

---
# Appendix E — Four-week practice calendar (printable)

```text
WEEK 1  Excavate & screen
  Mon  Timeline sweep: role 1–2              Tue  Timeline sweep: role 3+; prompt recall
  Wed  Artefact sweep                         Thu  Screen & score → shortlist
  Fri  Hubs, spokes, moments                  Sat/Sun  Coverage matrix v1; gap list

WEEK 2  Cards & facts
  Mon–Wed  Story cards (core 6–8)             Thu  Fact sheets; message ex-colleagues
  Fri  Level audit                            Sat/Sun  Fill gaps; company columns; pre-mortem hubs

WEEK 3  Speak & vary          (daily: 15-min shuffled retrieval drill, 5 questions aloud)
  Mon  Record story A (2 & 5 min)             Tue  Probe drill: story B
  Wed  Record story C                         Thu  Probe drill: story D
  Fri  Record story E                         Sat/Sun  Mock #1 (peer or AI), score, fix

WEEK 4  Mocks & calibrate     (daily: retrieval drill incl. hypotheticals & compound Qs)
  Mon  Mock #2 calibrated to target company   Tue  Fixes from mock #2
  Wed  Flagship deep-dive rehearsal + C4 diagram from memory
  Thu  Mock #3 (different partner/format)     Fri  Allocation plan; index card
  Day before  Light drill only; read index; sleep
```

---
# Appendix F — Day-of index card and tracking sheet

```text
INDEX CARD (pre-read only — not used during interviews)

STORY  HEADLINE                                   NUMBERS          MOMENTS
S1     ________________________________________   ______________   A__ B__ C__ D__ E__
S2     ________________________________________   ______________   A__ B__
S3     ________________________________________   ______________   A__ B__ C__
S4     ________________________________________   ______________   A__ B__
S5     ________________________________________   ______________   A__ B__
S6     ________________________________________   ______________   A__
S7     ________________________________________   ______________   A__ B__
S8     ________________________________________   ______________   A__
R1–R4  ________________________________________

PRIMARIES  Conf __ | Fail __ | Ambg __ | Tech __ | Ment __ | Infl __ | Flag __ | Fdbk __
BACKUPS    Conf __ | Fail __ | Ambg __ | Tech __ | Ment __ | Infl __ | Flag __ | Fdbk __


TRACKING SHEET (fill between rounds)

Round / interviewer role | Questions asked              | Story·moment used | Said anything not on fact sheet?
_________________________|______________________________|___________________|_________________________________
_________________________|______________________________|___________________|_________________________________

AFTER-ACTION REVIEW
Questions I didn't expect:
Where probes hit hardest:
Competency with no ready story:
One change to the bank per weakness:
```

---

*Next: **Module 36 — What senior-level coding rounds actually test**: given that your DS&A is already strong, how coding rounds at senior and staff level are really scored — code quality, edge cases, testing, API design and communication over algorithmic trivia — with .NET/C#-specific idioms that read as senior, and the new AI-enabled coding formats.*
