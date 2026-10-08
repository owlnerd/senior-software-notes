# Module 36 — What Senior-Level Coding Rounds Actually Test
*Phase 9: Coding Round, Right-Sized · Senior/Architect Interview Prep for .NET & C#*

> **State of practice verified on October 8, 2026.** The fundamentals in this module — what a coding round measures and what reads as senior — have been stable for a decade. What has changed fast since late 2025 is the *format* of coding rounds, mostly because of AI. The facts that calibrate the module:
>
> - **The rubric is still four dimensions.** Rubrics collated from Google, Amazon, Apple and Netflix by the Tech Interview Handbook (an ex-Meta staff engineer's guide, updated August 2026) score **communication, problem solving, technical competency and testing**, rolled into bands from *strong hire* to *strong no hire*. Exact names differ by company; the dimensions don't.
> - **Amazon** says SDE III coding questions need **syntactically correct code, no pseudocode**, and that the main evaluation criteria are code that is **scalable, robust and well-tested** — including checking edge cases and making sure no bad input slips through. Its listed objectives: efficiency, reliability, robustness, portability, maintainability, readability. The SDE III loop is **five 55-minute interviews**.
> - **Meta** began rolling out an **AI-enabled coding interview in October 2025**. It replaces one of the two onsite coding rounds (you still get one classic round with no AI); it is **60 minutes** in a **multi-file CoderPad project** with an AI chat panel and a choice of models; **C# is one of the supported languages** (with Java, C++, Python, Kotlin and TypeScript). Practitioner reports describe three phases — **fix a bug → implement the core feature → optimise for larger inputs** — scored on the **same four competencies as the classic round**: problem solving, code quality, verification and communication.
> - **Google** is piloting a **Gemini-assisted "code comprehension" round** — read, debug and optimise an existing codebase, with interviewers assessing *AI fluency* (prompting, validating output, debugging). Reports (May–July 2026) describe it as limited to **junior and mid-level roles on selected US teams** from the second half of 2026; the standard coding rounds remain. **Outside a confirmed pilot, assume no AI at Google.**
> - **Canva** has required candidates for backend, frontend and ML roles to use AI tools (Copilot, Cursor, Claude) since **June 2025**, replacing its algorithms-and-data-structures screen with an **AI-Assisted Coding** interview built on ambiguous, realistic problems.
> - **Shopify** (per candidate reports) runs **two AI coding rounds** — one at screening, one onsite — in **your own IDE with your own AI tools**, starting from an **empty GitHub repo**, on a problem that grows with follow-ups; design and tests are scored explicitly. **LinkedIn** runs **one AI-enabled and one traditional** coding round; its follow-ups push into concurrency and production-readiness.
> - **Stripe**-style practical loops (bug squash in an unfamiliar codebase, API integration, multi-part implementation) remain the best-known example of *work-sample* coding rounds; **code-review rounds** appear at several companies (including Google engineering-manager loops, where candidates can choose coding or code review).
> - **AI policy is per company and per round.** Anthropic's published guidance (last updated July 2025) is the cleanest statement of the conservative norm: live interviews are **"all you"** unless the interviewer says otherwise, and take-homes are done without AI unless permitted. Third-party "policy trackers" are frequently wrong or stale — **ask the recruiter for the policy of each round.**
> - **.NET today:** **.NET 10 (LTS, November 2025) with C# 14** is the current production release. **.NET 11 (STS) with C# 15** — headline feature: **union types** — is in preview, with GA expected at .NET Conf, **November 10–12, 2026**. Don't use preview features in an interview, and **check the interview environment's C# version** (CoderPad shows it under the pad's *Info* tab): online editors often lag the current SDK.
> - **Testing ecosystem:** **xUnit v3**, **MSTest 4**, **NUnit 4** and **TUnit** all run on **Microsoft.Testing.Platform**. **FluentAssertions moved to a commercial licence from v8 (January 2025)**; the community fork **AwesomeAssertions** stays Apache-2.0. Worth knowing because "which assertion library and why" is now a reasonable interview aside.
>
> Company formats are "verified on this date" and change quickly; the principles are stable.

## Orientation

Here is the sentence to carry through the whole module: **a senior coding round is a 45-minute work sample, not an algorithm quiz — the algorithm is the *floor*, and you are levelled on everything built on top of it: how you turn an ambiguous prompt into a precise contract, how you design the small API you're writing, how you find edge cases before they find you, how you verify your own code, how you narrate trade-offs, and — increasingly — how you direct and check an AI that writes some of the code for you.**

The curriculum entry reads: *What senior-level coding rounds actually test, given your DS&A is already strong — code quality, edge cases, testing, API design over algorithmic trivia.* Module 35 ended by promising exactly that, plus the .NET/C# idioms that read as senior and the new AI-enabled formats.

**Why this module is "right-sized."** Your data-structures-and-algorithms foundation is strong, so this module deliberately spends almost nothing on algorithm content. That's not because algorithms stopped mattering — at Meta, Amazon and Google the classic round is still there — but because, for a candidate with strong DS&A, **the marginal hour of preparation buys far more score on the other dimensions.** Strong algorithmists most often fail senior coding rounds for reasons that have nothing to do with algorithms: they code silently, skip clarification, ship code with no tests, leave edge cases to the interviewer, over-engineer, or (new in 2026) can't explain the AI's output they pasted.

How this connects to earlier modules:

- **Modules 1–2** (rubrics, IC vs architect evaluation) — the coding round is one row in the scoring packet; this module zooms into that row.
- **Modules 14–19** (CLR and memory, async and concurrency, modern C#, performance, ASP.NET Core, EF Core) — the *knowledge* behind the C# idioms in Part D. This module turns that knowledge into what you type in 45 minutes.
- **Module 25** (Polly) and **Module 13** (reliability) — the material behind the "integration round" playbook: timeouts, retries, idempotency.
- **Module 30** (design documents) — the same discipline of options-and-consequences, compressed into a two-sentence spoken trade-off.
- **Module 34** (STAR calibrated to seniority) — "narrate thinking, not just doing" applies even more in a coding round, where the interviewer can see the doing but not the thinking.
- **Module 37** (worked system-design problems) — several coding-round problems here (rate limiter, LRU cache, key-value store) reappear there at system scale.
- **Module 38** (mock interviews and self-scoring) — Appendix E's rubric feeds it.

Why it matters in interviews:

1. **The coding round is often the gate.** Phone screens are usually coding. A senior candidate with excellent design and behavioral skills can be filtered before anyone hears about them.
2. **Coding feedback is written in level language.** "Solved it, but needed prompting for edge cases and didn't test" is a mid-level write-up even when the solution was optimal. At senior level, *how* you solved it is the evidence.
3. **The format is in flux.** In one loop you may face a classic no-AI round and an AI-enabled round back to back. The skills overlap, but the behaviour that scores differs — and the most common 2026 failure is applying the wrong behaviour to the round.
4. **Architects are not exempt.** Many architect and staff loops still include coding, sometimes *only* one round — which makes it higher-variance, not lower-stakes. A weak coding signal from an architect candidate invites the question every hiring committee fears: *can this person still build?*

This module has six jobs:

1. **Explain what is actually measured** — the round as a work sample, the four (five) rubric dimensions, how seniority shifts their weights, how scoring and write-ups work, and the taxonomy of 2026 coding formats.
2. **Teach the craft inside the 45 minutes** — clarification, planning, time budgeting, decomposition, edge cases, testing, debugging, communication, getting unstuck, and follow-ups.
3. **Define code quality that reads as senior** — proportionate quality, API design, error handling, extensibility without over-engineering, and concurrency.
4. **Make you fluent in senior C#** — environment realities, collections, modern language features used with judgment, the idioms that read junior, and testing in .NET.
5. **Give a playbook per format** — classic algorithmic, practical multi-part, low-level design, code review, debugging, integration, take-home, and AI-enabled.
6. **Give you a preparation plan** — diagnosis, a light four-week plan that keeps your DS&A floor while building the other dimensions, and a day-of protocol.

Seven framings to carry through:

1. **The algorithm is the floor, not the score.** Being able to find the optimal approach gets you into the "hire" conversation. Everything else decides which level you're hired at.
2. **Make your thinking observable.** The interviewer scores what they can write down. Thinking you don't say is thinking they can't credit.
3. **Contract first, code second.** At senior level, the first ten minutes — clarifying, defining the signature, naming the edge cases — are where most of the level signal is produced.
4. **Proportionate quality.** Senior code is as simple as the problem allows and exactly as robust as the context requires. Over-engineering is a mistake, not a bonus.
5. **You own verification.** Never announce "done" before you've tested your own code. The interviewer finding your bug is worth far less than you finding it.
6. **Design for the follow-up you can see coming.** Most senior rounds add a requirement. Code that absorbs it without a rewrite is the clearest evidence of design skill.
7. **AI changes who types, not who's accountable.** In AI-enabled rounds, you are scored on your judgment: what you delegate, how you check it, and whether you can explain every line you kept.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | The round as a work sample | It predicts how you'll write code on the job — the algorithm is only part of that |
| 2 | The rubric dimensions | Communication · problem solving · technical competency · testing (+ verification) |
| 3 | How seniority shifts the weights | Baseline algorithm assumed; quality, judgment and verification differentiate |
| 4 | Scoring mechanics and write-ups | Interviewers write evidence; red flags weigh more than polish |
| 5 | The 2026 format taxonomy | Nine formats; identify which one you're in |
| 6 | Clarification as contract | Ask the questions that change the design |
| 7 | Planning out loud | State approach, complexity and plan before typing |
| 8 | The time budget | Working solution first; checkpoints every ten minutes |
| 9 | Decomposition and naming | Code that reads like your explanation |
| 10 | Edge cases, systematically | Partition inputs; walk the boundaries |
| 11 | Testing inside the round | Choose cases deliberately; dry-run like a debugger; write real tests when you can |
| 12 | Debugging under observation | Hypothesis, evidence, bisect — out loud |
| 13 | Communication | Narrate decisions, not keystrokes |
| 14 | Getting unstuck and taking hints | Hints are data; take them gracefully |
| 15 | Follow-ups and production thinking | Scale, failure, concurrency, operability |
| 16 | Proportionate code quality | As simple as possible, as robust as needed |
| 17 | API design in the small | Types, names, nullability, return-vs-throw, immutability |
| 18 | Errors and input validation | Fail fast at boundaries; choose exceptions or results deliberately |
| 19 | Extensibility without over-engineering | Seams where change is likely; YAGNI elsewhere |
| 20 | Concurrency in coding rounds | Correct first; the right primitive; known traps |
| 21 | C# in the interview environment | Version lag, harness shape, what you must type from memory |
| 22 | Collections fluency | The right BCL type and its exact semantics |
| 23 | Modern C# used with judgment | Records, patterns, collection expressions, LINQ — when they help |
| 24 | C# that reads junior | The tells interviewers notice |
| 25 | Testing in .NET | xUnit v3 theories, fakes, TimeProvider, the library landscape |
| 26 | The classic algorithmic round | Your floor; spend the surplus on quality |
| 27 | Practical multi-part rounds | Design for the next part; stay green |
| 28 | Low-level design / machine coding | Entities, responsibilities, seams — and working code |
| 29 | The code review round | Prioritise by severity; comment like a colleague |
| 30 | Debugging in an unfamiliar codebase | Orient, reproduce, root-cause, regression-test |
| 31 | The integration / API round | Docs, HTTP, JSON, timeouts, retries, idempotency |
| 32 | Take-home assignments | Scope, tests, README of decisions, timebox |
| 33 | AI-enabled rounds | You drive, the AI types; verify everything |
| 34 | Integrity and AI policy | Ask the rules per round; never covert |
| 35 | Diagnosing your gap | Score yourself on the rubric, not on problems solved |
| 36 | The four-week plan | Keep the floor; drill the differentiators |
| 37 | Day-of protocol | Environment check, opening script, after-action review |

---
# Part A — What is actually being measured

## Concept 1 — The coding round as a work sample

Start from the question every interview design has to answer: **what does this 45 minutes predict?**

A coding round is a **work sample test**: a small, standardised slice of the actual job, performed under observation. Personnel-selection research has long ranked work samples among the better predictors of job performance — far better than unstructured conversation — *provided* the sample resembles the work. That proviso explains almost everything about how coding rounds have evolved:

- The classic algorithmic puzzle was a **cheap, standardised proxy** for "can this person reason precisely and turn reasoning into correct code?" It was never a sample of the daily work itself. It persisted because it was cheap to run, easy to calibrate across thousands of interviewers, and hard to bluff.
- At junior level the proxy works reasonably well: the job *is* mostly turning specified behaviour into correct code.
- At senior level the job is different: clarifying vague requirements, designing small APIs other people will call, anticipating failure, reviewing others' code, debugging systems you didn't write, and — in 2026 — directing AI tools and checking their output. **The more senior the role, the less the classic puzzle samples the job**, so interviewers lean on everything *around* the puzzle to get signal.
- In 2025–26, AI tools broke the proxy outright: Canva reported that AI assistants solved its classic questions **in seconds, often without follow-up prompts**. Companies responded in two ways — hardening classic rounds (in-person or AI-off environments) and building **AI-enabled work samples** that look more like the job.

**What your DS&A strength buys you — and what it doesn't.**

| Your strength buys you | It does not buy you |
|---|---|
| Recognising the problem pattern in seconds | Credit for the clarifying questions you didn't ask |
| Choosing an optimal approach without hints | Credit for edge cases the interviewer had to point out |
| Accurate complexity analysis | Credit for code that works but reads badly |
| Time — the most valuable resource in the round | Credit for testing you didn't do |
| Confidence under follow-ups about optimisation | Credit for reasoning you kept in your head |

The middle row is the key: **time.** A candidate who recognises the pattern in two minutes rather than fifteen has, in effect, a thirteen-minute budget to spend on the dimensions that set level. The single most common way strong algorithmists under-level themselves is spending that surplus on *speed* — finishing early and saying "done" — rather than on *evidence*: a clean API, explicit edge-case handling, a test pass, a trade-off discussion, a follow-up anticipated.

**The interviewer's actual question.** Behind every rubric, a senior coding interviewer is answering: *"If this person opened a pull request on my team tomorrow, what would it look like — and what would reviewing it cost me?"* Hold that question in mind and most of this module follows from it.

**Interview-grade sentence:** *"I treat a coding round as a sample of how I'd work on a real change: the algorithm has to be right, but I spend the time it saves me on the things a reviewer would care about — a clear contract, explicit edge cases, readable code, and tests I've actually run."*

---

## Concept 2 — The rubric dimensions

Company rubrics use different labels, but they decompose into the same dimensions. Learn them as *what the interviewer is writing down*, because every behaviour in the rest of this module is aimed at one of them.

### The four classic dimensions

| Dimension | What the interviewer is checking | Typical evidence they write down |
|---|---|---|
| **Communication** | Do you clarify, explain your approach and trade-offs, and keep talking while coding? Can they follow your thinking without effort? | "Asked about input bounds and duplicates up front; explained why a heap beats sorting here" |
| **Problem solving** | Do you understand the problem, approach it systematically, reach an optimised solution, analyse complexity correctly — without major hints? Do you compare alternatives? | "Started with an O(n²) baseline, identified the bottleneck, moved to O(n log n) unprompted" |
| **Technical competency** (coding) | Do you translate the approach into working, clean, idiomatic code with good abstractions and naming, few bugs, no syntax errors? Do you know the language? | "Clean helper decomposition; used `PriorityQueue<TElement,TPriority>` correctly; idiomatic C#" |
| **Testing** (verification) | Do you test typical and corner cases, find and fix your own bugs, and verify systematically (e.g. stepping through state like a debugger)? | "Traced the empty and single-element cases; caught an off-by-one before I did" |

The Tech Interview Handbook's collated rubric — the most widely used public version — scores each from *strong hire* to *strong no hire*, and is explicit about one thing worth memorising: a *strong no hire* for testing is a candidate who **"announced they are done"** without testing typical cases or spotting glaring bugs.

### The fifth dimension, made explicit in 2026: verification and AI collaboration

AI-enabled rounds didn't invent new dimensions so much as **re-weight one of them**. Meta's AI-enabled round is reported to use the *same four competencies* as its classic round — but renames "testing" as **verification** and makes it central: do you run code frequently, check the AI's output before moving on, test edge cases? Google's pilot calls the cluster **AI fluency**: prompting, output validation, debugging.

The useful mental model: **verification is testing generalised to code you didn't write.** In a classic round, the untrusted code is your own first draft. In an AI-enabled round, it's also the model's output. Same skill, more surface.

### How the dimensions combine

Two scoring styles exist — a score per dimension summed or averaged, or a single holistic score informed by all of them — but in practice they behave similarly:

- **The dimensions are not independent.** Poor communication depresses problem-solving scores, because the interviewer can't see the problem solving. Weak testing depresses technical-competency scores, because unfound bugs are counted against the code.
- **They are not fungible at the extremes.** A brilliant algorithm doesn't offset a strong-no-hire on testing. Most hiring committees read a severe weakness on one dimension as a red flag, not as one low number among four (Concept 4).
- **Level shifts which dimension decides** (Concept 3).

**Interview-grade sentence:** *"I know coding rounds are scored on communication, problem solving, code quality and testing — and that in AI-enabled rounds testing becomes verification of any code, mine or the model's — so I make sure each of those is visible: I clarify and narrate, I compare approaches, I write code a reviewer would accept, and I never say I'm done before I've tested it."*

---

## Concept 3 — How seniority shifts the weights

The same four dimensions are used at every level. What changes is **what counts as meeting the bar** on each — and which dimension tends to decide the outcome.

### The shift, derived

At junior level, the open question is *"can they produce correct code at all?"* — so problem solving and technical competency dominate, and communication and testing are scored leniently.

At senior level, the interviewer **assumes** you can produce correct code for a medium problem; that's what years of experience are supposed to guarantee. So:

1. **Reaching the optimal solution becomes a threshold, not a differentiator.** Failing to reach it is still costly. Reaching it earns relatively little extra.
2. **Unprompted behaviours become the differentiator.** The same edge case is worth much more if you raise it than if the interviewer has to.
3. **Judgment becomes visible and scored.** Choosing the simpler of two correct designs; knowing when an abstraction is premature; explaining why you'd do something differently in production.
4. **Follow-ups probe deeper and wider.** "How would this behave with ten million items? With concurrent callers? If the input arrives as a stream?" — questions that test whether your design generalises.

### What "meets the bar" looks like by level

| Dimension | Mid-level (L4 / E4 / SDE II) | Senior (L5 / E5 / SDE III) | Staff+ (L6 / E6 / Principal) — when there is coding |
|---|---|---|---|
| **Communication** | Explains approach when asked | Drives the conversation: clarifies, proposes, narrates trade-offs unprompted | Frames the problem, states assumptions, makes the interviewer a collaborator; communicates priorities under time pressure |
| **Problem solving** | Reaches a working solution, possibly with hints | Reaches the optimal solution without major hints; compares alternatives | Same, *plus* recognises which constraints actually matter and when a "worse" algorithm is the better engineering choice |
| **Code quality** | Working code; some clumsiness | Clean, idiomatic, well-decomposed; sensible API and error handling | Same, *plus* the API shape anticipates the obvious extensions; nothing gratuitous |
| **Testing / verification** | Tests when prompted | Tests unprompted; finds own bugs; enumerates edge cases systematically | Same, *plus* chooses what *not* to test given time, and explains the testing strategy for production |
| **Follow-ups** | Handles optimisation follow-ups | Handles scale and concurrency follow-ups with concrete changes | Connects to operability, failure modes, rollout and the system around the code |

**A consequence that surprises strong algorithmists:** at senior level, a candidate who solves a problem optimally in 25 minutes and stops can score *below* a candidate who reaches the same solution in 35 minutes having clarified the contract, named the edge cases, tested and discussed production concerns. The first candidate produced less evidence.

### The staff/architect special case

Many staff and architect loops have **one** coding round or none. With one round there's no averaging: a weak performance is the only coding evidence in the packet. Three implications:

- **Treat it as a must-not-fail round**, not a formality. The bar is usually "solid senior coding", not "staff-level coding" — what's being checked is that you *can still build*.
- **Don't over-design to show seniority.** Turning a 45-minute coding problem into an architecture lecture reads as avoidance. Show seniority through judgment *inside* the code: the contract, the error handling, the seam where the follow-up will land.
- **Expect production follow-ups** (concurrency, observability, rollout). These are where staff-level signal lives in a coding round.

**Interview-grade sentence:** *"At senior level I assume reaching the optimal solution is the threshold, not the score — what differentiates is what I do unprompted: clarifying the contract, raising edge cases before the interviewer does, testing my own code, choosing the simpler design and explaining why, and handling scale and concurrency follow-ups with concrete changes."*

---

## Concept 4 — Scoring mechanics and the write-up

To influence a score you need to know how it's produced. In most structured loops:

1. **The interviewer takes notes during the round** — often close to verbatim for what you say, plus the code itself.
2. **They write feedback afterwards**, usually within a day: a summary, evidence per dimension, a level judgment, and a hire recommendation.
3. **A debrief or hiring committee reads all the feedback together**, often without having met you.

### Three consequences

**1. Evidence must be writable.** "Seemed smart" isn't feedback. "Identified without prompting that the input could contain duplicates and asked whether to dedupe" is. Every behaviour in Part B is designed to produce sentences the interviewer can write down. **If you think it, say it** — unspoken reasoning cannot be written up.

**2. Red flags weigh more than strengths.** Interviewers and committees are, sensibly, loss-averse: a false-positive hire is far costlier than a false-negative rejection. Some behaviours read as red flags regardless of the rest of the round:

| Red flag | Why it's weighted heavily |
|---|---|
| Declaring "done" without testing; interviewer finds an obvious bug | Predicts shipping untested code |
| Arguing with the interviewer about a correct observation | Predicts being hard to review |
| Silent coding for long stretches | Predicts poor collaboration; also removes all evidence |
| Unable to explain code you wrote (or pasted from an AI) | Predicts not owning your output |
| Pseudocode when real code was required | Can't tell if you can actually write it (Amazon says so explicitly) |
| Ignoring a stated requirement | Predicts not reading specs |
| Covert AI use in a no-AI round | An integrity failure; usually terminal |

**3. Level is inferred from the write-up's language.** Committees calibrate on phrases. Compare:

> *"Candidate solved the problem with the optimal approach. Needed a nudge on the empty-input case. Did not write tests. Code was readable."* → reads **mid-level / lean hire**.

> *"Candidate clarified input bounds and duplicate handling before coding, proposed two approaches with trade-offs and chose the simpler one given n ≤ 10⁵, wrote clean decomposed code, enumerated and traced five edge cases unprompted including the empty input, found and fixed an off-by-one during their own trace, and discussed how they'd make it thread-safe."* → reads **senior / strong hire**.

Same algorithm. Different evidence.

### Interviewer variance, and what you can do about it

Interviewers vary — in strictness, in which dimension they weight, in how much they help. You can't control that, but you can **reduce variance in your own evidence**: if you reliably produce each dimension's evidence in every round, a strict interviewer still has something to write and a lenient one has more. That reliability is what Appendix A's protocol is for.

**Interview-grade sentence:** *"I assume the interviewer can only credit what they can write down and that red flags outweigh polish, so I make my reasoning audible, test before I say I'm done, never argue with a correct observation, and make sure I can explain every line on the screen."*

---

## Concept 5 — The 2026 format taxonomy

"Coding round" now covers at least nine formats. Each samples a different part of the job, and **the behaviour that scores differs by format.** The first skill is recognising which one you're in — ideally before the day, by asking the recruiter (Concept 37).

| # | Format | What it looks like | What it samples | Where you'll see it (examples) | Playbook |
|---|---|---|---|---|---|
| 1 | **Classic algorithmic** | One or two problems in 45 min, blank editor, no AI | Precise reasoning → correct code | Meta (one classic round), Amazon, Google, most big-tech phone screens | Concept 26 |
| 2 | **Practical / multi-part** | One problem in 3–4 escalating parts (parse → compute → extend → handle errors) | Building and extending working software | Stripe-style loops; many product companies | Concept 27 |
| 3 | **Low-level design / machine coding** | Design classes for a small system (parking lot, rate limiter, LRU cache) and implement the core | Object modelling, API design, extensibility | Amazon (low-level design is in its SDE III prep material), Uber, Shopify's AI rounds in practice | Concept 28 |
| 4 | **Code review** | Review a PR or snippet; leave comments; discuss priorities | Judgment, quality bar, communication | Several companies; Google EM loops (choice of coding or code review) | Concept 29 |
| 5 | **Debugging / bug squash** | Find and fix bugs in an unfamiliar, realistic codebase | Code reading, hypothesis-driven debugging | Stripe; Meta's AI round phase 1; Google's comprehension pilot | Concept 30 |
| 6 | **Integration / API** | Call a real or mock API from docs, transform JSON, handle failures | Working with third-party systems | Stripe-style loops | Concept 31 |
| 7 | **Refactoring** | Improve messy, working code without changing behaviour | Taste, safety, incremental change | Some practical loops | Concepts 16, 30 |
| 8 | **Take-home** | A few hours on a small project; review conversation afterwards | Unpressured work quality and decisions | Many mid-size companies and startups | Concept 32 |
| 9 | **AI-enabled** | Multi-file project or empty repo with an AI assistant; often phased | Directing and verifying AI; code comprehension | Meta, Canva, Shopify, LinkedIn; Google pilot | Concept 33 |

**Notice how the formats differ on three axes** — they predict what scores:

| Axis | Low end | High end | What shifts |
|---|---|---|---|
| **Who wrote the starting code?** | Blank editor (1, 3, 8) | Someone else's codebase (4, 5, 7, 9-Meta) | Code *reading* skill matters more than code writing |
| **How open is the problem?** | Precisely specified (1) | Ambiguous and evolving (2, 3, 9-Shopify/Canva) | Clarification and design-for-change matter more |
| **Who types?** | You (1–8) | You and an AI (9) | Verification and explanation matter more |

**Why this matters for preparation:** a candidate who practises only format 1 is practising the one format where their DS&A strength already gives them the most margin. Your preparation (Part F) should be weighted toward the formats where you have the least practice — usually 4, 5 and 9.

**Interview-grade sentence (to the recruiter):** *"Could you tell me the format of each coding round — classic algorithmic, practical, design, code review or debugging — whether AI tools are allowed in any of them, and which environment and language versions you use?"*

---
# Part B — The craft inside the 45 minutes

Part B uses one running example so the techniques are concrete:

> *"Given a list of log lines, return the k most frequent error codes."*

It looks trivial to a strong algorithmist — hash map plus heap or bucket sort — which is exactly why it's useful: at senior level the algorithm is not where this round is won.

## Concept 6 — Clarification as contract

**Clarifying questions are not politeness; they are requirements engineering in miniature.** The output of clarification is a **contract**: inputs, outputs, invariants, error behaviour and constraints, agreed with the interviewer before you write code.

### The contract template

Ask — or state as an assumption and confirm — along six axes:

| Axis | Questions | For the running example |
|---|---|---|
| **Input shape** | Type? Format? Already parsed? Where from? | "Are lines raw strings like `2026-10-08T10:00Z ERROR E1042 ...`, or already parsed?" |
| **Size and distribution** | n? Distinct values? Skew? Fits in memory? Streaming? | "How many lines — thousands or billions? Roughly how many distinct codes?" |
| **Validity** | Can input be malformed, null, empty? What should happen? | "What if a line has no code, or a malformed one — skip, count separately, or fail?" |
| **Output semantics** | Order? Ties? Fewer than k? Exact or approximate? | "If two codes tie, any order or a deterministic tie-break? If there are fewer than k codes, return all?" |
| **Parameters** | Ranges and invalid values | "Can k be 0, negative, or larger than the number of codes?" |
| **Context** | Called once or repeatedly? Concurrently? Latency/memory budget? | "Is this a one-off batch or an API called frequently on a sliding window?" |

**Senior clarifying is selective.** Asking all thirty possible questions wastes five minutes and signals that you can't prioritise. Ask the ones **whose answers would change your design** and state assumptions for the rest:

> *"Two questions that change the design: roughly how many lines — does it fit in memory? — and do ties need a deterministic order? For the rest I'll assume: lines are raw strings, malformed lines are skipped and counted, k ≤ 0 is an argument error, and if there are fewer than k codes I return them all. Tell me if any of those is wrong."*

That answer does four things at once: it shows you know which questions matter (n and determinism drive the algorithm and the output), it makes the remaining decisions explicit and checkable, it produces writable evidence, and it costs about thirty seconds.

### Why "state an assumption" beats "ask"

- It **shows a default** — the interviewer learns what you'd do, not just that you noticed.
- It **keeps momentum** — the interviewer can say "fine" instead of making a decision for you.
- It mirrors real work: senior engineers propose; they don't wait to be told.

**The one trap:** don't *assume away* the interesting part. If the problem says "logs", the interviewer may be planning a streaming follow-up. "I'll assume it fits in memory for the first version, and we can talk about streaming afterwards" is fine; "I'll assume it's small" without saying so is not.

### Write the contract down

In the editor, as a comment block, before any code:

```csharp
// Contract
// Input:  IEnumerable<string> lines (raw log lines), int k
// Output: up to k (code, count) pairs, highest count first; ties broken by code, ordinal
// Errors: k <= 0 -> ArgumentOutOfRangeException; null lines -> ArgumentNullException
// Lines without a parsable code are skipped (and counted, for diagnostics)
// Assume: fits in memory for v1; streaming is a follow-up
```

It costs a minute and pays back three ways: the interviewer can correct it early, you can test against it at the end, and it is literally written evidence of clarification on the screen.

**Interview-grade sentence:** *"I clarify by writing a contract — input shape, size, validity, output semantics, parameter ranges and context — and I only ask the questions whose answers would change my design; for everything else I state my default as an assumption and let the interviewer correct it."*

---

## Concept 7 — Planning out loud

After the contract and before typing, **state the approach in a few sentences, with complexity, and get a nod.** This is where problem-solving evidence is produced most efficiently, and where most expensive mistakes are caught.

### The planning script

1. **Baseline:** the simplest correct approach and its cost. *"Count with a dictionary, sort all codes by count — O(n + m log m) for n lines and m distinct codes."*
2. **Bottleneck:** what dominates under the stated constraints. *"m is small relative to n here, so the sort is cheap; the cost is reading and parsing n lines."*
3. **Options:** the alternatives and when they win. *"A size-k min-heap makes the selection O(m log k), which matters if m is huge and k small; bucket sort by count is O(n + m) but allocates up to n buckets."*
4. **Choice and why:** *"Given m is in the thousands, I'll sort — it's the simplest to get right and the asymptotic gain from the heap is irrelevant at this size. If m were in the millions I'd use the heap."*
5. **Plan of the code:** *"Three pieces: a parser that extracts the code from a line, a counting pass, and a selection step. I'll write them as separate methods."*

**Choosing the simpler algorithm on purpose is a senior signal**, as long as you say *why* and name when you'd switch. It shows you treat complexity as a means, not a trophy. Strong algorithmists often reach for the cleverest solution by reflex — which spends time and adds bug surface for no gain at the stated size.

### When to skip ahead

If the problem is clearly a known pattern and the interviewer is time-conscious, compress: *"This is top-k frequency — count, then select. With m small I'll sort; heap if m is large. Sound good?"* Fifteen seconds; same evidence.

### When to start with brute force in code

Write the brute-force version first only when (a) the optimal approach is genuinely risky to get right, or (b) the interviewer asks for it. Otherwise, *describing* the baseline and coding the chosen approach is the better use of time.

**Interview-grade sentence:** *"Before I type I state a baseline with its complexity, name the bottleneck under the constraints we agreed, give the alternatives and when each wins, choose — often the simpler one, with the condition under which I'd switch — and outline how I'll split the code; then I check the interviewer is happy with the plan."*

---

## Concept 8 — The time budget

A 45-minute round is really about 35–40 working minutes after introductions and your questions at the end. Budget them explicitly — in your head, with a glance at the clock at each checkpoint.

### A default budget for a single medium problem

```text
00–05  Clarify, write the contract                         (Concept 6)
05–10  Plan out loud: baseline → choice → code outline       (Concept 7)
10–25  Write the code, narrating decisions                  (Concepts 9, 13)
25–32  Test: enumerate cases, trace, fix                    (Concepts 10–11)
32–40  Follow-ups: optimisation, scale, concurrency         (Concept 15)
40–45  Your questions
```

For two problems in 45 minutes (common at Meta's classic round), halve everything and compress clarification to the design-changing questions only.

For a 60-minute AI-enabled round, expect five or so minutes of platform orientation, and plan around phases rather than a fixed split (Concept 33).

### Three rules

1. **Working first, then better.** A correct, slightly clumsy solution at minute 25 beats an elegant one that's still broken at minute 40. Refactor after it works, if time allows — and say that's what you're doing.
2. **Checkpoint every ten minutes.** Ask yourself: *am I where the budget says? If not, what do I cut?* Usually: compress planning, simplify the API, or skip a non-essential feature — and say so: *"I'm going to skip input validation for now and come back to it if we have time."*
3. **Protect the testing slot.** It is the most commonly sacrificed slot and the one whose absence most often produces red-flag feedback ("announced done; obvious bug"). If you're behind, cut code polish, not testing.

### Time sinks to recognise early

| Sink | Fix |
|---|---|
| Parsing input elegantly | Hard-code or simplify parsing; say you would make it robust in production |
| Fighting the environment (imports, compiler errors) | Know the harness by heart (Concept 21) |
| Perfecting naming before the code works | Name decently once; rename at the end |
| Debugging by staring | Switch to a systematic trace or print (Concept 12) |
| Over-long clarification | Ask only design-changing questions; assume the rest |

**Interview-grade sentence:** *"I budget the round — clarify and plan in the first ten minutes, a working solution by about minute twenty-five, then testing and follow-ups — check the clock every ten minutes, cut polish before I cut testing, and say out loud what I'm deferring so the interviewer knows it's a choice."*

---

## Concept 9 — Decomposition and naming

Interviewers read your code while you write it. **Code that reads like your explanation** lets them follow without effort, which helps every dimension at once.

### Decompose by responsibility, not by line count

For the running example, the explanation was "parse, count, select". The code should have exactly that shape:

```csharp
public static IReadOnlyList<(string Code, int Count)> TopErrorCodes(IEnumerable<string> lines, int k)
{
    ArgumentNullException.ThrowIfNull(lines);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(k);

    var counts = CountCodes(lines);
    return SelectTop(counts, k);
}
```

Then `CountCodes`, `TryParseCode` and `SelectTop` as small methods. The top-level method is now a readable summary; each helper can be explained, tested and changed independently — which is exactly what the follow-ups will need ("now the format changes", "now it's streaming", "now we need the top k per hour").

**How much decomposition?** Enough that each method does one thing you can name. A 40-line method in an interview is usually two or three methods waiting to be extracted; ten one-line methods is fragmentation. As a rough guide: the top-level method fits on screen and reads as a sentence.

### Names

- **Intention-revealing:** `counts`, `TryParseCode`, `SelectTop` — not `d`, `helper`, `process`.
- **Domain language from the problem:** if the interviewer said "error codes", call them codes, not keys.
- **Loop variables and short-lived locals can be short** (`i`, `line`); fields, parameters and methods should not be.
- **Booleans read as predicates:** `isMalformed`, `hasCapacity`.
- **Rename when meaning changes.** If `result` has become "the codes we've already seen", rename it `seen`.

### Structure that helps the follow-up

Make the **variable part** of the problem a parameter or a seam, *if* it's cheap. Here, the parsing rule is the most likely thing to change, so a separate `TryParseCode` method is enough of a seam — no interface needed (Concept 19).

**Interview-grade sentence:** *"I decompose along the lines of my explanation — here parse, count and select — so the top-level method reads as a summary, each helper can be tested and changed alone, and the follow-up usually lands in exactly one place; and I use names from the problem's own vocabulary."*

---

## Concept 10 — Edge cases, systematically

"Think about edge cases" is advice everyone has heard and few apply systematically. Two techniques from software testing make it mechanical: **equivalence partitioning** and **boundary value analysis**.

### Equivalence partitioning

Divide each input into classes where the code should behave the same way; pick one representative per class. For the running example:

| Input | Partitions |
|---|---|
| `lines` | null · empty · one line · many lines · contains malformed lines · all malformed |
| codes | one distinct · fewer than k distinct · exactly k · more than k · ties at the k-th position |
| `k` | negative · zero · one · equal to distinct count · larger than distinct count |
| line content | well-formed · missing code · extra whitespace · lowercase/uppercase variants (is `e1042` the same code?) · very long line |

### Boundary value analysis

Bugs cluster at the edges of partitions: **0, 1, n−1, n, n+1**; first and last elements; the k-th position; empty and full. For any index or count, test the boundary and one step either side.

### The general edge-case checklist

Keep this in your head (Appendix B has the long version):

| Family | Cases |
|---|---|
| **Size** | empty, single element, two elements, very large |
| **Values** | zero, negative, minimum/maximum (`int.MaxValue` — overflow!), duplicates, all equal |
| **Order** | sorted, reverse-sorted, already-processed |
| **Structure** | null, cycles (graphs, linked lists), disconnected components, self-references |
| **Text** | empty string, whitespace, case, Unicode (surrogate pairs, combining marks), culture-sensitive comparison |
| **Time** | time zones, DST transitions, leap years, clock going backwards, equal timestamps |
| **Concurrency** | simultaneous calls, re-entrancy, cancellation mid-way |
| **Contract** | invalid parameters, partially valid input, the "fewer than requested" case |

### Say them, then decide

Edge-case handling has three parts, and each is evidence:

1. **Enumerate** — out loud, early: *"Edge cases I see: empty input, k larger than the number of codes, ties at position k, and malformed lines."*
2. **Decide the behaviour** — tie each to the contract: *"Empty input returns an empty list; k too large returns all; ties broken by code; malformed lines skipped."*
3. **Verify** — after coding, trace at least the riskiest ones (Concept 11).

Raising an edge case the interviewer was about to ask about is one of the cheapest strong signals in the round.

### Integer overflow: the C#-specific trap

`int` arithmetic in C# is **unchecked by default** — overflow wraps silently. Classic interview bugs: `(lo + hi) / 2` on large indices (use `lo + (hi - lo) / 2`), summing counts into an `int`, multiplying two `int`s before widening. Mention `long` or a `checked` context when values can be large; it's a small sentence with high signal.

**Interview-grade sentence:** *"I find edge cases systematically: partition each input into classes that should behave the same, then test the boundaries of each class — empty, one, the k-th position, just past the end — plus the families that bite in practice like overflow, ties, Unicode and time; I say them early, tie each to the contract, and trace the risky ones after coding."*

---

## Concept 11 — Testing inside the round

There are three levels of testing available in a coding round. Use the highest the environment allows.

### Level 1 — Trace like a debugger

When you can't run code (whiteboard, some phone screens), **trace by hand**: pick a small input, step through line by line, and write the state of key variables as comments or aloud. This is the "systematic verification" the rubric names.

Rules that make tracing effective:

- **Use a small, non-trivial input** — three to five elements, chosen to exercise a branch (a tie, a duplicate, a boundary).
- **Trace the code on screen, not the code in your head.** The bug is the difference between them.
- **Track state explicitly:** `i=2, counts={E1:2, E2:1}, heap=[E2:1]`.

### Level 2 — Run it

When you can run code (CoderPad, most online editors, every AI-enabled round), **run early and often**: after the first meaningful piece works, not only at the end. A quick harness in `Main` is enough:

```csharp
static void Check<T>(string name, T expected, T actual)
{
    var ok = EqualityComparer<T>.Default.Equals(expected, actual);
    Console.WriteLine($"{(ok ? "PASS" : "FAIL")} {name}: expected {expected}, got {actual}");
}
```

(For collections, compare with `SequenceEqual` or format them to strings first — `EqualityComparer<T>.Default` compares lists by reference.)

### Level 3 — Write real tests

If the environment has a test framework (CoderPad's C# environment has historically supported NUnit; your own IDE in a Shopify-style round supports whatever you set up), write a handful of **parameterised tests**. Shopify interviewers have reportedly reacted well to candidates proposing parameterised tests over repetitive individual ones; in xUnit that's `[Theory]` with `[InlineData]` (Concept 25).

### Which cases, in which order

1. **The example from the prompt** — confirms basic understanding.
2. **The riskiest edge case** — the one most likely to break *your* implementation (ties at k; empty).
3. **A boundary** — k equal to the number of codes.
4. **An invalid input** — k = 0, null — to show the guard clauses work.
5. **A larger input** — if runnable, to sanity-check performance.

Five tests, chosen deliberately and explained in one sentence each, are worth more than twenty generated without thought.

### Testing as a design check

If something is hard to test — because it reads the clock, the file system, or a static — that's information about the design. Say so: *"This reads `DateTime.UtcNow` directly, which makes it hard to test; in real code I'd inject `TimeProvider`."* (Concept 25.)

**Interview-grade sentence:** *"I test at the highest level the environment allows — trace by hand on a small, branch-exercising input if I can't run, run early and often if I can, and write a few parameterised tests if there's a framework — choosing cases deliberately: the prompt's example, the riskiest edge case for my implementation, a boundary, an invalid input and a larger input."*

---

## Concept 12 — Debugging under observation

Everyone writes bugs in interviews. **How you find them is evidence**; how you behave while finding them is evidence too.

### The method: hypothesis → evidence → narrowing

1. **Read the symptom precisely.** The compiler error or exception message, the failing test's expected vs actual. Out loud: *"Expected `[E1, E2]`, got `[E2, E1]` — so counting works but ordering is wrong."*
2. **Form a hypothesis about the mechanism**, not a guess about the line: *"The comparer must be sorting ascending, or the tie-break is inverted."*
3. **Get evidence cheaply:** re-read the specific code, print an intermediate value, or trace one step.
4. **Narrow the search space** — bisect: is the bug before or after this point? Check the intermediate state at the midpoint.
5. **Fix the cause, not the symptom.** Then **re-run the full set**, not just the failing case — a fix can break something else.

### What to avoid

- **Random edits** ("let me try `<=` instead"). Interviewers recognise flailing instantly.
- **Silent staring.** Narrate what you're checking, even when you don't know yet.
- **Blaming the environment** before you've ruled out your code.
- **Defensiveness** if the interviewer points at the bug. *"Good catch — yes, that's the off-by-one"* and fix it.

### Interviewer hints during debugging

If the interviewer asks "what happens when the list is empty?", that's almost always a hint that something breaks there. Trace that case immediately rather than asserting it works.

**Interview-grade sentence:** *"When something fails I read the symptom precisely, form a hypothesis about the mechanism rather than guessing a line, get cheap evidence — a trace or a printed intermediate value — bisect if needed, fix the cause, and rerun everything; and I narrate the whole time, because how I debug is part of what's being assessed."*

---

## Concept 13 — Communication

Communication is the dimension most under-practised by strong technical candidates — and the one that most often turns a correct solution into a mid-level write-up.

### What to say: decisions, not keystrokes

| Low-value narration | High-value narration |
|---|---|
| "Now I'm writing a for loop…" | "I'm iterating once to count, so this pass is O(n)." |
| "Let me add an if…" | "This guard handles the malformed line — we agreed to skip those." |
| "OK, a dictionary…" | "Dictionary keyed by code with ordinal comparison, since codes are case-sensitive identifiers." |
| (silence for three minutes) | "I'm thinking about whether the tie-break needs a stable sort — it doesn't, because I'm sorting by a total order on (count, code)." |

Narrate **why**, not **what**: the interviewer can see the what.

### The cadence

- **Before a block:** one sentence of intent. *"Next, selection: sort by count descending, then code."*
- **During:** the occasional decision or trade-off.
- **After:** one sentence of status. *"That's the core done; let me test it before anything else."*
- **When stuck:** say where you are and what you're considering. Silence is the worst choice; "I'm considering two options…" is fine.

### Treat the interviewer as a collaborator

Senior engineers pair. Behave like it:

- **Check in at decision points:** *"I'm planning to sort rather than use a heap — any objection?"*
- **Listen to every interviewer sentence as possible signal.** "Interesting choice" often means "reconsider". "Are you sure about that line?" means "there's a bug there".
- **Accept corrections gracefully and immediately.** Arguing with a correct observation is a red flag (Concept 4). If you think the interviewer is wrong, check your reasoning out loud and politely: *"Let me trace it to be sure — for [1,1,2]… actually I think it's fine because… does that match what you were thinking?"*

### Communication in AI-enabled rounds

The hardest new skill: two conversations at once — with the AI and with the interviewer. Practitioners report candidates finding this harder than the code. The rule: **the interviewer must hear every intent before you prompt, and every judgment after you read the response** (Concept 33).

**Interview-grade sentence:** *"I narrate decisions rather than keystrokes — intent before each block, trade-offs as I make them, status after — I check in with the interviewer at decision points, treat their questions as signal, and accept a correct correction immediately; silence is the one thing I avoid, because unspoken reasoning can't be scored."*

---

## Concept 14 — Getting unstuck and taking hints

Even strong candidates get stuck — on an unfamiliar problem, a subtle bug, or a follow-up that breaks the design. **Being stuck is not the problem; how you get unstuck is the evidence.**

### Unsticking techniques (in order)

1. **Restate the problem and constraints** aloud — half the time the missing piece is a constraint you dropped.
2. **Solve a smaller instance by hand** — n = 3 — and watch what *you* do; that's the algorithm.
3. **Relax a constraint** — "if the input were sorted…", "if there were no duplicates…" — then ask what it costs to restore it.
4. **Change the representation** — graph instead of grid, intervals as events, counts instead of elements.
5. **Work backwards** from the desired output.
6. **Fall back to a correct baseline** and optimise from it — a working O(n²) beats a broken O(n log n).

### Hints

Interviewers give hints because they want you to succeed and because they need to see more of your skills. A hint costs some problem-solving credit, but **ignoring or fighting a hint costs much more.**

- **Take it explicitly:** *"That's a good point — if I think of each meeting as two events, start and end, then sorting them gives me…"*
- **Show you understood it,** don't just apply it: explain why it helps.
- **Don't apologise repeatedly.** Once, briefly, if at all — then move on.

### When the interviewer is silent

Some interviewers deliberately don't help. Keep narrating, state your current best approach, and ask a targeted question if genuinely blocked: *"I'm deciding between X and Y; is there a constraint I'm missing that favours one?"*

**Interview-grade sentence:** *"When I'm stuck I restate the constraints, solve a tiny instance by hand, relax a constraint or change the representation, and fall back to a correct baseline if I need to — out loud throughout; and when the interviewer gives a hint I take it explicitly and show I understand why it helps, because fighting a hint costs far more than taking one."*

---

## Concept 15 — Follow-ups and production thinking

At senior level, the follow-up questions are often where the level decision is actually made. LinkedIn candidates, for instance, report the initial problem being familiar and the real evaluation happening in concurrency and production-readiness follow-ups.

### The follow-up families — and the senior answer shape

| Family | Typical question | What a senior answer contains |
|---|---|---|
| **Scale** | "What if there are a billion lines?" | Where it breaks (memory for counts? time?); concrete change (streaming read, external sort, sharded counting with merge); approximate alternatives (Count-Min Sketch + heap, Space-Saving) and their error trade-offs |
| **Streaming / online** | "What if lines arrive continuously and we need the top k for the last hour?" | Time-bucketed counts, sliding window eviction, update cost, what "exact" means in a window |
| **Concurrency** | "Multiple threads call this" / "make it thread-safe" | Shared state identified; primitive chosen with reason (Concept 20); contention discussed |
| **Failure** | "What if the source is unavailable / a line is corrupt?" | Contract for partial failure; retries and idempotency where relevant (Module 25) |
| **Changing requirements** | "Now codes come in two formats" | The seam that absorbs it (Concept 19) — ideally one method changes |
| **Production readiness** | "How would you ship this?" | Tests, logging/metrics of skipped lines, configuration, performance check, rollout — briefly, proportionately |
| **Optimisation** | "Can you do better?" | Exact bound you're at, the theoretical bound, whether the improvement matters at the agreed size |

### Answer with concrete changes, not vocabulary

*"I'd use a distributed approach"* is noise. *"If workers split the input by file, each one sees every code, so merging their local top-k lists is wrong — a code can be (k+1)-th on every worker and first overall. So I'd repartition by hash of the code: each code's count then lives on exactly one worker, and merging the per-worker top-k lists with a k-way heap is exact"* is a senior answer: specific, with the correctness trap named and avoided.

### Know when the follow-up is a design discussion

If the follow-up is big ("make it a service"), switch register: two or three sentences of design, then ask whether they want you to code any part. Don't start rewriting the whole solution unless asked.

**Interview-grade sentence:** *"I treat follow-ups as where the level is decided: I say exactly where the current solution breaks, propose a concrete change — streaming, sharding with a correct merge, an approximate sketch with its error bound, a specific concurrency primitive — name its correctness caveats, and say whether the improvement matters at the size we agreed."*

---
# Part C — Code quality that reads as senior

## Concept 16 — Proportionate code quality

"Write clean code" is not a usable instruction in a 45-minute round, because clean code has costs and the round has a budget. The senior skill is **proportionality**: code exactly as simple as the problem allows and exactly as robust as the context requires.

### The quality hierarchy

When time forces trade-offs, spend it in this order — it matches what reviewers weight on real pull requests:

1. **Correctness** — including the edge cases in the contract.
2. **Clarity** — someone else can read it once and understand it: structure, names, no cleverness for its own sake.
3. **Robustness at the boundaries** — validation of inputs from callers; explicit handling of the agreed failure cases.
4. **Appropriate efficiency** — meets the agreed constraints; no accidental quadratic behaviour.
5. **Extensibility where change is predictable** — a seam where the follow-up will land (Concept 19).
6. **Polish** — consistent style, final renames, removing dead code.

Junior candidates often invert the order — polishing a design while the code is still wrong. Over-confident candidates skip 2 and 3. Both read badly.

### Over-engineering is a mistake, not a bonus

Interviewers see two failure modes in roughly equal measure: **under-engineered** (one long method, magic numbers, no validation) and **over-engineered** (an interface, a factory and a strategy for a function that's called once). The second is often committed by experienced engineers trying to *show* seniority — and it costs time, adds bug surface, and signals poor judgment about cost.

| Situation | Under-engineered | Proportionate | Over-engineered |
|---|---|---|---|
| Parsing a code from a line | Inline `Split(' ')[2]` with no checks | `TryParseCode` method with a clear failure path | `ICodeParser` interface + factory + registry |
| Ordering results | Inline lambda with a sign bug | Named comparison or `OrderByDescending(...).ThenBy(...)` | A custom `IComparer<T>` class hierarchy |
| Config value (k) | Magic number | Parameter with a guard clause | Options pattern with validation attributes |
| LRU cache follow-up "pluggable eviction" | `if (policy == "lru")` strings | `IEvictionPolicy` *when asked* | Policy interface built before anyone asked |

**The test for any abstraction in a round:** *can I name the second implementation, and is it likely within the scope of this conversation?* If not, don't build it — mention it: *"If we needed other eviction policies I'd extract an interface here; for now one is enough."* That sentence earns the design credit without the cost.

### Production gaps: name them, don't fill them

You will deliberately leave things out — logging, configuration, full validation, cancellation. **Say what and why:**

> *"In production I'd add a metric for skipped lines and make the parsing rule configurable; I'm leaving those out to keep this focused."*

That converts a gap into evidence of judgment.

**Interview-grade sentence:** *"I aim for proportionate quality: correctness first, then clarity, then robustness at the boundaries, efficiency that meets the agreed constraints, and a seam only where the next change is predictable; I don't build abstractions whose second implementation I can't name — I mention them instead — and I say out loud which production concerns I'm leaving out."*

---

## Concept 17 — API design in the small

Most coding rounds ask you to write a function or a small class. Its **signature and public surface** are an API, and senior interviewers read them as evidence of how you'd design the APIs your team will call for years. This is the clearest place where "senior" is visible in a coding round.

### Design the surface first

Before implementing, write the signature (or class outline) and say why:

```csharp
public sealed record CodeCount(string Code, int Count);

public static IReadOnlyList<CodeCount> TopErrorCodes(IEnumerable<string> lines, int k)
```

Six decisions are visible in two lines — each a sentence you can say:

| Decision | Choice | Why |
|---|---|---|
| **Input type** | `IEnumerable<string>` | Accepts arrays, lists and lazily-read files (`File.ReadLines`) — prepares for the streaming follow-up |
| **Output type** | `IReadOnlyList<CodeCount>` | Caller gets order and count but can't mutate our result; not a lazy `IEnumerable` that could re-execute |
| **Result element** | A `record` | Named fields beat `(string, int)` tuples in a public API; value equality makes tests trivial |
| **Static vs instance** | Static function | No state needed; becomes a class when the follow-up adds state (sliding window) |
| **`sealed`** | Yes | Not designed for inheritance; sealing is the safe default |
| **Parameter order** | Data first, options after | Conventional; reads naturally |

### Principles that read as senior

1. **Make illegal states unrepresentable — within reason.** Prefer types that can't hold invalid values: an enum over a string mode, a `record` with validation in the constructor, a `TimeSpan` over an `int seconds`. *"I'll take a `TimeSpan` rather than an int of milliseconds — it removes a unit bug."*
2. **Be honest about nullability.** With nullable reference types enabled, `string?` means "can be null" and `string` means "won't be". Annotate return values truthfully; a `FindUser` that can fail returns `User?` or uses the Try pattern.
3. **Accept the most general input, return the most specific *useful* output** (within reason): `IEnumerable<T>` or `IReadOnlyCollection<T>` in, `IReadOnlyList<T>` or a concrete immutable type out.
4. **Avoid boolean parameters** that change behaviour (`Process(data, true)`): a named enum or two methods read better.
5. **Choose return-vs-throw deliberately** (Concept 18): expected failures use `bool TryX(..., out T)` or a result type; programming errors throw.
6. **Minimise the public surface.** Helpers are `private`. Fields are `private readonly`. Expose behaviour, not data structures — an LRU cache exposes `TryGet` and `Set`, never its internal `LinkedList`.
7. **Async APIs take a `CancellationToken`** as the last parameter and return `Task<T>`/`ValueTask<T>` — and never block (Module 15).
8. **Consistent with the BCL.** `TryGetValue`, `Count`, `Add` throwing on duplicates vs `TryAdd` returning `false` — follow these conventions so callers' intuitions transfer.

### Class APIs: invariants at the boundary

For a class, say its **invariants** and enforce them in the constructor and public methods:

```csharp
public sealed class LruCache<TKey, TValue> where TKey : notnull
{
    public LruCache(int capacity)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(capacity);
        Capacity = capacity;
    }

    public int Capacity { get; }
    public int Count => _map.Count;

    public bool TryGet(TKey key, [MaybeNullWhen(false)] out TValue value) { /* … */ }
    public void Set(TKey key, TValue value) { /* … */ }
}
```

*"The invariants are: capacity is positive, Count never exceeds Capacity, and the map and the recency list always contain the same keys. The constructor enforces the first; every public method preserves the other two."* Saying invariants out loud is rare and distinctive — and it gives you a precise checklist when testing.

(`[MaybeNullWhen(false)]` lives in `System.Diagnostics.CodeAnalysis`; it tells the nullability analysis that `value` may be default when the method returns false — exactly how `Dictionary.TryGetValue` is annotated.)

**Interview-grade sentence:** *"I design the signature before the body and explain it — general input such as `IEnumerable<T>`, a specific read-only output, a record rather than a tuple, honest nullability, return-or-throw chosen deliberately, a minimal public surface, BCL-style naming — and for a class I state its invariants out loud and enforce them at every public boundary."*

---

## Concept 18 — Errors and input validation

Error handling is where interview code most often reveals production habits — good or bad.

### Two kinds of failure

| Kind | Examples | Mechanism in C# |
|---|---|---|
| **Programming errors** (caller broke the contract) | null argument, `k <= 0`, index out of range, calling `Dequeue` on an empty queue | Throw: `ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException` — fail fast |
| **Expected outcomes** (the world is as it is) | key not found, line not parsable, user input invalid, remote call failed | `bool TryX(out T)`, a nullable return, or a result type; exceptions only for exceptional cases at system boundaries |

The guiding rule is the .NET design guideline: **don't use exceptions for normal control flow.** A parser that expects malformed lines shouldn't throw and catch per line — that's slow and obscures intent. It should return `false`.

### Guard clauses with the modern throw helpers

.NET 6+ and 8+ added static throw helpers that are shorter, consistent, and produce correct parameter names via `CallerArgumentExpression`:

```csharp
ArgumentNullException.ThrowIfNull(lines);                      // .NET 6+
ArgumentException.ThrowIfNullOrWhiteSpace(name);               // .NET 8+
ArgumentOutOfRangeException.ThrowIfNegativeOrZero(k);          // .NET 8+
ArgumentOutOfRangeException.ThrowIfGreaterThan(k, MaxK);       // .NET 8+
ObjectDisposedException.ThrowIf(_disposed, this);              // .NET 7+
```

Using these fluently is a small but noticeable "current C#" signal — the kind of thing a .NET interviewer notices in the first minute.

### Validation belongs at boundaries

Validate where untrusted data enters (public methods, parsers, API handlers) — **once**. Internal private helpers can assume their inputs were validated. Re-validating everywhere is noise; validating nowhere is a bug.

### The Try pattern, correctly

```csharp
private static bool TryParseCode(string line, [NotNullWhen(true)] out string? code)
{
    // Format: "<timestamp> <LEVEL> <CODE> <message…>"
    code = null;
    if (string.IsNullOrWhiteSpace(line)) return false;

    var parts = line.Split(' ', 4, StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
    if (parts.Length < 3 || !parts[1].Equals("ERROR", StringComparison.Ordinal)) return false;

    code = parts[2];
    return true;
}
```

Points to say: the `out` is assigned on every path; `[NotNullWhen(true)]` lets callers use `code` without a null check after a successful call; comparison is **ordinal** (identifiers, not human text — Concept 24); `Split` with a count limits work on long lines.

### Result types vs exceptions — know the debate

Some teams use result types (`Result<T, Error>` via libraries or hand-rolled, or — from C# 15 in preview — union types) for expected failures across layers. It's a legitimate design; in an interview, **pick one approach, be consistent, and be able to say why.** Don't introduce a result-type library into a 45-minute round unless the problem is about error propagation.

### Never swallow

`catch (Exception) { }` — or `catch { return null; }` — is among the most-noticed red flags in .NET code review. If you catch, catch **specific** exceptions, at a boundary where you can do something meaningful (retry, translate, log and rethrow with `throw;` to preserve the stack).

**Interview-grade sentence:** *"I separate programming errors, which throw fast with the .NET throw helpers at public boundaries, from expected outcomes like an unparsable line or a missing key, which return through the Try pattern or a result type; I validate once where untrusted data enters, never use exceptions for normal control flow, and never swallow — I catch specific exceptions only where I can act on them."*

---

## Concept 19 — Extensibility without over-engineering

Many senior rounds are built around a **follow-up that changes the requirements**: "now support multiple robots", "now evict by frequency", "now the log format changes". Shopify's AI rounds, for example, are reported to grow the problem with follow-ups and to score whether the architecture *absorbs* new requirements without a rewrite. Your code's response to that follow-up is the most direct evidence of design skill the round produces.

### Predict the axis of change

Ask: *what is the most likely next requirement?* Usually it's visible in the problem:

| Problem | Likely axis of change | Cheap seam now |
|---|---|---|
| Parse logs | Format | Parsing in one method (`TryParseCode`) |
| LRU cache | Eviction policy; thread safety; TTL | Eviction logic in one private method; time from `TimeProvider` |
| Rate limiter | Algorithm (fixed window → sliding → token bucket); per-key limits | Algorithm behind a small interface *if* the prompt hints at several |
| Robot on a grid | More robots; obstacles; commands | Command handling as a switch over a command type; grid as its own class |
| Order book / ledger | New transaction types | Pattern-matched handling per type |

**Seams, from cheapest to most expensive:** a private method → a parameter (`Func<T, bool>`, a comparer) → a small interface with one implementation → a pluggable strategy with configuration. **Choose the cheapest seam that would make the likely change a one-place edit.**

### SOLID, proportionately

The SOLID principles are useful vocabulary in these rounds, if applied as heuristics rather than rules:

- **Single responsibility** → decomposition by reason to change (Concept 9).
- **Open/closed** → the follow-up adds a new case without editing the existing ones. In modern C# this is often a `switch` expression over a closed set of types — or an interface when the set is open.
- **Liskov** → don't create a subtype that breaks the base's contract (a `ReadOnlyCache` that throws on `Set` is the classic smell).
- **Interface segregation** → small interfaces (`IClock`-like, `IEvictionPolicy`) rather than a do-everything service.
- **Dependency inversion** → depend on `TimeProvider`, not `DateTime.UtcNow`; inject the dependency through the constructor.

Say them only when they explain a decision: *"I'm passing in `TimeProvider` so the TTL logic is testable"* beats *"I'm applying dependency inversion."*

### When the follow-up arrives

1. **Restate it** and say where it lands: *"Frequency-based eviction — that only changes `EvictOne`. Everything else stays."*
2. **If your seam fits, use it** — this is the payoff moment; let the interviewer see it.
3. **If it doesn't, refactor deliberately**: *"This needs eviction to be pluggable; I'll extract `IEvictionPolicy` with `OnAccess`, `OnInsert` and `SelectVictim`."* A clean refactor under observation is also strong evidence.
4. **Keep it running** — re-run the tests after the change.

**Interview-grade sentence:** *"I predict the most likely next requirement from the problem itself and put the cheapest seam there — usually just a separate method or a parameter, an interface only when I can name the second implementation — so when the follow-up comes it's a one-place change; and if it doesn't fit, I refactor deliberately and rerun the tests."*

---

## Concept 20 — Concurrency in coding rounds

Concurrency follow-ups are increasingly common at senior level — "make it thread-safe" is the classic LRU and rate-limiter follow-up — and .NET has specific traps interviewers know. Module 15 covered the theory; here is what you need *in the round*.

### Step 1: identify shared mutable state and invariants

Before choosing a primitive, say **what** must be protected: *"The map and the linked list must change together — a reader must never see a key in the map whose node is no longer in the list. So the unit of atomicity is the whole `Get` or `Set`, not individual data structures."*

That sentence is the senior part. Choosing a primitive without it is cargo culting.

### Step 2: choose the primitive

| Need | Primitive | Notes to say |
|---|---|---|
| A compound invariant across several structures, short critical sections | `lock` (C# 13+: can use the new `System.Threading.Lock` type) | Simplest correct choice; uncontended locks are cheap; never `await` inside a `lock` (it won't compile) |
| Independent per-key updates in a map | `ConcurrentDictionary<TKey, TValue>` | Atomic per operation only — compound read-modify-write still needs care |
| Async code that must serialise access | `SemaphoreSlim(1, 1)` with `WaitAsync` / `Release` in `finally` | The async-compatible mutex |
| Many readers, rare writers, no mutation on read | `ReaderWriterLockSlim` | **Not** for LRU — an LRU "read" mutates recency order, so every `Get` is a write |
| Producer/consumer pipelines | `Channel<T>` (bounded) | Back-pressure via `BoundedChannelFullMode` (Module 15) |
| Lazy one-time initialisation | `Lazy<T>` (default mode is thread-safe) | Avoids double-checked locking bugs |
| Simple counters | `Interlocked.Increment` / `Add` / `CompareExchange` | Lock-free for single variables only |

### Step 3: know the traps

1. **`ConcurrentDictionary.GetOrAdd(key, factory)` can run the factory more than once** under contention; only one result is stored, but side effects (expensive calls, I/O) may happen twice. Fix: store `Lazy<TValue>` values — `GetOrAdd(key, k => new Lazy<TValue>(() => Create(k))).Value`.
2. **Check-then-act races:** `if (!dict.ContainsKey(k)) dict[k] = v;` is not atomic even on a `ConcurrentDictionary`. Use `TryAdd`, `AddOrUpdate` or `GetOrAdd`.
3. **`AddOrUpdate`'s update delegate may also run more than once** — it must be pure.
4. **Locking on `this`, a `Type` or a string** — lock on a private dedicated object.
5. **Blocking on async** — `.Result`, `.Wait()` — under load starves the ThreadPool (Module 15).
6. **Lock granularity talk:** striping (N locks by key hash) reduces contention at the cost of complexity; mention it as the next step, don't build it unless asked.

### Step 4: say how you'd test it

*"I'd hammer it from many tasks with `Parallel.ForAsync` and assert the invariants afterwards — Count ≤ Capacity, map and list consistent. Concurrency tests find bugs probabilistically, so I'd also keep the critical section small enough to reason about by inspection."*

Worked example 2 implements the thread-safe LRU follow-up in full.

**Interview-grade sentence:** *"For a thread-safety follow-up I first name the shared state and the invariant that must hold across it — that sets the unit of atomicity — then pick the simplest primitive that protects it, usually a `lock` around the compound operation; I know the .NET traps, like `GetOrAdd` running factories more than once and check-then-act races, and I say how I'd test it."*

---
# Part D — C# and .NET idioms that read as senior

A .NET interviewer reads your C# the way a native speaker hears an accent. Correct-but-dated C# ("Java with capital letters") is not penalised much at mid-level; at senior level, **fluent, current, idiomatic C# — used with judgment — is part of the technical-competency score.** The rubric's advanced signal is literally "strong knowledge of language constructs and paradigms".

## Concept 21 — C# in the interview environment

### Should you use C#?

Usually yes, if the role is .NET — interviewers on .NET teams read it best, and fluency signals relevance. Two exceptions: (1) the environment's C# support is poor or ancient (check — see below), and (2) the round is language-agnostic and you're *noticeably* faster in another language. Meta's AI-enabled round supports C#; most online editors do.

### Know the environment's version

Online interview editors lag the SDK. CoderPad's public C# page has at times described a runtime several versions old; the current version is shown under the pad's **Info** tab and its packages under **Packages**. Before the interview — ideally in a practice pad — check:

- **Language version.** Collection expressions (`[1, 2, 3]`) need C# 12; the `field` keyword and extension members need C# 14; list patterns need C# 11. If they're unavailable, don't discover it at minute 20.
- **Entry point.** Some environments expect `static void Main` on a class (CoderPad's C# guidance has historically said a class named `Solution`); some accept top-level statements.
- **Test framework.** NUnit has historically been the one bundled in CoderPad's C# environment. Know enough NUnit (`[TestFixture]`, `[Test]`, `[TestCase(…)]`, `Assert.That(actual, Is.EqualTo(expected))`) to use it if it's there.
- **Nullable reference types** — on or off? Warnings will appear differently.

### What you must be able to type from memory

No IntelliSense is a real possibility (whiteboard, or an editor with completion disabled). Be able to write without lookup:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public static class Solution
{
    public static void Main()
    {
        Check("example", 2, Add(1, 1));
    }

    static int Add(int a, int b) => a + b;

    static void Check<T>(string name, T expected, T actual)
    {
        bool ok = EqualityComparer<T>.Default.Equals(expected, actual);
        Console.WriteLine($"{(ok ? "PASS" : "FAIL")} {name}: expected {expected}, got {actual}");
    }
}
```

Plus the exact signatures of the collections in Concept 22, the throw helpers in Concept 18, and the common LINQ operators. **Syntax errors are scored**: Amazon says no pseudocode, and the rubric's technical-competency band explicitly mentions syntax errors.

### Language choice in AI-enabled rounds

The AI will happily write any language, so the temptation is to pick the one the model seems best at. Don't. **Pick the language you can review fastest** — in an AI-enabled round, your bottleneck is reading and verifying, not typing.

**Interview-grade sentence:** *"I use C# for .NET roles, but I check the interview environment beforehand — its C# version, entry-point convention, bundled test framework and nullable setting — and I can write the harness, the core collections and the throw helpers from memory, because syntax errors are scored and IntelliSense isn't guaranteed."*

---

## Concept 22 — Collections fluency

Knowing data structures is your strength; knowing **the exact semantics of the .NET types that implement them** is what reads as fluent C#. Interviewers notice when a candidate knows, without checking, that `PriorityQueue` is a min-heap with no decrease-key.

### The essential types and their exact semantics

| Type | Use for | Semantics worth saying out loud |
|---|---|---|
| `Dictionary<TKey,TValue>` | Hash map | O(1) average; **enumeration order is not guaranteed** — never rely on it; indexer *throws* on a missing key; `TryGetValue` avoids double lookup; `TryAdd` vs `Add` (throws on duplicate); pass a comparer (`StringComparer.Ordinal`, `OrdinalIgnoreCase`) for string keys |
| `HashSet<T>` | Membership, dedupe | `Add` returns `false` if present — use it instead of `Contains` + `Add` |
| `List<T>` | Dynamic array | `Insert`/`RemoveAt(0)` are O(n); `BinarySearch` returns the bitwise complement (`~index`) of the insertion point when not found |
| `Queue<T>` / `Stack<T>` | BFS / DFS, undo | `TryDequeue`, `TryPeek`, `TryPop` avoid exceptions on empty |
| `PriorityQueue<TElement,TPriority>` (.NET 6+) | Heap | **Min-heap** on `TPriority`; for a max-heap pass `Comparer<int>.Create((a, b) => b.CompareTo(a))`; **not stable** (equal priorities dequeue in no particular order); **no decrease-key** — re-enqueue and skip stale entries ("lazy deletion"); `EnqueueDequeue` for fixed-size top-k; `UnorderedItems` for enumeration |
| `SortedDictionary<TKey,TValue>` / `SortedSet<T>` | Ordered map/set (red-black tree) | O(log n) operations; `SortedSet.Min`/`Max`; `GetViewBetween(lo, hi)` for range queries; no "floor/ceiling" method by name — use `GetViewBetween` |
| `SortedList<TKey,TValue>` | Small ordered maps | Array-backed; O(n) insert — fine for small, read-heavy maps |
| `LinkedList<T>` | O(1) removal given a node | Holds `LinkedListNode<T>` references — the basis of LRU (Worked example 2) |
| `ConcurrentDictionary`, `Channel<T>` | Concurrency | Concept 20 |
| `FrozenDictionary` / `FrozenSet` (.NET 8+) | Build-once, read-many lookups | Slower to create, faster to read — config tables, keyword sets |
| `ImmutableArray<T>`, `ImmutableList<T>` | Persistent / shared-safe collections | Thread-safe by immutability; `ImmutableList` is a tree — O(log n) indexer |

### Small idioms with outsized signal

```csharp
// One lookup instead of two (ContainsKey + indexer):
if (map.TryGetValue(key, out var existing)) { /* … */ }

// Counting without a double lookup (.NET 6+, using System.Runtime.InteropServices):
ref int slot = ref CollectionsMarshal.GetValueRefOrAddDefault(counts, code, out _);
slot++;

// Simple, readable alternative most interviewers prefer in a round:
counts[code] = counts.GetValueOrDefault(code) + 1;

// .NET 9+: counting in one LINQ call
var countsByCode = codes.CountBy(c => c);   // IEnumerable<KeyValuePair<string,int>>

// Fixed-size top-k with a min-heap of size k:
var heap = new PriorityQueue<string, int>(k);
foreach (var (code, count) in counts)
{
    if (heap.Count < k)
        heap.Enqueue(code, count);
    else if (heap.TryPeek(out _, out int minCount) && count > minCount)
        heap.EnqueueDequeue(code, count);      // push, then pop the current minimum
}
// heap now holds the k largest; dequeuing yields them smallest-first, so reverse for output.
// Ties at the k-th count are resolved arbitrarily here — say so, or include the code in the priority.
```

*The trap this avoids:* `PriorityQueue.Peek()` returns the **element**, not the priority, so `count > heap.Peek()` doesn't compile for `<string, int>` — you need `TryPeek(out element, out priority)`. And `EnqueueDequeue` does the push-then-pop in one efficient operation. That exact distinction is what separates "knows heaps" from "knows .NET's heap". Worked example 1 shows a version with a deterministic tie-break.

**Choose readability over micro-optimisation by default.** `CollectionsMarshal` is worth *mentioning* in a performance follow-up (Module 17); in the first version, `GetValueOrDefault(code) + 1` is clearer and almost always fast enough.

### Know your .NET version boundaries

`PriorityQueue`, `ArgumentNullException.ThrowIfNull` (.NET 6) · `ThrowIfNegativeOrZero`, `TimeProvider`, `FrozenDictionary` (.NET 8) · `CountBy`, `AggregateBy`, `Index()` LINQ operators, `System.Threading.Lock` (.NET 9) — if the environment is older, you'll need the fallback. Know one.

**Interview-grade sentence:** *"I know the exact semantics of the .NET collections, not just the data structures — `Dictionary` order isn't guaranteed and the indexer throws, `PriorityQueue` is an unstable min-heap without decrease-key so I use lazy deletion or `EnqueueDequeue` for top-k, `SortedSet.GetViewBetween` gives range queries — and I default to the readable idiom, mentioning the low-allocation one only when performance comes up."*

---

## Concept 23 — Modern C# used with judgment

Module 16 covered modern C# in depth. In a coding round, the question is narrower: **which features make interview code clearer, and which are traps or show-off?**

### Features that usually help

| Feature (version) | Helps with | Example |
|---|---|---|
| **Records** (C# 9; `record struct` C# 10) | Value objects, results, test expectations | `public sealed record CodeCount(string Code, int Count);` — equality and `ToString` for free |
| **Switch expressions** (C# 8) and **pattern matching** (property, relational, list patterns — C# 9–11) | Dispatch over commands/types; concise conditions | `cmd switch { Move m => …, Turn { Direction: Left } => …, _ => throw … }` |
| **Collection expressions** (C# 12) | Concise test data and literals | `int[] input = [3, 1, 2];` `List<string> empty = [];` |
| **Target-typed `new`** (C# 9) | Less repetition | `Dictionary<string, int> counts = new();` |
| **`init` / `required`** (C# 9 / 11) | Immutable configuration objects | `public required int Capacity { get; init; }` |
| **Tuples with names / deconstruction** | Private multi-value returns, `foreach (var (k, v) in dict)` | Prefer records in *public* APIs |
| **Nullable reference types** | Honest contracts | `string?`, `[NotNullWhen(true)]` |
| **Raw string literals** (C# 11) | Multi-line test input, JSON | `"""` … `"""` |
| **`field` keyword** (C# 14) | Validation in auto-properties without a backing field | `public int Capacity { get; set => field = value > 0 ? value : throw …; }` — only if the environment supports C# 14 |

### Features to use carefully

- **Primary constructors on classes (C# 12).** Convenient — but the parameters are **captured mutable state, not `readonly` fields**, and they're visible throughout the class. In a class with invariants, a primary-constructor parameter can be reassigned by mistake. For an interview class with validation, an explicit constructor and `private readonly` fields are often clearer; if you use a primary constructor, assign to `readonly` fields explicitly. Saying this out loud is a fluency signal.
- **LINQ.** Excellent for clarity in transformations: `counts.OrderByDescending(p => p.Value).ThenBy(p => p.Key, StringComparer.Ordinal).Take(k)`. Dangerous when it hides cost:
  - **Multiple enumeration** of an `IEnumerable<T>` that's expensive or lazy (a file read, a database query) — materialise once with `ToList()`/`ToArray()` if you'll enumerate twice.
  - **Quadratic patterns** — `list.Where(x => other.Contains(x))` with `other` a `List` is O(n·m); use a `HashSet`.
  - **`Count() > 0` instead of `Any()`**; `First()` where `FirstOrDefault()` plus a check was meant.
  - **Long chains in hot loops** — allocations and delegate calls; fine in a round unless performance is the topic.
- **`var`.** Use it when the type is obvious from the right-hand side; spell the type when it isn't (`IReadOnlyList<CodeCount> result = …` documents the contract).
- **Expression-bodied members** — great for one-liners, worse when they hide side effects.

### Features that are usually show-off in a round

- **Preview language features** (C# 15 unions in .NET 11 previews) — interesting to *discuss*, risky to *use*.
- **`Span<T>`, `stackalloc`, `unsafe`** — unless the problem is explicitly about performance or parsing at scale. Mentioning them in the optimisation follow-up is the right register (Module 17).
- **Source generators, reflection, `dynamic`** — almost never.

**Interview-grade sentence:** *"I use modern C# where it makes interview code clearer — records for results, switch expressions and patterns for dispatch, collection expressions for test data, honest nullability — I'm careful with primary constructors because their parameters are mutable captured state and with LINQ where it hides multiple enumeration or quadratic lookups, and I keep `Span`, `stackalloc` and preview features for the performance discussion rather than the first version."*

---

## Concept 24 — C# that reads junior

These are the tells .NET interviewers notice — each one small, and each one shifting the write-up from "senior" toward "mid-level". Most are also standard code-review findings, which is why they matter in code-review rounds too (Concept 29).

| Tell | Why it's a problem | Senior alternative |
|---|---|---|
| `catch (Exception) { }` or `catch { return null; }` | Swallows bugs; hides failures | Catch specific exceptions where you can act; `throw;` to rethrow |
| `throw ex;` | Resets the stack trace | `throw;` |
| `.Result` / `.Wait()` / `GetAwaiter().GetResult()` on tasks | Deadlocks with a sync context; ThreadPool starvation under load (Module 15) | `await` all the way |
| `async void` (outside event handlers) | Exceptions crash the process; can't be awaited | `async Task` |
| `new HttpClient()` per request | Socket exhaustion; ignores DNS changes | `IHttpClientFactory` or a long-lived shared instance with `PooledConnectionLifetime` |
| `DateTime.Now` in logic | Local time zone, DST bugs, untestable | `TimeProvider` (.NET 8) injected; `UtcNow` / `DateTimeOffset` if you must |
| `new Random()` in a loop | Wasteful; historically produced identical sequences | `Random.Shared` (.NET 6) |
| Culture-sensitive string comparison for identifiers (`ToLower()`, `string.Compare(a, b)` with defaults, `StartsWith(string)` without a comparison) | Wrong results under some cultures (the Turkish "i" problem); slower | `StringComparison.Ordinal` / `OrdinalIgnoreCase`; `StringComparer.Ordinal` for dictionaries |
| String concatenation in a loop | O(n²) copying | `StringBuilder`, `string.Join`, `string.Create` |
| `ContainsKey` followed by the indexer | Two lookups | `TryGetValue` |
| `Count() > 0` on an `IEnumerable` | May enumerate everything | `Any()` (or the `Count` property on collections) |
| Re-enumerating a lazy `IEnumerable` | Repeated work or repeated I/O | Materialise once |
| Mutable `static` state | Hidden coupling; thread-unsafe; untestable | Instance state with explicit lifetime |
| Magic numbers | Unreadable; brittle | Named constants or parameters |
| Public mutable fields / exposing internal `List<T>` | Callers can break invariants | Private fields; expose `IReadOnlyList<T>` or behaviour |
| Mutable `struct` | Copies silently lose mutations | `readonly struct` or a class |
| Mutating a `Dictionary` key's hash-relevant fields | The entry becomes unfindable | Immutable keys (records, strings) |
| `int` arithmetic that can overflow | Silent wraparound (unchecked by default) | `long`, `checked`, or `lo + (hi - lo) / 2` |
| `lock (this)` / `lock (typeof(X))` / `lock ("str")` | Others can lock the same object | A private lock object |
| Ignoring `IDisposable` | Leaked handles/connections | `using` declarations (`using var stream = …;`) |
| Everything `public` | No encapsulation | `private` by default, `sealed` by default |

**Use the list in both directions.** When writing, avoid them. When reviewing (Concept 29) or reading AI-generated code (Concept 33), *look for* them — AI models still produce several of these regularly (`new HttpClient()`, culture-default comparisons, `DateTime.Now`, broad catches), which makes catching them out loud one of the easiest verification signals in an AI-enabled round.

**Interview-grade sentence:** *"I keep a mental list of the C# tells that read junior — swallowed exceptions, blocking on async, `async void`, `HttpClient` per call, `DateTime.Now` in logic, culture-sensitive comparisons for identifiers, string concatenation in loops, double dictionary lookups, mutable statics and silent overflow — I avoid them when writing and I actively look for them when reviewing someone else's code or an AI's."*

---

## Concept 25 — Testing in .NET

When the round allows real tests (your own IDE, a take-home, a pair-programming round, Shopify-style AI rounds), fluent .NET testing is a strong differentiator. It's also the subject of common follow-up questions: *"How would you test this?"*

### The frameworks — 2026 state

| Framework | Status | Interview-relevant notes |
|---|---|---|
| **xUnit v3** | The most common default for new projects | `[Fact]`, `[Theory]` + `[InlineData]` / `[MemberData]` / `TheoryData<T>`; constructor = setup, `IDisposable`/`IAsyncLifetime` = teardown; new instance per test; native Microsoft.Testing.Platform support |
| **NUnit 4** | Mature, widespread | `[Test]`, `[TestCase]`; constraint model `Assert.That(x, Is.EqualTo(y))` — and historically the framework in CoderPad's C# environment |
| **MSTest 4** | Microsoft's own; much modernised | `[TestMethod]`, `[DataRow]` |
| **TUnit** | Newer, built entirely on Microsoft.Testing.Platform; source-generated | Fine to mention; check team conventions before adopting |

All four run on **Microsoft.Testing.Platform**, the newer test runner that replaces VSTest over time.

### A parameterised test in xUnit

```csharp
public class TopErrorCodesTests
{
    [Theory]
    [InlineData(1, new[] { "E1" })]
    [InlineData(2, new[] { "E1", "E2" })]
    [InlineData(5, new[] { "E1", "E2", "E3" })]          // k larger than distinct codes
    public void Returns_most_frequent_codes_in_order(int k, string[] expected)
    {
        string[] lines =
        [
            "t1 ERROR E1 a", "t2 ERROR E2 b", "t3 ERROR E1 c",
            "t4 ERROR E3 d", "t5 INFO  I9 e", "garbage",
        ];

        var result = LogAnalysis.TopErrorCodes(lines, k);

        Assert.Equal(expected, result.Select(r => r.Code));
    }

    [Fact]
    public void Ties_are_broken_by_code_ordinal()
    {
        var result = LogAnalysis.TopErrorCodes(["t ERROR B x", "t ERROR A y"], 2);
        Assert.Equal(new[] { "A", "B" }, result.Select(r => r.Code));
    }

    [Fact]
    public void Rejects_non_positive_k() =>
        Assert.Throws<ArgumentOutOfRangeException>(() => LogAnalysis.TopErrorCodes([], 0));
}
```

What to point out: one theory covers the boundary partitions; the tie-break has its own named test because it's a contract decision; the invalid-input test documents the guard clause; test names describe behaviour. (Collection expressions in test bodies need C# 12; on older environments use `new[] { … }`.)

### Testing time, randomness and I/O

- **Time:** inject `TimeProvider` (built into .NET 8+) and use `FakeTimeProvider` from the `Microsoft.Extensions.TimeProvider.Testing` package to advance time deterministically — the standard answer for TTL caches, rate limiters and token buckets.
- **Randomness:** inject a `Random` with a fixed seed, or an abstraction.
- **I/O:** separate parsing (pure, unit-tested) from reading (thin, integration-tested); in take-homes, `WebApplicationFactory<T>` for ASP.NET Core and Testcontainers for real databases (Module 19).

### Test doubles and assertion libraries — the licensing landscape

This is a legitimate 2026 interview aside ("what do you use, and why?"), and the honest answer involves licensing:

- **Mocking:** **NSubstitute** and **Moq** are the two common choices. Moq briefly shipped a telemetry component (SponsorLink) in August 2023 that collected hashed developer email addresses at build time; it was removed after a backlash within days, but many teams switched to NSubstitute or reviewed their supply-chain policy as a result. A senior answer also notes **prefer fakes over mocks** where a simple in-memory implementation exists, and avoid mocking types you don't own.
- **Assertions:** **FluentAssertions moved to a commercial licence from version 8** (January 2025; earlier versions remain Apache-2.0). Teams either pinned v7, paid, switched to the Apache-2.0 community fork **AwesomeAssertions**, moved to **Shouldly**, or went back to built-in asserts. Knowing this is a small "keeps up with the ecosystem" signal — and a nice link to build-vs-buy reasoning (Module 33).
- **Snapshot testing:** **Verify** for complex outputs (JSON, rendered text).

### Test naming and structure

Arrange–Act–Assert; one behaviour per test; names that read as specifications (`Ties_are_broken_by_code_ordinal`). In an interview, five well-chosen, well-named tests beat twenty generated ones.

**Interview-grade sentence:** *"When the round allows real tests I write a few parameterised xUnit theories over the input partitions plus named tests for contract decisions like tie-breaking and invalid input; I make time testable with `TimeProvider` and `FakeTimeProvider`, prefer fakes over mocks, and I can talk about the 2026 tooling landscape — xUnit v3 on Microsoft.Testing.Platform, NSubstitute versus Moq, and FluentAssertions' move to a commercial licence with AwesomeAssertions as the Apache fork."*

---
# Part E — Format-specific playbooks

Each concept in this part follows the same shape: **what the format samples**, **the playbook** (what to do, in order), **what scores at senior level**, and **the common failure**.

## Concept 26 — The classic algorithmic round

**What it samples.** Precise reasoning turned into correct code under time pressure. Still the most common format in big-tech phone screens and still present in Meta's onsite (the non-AI round), Amazon's and Google's loops.

**The playbook for a candidate with strong DS&A:**

1. **Don't skip the contract** because you recognise the problem. Recognition is exactly when candidates miss the twist ("the input may contain negative weights", "return indices, not values").
2. **Name the pattern and the complexity in one breath**, then confirm: *"This is a sliding window with a frequency map — O(n) time, O(alphabet) space. Shall I go with that?"*
3. **Code it cleanly at a steady pace.** Speed is valued — Meta's classic round in particular is reported to favour pace, often with two problems in 45 minutes — but clean code at a steady pace beats fast code you then have to debug.
4. **Test unprompted** with a trace, then edge cases.
5. **Spend the surplus on evidence** — a second approach and why you didn't choose it, the trade-off at a different input size, a follow-up anticipated. *Not* on silence after "done".

**What scores at senior level:** an optimal solution reached without hints, *plus* the other three dimensions at full strength. Interviewers often have a second question or a harder follow-up ready if you finish early — finishing early is an opportunity, not the end.

**The common failure for strong algorithmists:** treating the round as solved once the algorithm is known — code dashed off with single-letter names, no tests, "done" at minute 18, then an interviewer-found bug. The write-up says "fast but careless".

**Keeping the floor without over-investing.** You don't need hundreds of problems. You need the core patterns *fluent in C#* (Concept 36): two pointers, sliding window, hashing, prefix sums, binary search on answers, BFS/DFS on grids and graphs, topological sort, union–find, heaps/top-k, intervals, monotonic stack, tries, backtracking, and basic DP. Grind 75 or NeetCode 150 subsets are enough as maintenance.

**Interview-grade sentence:** *"In a classic algorithmic round I still write the contract, name the pattern and complexity in one sentence, code at a steady clean pace, test without being asked, and spend any time I've saved on a second approach or a follow-up — because at senior level the optimal algorithm is the expectation, not the score."*

---

## Concept 27 — Practical multi-part rounds

**What it samples.** Building working software incrementally — closer to real feature work. Typical shape: *part 1* parse some input; *part 2* compute something; *part 3* a new rule or data shape; *part 4* errors, scale or an API. You usually don't see later parts in advance. Stripe-style loops are the best-known example; many product companies use the shape.

**The playbook:**

1. **Optimise part 1 for change, not for brevity.** The parser you write now will be extended twice. Parse into a small domain type (`record Transaction(...)`) rather than passing raw strings around.
2. **Keep it running and green between parts.** Run after each part; keep earlier part's checks in `Main` as a regression suite. Interviewers notice when part 3 silently breaks part 1.
3. **Read the whole prompt for each part** — these prompts often contain specific rules ("refunds can't exceed the original charge", "amounts are in minor units") that are easy to skim past. Restate the new rules before coding.
4. **Refactor at part boundaries, briefly and visibly.** *"Part 3 adds currencies, so I'll pull amount handling into a `Money` record before adding the new rule."*
5. **Handle bad data per the spec** — practical rounds often have deliberately malformed records.

**What scores at senior level:** each part's change is small because the earlier design was right; domain types instead of primitive soup; correctness on the fiddly rules; consistent progress. Completing all parts matters more here than in other formats — but not at the cost of tests.

**The common failure:** a fast, monolithic part 1 that has to be rewritten for part 2, burning the time needed for parts 3 and 4.

**Interview-grade sentence:** *"In a multi-part round I write part one for change — parse into small domain types, keep each step separately testable — keep earlier parts' checks running as a regression suite, restate each new part's rules before coding, and refactor briefly and visibly at the boundaries, so each later part is a small change rather than a rewrite."*

---

## Concept 28 — Low-level design and machine coding

**What it samples.** Object modelling and API design at class level, with working code. Typical prompts: parking lot, elevator, rate limiter, LRU/LFU cache, in-memory key-value store with TTL, library or booking system, vending machine, logger with levels and sinks, task scheduler. Amazon includes low-level design in its SDE III preparation material; Shopify's AI rounds behave like LLD with AI; many companies run "machine coding" rounds of 60–90 minutes.

**The playbook:**

1. **Requirements first — functional and non-functional — in two minutes.** *"Operations: park, unpark, find free spot by size. Constraints: single process, concurrent gates, sizes small/medium/large."* Ask which features are in scope; propose a cut.
2. **Entities and responsibilities.** List the nouns and give each **one responsibility**; list the verbs and assign each to the entity that owns the data. *"`ParkingLot` allocates; `Level` tracks its spots; `Ticket` is an immutable record; pricing is a separate policy."*
3. **Public API before internals.** Write the signatures of the 3–5 public operations (Concept 17).
4. **Identify the axis of change** and put one seam there (Concept 19): pricing policy, eviction policy, allocation strategy.
5. **Implement the core path end to end** first — one happy path working — then edge cases, then the secondary features.
6. **Concurrency and persistence: discuss, then implement if asked** (Concept 20).
7. **Demonstrate it**: a short driver in `Main` or a few tests.

**What scores at senior level:** a small number of cohesive types with clear responsibilities; immutability where it simplifies (records for tickets, events); the right seam; working code. Interviewers are explicit that they want **working code, not UML**.

**The common failures:** an elaborate class hierarchy with no running code; a "god class"; patterns applied for their own sake (a Singleton `ParkingLot` and an Abstract Factory for vehicles); or the opposite — one big procedural method.

**Patterns that genuinely recur in LLD rounds** (and their C# expression): **Strategy** (an interface or `Func<>` for pricing/eviction/allocation), **State** (a `switch` over an enum or state records for vending machines/elevators), **Observer** (events or `IObservable<T>`, or simply a callback list), **Command** (records for commands with a dispatcher), **Decorator** (wrapping a sink or a cache). Name them only when they explain a choice.

**Interview-grade sentence:** *"In a low-level design round I spend two minutes on scope, list the entities with one responsibility each, write the public API before the internals, put a single seam where change is likely — usually a policy like pricing or eviction — and get one path working end to end before adding features, because they want working, cohesive code rather than a diagram of patterns."*

---

## Concept 29 — The code review round

**What it samples.** Judgment and quality bar: can you find what matters in someone else's code, prioritise it, and communicate it constructively? Increasingly used for senior roles because, with AI writing more code, **reviewing it is a larger share of the job**. Formats vary: a PR in a web tool, a snippet in a shared doc with comments, or verbal review on a screen share; often 30–45 minutes.

**The playbook:**

1. **Understand intent first.** Read the description or ask: *"What is this change supposed to do, and what's the context — a hot path, a public API, a one-off script?"* Context sets the bar (Concept 16).
2. **Skim the whole change before commenting** — structure, size, tests present?
3. **Review in priority order** — the same order as a production review:

   | Priority | Look for |
   |---|---|
   | **1. Correctness** | Logic errors, edge cases, off-by-one, null handling, wrong comparisons, broken invariants |
   | **2. Security** | Injection (SQL, command, path), secrets in code, missing authorisation, unsafe deserialisation, logging sensitive data |
   | **3. Concurrency and resources** | Races, shared mutable state, blocking on async, undisposed resources, `HttpClient` per call |
   | **4. Failure handling** | Swallowed exceptions, missing timeouts/retries, partial failure, idempotency |
   | **5. Performance** | Accidental O(n²), N+1 queries (Module 19), allocations in hot paths, unbounded growth |
   | **6. API and design** | Leaky abstractions, mutable outputs, confusing names, wrong layer |
   | **7. Tests** | Missing, weak, or testing implementation rather than behaviour |
   | **8. Readability and style** | Naming, structure, comments — last, and lightly |

4. **Label severity** on each comment. A widely used convention is **Conventional Comments** — `issue (blocking):`, `suggestion:`, `question:`, `nitpick (non-blocking):`, `praise:`. In the round, simply say "blocking" vs "nit".
5. **Comment like a colleague.** Explain *why*, suggest a fix, ask when unsure: *"`question`: is `items` ever empty here? If so, `First()` throws — `FirstOrDefault` plus a check, or a guard at the top?"*
6. **Say what's good.** Praise is part of a review, not flattery — it shows you'd be a reviewer people want.
7. **Summarise** at the end: *"Two blocking issues — the race on the cache and the SQL injection — one important (no timeout on the HTTP call), and a few nits. With the first two fixed I'd approve."*

**What scores at senior level:** finding the important issues (security, concurrency, correctness) rather than twenty style nits; correct prioritisation; constructive, specific language; knowing what to let go. Google's own engineering practices guide puts it well: approve once the change **definitely improves overall code health**, even if it isn't perfect.

**The common failure:** a flood of style comments while the SQL injection on line 12 goes unmentioned — exactly what practitioners mean by "senior code review is not about finding bugs" in the narrow sense: it's about finding what *matters*.

Worked example 4 is a full C# review exercise.

**Interview-grade sentence:** *"In a code review round I first establish intent and context, skim the whole change, then review in priority order — correctness, security, concurrency and resources, failure handling, performance, API, tests, then style — label each comment blocking or not, explain why and suggest a fix, mention what's good, and close with a summary and a decision, because the skill is finding what matters, not everything."*

---

## Concept 30 — Debugging in an unfamiliar codebase

**What it samples.** Code comprehension and systematic debugging — the daily reality of senior work. Appears as Stripe-style "bug squash", phase 1 of Meta's AI-enabled round (practitioners report bugs such as wrong type conversions, off-by-one errors, incorrect conditionals, or an iteration cap masking a missing visited set), and Google's comprehension pilot.

**The playbook:**

1. **Orient before you touch anything** (5 minutes is well spent; candidates consistently report skipping it as their biggest regret):
   - Read the README / problem statement and **the tests** — tests are the executable specification.
   - Map the structure: entry point, core types, data models, where the failing behaviour lives.
   - Read the **data models** carefully — many planted bugs are type or unit mismatches.
2. **Reproduce.** Run the failing test or scenario; read the exact failure.
3. **Hypothesise and narrow** (Concept 12): what mechanism produces this output? Bisect with prints or the debugger.
4. **Fix the root cause**, not the symptom. If the fix is "raise the iteration cap", it's the symptom; the cause is the missing visited set.
5. **Add a regression test** for the bug — even a one-line check — and rerun everything.
6. **Report what else you noticed** — suboptimal complexity, a latent bug, a confusing name — without fixing everything. Practitioners report interviewers reacting well to proactively spotted issues beyond the planted one.

**Reading-code techniques that speed you up:** start from the test that fails and follow calls inward; read signatures and types before bodies; keep a scratch list of "facts" (what each type means, invariants you've inferred); use "find usages" if the editor has it; ask the AI to summarise a file *only where AI is allowed* — and verify its summary against the code.

**What scores at senior level:** a calm, structured process; a root-cause fix with a test; noticing adjacent issues; explaining the codebase's design back to the interviewer.

**Interview-grade sentence:** *"In a debugging round I orient first — the tests as the specification, the structure, and especially the data models — then reproduce, hypothesise and bisect, fix the root cause rather than the symptom, add a regression test, rerun everything, and mention the adjacent issues I noticed without trying to fix the whole codebase."*

---

## Concept 31 — The integration / API round

**What it samples.** Working with third-party systems: reading documentation, making HTTP calls, transforming JSON, and — most importantly — **handling the ways remote calls fail**. Candidates of Stripe-style integration rounds consistently report heavy emphasis on error handling.

**The playbook in .NET:**

1. **Read the docs for the contract**: authentication, pagination, rate limits, error format, idempotency support.
2. **Model the payloads** as records and deserialise with `System.Text.Json` (`JsonSerializer.Deserialize<T>` or `HttpClient.GetFromJsonAsync<T>`); set `JsonSerializerOptions` (case-insensitivity or `JsonSerializerDefaults.Web`) explicitly.
3. **Use one `HttpClient`** (or `IHttpClientFactory` in a real app) — never one per call (Concept 24).
4. **Timeouts and cancellation:** pass a `CancellationToken`; set a timeout; don't let the call hang forever.
5. **Check status codes deliberately:** `EnsureSuccessStatusCode()` is fine for a first version; better, branch on 4xx (caller's fault — don't retry, except 429) vs 5xx and timeouts (transient — retry with backoff and jitter).
6. **Retries only for idempotent operations** — or with an **idempotency key** for non-idempotent ones like creating a payment (Modules 11 and 25). This sentence alone is a strong senior signal in a payments-flavoured round.
7. **Pagination:** loop until the cursor/`has_more` says stop; don't assume one page.
8. **Rate limits:** honour `429` and `Retry-After`.
9. **Partial failure:** decide and say what happens if item 37 of 100 fails — skip and report, stop, or retry later.

```csharp
public sealed record Charge(string Id, long AmountMinor, string Currency, string Status);
public sealed record Page<T>(IReadOnlyList<T> Data, bool HasMore, string? NextCursor);

public async Task<IReadOnlyList<Charge>> GetAllChargesAsync(CancellationToken ct)
{
    var all = new List<Charge>();
    string? cursor = null;
    do
    {
        var url = cursor is null ? "charges" : $"charges?starting_after={Uri.EscapeDataString(cursor)}";
        var page = await _http.GetFromJsonAsync<Page<Charge>>(url, _json, ct)
                   ?? throw new InvalidOperationException("Empty response body.");
        all.AddRange(page.Data);
        cursor = page.HasMore ? page.NextCursor : null;
    } while (cursor is not null);
    return all;
}
```

Things to say about it: `_json` is a single cached `JsonSerializerOptions` with `PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower` (.NET 8+) so `has_more` maps to `HasMore` — and caching options matters, because creating them per call is expensive; amounts in **minor units as `long`** (never `double` for money); escaping the cursor; `null` body handled; the loop terminates when `HasMore` is false; in production you'd add a Polly/`Microsoft.Extensions.Http.Resilience` pipeline for transient retries with timeouts (Module 25) and a cap on total pages.

**What scores at senior level:** correct failure taxonomy (retryable vs not), idempotency, timeouts, no resource leaks, clean modelling — and a working call.

**Interview-grade sentence:** *"In an integration round I read the docs for auth, pagination, rate limits, errors and idempotency, model payloads as records with System.Text.Json, use a shared `HttpClient` with timeouts and cancellation, distinguish non-retryable 4xx from retryable 429, 5xx and timeouts, retry only idempotent calls or use an idempotency key, follow pagination to the end, and decide explicitly what partial failure means."*

---

## Concept 32 — Take-home assignments

**What it samples.** Your work when unhurried: structure, tests, decisions and communication in writing — followed, usually, by a review conversation where you defend it.

**The playbook:**

1. **Clarify scope and the AI policy in writing** before you start (Concept 34). Many companies now say "use the tools you'd use on the job, and be ready to explain your choices"; some forbid AI.
2. **Respect the timebox.** If they say four hours, a 20-hour submission signals poor prioritisation (and is unfair to other candidates). Spend the time on quality of the core, not breadth of features.
3. **Structure like a real repo:** a solution with `src/` and `tests/`, a clear entry point, `dotnet test` passing, no warnings, `.editorconfig` if you like, nullable enabled.
4. **Tests that show judgment:** unit tests for the domain logic, one or two integration tests (e.g. `WebApplicationFactory`) for the API surface — not exhaustive coverage.
5. **Write the README as a short design document** (Module 30): how to run it; assumptions made; key decisions with alternatives considered; what you'd do with more time; known limitations. *This is often read before the code.*
6. **Commit history that tells the story** — small commits with clear messages beat one "initial commit".
7. **Prepare for the review conversation:** expect "why this structure?", "what would you change for production?", "extend it live with X". If AI helped, be ready to explain every line.

**What scores at senior level:** a focused, well-tested core; explicit, defensible decisions; honest limitations; a design that's easy to extend in the follow-up session.

**The common failure:** over-building (microservices, message brokers and Kubernetes manifests for a CRUD task) — the take-home equivalent of over-engineering (Concept 16) — or a code-only submission with no README, which forces the reviewer to guess your reasoning.

**Interview-grade sentence:** *"For a take-home I confirm scope and AI policy in writing, respect the timebox, build a focused core with a real repo structure and judgment-driven tests, and treat the README as a short design document — assumptions, decisions with alternatives, limitations and what I'd do next — because the reviewer often reads it before the code."*

---

## Concept 33 — AI-enabled rounds

**What it samples.** How you work with AI tools on realistic code: decomposition, delegation, verification, code comprehension and communication. Formats differ substantially:

| Company (as reported in 2025–26) | Environment | Shape | Distinctive emphasis |
|---|---|---|---|
| **Meta** | CoderPad, multi-file project, AI chat panel (can't edit files), several models | 60 min; phases: bug fix → core implementation → optimise for larger inputs | Same four competencies as classic coding; AI reportedly less helpful than in practice; not finishing phase 3 is survivable |
| **Shopify** | Your own IDE and AI tools; empty GitHub repo; screen share | Evolving open-ended problem (e.g. LRU cache, robot grid, `tail`) with follow-ups | Design and file structure; tests; "you drive, AI assists"; cleaning AI output |
| **LinkedIn** | CoderPad with AI panel | Familiar problem + deep follow-ups | Concurrency and production-readiness; verification |
| **Canva** | Your preferred AI tools | Ambiguous, realistic product problems | Requirement clarification; review and fix AI code |
| **Google (pilot)** | Gemini in a comprehension round | Read, debug, optimise existing code | Prompting, output validation, debugging |

### The workflow that scores

1. **Orient yourself, not the AI.** Read the code and tests yourself first (Concept 30). In Meta's format, debugging phase 1 *without* AI is reported to send an early "can work independently" signal.
2. **Plan before prompting** — say the approach to the interviewer: *"This is BFS with a visited set keyed by position and gate state. I'll write the traversal skeleton myself and ask the AI for the parsing boilerplate."*
3. **Delegate small, well-specified units.** Information-rich prompts with types, constraints and the exact behaviour: *"Write a C# method `static IReadOnlyList<Point> ParseGrid(string[] rows)` that returns the coordinates of `#` cells; rows may have trailing whitespace; throw `FormatException` on unknown characters."* Not *"solve this"*.
4. **Review every response before using it** — out loud: *"It used `Split(' ')` without `RemoveEmptyEntries`, which breaks on double spaces — I'll fix that."* Look for the Concept 24 tells; AI output is a reliable source of them.
5. **Run after every integration.** The rhythm reported as optimal: **prompt → review → run → confirm → move on.** Never stack three AI-generated pieces before running.
6. **Write the core insight yourself** when it's the crux — or when the AI is unhelpful. Practitioners report that models in interview environments may be deliberately constrained; plan to carry the algorithm yourself.
7. **Reject what you can't explain.** LinkedIn candidates have reportedly received positive feedback for saying *"I don't want to use this response; I'll write it myself"* — even when the AI's answer was correct.
8. **Use AI for what it's good at:** boilerplate, data-structure implementations you've specified, test-case generation, quick benchmarks of two approaches, explaining unfamiliar APIs.
9. **Narrate both conversations** — intent before each prompt, judgment after each response (Concept 13).

### What interviewers say they penalise

- Prompting your way through without understanding ("don't prompt your way out of it", as one internal description is reported to put it).
- Pasting without review; not running.
- Being unable to explain code on screen.
- Letting the AI choose the approach.
- Silence while the AI works.

### Preparation specifics

- **Practise in the real environment** — Meta provides a practice environment ("the puzzle") to scheduled candidates; ask for it. For Shopify-style rounds, practise greenfield work in your own setup.
- **Practise with a weak or no AI** so a constrained model doesn't throw you.
- **Practise reading unfamiliar code fast** — open-source .NET repositories are ideal (Resources).
- **Pick the language you review fastest** (Concept 21).

Worked example 6 shows a full AI-enabled round transcript.

**Interview-grade sentence:** *"In an AI-enabled round I orient and plan myself, delegate small well-specified pieces with information-rich prompts, review every response out loud before I use it, run after every integration, write the crux myself, reject anything I can't explain, and narrate both conversations — because they're scoring my judgment about the AI's output, not the volume of code it produces."*

---

## Concept 34 — Integrity and AI policy

The AI policy varies **by company and by round** — sometimes within one loop (Meta: one AI round, one non-AI round). Getting it wrong is costly either way: using AI where it's forbidden is an integrity failure; not using it where it's expected (Canva says it *insists*) under-samples your skills.

**Rules:**

1. **Ask for each round's policy** in advance, and again at the start if unclear: *"Should I treat this round as AI-allowed or AI-off?"* It's a normal, professional question.
2. **Default to "no AI" unless told otherwise.** That's Anthropic's published norm for live interviews and the safe assumption elsewhere.
3. **Never use covert tools.** Companies have invested in detection and in formats that make covert assistance obvious (follow-up depth, in-person rounds); and a discovered covert tool ends the process — and can follow you.
4. **In take-homes, disclose AI use** if allowed, and be ready to explain everything.
5. **Preparation is different from performance.** Using AI to *prepare* — generating practice problems, reviewing your practice code, running mock interviews — is broadly encouraged.

**Interview-grade sentence:** *"I ask the AI policy for each round — it differs even within one loop — default to no AI unless told otherwise, never use covert assistance, disclose AI use in take-homes where it's permitted, and use AI freely for preparation, which is where it belongs unless the interviewer says otherwise."*

---
# Part F — Preparation

## Concept 35 — Diagnosing your gap

Strong-DS&A candidates usually measure preparation in **problems solved**. That metric measures the dimension you're already strong in. Measure instead **against the rubric**.

### A self-diagnosis in one session

Record yourself (screen and voice) solving two medium problems in 45 minutes, as if in a real round, in C#, in an online editor with no IntelliSense. Then score the recording on Appendix E's rubric, dimension by dimension, using **observable evidence only**:

| Question | Evidence to look for in the recording |
|---|---|
| Did I write a contract before coding? | A comment block or explicit spoken assumptions in the first five minutes |
| Did I state approach and complexity before typing? | A plan sentence with Big-O |
| Did I narrate decisions? | Count silent stretches longer than 60 seconds |
| Did I enumerate edge cases unprompted? | Named cases before the interviewer (or before minute 30) |
| Did I test before saying "done"? | A trace or a run of deliberately chosen cases |
| Did I find my own bugs? | Bugs found by me vs left in |
| Was the code idiomatic, current C#? | Concept 24 tells present? Throw helpers, records, correct collection use? |
| Did I compile first time? | Count syntax errors |
| Did I anticipate the follow-up? | Mentioned scale/concurrency/extension without being asked? |

**Typical findings for strong algorithmists:** silent stretches; edge cases left to the end or omitted; "done" before testing; dated or Java-flavoured C#; and little or no follow-up discussion. Each is a skill gap with a specific drill (Concept 36), not a talent gap.

### Diagnose by format, too

Rate your comfort 1–5 for each format in Concept 5. The low scores — typically code review, debugging in an unfamiliar codebase, and AI-enabled — are where preparation hours pay most.

**Interview-grade sentence (for yourself):** *"I measure my coding-round readiness against the rubric, not by problems solved: I record a realistic mock, score observable evidence per dimension — contract, plan, narration, edge cases, testing, own-bug finding, idiomatic C#, follow-ups — and rate my comfort with each format, then practise the lowest scores."*

---

## Concept 36 — The four-week plan

Designed for a candidate with strong DS&A, alongside other preparation, at **30–45 minutes on weekdays and 1–2 hours at weekends**. It keeps the algorithmic floor with light maintenance and spends most time on the differentiators. The learning principles are Module 35's: **retrieval over rereading, spacing, interleaving, and feedback.**

### Week 1 — Calibrate and build the C# reflexes

| Day | Activity |
|---|---|
| 1 | Self-diagnosis recording and scoring (Concept 35) |
| 2 | **C# fluency drill:** write from memory the harness, the collections table (Concept 22), throw helpers and common LINQ; check against docs; repeat on day 4 |
| 3 | Two classic problems in C#, *full protocol* (Appendix A): contract, plan, narrate, test — aloud, recorded |
| 4 | Fluency drill repeat; one problem rewritten from Java-flavoured to idiomatic modern C# |
| 5 | **Edge-case drill:** for five problems, write only the contract and the edge-case partitions — no code (15 min) |
| Weekend | Mock #1 (peer or AI interviewer) in classic format; score on Appendix E |

### Week 2 — Quality, design and tests

| Day | Activity |
|---|---|
| 1 | **LLD:** LRU cache with tests, then the thread-safe follow-up (Worked example 2) — 60 min |
| 2 | **Practical multi-part:** a four-part problem (Worked example 3 or self-made) with a running regression harness |
| 3 | **Testing drill:** take an earlier solution, write xUnit theories over its partitions; introduce `TimeProvider` into a TTL problem |
| 4 | **LLD:** rate limiter or key-value store with TTL — design surface first, seam for the algorithm |
| 5 | Two classic problems (maintenance), full protocol |
| Weekend | Mock #2 — LLD or practical format; focus on follow-ups |

### Week 3 — Reading, reviewing, debugging

| Day | Activity |
|---|---|
| 1 | **Code-review drill:** Worked example 4 cold, then compare with the model review; time yourself (30 min) |
| 2 | **Real-world review:** read three merged PRs in an open-source .NET repo (dotnet/runtime, dotnet/aspnetcore, a smaller library) and their review comments — what did reviewers prioritise? |
| 3 | **Debugging drill:** take an open-source repo, check out a commit with a known fixed bug (from an issue), and find it from the failing test without looking at the fix |
| 4 | **Integration drill:** call a public API (e.g. a free REST API) with pagination, timeouts and error handling (Concept 31) |
| 5 | Two classic problems (maintenance) |
| Weekend | Mock #3 — code review or bug-squash format |

### Week 4 — AI-enabled and calibration

| Day | Activity |
|---|---|
| 1 | **AI-enabled drill (CoderPad-style):** a multi-file starter project, AI in a *chat panel only*; three phases: fix a bug, implement, optimise — narrate aloud (60 min) |
| 2 | **AI-enabled drill (Shopify-style):** empty repo, your IDE and AI tools; build and extend a small app with tests in 45 min |
| 3 | **Weak-AI drill:** repeat day 1's format with a small/weak model or AI off — carry the core yourself |
| 4 | Mock #4 calibrated to the target company's actual format |
| 5 | Light review: Appendix A, B, C, F cards; one classic problem |
| Day before | Environment check (Concept 37); no new material |

### Maintenance after week 4

Two or three classic problems a week, always under the full protocol, plus one review or debugging exercise. Keep a **mistake log** — every bug you wrote and every rubric miss — and review it weekly; it converges on your personal edge-case list.

### The maintenance problem set

You don't need a large set. Choose ~40–60 problems that cover the core patterns (Concept 26) from Grind 75 or NeetCode 150, and re-solve them in C# under the protocol, spaced over weeks. Re-solving a known problem with perfect process is better practice for a senior round than solving a new one with poor process.

**Interview-grade sentence (for yourself):** *"My four weeks keep the algorithmic floor with a few protocol-driven problems a week and spend the rest on the differentiators — C# fluency from memory, edge-case and testing drills, low-level design with follow-ups, code review, debugging unfamiliar code, integration work, and AI-enabled rounds in both CoderPad and own-IDE styles — with a scored mock every weekend and a mistake log I review weekly."*

---

## Concept 37 — Day-of protocol

### Before the day

- **Ask the recruiter:** format of each round; AI policy per round; environment (CoderPad, HackerRank, own IDE, whiteboard); language and version; whether code must run; duration. For Meta's AI round, ask for the practice environment.
- **Environment check** in a practice pad: C# version, entry-point convention, test framework, how to run, keyboard shortcuts. For own-IDE rounds: IDE updated, AI tools logged in and working, a clean empty project template, `dotnet --info` working, notifications off, screen-sharing tested.
- **Have your harness memorised** (Concept 21) — or, for own-IDE rounds, a template project ready.

### The opening script (first 60 seconds of a coding round)

> *"Before I start — is this a round where I can use AI tools or not? And should the code run, or is this more of a whiteboard-style discussion? … Great. Let me restate the problem to check I've got it…"*

Then the contract (Concept 6).

### During

Run Appendix A's protocol. Glance at the clock at each checkpoint (Concept 8). If you're going wrong — behind time, wrong approach — **say so and correct course**; a visible recovery is evidence too.

### The last five minutes

Your questions for the interviewer. Good ones for a coding round: *"How does the team review code — what does a good PR look like here?"*, *"How much of the team's code is AI-assisted now, and how has review changed?"*, *"What does the on-call / production ownership model look like?"* They show you think about the work beyond the puzzle.

### After — the after-action review

Within an hour, write down: the problem(s), what you did in each phase, where you lost time, bugs you wrote, edge cases you missed, follow-ups asked, and one change for next time. Add bugs and misses to your mistake log. (Module 35, Concept 36 uses the same structure for behavioral rounds.)

**Interview-grade sentence:** *"Before the day I confirm each round's format, AI policy and environment and check the C# version in a practice pad; I open every coding round by confirming the AI policy and whether code must run, then restate the problem and write the contract; I close with questions about how the team reviews code; and I write an after-action review within the hour."*

---
# Worked examples

The examples are written to compile on .NET 8 or later with nullable reference types enabled (a few lines need .NET 9 and are marked). They're shown the way you'd want them to look at the end of a round — with the narration that produced them.

## Worked example 1 — The same problem, two rounds, two write-ups

**Prompt:** *"Given a list of log lines, return the k most frequent error codes."*

### Round A — strong algorithmist, mid-level evidence (condensed transcript)

> **Candidate:** "OK, hash map and a heap." *(types for 9 minutes in silence)* "Done. It's O(n log k)."
> **Interviewer:** "What happens if two codes have the same count?"
> **Candidate:** "Uh — the heap picks one. Is that OK?"
> **Interviewer:** "What if a line doesn't have a code?"
> **Candidate:** "It'd throw on the split… let me add a check." *(fixes)*
> **Interviewer:** "Can you run it?" *(it prints the right answer for the example)*

The code worked; it used a `PriorityQueue` correctly. The write-up:

> *"Recognised the pattern immediately and reached O(n log k). Did not clarify ties or malformed input; both raised by me. Silent while coding. No testing beyond running the example. Code readable but monolithic. **Lean hire at mid-level; not a senior signal.**"*

### Round B — same candidate, senior evidence

**Minutes 0–4 — contract.**

> "Two questions that change the design: roughly how many lines and distinct codes — and do ties need a deterministic order? … Millions of lines, thousands of codes, deterministic ties please. Then I'll assume: raw lines in the format `timestamp LEVEL CODE message`; only `ERROR` lines count; malformed lines are skipped; `k ≤ 0` is an argument error; fewer than k codes returns them all; ties broken by code, ordinal." *(writes the contract comment — Concept 6)*

**Minutes 4–7 — plan.**

> "Counting is O(n) and dominates. With thousands of distinct codes, sorting them is trivial, so I'll sort with a two-key order — count descending, then code. A size-k heap would be O(m log k) and matters only if m were huge — I'll mention how I'd do that at the end. Three pieces: parse, count, select."

**Minutes 7–20 — code, narrated.**

```csharp
using System.Diagnostics.CodeAnalysis;

public sealed record CodeCount(string Code, int Count);

public static class LogAnalysis
{
    // Contract: up to k (code, count) pairs, count desc, then code (ordinal) asc.
    // Malformed / non-ERROR lines are skipped. k <= 0 throws.
    public static IReadOnlyList<CodeCount> TopErrorCodes(IEnumerable<string> lines, int k)
    {
        ArgumentNullException.ThrowIfNull(lines);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(k);

        var counts = CountCodes(lines);
        return SelectTop(counts, k);
    }

    private static Dictionary<string, int> CountCodes(IEnumerable<string> lines)
    {
        var counts = new Dictionary<string, int>(StringComparer.Ordinal);
        foreach (var line in lines)
        {
            if (TryParseCode(line, out var code))
                counts[code] = counts.GetValueOrDefault(code) + 1;
        }
        return counts;
    }

    private static IReadOnlyList<CodeCount> SelectTop(Dictionary<string, int> counts, int k) =>
        counts
            .OrderByDescending(pair => pair.Value)
            .ThenBy(pair => pair.Key, StringComparer.Ordinal)
            .Take(k)
            .Select(pair => new CodeCount(pair.Key, pair.Value))
            .ToList();

    private static bool TryParseCode(string? line, [NotNullWhen(true)] out string? code)
    {
        code = null;
        if (string.IsNullOrWhiteSpace(line)) return false;

        var parts = line.Split(' ', 4, StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
        if (parts.Length < 3 || !parts[1].Equals("ERROR", StringComparison.Ordinal)) return false;

        code = parts[2];
        return true;
    }
}
```

Narration samples while typing: *"Ordinal comparer — codes are identifiers, not words."* · *"`GetValueOrDefault` keeps this to one readable line; `CollectionsMarshal` would avoid the second lookup if this were hot."* · *"`ToList` so the caller gets a materialised read-only list, not a lazy query that re-sorts every time it's enumerated."* · *"`Split` with a count of 4 so a long message isn't split needlessly."*

**Minutes 20–28 — test.** Five cases, stated then run: the prompt's example; ties at position k (`E2` and `E3` both 1 with k = 2 → `E2`); k larger than the number of codes; only malformed lines → empty list; `k = 0` → `ArgumentOutOfRangeException`. During the trace the candidate notices that `"error"` in lowercase is skipped and says: *"That follows the contract — level is case-sensitive. If the real logs mix case I'd switch to `OrdinalIgnoreCase` for the level only."*

**Minutes 28–38 — follow-ups.**

> **Interviewer:** "Suppose there are millions of distinct codes and k is 10."
> **Candidate:** "Then sorting m items is wasteful; a size-k min-heap is O(m log k). The subtle part is keeping the tie-break deterministic, so the heap's priority must be the full ordering, not just the count:"

```csharp
private static IReadOnlyList<CodeCount> SelectTopWithHeap(Dictionary<string, int> counts, int k)
{
    // "Worst" element at the top of the min-heap: lowest count; among equal counts, the larger code.
    var worstFirst = Comparer<(int Count, string Code)>.Create((a, b) =>
    {
        int byCount = a.Count.CompareTo(b.Count);
        return byCount != 0 ? byCount : string.CompareOrdinal(b.Code, a.Code);
    });

    var heap = new PriorityQueue<string, (int Count, string Code)>(k, worstFirst);
    foreach (var (code, count) in counts)
    {
        (int Count, string Code) candidate = (count, code);
        if (heap.Count < k)
            heap.Enqueue(code, candidate);
        else if (heap.TryPeek(out _, out var worst) && worstFirst.Compare(candidate, worst) > 0)
            heap.EnqueueDequeue(code, candidate);
    }

    // Dequeue yields worst-first, so fill the result from the end.
    var result = new CodeCount[heap.Count];
    for (int i = result.Length - 1; i >= 0; i--)
    {
        heap.TryDequeue(out _, out var top);
        result[i] = new CodeCount(top.Code, top.Count);
    }
    return result;
}
```

> **Interviewer:** "And if the logs don't fit on one machine?"
> **Candidate:** "If workers split the input by file, merging their local top-k lists is wrong — a code can be (k+1)-th everywhere and first overall. So I'd partition by hash of the code, so each code's full count lives on one worker; then each worker's top-k is exact and a k-way merge of those lists gives the global answer. If approximate is acceptable and memory is the issue, a Count-Min Sketch plus a heap, or the Space-Saving algorithm, gives bounded error in small memory."

The write-up:

> *"Clarified the two design-relevant questions (size, tie determinism) and stated defaults for the rest, as a written contract. Planned with complexity and chose the simpler sort for the stated size, naming when a heap would win. Clean decomposition, idiomatic modern C# (throw helpers, ordinal comparers, records, Try pattern with nullability attributes). Tested five deliberately chosen cases unprompted, including ties and invalid k. Follow-ups: correct deterministic heap and a correct partitioned merge, with the classic incorrect merge identified. **Strong hire, senior.**"*

**What changed between A and B:** not the algorithm. The candidate spent roughly fifteen extra minutes on contract, narration, tests and follow-ups — the time their pattern recognition had saved them.

---

## Worked example 2 — LRU cache, then "make it thread-safe", then "add expiry"

**Prompt:** *"Implement an LRU cache with get and put in O(1)."* A favourite in LLD, LinkedIn-style and Shopify-style rounds precisely because everyone knows the algorithm — the evaluation is in the API, the invariants and the follow-ups.

### Version 1 — single-threaded, with a deliberate API

Narration of the surface first (Concept 17): *"Generic `LruCache<TKey, TValue>` with `TKey : notnull`; `TryGet` in the BCL Try style rather than returning null, because `TValue` may legitimately be null or a value type; `Set` that inserts or updates; `Capacity` and `Count`. Invariants: capacity positive; Count ≤ Capacity; map and list hold exactly the same keys; list head is most recently used."*

### Version 2 — the thread-safety follow-up

> **Interviewer:** "Now many threads use it."
> **Candidate:** "The invariant spans two structures — the map and the recency list — and in an LRU even a read mutates the list, so a reader-writer lock doesn't help: every operation is a write. The simplest correct design is one lock around each public operation. Uncontended locks are cheap; if contention showed up in profiling, I'd stripe the cache into N independent shards by key hash, at the cost of approximate global LRU."

```csharp
using System.Diagnostics.CodeAnalysis;

public sealed class LruCache<TKey, TValue> where TKey : notnull
{
    private readonly record struct Entry(TKey Key, TValue Value);

    private readonly Dictionary<TKey, LinkedListNode<Entry>> _map;
    private readonly LinkedList<Entry> _recency = new();   // First = most recently used
    private readonly Lock _gate = new();                     // .NET 9+; on .NET 8 use: private readonly object _gate = new();

    public LruCache(int capacity, IEqualityComparer<TKey>? comparer = null)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(capacity);
        Capacity = capacity;
        _map = new Dictionary<TKey, LinkedListNode<Entry>>(capacity, comparer);
    }

    public int Capacity { get; }

    public int Count
    {
        get { lock (_gate) return _map.Count; }
    }

    public bool TryGet(TKey key, [MaybeNullWhen(false)] out TValue value)
    {
        lock (_gate)
        {
            if (_map.TryGetValue(key, out var node))
            {
                MoveToFront(node);
                value = node.Value.Value;
                return true;
            }
        }
        value = default;
        return false;
    }

    public void Set(TKey key, TValue value)
    {
        lock (_gate)
        {
            if (_map.TryGetValue(key, out var existing))
            {
                existing.Value = new Entry(key, value);
                MoveToFront(existing);
                return;
            }

            if (_map.Count == Capacity)
                EvictLeastRecentlyUsed();

            _map[key] = _recency.AddFirst(new Entry(key, value));
        }
    }

    // Callers hold _gate.
    private void MoveToFront(LinkedListNode<Entry> node)
    {
        if (node == _recency.First) return;
        _recency.Remove(node);
        _recency.AddFirst(node);
    }

    private void EvictLeastRecentlyUsed()
    {
        var lru = _recency.Last!;     // non-null: called only when Count == Capacity > 0
        _recency.RemoveLast();
        _map.Remove(lru.Value.Key);
    }
}
```

Points to make out loud:

- **Version 1 is this code without `_gate`** — adding thread safety touched only the public methods, because the invariant-preserving logic was already isolated in private helpers.
- `LinkedList<T>.Remove(node)` and `AddFirst(node)` are O(1) given the node — that's why the map stores nodes.
- `Entry` is a `readonly record struct` — no extra allocation per entry beyond the list node; replacing `node.Value` updates the value in place.
- **The `!` on `_recency.Last`** is justified by the invariant; say so rather than leaving it unexplained.
- **Never call user code inside the lock** — if a follow-up adds an eviction callback, collect evicted entries and invoke callbacks after releasing the lock, or a callback that touches the cache deadlocks or re-enters.

**Testing the concurrent version:**

```csharp
[Fact]
public void Concurrent_use_never_exceeds_capacity()
{
    var cache = new LruCache<int, int>(capacity: 64);

    Parallel.For(0, 200_000, i =>
    {
        int key = i % 500;
        if (i % 3 == 0) cache.Set(key, i);
        else cache.TryGet(key, out _);
    });

    Assert.InRange(cache.Count, 1, cache.Capacity);
}
```

*"This finds gross races probabilistically. To check the map–list invariant directly I'd add an internal `AssertInvariants()` exposed to tests with `InternalsVisibleTo`, and call it after the parallel run."*

### Version 3 — "Add a time-to-live"

> **Candidate:** "Expiry needs a clock, and I want it testable, so I'll inject `TimeProvider`. Each entry stores its expiry; `TryGet` treats an expired entry as a miss and removes it. That's lazy expiry — memory for expired-but-unread entries is reclaimed only on access or eviction; if that mattered I'd add a periodic sweep using `TimeProvider.CreateTimer`."

The change, in diff form:

```csharp
private readonly record struct Entry(TKey Key, TValue Value, DateTimeOffset ExpiresAt);
private readonly TimeProvider _time;
private readonly TimeSpan _ttl;

// constructor gains: TimeSpan ttl, TimeProvider? time = null
//   ArgumentOutOfRangeException.ThrowIfLessThanOrEqual(ttl, TimeSpan.Zero);
//   _ttl = ttl; _time = time ?? TimeProvider.System;

// in TryGet, inside the lock:
if (_map.TryGetValue(key, out var node))
{
    if (_time.GetUtcNow() >= node.Value.ExpiresAt)
    {
        _recency.Remove(node);
        _map.Remove(key);
    }
    else
    {
        MoveToFront(node);
        value = node.Value.Value;
        return true;
    }
}

// in Set: new Entry(key, value, _time.GetUtcNow() + _ttl)
```

And the test that only `TimeProvider` makes possible:

```csharp
[Fact]
public void Entries_expire_after_ttl()
{
    var time = new FakeTimeProvider();     // Microsoft.Extensions.TimeProvider.Testing
    var cache = new LruCache<string, int>(capacity: 10, ttl: TimeSpan.FromMinutes(5), time: time);

    cache.Set("a", 1);
    time.Advance(TimeSpan.FromMinutes(4));
    Assert.True(cache.TryGet("a", out _));

    time.Advance(TimeSpan.FromMinutes(2));
    Assert.False(cache.TryGet("a", out _));
}
```

### Version 4 — the question that tests architectural judgment

> **Interviewer:** "Would you use this in production?"
> **Candidate:** "For an in-process cache in a .NET service, I'd normally use what the platform gives me — `HybridCache` is the current recommended default, with stampede protection and an optional distributed tier (Module 10) — and reserve a hand-written cache for a hot path with a specific eviction requirement the library can't meet. Writing it here is about the data structures and the concurrency; in production, owning a cache is a maintenance cost I'd want a reason for."

That last answer is what turns a well-known exercise into a senior signal.

---

## Worked example 3 — A practical multi-part round: a merchant ledger

**Part 1 (as given):** *"Each line is `id,account,type,amount` where type is `charge` or `refund` and amount is in minor units. Print each account's balance."*
**Part 2 (revealed later):** *"Refunds have a sixth field naming the original charge. A refund may not exceed what remains refundable on that charge. Invalid transactions are rejected and reported."*
**Part 3:** *"Add a currency field. Balances are per account per currency."*
**Part 4:** *"The feed may replay transactions. Process each id at most once."*

### The design decision in part 1 that pays off later

> "Even for part 1 I'll parse into a `Transaction` record and apply it to a `Ledger` class, rather than summing strings in a loop — this looks like it'll grow, and I want each rule to have one home."

### The final code (after part 4), with what each part changed

```csharp
using System.Diagnostics.CodeAnalysis;

public enum TxType { Charge, Refund }

public sealed record Money(long Minor, string Currency);                         // part 3
public sealed record Transaction(string Id, string Account, TxType Type, Money Amount, string? RefundOf);
public sealed record Rejection(string TransactionId, string Reason);

public sealed class Ledger
{
    private readonly Dictionary<(string Account, string Currency), long> _balances = new();
    private readonly Dictionary<string, Transaction> _charges = new(StringComparer.Ordinal);
    private readonly Dictionary<string, long> _refunded = new(StringComparer.Ordinal);       // part 2
    private readonly HashSet<string> _seen = new(StringComparer.Ordinal);                     // part 4
    private readonly List<Rejection> _rejections = [];

    public IReadOnlyList<Rejection> Rejections => _rejections;

    public long Balance(string account, string currency) =>
        _balances.GetValueOrDefault((account, currency));

    public void Apply(Transaction tx)
    {
        if (!_seen.Add(tx.Id)) return;                       // part 4: replays are ignored

        string? error = tx.Type switch
        {
            TxType.Charge => ApplyCharge(tx),
            TxType.Refund => ApplyRefund(tx),
            _ => "unknown transaction type",
        };

        if (error is not null)
            _rejections.Add(new Rejection(tx.Id, error));
    }

    private string? ApplyCharge(Transaction tx)
    {
        if (tx.Amount.Minor <= 0) return "amount must be positive";
        _charges[tx.Id] = tx;
        Post(tx.Account, tx.Amount);
        return null;
    }

    private string? ApplyRefund(Transaction tx)                                        // part 2
    {
        if (tx.RefundOf is null || !_charges.TryGetValue(tx.RefundOf, out var original))
            return "refund references an unknown charge";
        if (original.Account != tx.Account || original.Amount.Currency != tx.Amount.Currency)
            return "refund does not match the original charge";
        if (tx.Amount.Minor <= 0)
            return "amount must be positive";

        long alreadyRefunded = _refunded.GetValueOrDefault(original.Id);
        if (alreadyRefunded + tx.Amount.Minor > original.Amount.Minor)
            return "refund exceeds the refundable amount";

        _refunded[original.Id] = alreadyRefunded + tx.Amount.Minor;
        Post(tx.Account, tx.Amount with { Minor = -tx.Amount.Minor });
        return null;
    }

    private void Post(string account, Money amount)
    {
        var key = (account, amount.Currency);
        _balances[key] = checked(_balances.GetValueOrDefault(key) + amount.Minor);
    }
}

public static class TransactionParser
{
    // id,account,type,amount,currency[,refundOf]
    public static bool TryParse(string line, [NotNullWhen(true)] out Transaction? tx, out string? error)
    {
        tx = null;
        error = null;
        var f = line.Split(',', StringSplitOptions.TrimEntries);
        if (f.Length is < 5 or > 6) { error = "wrong field count"; return false; }
        if (!Enum.TryParse<TxType>(f[2], ignoreCase: true, out var type)) { error = "unknown type"; return false; }
        if (!long.TryParse(f[3], System.Globalization.NumberStyles.Integer,
                           System.Globalization.CultureInfo.InvariantCulture, out var minor)) { error = "bad amount"; return false; }

        string? refundOf = f.Length == 6 && f[5].Length > 0 ? f[5] : null;
        tx = new Transaction(f[0], f[1], type, new Money(minor, f[4].ToUpperInvariant()), refundOf);
        return true;
    }
}
```

### Narration that earns the senior write-up

- **Part 1:** *"Amounts are minor units, so `long`, never `double`. `checked` on posting — a silent overflow in a ledger is the worst kind of bug."*
- **Part 2:** *"The refund rules each get one guard with a specific reason — the report is part of the output. Rejected refunds don't move money."* · *"I track cumulative refunded amount per charge so partial refunds work."*
- **Part 3:** *"Only `Money`, the balance key and the parser changed; the rules didn't — and I added the currency check to refunds, which part 3 implies but doesn't state. Flagging that as an assumption."*
- **Part 4:** *"`HashSet.Add` returning false is the idempotency check. Decision to confirm: a replay of a *rejected* transaction is also ignored. And if a replay arrives with the same id but different content, that's suspicious — in production I'd store a hash of the content and reject mismatches rather than silently ignoring them."*
- **Throughout:** earlier parts' checks stay in `Main` and are rerun after each part. `Enum.TryParse` with `ignoreCase` and `long.TryParse` with `InvariantCulture` are deliberate — parsing must not depend on the server's culture. And `Enum.TryParse` happily accepts numeric strings such as `"7"`, producing an undefined `TxType` value — which is why the `switch` keeps a default arm that rejects it (or the parser could check `Enum.IsDefined`).

**What to notice:** part 3 touched three small places; part 4 touched one line. That's the design evidence the interviewer was waiting for.

---

## Worked example 4 — A code review round in C#

**Prompt:** *"This was submitted as a PR to a pricing service that handles a few hundred requests per second. Review it."*

```csharp
public class PriceService
{
    public static Dictionary<string, decimal> Cache = new Dictionary<string, decimal>();

    public decimal GetPrice(string sku, string currency)
    {
        if (Cache.ContainsKey(sku + currency))
            return Cache[sku + currency];

        var client = new HttpClient();
        var json = client.GetStringAsync(
            "https://prices.internal/api/price?sku=" + sku + "&cur=" + currency).Result;
        var price = JsonSerializer.Deserialize<PriceDto>(json).Amount;

        // night-time discount
        if (DateTime.Now.Hour >= 22 || DateTime.Now.Hour < 6)
            price = price * 0.9m;

        Cache[sku + currency] = price;
        return price;
    }

    public async void RefreshAll(List<string> skus)
    {
        foreach (var sku in skus)
        {
            try { GetPrice(sku, "EUR"); }
            catch (Exception) { }
        }
    }
}
```

### Step 1 — intent and context (said aloud)

> "It returns a SKU's price in a currency, caching it, with a night-time discount. It's on a hot path at a few hundred requests per second, so concurrency and resource use matter more than style."

### Step 2 — the review, in priority order

**Blocking**

1. **`issue (blocking)` — the discounted price is cached.** A price fetched at 22:30 is stored *after* the 10% discount and then served all day. That's a revenue bug, not a style issue. The discount must be applied *after* reading from the cache — and arguably belongs in a pricing-rules component, not in the fetch path.
2. **`issue (blocking)` — the static `Dictionary` is shared across concurrent requests.** `Dictionary<,>` isn't thread-safe; concurrent writes can corrupt it (including infinite loops on enumeration or lookup in some runtimes). It's also a public mutable static field anyone can modify.
3. **`issue (blocking)` — `new HttpClient()` per call plus `.Result`.** Per-call clients exhaust sockets under load; `.Result` blocks a ThreadPool thread per request and will starve the pool at this traffic. Make it `async` end to end with a client from `IHttpClientFactory`.
4. **`issue (blocking)` — the query string isn't encoded.** A SKU containing `&` or `#` changes the request — at best a wrong price, at worst a parameter-injection vector. Use `Uri.EscapeDataString`.

**Important**

5. **`issue` — the cache never expires and grows without bound.** Prices change; memory isn't infinite. Use `HybridCache` or `IMemoryCache` with an expiration and a size limit.
6. **`issue` — cache-key collision.** `sku + currency` makes `("AB", "CEUR")` and `("ABC", "EUR")` the same key. Use a delimiter or a tuple key.
7. **`issue` — `Deserialize` can return `null`** (e.g. the body `null`), so `.Amount` can throw a `NullReferenceException`; there's no timeout or cancellation on the HTTP call.
8. **`issue` — `async void RefreshAll` with no awaits, swallowing every exception.** It isn't actually asynchronous, can't be awaited, hides all failures, and hard-codes EUR. Either `async Task` that awaits `GetPriceAsync` with logging per failure, or remove it.

**Suggestions**

9. **`suggestion` — `DateTime.Now` uses the server's local time zone and is untestable.** Which time zone defines "night" — the customer's, the store's? Inject `TimeProvider` and make the rule explicit.
10. **`suggestion` — `ContainsKey` then indexer** is a double lookup; moot once the cache is replaced.
11. **`nitpick` —** class can be `sealed`; magic `0.9m` and hour values deserve names.

**Praise**

12. **`praise` — `decimal` for money.** Correct choice; keep it.

### Step 3 — summary and decision

> "Four blocking issues: cached discounts, a thread-unsafe shared cache, blocking HTTP with a client per call, and unencoded query parameters. I wouldn't approve as is. Happy to pair on it — here's roughly the shape I'd suggest:"

```csharp
public sealed class PriceService
{
    private readonly HttpClient _http;            // typed client registered via IHttpClientFactory
    private readonly HybridCache _cache;          // Microsoft.Extensions.Caching.Hybrid
    private readonly IPricingRules _rules;        // owns time-dependent rules; uses TimeProvider

    public PriceService(HttpClient http, HybridCache cache, IPricingRules rules)
    {
        _http = http;
        _cache = cache;
        _rules = rules;
    }

    public async Task<decimal> GetPriceAsync(string sku, string currency, CancellationToken ct)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(sku);
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);

        decimal basePrice = await _cache.GetOrCreateAsync(
            $"price:{sku}:{currency}",
            async token => await FetchBasePriceAsync(sku, currency, token),
            new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(5) },
            cancellationToken: ct);

        return _rules.Apply(basePrice, currency);   // discounts applied after the cache, never cached
    }

    private async Task<decimal> FetchBasePriceAsync(string sku, string currency, CancellationToken ct)
    {
        var url = $"api/price?sku={Uri.EscapeDataString(sku)}&cur={Uri.EscapeDataString(currency)}";
        var dto = await _http.GetFromJsonAsync<PriceDto>(url, ct)
                  ?? throw new InvalidOperationException($"No price returned for {sku}/{currency}.");
        return dto.Amount;
    }
}
```

*"The typed client gets its base address, timeout and a resilience pipeline from registration (Module 25). `HybridCache` also gives stampede protection — a hundred concurrent misses on one SKU make one upstream call."*

**What to notice:** the review led with a business-correctness bug that a style-focused reviewer would miss entirely; every comment says *why*; severity is explicit; the reviewer offered a path forward rather than a verdict alone.

---

## Worked example 5 — A Meta-style AI-enabled round (with a debugging phase)

**Setup:** a CoderPad project in C# — `Grid.cs` (parses a maze of `.` open, `#` wall, `S` start, `G` goal), `Solver.cs`, `SolverTests.cs`, and a `data/` folder with small, medium and large maze files. An AI chat panel is available; it can read the files but can't edit them. 60 minutes.

### Phase 1 — "Some tests fail or time out. Find and fix the bug." (minutes 5–17)

The candidate **reads first, without the AI** (Concept 30): *"Tests first — they call `ShortestPath` and expect step counts. `Grid` stores cells in a `char[,]` indexed `[row, col]`; `Point` is a `record struct (int Row, int Col)`. Now the solver:"*

```csharp
public int ShortestPath(Grid grid, Point start, Point goal)
{
    var queue = new Queue<(Point P, int Dist)>();
    queue.Enqueue((start, 0));
    int iterations = 0;

    while (queue.Count > 0)
    {
        if (++iterations > 10_000)
            throw new InvalidOperationException("Too many iterations");

        var (p, dist) = queue.Dequeue();
        if (p == goal) return dist;

        foreach (var next in grid.Neighbours(p))
            queue.Enqueue((next, dist + 1));
    }
    return -1;
}
```

> "There's no visited set, so cells are re-enqueued from every neighbour — the queue grows exponentially in open areas, and the iteration cap converts that into an exception. The cap is masking the bug, not fixing it. Fix: mark cells visited when *enqueued*, which also guarantees each cell is processed once at its shortest distance. I'll remove the cap — BFS on a finite grid with a visited set terminates by construction."

```csharp
var visited = new HashSet<Point> { start };
// …
foreach (var next in grid.Neighbours(p))
    if (visited.Add(next))
        queue.Enqueue((next, dist + 1));
```

Runs the tests: small and medium pass. *"Complexity is now O(rows × cols) time and space. One more thing I noticed: `Neighbours` uses `yield return`, which allocates an enumerator per cell — fine for now, might matter in phase 3."*

### Phase 2 — "Support one-way cells `>`, `<`, `^`, `v`: they can only be *entered* moving in that direction." (minutes 17–38)

The candidate states the design, then delegates a well-specified unit:

> "This only changes neighbour generation: moving from `p` to `q` in direction `d` is allowed if `q` is open, or if `q` is a one-way cell whose arrow equals `d`. Visited stays per cell, because the rule depends on the move, not on state carried along the path. I'll ask the AI for the direction table and the neighbour method, then check it."

**Prompt:**
> *"In C#, given `char[,] cells` indexed [row, col] and `record struct Point(int Row, int Col)`, write `IEnumerable<Point> Neighbours(Point p)` for a 4-connected grid. A move from p to q in direction d (Up/Down/Left/Right) is allowed if q is in bounds and cells[q] is '.', 'S' or 'G', or if cells[q] is a one-way arrow ('^','v','<','>') matching d. Walls '#' are never enterable. Use a static readonly array of (dRow, dCol, arrow) tuples."*

**AI response (excerpt):**

```csharp
private static readonly (int DRow, int DCol, char Arrow)[] Moves =
    [(-1, 0, '^'), (1, 0, 'v'), (0, -1, '<'), (0, 1, '>')];

public IEnumerable<Point> Neighbours(Point p)
{
    foreach (var (dr, dc, arrow) in Moves)
    {
        var q = new Point(p.Row + dr, p.Col + dc);
        if (q.Row < 0 || q.Col < 0 || q.Row >= _cells.GetLength(0) || q.Col >= _cells.GetLength(1)) continue;
        char c = _cells[p.Row, p.Col];
        if (c == '.' || c == 'S' || c == 'G' || c == arrow) yield return q;
    }
}
```

The candidate reviews aloud:

> "Bounds check is right; the direction table is right. But it reads the cell at **`p`**, not **`q`** — it's checking the cell we're leaving. The rule is about the cell we're *entering*. As written, any move out of an open cell is allowed — including straight into a wall — and arrow cells can be entered from any side. Fix: `_cells[q.Row, q.Col]`. I'll also add two tests — a wall next to the start, and an arrow that must block entry from the opposite side — because those are exactly the cases this bug would get wrong."

Adds the test, fixes the line, runs: all pass. Time check at minute 36: *"Phase 2 done. Moving on."*

### Phase 3 — "The large maze (3000 × 3000) must finish in under two seconds." (minutes 38–55)

Runs it: about eight seconds.

> "Asymptotically we're already optimal — O(cells). So this is constant factors: `HashSet<Point>` hashing nine million structs, tuple allocations in the queue, and an enumerator allocated per cell. I'll switch to a flat representation: cell index `r * cols + c` as an `int`, a `bool[]` for visited, an `int[]` for distances, and a `Queue<int>`, with the neighbour loop inlined. I know exactly what I want, so I'll write the loop and ask the AI only to produce the index-arithmetic version of the arrow rule — then check it against the tests we already have."

```csharp
public int ShortestPathFast(char[,] cells, Point start, Point goal)
{
    int rows = cells.GetLength(0), cols = cells.GetLength(1);
    var dist = new int[rows * cols];
    Array.Fill(dist, -1);

    int s = start.Row * cols + start.Col, g = goal.Row * cols + goal.Col;
    var queue = new Queue<int>();
    dist[s] = 0;
    queue.Enqueue(s);

    while (queue.TryDequeue(out int cur))
    {
        if (cur == g) return dist[cur];
        int r = cur / cols, c = cur % cols;

        foreach (var (dr, dc, arrow) in Moves)
        {
            int nr = r + dr, nc = c + dc;
            if ((uint)nr >= (uint)rows || (uint)nc >= (uint)cols) continue;  // one compare per axis

            char target = cells[nr, nc];
            bool enterable = target is '.' or 'S' or 'G' || target == arrow;
            int next = nr * cols + nc;
            if (enterable && dist[next] < 0)
            {
                dist[next] = dist[cur] + 1;
                queue.Enqueue(next);
            }
        }
    }
    return -1;
}
```

> "`dist` doubles as the visited set. The `(uint)` cast folds the negative and upper-bound checks into one comparison. `foreach` over a static array of tuples doesn't allocate."

Large maze: about 0.6 seconds. Tests: green. The candidate closes: *"If we needed more, I'd profile before guessing — likely the `char[,]` access; a flattened `char[]` or a `ReadOnlySpan<char>` per row would be next. And if many queries share a start, I'd run BFS once and answer each query from the `dist` array."*

### What the interviewer can write down

> *"Read the code and tests before using the AI; diagnosed the missing visited set and identified the iteration cap as masking it; removed the cap with a correct argument for termination. Specified the AI request precisely; caught that its neighbour rule checked the source cell instead of the target, explained the consequence, added a test that would have caught it. Phase 3: recognised the solution was already asymptotically optimal and targeted constant factors with a flat-array BFS; verified with existing tests. Narrated throughout. **Strong hire.**"*

Notice what the AI did and didn't do: it produced boilerplate and a table; the candidate owned the diagnosis, the rule, the review, the optimisation insight and the verification.

---
# Common interview questions with model answers

These are the questions *inside* coding rounds — the follow-ups and probes that decide level — plus a few meta-questions architects get about coding.

**Q1. "Here's the problem. How would you approach it?"** *(opening)*
> "Let me check I understand it and pin down the contract first." — then two or three design-changing questions, stated defaults for the rest, a written contract, a baseline with complexity, the chosen approach and why, and a one-line outline of the code.
*Why it works:* the first five minutes produce communication and problem-solving evidence before a single line of code (Concepts 6–7).

**Q2. "Why that data structure?"**
> "I only need O(1) membership checks and never iterate in order, so a `HashSet<T>`. A sorted structure like `SortedSet<T>` would give ordered iteration at O(log n) per operation, which we don't need. If we later needed 'smallest active key', I'd switch to `SortedSet` or a heap — that's the change I'd expect."
*Key signal:* the alternative, its cost, and the condition that would change your mind.

**Q3. "What's the complexity — and can you do better?"**
> "Time O(n log n) from the sort, space O(n). The lower bound for comparison-based sorting is n log n, but we don't need a full order — only the top k — so a size-k heap makes it O(n log k). At our size, n ≈ 10⁴, that's a small constant-factor gain and adds code; I'd only do it if profiling showed this mattered or k were tiny relative to n."
*Key signal:* knowing the bound you're at, the theoretical bound, *and* whether the improvement matters.

**Q4. "How would you test this?"**
> "In three layers. Partition the inputs — empty, one element, ties at the boundary, k larger than the input, invalid k — and test each class plus its boundaries; I've traced the riskiest two already. Then a property-style check: for random inputs, the result should match a brute-force reference implementation. And for production, the parsing would be unit-tested separately from file reading, with one integration test over a sample log."
*Key signal:* a strategy, not a list; the brute-force oracle is a particularly strong touch.

**Q5. "What would you change before shipping this?"**
> "Four things, briefly: metrics for skipped lines so we notice format drift; making the parsing rule configurable or at least isolated; a cap or streaming read if inputs can be unbounded; and tests in CI. I'd *not* add abstractions we don't need yet."
*Key signal:* proportionate production thinking (Concept 16) — operability, failure, tests — and restraint.

**Q6. "Make it thread-safe."**
> "The invariant is that the map and the list change together, and every operation — even reads — mutates the list, so the unit of atomicity is each public method. One `lock` around each, on a private lock object; no user callbacks inside the lock. A `ReaderWriterLockSlim` wouldn't help because there are no pure reads. If contention showed up, I'd shard by key hash into independent caches."
*Key signal:* invariant first, primitive second, known traps named (Concept 20; Worked example 2).

**Q7. "Now it's a billion records."**
> "Two things break: memory for the counts and single-machine time. If the distinct keys fit in memory, a streaming single pass still works — `File.ReadLines` instead of loading everything. If not, partition by hash of the key across workers so each key's count is local, then merge per-worker top-k — merging top-k lists from workers that split by *input* would be wrong. If approximate is acceptable, a Count-Min Sketch with a heap bounds the memory."
*Key signal:* what breaks, a concrete change, and the correctness trap (Concept 15).

**Q8. "Why throw an exception there instead of returning null?"**
> "A non-positive k is a caller bug — the contract says k is positive — so failing fast with `ArgumentOutOfRangeException` surfaces it immediately. A malformed log line, on the other hand, is expected data, so that path returns `false` from `TryParseCode` and we skip it. Exceptions for contract violations, Try-pattern or results for expected outcomes."
*Key signal:* a principled distinction, matching .NET design guidelines (Concept 18).

**Q9. "Why a record? Why not a tuple?"**
> "It's part of a public return type, so named members are clearer for callers than `Item1`/`Item2` or even named tuple elements, which don't survive across all boundaries cleanly. Value equality makes tests a one-line `Assert.Equal`. Inside a private method I'd happily use a tuple."
*Key signal:* fluency with modern C# *and* a sense of where each feature belongs (Concept 23).

**Q10. "Here's a function an AI wrote. Would you merge it?"** *(AI-enabled or code-review round)*
> "Let me read it against the requirement first… It handles the main case, but: it compares strings with the culture-sensitive default where these are identifiers, so it should be ordinal; it creates an `HttpClient` per call; and it catches `Exception` and returns null, which would hide outages. I'd fix those three and add a test for the case-sensitivity behaviour before merging."
*Key signal:* reviewing AI output as untrusted code with the same bar as any PR — and finding the Concept 24 tells.

**Q11. "Explain this code." (pointing at something you pasted from the AI)**
> Line by line, in your own words, including *why* each non-obvious line is there. If you can't: *"I can't fully justify this part, so I'm going to replace it with something I can."*
*Key signal:* ownership. Being unable to explain code on the screen is one of the most damaging signals in an AI-enabled round (Concepts 4, 33).

**Q12. "You have five minutes left and it's not working. What now?"** *(sometimes asked explicitly; often just happens)*
> "Let me tell you where I am: the parsing and counting work and are tested; selection has a bug in the tie-break I haven't isolated. Rather than flail, I'll explain the fix I'd try — I believe the comparer is inverted — and the two tests that would confirm it."
*Key signal:* calm triage and an honest status. Interviewers credit a clear account of what works, what doesn't and what you'd do next far more than a frantic last edit.

**Q13. "Are you done?"**
> "Functionally yes — let me run through the edge cases before I say it's finished." *(then test)*
*Key signal:* never answer "yes" before testing. The question itself is often a hint.

**Q14. "How do you use AI tools when you code day to day?"** *(also a behavioral question — Module 34)*
> "For boilerplate, test scaffolding, unfamiliar APIs and first drafts of well-specified functions — and I review everything as if it came from a new colleague: I read it, run it, and test the edge cases I care about. I'm careful with anything security-sensitive or concurrent, and I don't use it for the core design decisions. The time it saves I spend on review and tests."
*Key signal:* calibrated, specific, accountable — neither dismissive nor uncritical.

**Q15. "When did you last write production code?"** *(architect and staff loops)*
> Answer honestly and specifically: *"Last month — I wrote the outbox dispatcher for our ordering service and reviewed most PRs on the integration platform. I deliberately keep one or two implementation tasks a quarter so my design decisions stay grounded."* If it's been longer, say so and say how you stay close to the code (reviews, prototypes, spikes, a personal project).
*Key signal:* the interviewer is testing whether your designs are grounded in current reality. A strong coding round is the best answer.

**Q16. "ConcurrentDictionary is thread-safe, so this is fine, right?"** *(code-review probe)* — shown `cache.GetOrAdd(key, k => LoadFromDatabase(k))`.
> "Each individual operation is thread-safe, but the value factory can run more than once for the same key under contention — only one result is stored, but we could hit the database twice. If the load is expensive or has side effects, store `Lazy<T>` values so the factory runs once — or, for async loads, use `HybridCache`, which handles stampedes for us."
*Key signal:* knowing the precise guarantee, not the slogan (Concept 20).

---
# Mistakes vs senior signals

| Area | Common mistake | Senior signal |
|---|---|---|
| Opening | Starts coding the moment the problem is recognised | Writes a contract; asks only design-changing questions; states defaults |
| Planning | Codes without stating the approach | Baseline → bottleneck → options → choice, with complexity |
| Algorithm choice | Cleverest solution by reflex | Simplest solution that meets the agreed constraints; says when to switch |
| Time | Perfects one part; runs out of time for testing | Working version by ~25 min; protects the testing slot |
| Narration | Silent, or narrates keystrokes | Narrates decisions and trade-offs; checks in at decision points |
| Structure | One long method | Decomposition that mirrors the explanation |
| API | Tuples, mutable outputs, booleans that change behaviour | Records, read-only outputs, honest nullability, BCL conventions |
| Errors | `catch (Exception) {}`; exceptions as control flow | Throw helpers for contract violations; Try pattern for expected failures |
| Edge cases | Left to the interviewer | Partitioned and named early; tied to the contract |
| Testing | "Done" after the happy path | Deliberate cases, traced or run; own bugs found |
| Debugging | Random edits | Hypothesis → evidence → bisect → root cause → rerun |
| Hints | Ignores or argues | Takes them explicitly; shows understanding |
| Over-engineering | Interfaces and factories for one implementation | Cheapest seam on the likely axis of change; mentions the rest |
| Concurrency | "Use ConcurrentDictionary" | Names the invariant; picks the primitive; knows `GetOrAdd`'s guarantee |
| C# fluency | Dated or Java-flavoured C# | Current, idiomatic, version-aware; knows BCL semantics exactly |
| C# tells | `.Result`, `new HttpClient()`, `DateTime.Now`, culture-default comparisons | Async all the way, factory clients, `TimeProvider`, ordinal comparisons |
| Follow-ups | Vocabulary ("distribute it") | What breaks, a concrete change, the correctness caveat |
| Code review | Twenty style nits | Correctness and security first; severity labels; constructive *why* |
| Unfamiliar code | Edits before reading | Orients via tests and data models; reproduces first |
| Integration | Happy-path HTTP | Timeouts, retry taxonomy, idempotency keys, pagination, partial failure |
| Take-home | Over-built; no README | Focused core, tests, README as a decision record |
| AI-enabled | "Solve this"; pastes unread | Plans, delegates small units, reviews aloud, runs every time |
| AI policy | Assumes | Asks per round; never covert |
| Architect coding | Lectures on architecture instead of coding | Solid senior code with judgment visible inside it |
| After the round | Moves on | After-action review; mistake log |

---
# Practice exercises

1. **Self-diagnosis.** Record a 45-minute two-problem mock in C# without IntelliSense; score it on Appendix E with evidence only (Concept 35).
2. **Contracts only.** For ten problems from your maintenance set, write only the contract and the edge-case partitions in five minutes each. Compare with the problem's official edge cases.
3. **Two write-ups.** Take any problem you've solved; write the mid-level and the senior interviewer write-up for it (Worked example 1). Then solve it again so the senior one would be true.
4. **C# from memory.** Without an IDE, write the harness, `PriorityQueue` top-k with `EnqueueDequeue`, a `TryParse` method with `[NotNullWhen(true)]`, and an xUnit theory. Compile afterwards; count errors; repeat until zero.
5. **Modernise.** Take a C# solution you wrote years ago (or a Java solution) and rewrite it in idiomatic modern C# — records, patterns, throw helpers, ordinal comparisons — explaining each change aloud.
6. **LRU, three ways.** Implement Worked example 2's versions 1–3 from scratch in 60 minutes, with tests, including `FakeTimeProvider`.
7. **Multi-part ledger.** Implement Worked example 3 part by part, revealing each part to yourself only after the previous one passes its tests. Note which parts forced rewrites.
8. **LLD drills.** Rate limiter (fixed window → sliding log → token bucket behind a seam), parking lot, in-memory key-value store with TTL and transactions. 45 minutes each; working code required.
9. **Code review, cold.** Do Worked example 4 without reading the model review; time-box to 25 minutes; compare priorities. Then review three real merged PRs in an open-source .NET repository and compare with the actual review comments.
10. **Plant and find.** Ask a friend — or an AI — to plant three bugs (an off-by-one, a culture-sensitive comparison, a missing visited set) in a small codebase you haven't seen; find them in 20 minutes using the Concept 30 process.
11. **Integration.** Call a public paginated REST API with `HttpClient`, records and `System.Text.Json`; add a timeout, retry for 429/5xx with jitter via `Microsoft.Extensions.Http.Resilience`, and a test with a fake `HttpMessageHandler`.
12. **AI-enabled, chat-only.** Set up a multi-file C# starter project; use an AI only through a chat window (no inline completion); do bug fix → feature → optimise in 60 minutes, narrating both conversations (Worked example 5).
13. **AI-enabled, own IDE.** Empty repo, your IDE and AI agent; build a small app (robot on a grid; `tail -n`) with tests in 45 minutes; then have a partner add two follow-up requirements.
14. **Weak-AI drill.** Repeat exercise 12 with a small model or AI off for the core algorithm; note where you relied on it.
15. **Mock calibration.** One mock per target company's actual format (classic, practical, LLD, code review, AI-enabled), scored on Appendix E by the partner and by you; reconcile the two scores and log one fix.

---
# Free resources and learning material

All free to read online unless marked *(book)* or *(partly paid)*. Start with the ★ items. Company formats and policies were checked on October 8, 2026; they change quickly, so confirm with your recruiter.

### How coding rounds are scored
- ★ [How candidates are evaluated in coding interviews — Tech Interview Handbook](https://www.techinterviewhandbook.org/coding-interview-rubrics/) — the four dimensions and their bands, collated from big-tech rubrics; includes a printable practice rubric.
- ★ [Coding interview best practices cheatsheet — Tech Interview Handbook](https://www.techinterviewhandbook.org/coding-interview-cheatsheet/) — before/during/after behaviours mapped to the rubric.
- [Techniques to solve coding questions — Tech Interview Handbook](https://www.techinterviewhandbook.org/coding-interview-techniques/).
- [Picking a programming language — Tech Interview Handbook](https://www.techinterviewhandbook.org/programming-languages-for-coding-interviews/).
- [Mock coding interviews — Tech Interview Handbook](https://www.techinterviewhandbook.org/mock-interviews/).
- ★ [SDE III / Senior SDE interview prep — Amazon](https://amazon.jobs/content/en/how-we-hire/sde-iii-interview-prep) — syntactically correct code, edge cases, the six coding objectives, and low-level design.
- [How we hire — Google](https://www.google.com/about/careers/applications/how-we-hire) and [Interview tips — Google](https://www.google.com/about/careers/applications/interview-tips).

### AI-enabled coding rounds
- ★ [Meta's AI-enabled coding interview: how to prepare — Hello Interview](https://www.hellointerview.com/blog/meta-ai-enabled-coding) — environment, three phases, the four competencies, candidate reports.
- ★ [Using AI in Meta's AI-assisted coding interview, with real prompts — interviewing.io](https://interviewing.io/blog/how-to-use-ai-in-meta-s-ai-assisted-coding-interview-with-real-prompts-and-examples).
- ★ [Shopify's AI coding interview — Hello Interview](https://www.hellointerview.com/blog/shopify-ai-enabled-coding) — own IDE, empty repo, design and testing emphasis.
- [LinkedIn's AI-enabled coding interview — Hello Interview](https://www.hellointerview.com/blog/linkedin-ai-enabled-coding) — familiar problems, deep concurrency and production follow-ups.
- ★ [Yes, you can use AI in our interviews. In fact, we insist — Canva Engineering](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/) — the clearest first-party explanation of why and how AI-enabled interviews are designed.
- [Google's AI-assisted coding interview (2026 guide) — Aced (formerly Exponent)](https://www.tryexponent.com/blog/google-ai-coding-interview) — the Gemini code-comprehension pilot.
- [AI coding interview guide — Hello Interview](https://www.hellointerview.com/learn/ai-coding/overview/introduction) *(partly paid)* — with free fundamentals chapters:
  - [Codebase orientation](https://www.hellointerview.com/learn/ai-coding/fundamentals/codebase-orientation)
  - [Driving the AI](https://www.hellointerview.com/learn/ai-coding/fundamentals/driving-the-ai)
  - [Verification and testing](https://www.hellointerview.com/learn/ai-coding/fundamentals/verification-and-testing)
  - [Communication](https://www.hellointerview.com/learn/ai-coding/fundamentals/communication)
- [The AI coding interview: a complete 2026 guide — PracHub](https://prachub.com/resources/ai-coding-interview-guide) and [Meta's AI-enabled coding interview — PracHub](https://prachub.com/resources/meta-ai-coding-interview).
- [How hard is it to cheat with ChatGPT in technical interviews? — interviewing.io](https://interviewing.io/blog/how-hard-is-it-to-cheat-with-chatgpt-in-technical-interviews) — the experiment behind many companies' format changes.
- ★ [Guidance on candidates' AI usage — Anthropic](https://www.anthropic.com/candidate-ai-guidance) — the clearest statement of the "AI for preparation, not live answers unless told" norm.

### Low-level design and practical rounds
- [Low-level design in a hurry — Hello Interview](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/introduction) *(partly paid)*.
- [Concurrency for low-level design — Hello Interview](https://www.hellointerview.com/learn/low-level-design/concurrency/intro) *(partly paid)*.
- [How to prepare for a low-level design interview — Hello Interview](https://www.hellointerview.com/blog/how-to-prepare-lld).
- ★ [awesome-low-level-design — GitHub](https://github.com/ashishps1/awesome-low-level-design) — problems and solutions, several with C# implementations.
- ★ [Design patterns with C# examples — Refactoring.Guru](https://refactoring.guru/design-patterns/csharp) — Strategy, State, Observer, Command, Decorator with C# code.
- [Refactoring catalog — Refactoring.Guru](https://refactoring.guru/refactoring/catalog) — vocabulary for refactoring rounds.

### Code review rounds
- ★ [Google engineering practices: code review](https://google.github.io/eng-practices/review/) — especially [the standard of code review](https://google.github.io/eng-practices/review/reviewer/standard.html), [what to look for](https://google.github.io/eng-practices/review/reviewer/looking-for.html) and [how to write comments](https://google.github.io/eng-practices/review/reviewer/comments.html).
- ★ [Conventional Comments](https://conventionalcomments.org/) — labels for severity and intent.
- [How to crack the Google code review interview — IGotAnOffer](https://igotanoffer.com/en/advice/google-code-review-interview).
- [Code review practice — Hello Interview](https://www.hellointerview.com/practice/code-review) *(partly paid)*.
- [.NET code analysis overview — Microsoft Learn](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/overview) — the analyzer rules (CA*) encode many of the Concept 24 tells.

### C# and .NET — current state
- [What's new in C# 14 — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14), [C# 13](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13), [C# 12](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12) — know what's available in which version.
- [What's new in .NET 10 — Microsoft Learn](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview).
- [What's new in .NET 11 (preview) — Microsoft Learn](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/overview) and [.NET Conf 2026 announcement](https://devblogs.microsoft.com/dotnet/dotnet-conf-2026/).
- [Collection expressions — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions).
- [Pattern matching overview — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching).
- [Primary constructors tutorial — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/whats-new/tutorials/primary-constructors) — including the captured-parameter semantics.
- ★ [Nullable reference types — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/nullable-references) and [nullable-analysis attributes](https://learn.microsoft.com/dotnet/csharp/language-reference/attributes/nullable-analysis) (`NotNullWhen`, `MaybeNullWhen`).
- [C# coding conventions — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions).

### Collections and concurrency
- ★ [PriorityQueue<TElement,TPriority> — API reference](https://learn.microsoft.com/dotnet/api/system.collections.generic.priorityqueue-2) — read the remarks on ordering and `EnqueueDequeue`.
- [Selecting a collection class — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/collections/selecting-a-collection-class).
- [CollectionsMarshal — API reference](https://learn.microsoft.com/dotnet/api/system.runtime.interopservices.collectionsmarshal).
- [Frozen collections — API reference](https://learn.microsoft.com/dotnet/api/system.collections.frozen).
- [Thread-safe collections — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/collections/thread-safe/).
- ★ [ConcurrentDictionary.GetOrAdd — API reference](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentdictionary-2.getoradd) — the remarks state that the factory can be called more than once.
- [The lock statement — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/lock) — including the `System.Threading.Lock` type.
- ★ [Async guidance — David Fowler](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md) — the canonical list of async anti-patterns.

### Errors, strings, time, HTTP
- ★ [Best practices for exceptions — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions).
- [Framework design guidelines — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/design-guidelines/) and [exception throwing](https://learn.microsoft.com/dotnet/standard/design-guidelines/exception-throwing) — the source of BCL API conventions.
- ★ [Best practices for comparing strings — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/base-types/best-practices-strings) — ordinal vs culture-sensitive, the Turkish "i".
- ★ [What is TimeProvider? — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview) and [FakeTimeProvider — API reference](https://learn.microsoft.com/dotnet/api/microsoft.extensions.time.testing.faketimeprovider).
- ★ [HttpClient guidelines — Microsoft Learn](https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines) and [IHttpClientFactory — Microsoft Learn](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory).
- [Build resilient HTTP apps — Microsoft Learn](https://learn.microsoft.com/dotnet/core/resilience/http-resilience).
- [System.Text.Json overview — Microsoft Learn](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview).
- [HybridCache in ASP.NET Core — Microsoft Learn](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid).
- [Falsehoods programmers believe about time — Noah Sussman](https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time) and [about names — Patrick McKenzie](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/) — edge-case generators.

### Testing in .NET
- ★ [Unit testing best practices for .NET — Microsoft Learn](https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices).
- [Microsoft.Testing.Platform overview — Microsoft Learn](https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro) and [MTP supported by all major frameworks — .NET Blog](https://devblogs.microsoft.com/dotnet/mtp-adoption-frameworks/).
- ★ [xUnit.net v3 getting started](https://xunit.net/docs/getting-started/v3/getting-started) and [Microsoft Testing Platform support in xUnit v3](https://xunit.net/docs/getting-started/v3/microsoft-testing-platform).
- [NUnit documentation](https://docs.nunit.org/) — worth a skim because CoderPad's C# environment has historically bundled NUnit.
- [TUnit — GitHub](https://github.com/thomhurst/TUnit).
- [NSubstitute](https://nsubstitute.github.io/).
- [AwesomeAssertions](https://awesomeassertions.org/) — the Apache-2.0 fork of FluentAssertions; [FluentAssertions repository](https://github.com/fluentassertions/fluentassertions) for its licence change.
- [Shouldly — GitHub](https://github.com/shouldly/shouldly) and [Verify (snapshot testing) — GitHub](https://github.com/VerifyTests/Verify).
- [Testcontainers for .NET](https://dotnet.testcontainers.org/) — for take-homes with real dependencies.
- ★ [The practical test pyramid — Martin Fowler's site](https://martinfowler.com/articles/practical-test-pyramid.html) and [Mocks aren't stubs — Martin Fowler](https://martinfowler.com/articles/mocksArentStubs.html).
- [Equivalence partitioning](https://en.wikipedia.org/wiki/Equivalence_partitioning) and [boundary-value analysis — Wikipedia](https://en.wikipedia.org/wiki/Boundary-value_analysis).
- [FsCheck — property-based testing for .NET](https://fscheck.github.io/FsCheck/).

### Reading unfamiliar code (practice material)
- [dotnet/runtime](https://github.com/dotnet/runtime) and [dotnet/aspnetcore](https://github.com/dotnet/aspnetcore) — read merged PRs *with their review threads*.
- [dotnet/eShop](https://github.com/dotnet/eShop) — a realistic multi-service reference app.
- [Polly — GitHub](https://github.com/App-vNext/Polly) — a compact, well-tested library to practise orienting in.
- [DeepWiki](https://deepwiki.com/) — AI-generated overviews of GitHub repos; useful for practising "verify the AI's summary against the code".
- [Visual Studio debugger feature tour — Microsoft Learn](https://learn.microsoft.com/visualstudio/debugger/debugger-feature-tour).

### Keeping the algorithmic floor (maintenance only)
- ★ [Grind 75 — Tech Interview Handbook](https://www.techinterviewhandbook.org/grind75/) — configurable by weeks and hours.
- [NeetCode practice (NeetCode 150)](https://neetcode.io/practice).
- [Algorithms study cheatsheets — Tech Interview Handbook](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/).
- [LeetCode](https://leetcode.com/) *(partly paid)* — C# supported.
- [cp-algorithms](https://cp-algorithms.com/) — rigorous references when a follow-up goes deep.
- [Exercism C# track](https://exercism.org/tracks/csharp) — free mentored exercises; excellent for idiomatic modern C# rather than speed.

### Mocks
- [Aced (formerly Exponent) practice](https://www.aced.io/practice) — peer and AI mock interviews, free to start (the platform Pramp moved to in 2024).
- [interviewing.io](https://interviewing.io/) *(paid mocks; free guides)*.
- An AI assistant as a mock interviewer — instruct it to withhold solutions, ask only follow-ups, and flag untested claims.

### Books
- *(book)* John Ousterhout — *A Philosophy of Software Design* — deep modules, complexity; the best single book on "proportionate" design.
- *(book)* Krzysztof Cwalina, Jeremy Barton, Brad Abrams — *Framework Design Guidelines* (3rd ed.) — the reasoning behind BCL API conventions.
- *(book)* Bill Wagner — *Effective C#* — idioms that read as senior.
- *(book)* Jon Skeet — *C# in Depth* — language semantics, precisely.
- *(book)* Vladimir Khorikov — *Unit Testing Principles, Practices, and Patterns* — fakes vs mocks, what to test; examples in C#.
- *(book)* Dustin Boswell & Trevor Foucher — *The Art of Readable Code*.
- *(book)* Gayle Laakmann McDowell, Mike Mroczka, Aline Lerner, Nil Mamano — *Beyond Cracking the Coding Interview* (2025) — the current-generation interview book, including how interviewers think.
- *(book)* David J. Agans — *Debugging: The 9 Indispensable Rules*.

### Previous modules to revisit
- Modules 14–17 — CLR, async/concurrency, modern C#, performance: the knowledge behind Part D.
- Module 10 — caching (HybridCache) for the LRU "would you use this in production?" answer.
- Modules 11, 13, 25 — idempotency, retries, Polly for the integration round.
- Module 30 — options-and-consequences reasoning, compressed into spoken trade-offs.

---
# Quick-recall sheet

**One sentence.** A senior coding round is a 45-minute work sample: the algorithm is the floor; level comes from the contract, the API, edge cases, verification, trade-offs, follow-ups — and, in AI rounds, judgment about AI output.

**Rubric.** Communication · problem solving · technical competency · testing → verification in AI rounds. Dimensions interact; red flags outweigh polish; only writable evidence counts.

**Seniority shift.** Optimal solution = threshold. Differentiators: unprompted clarification, edge cases, tests, simpler-design judgment, scale/concurrency follow-ups. Staff/architect: one round, must-not-fail; show judgment *inside* the code.

**Red flags.** "Done" untested · arguing with correct feedback · long silence · can't explain code · pseudocode · ignored requirement · covert AI.

**Formats.** Classic · practical multi-part · LLD · code review · debugging · integration · refactoring · take-home · AI-enabled. Axes: who wrote the code · how open · who types.

**Contract.** Input shape · size · validity · output semantics (order, ties, fewer than k) · parameter ranges · context. Ask design-changing questions; state the rest as defaults; write it down.

**Plan.** Baseline → bottleneck → options → choice (often simpler, with switch condition) → code outline. Get a nod.

**Time.** 0–5 contract · 5–10 plan · 10–25 code · 25–32 test · 32–40 follow-ups · 40–45 questions. Working first. Checkpoint every 10 min. Cut polish, not testing.

**Edge cases.** Partition, then boundaries (0, 1, k, n, n+1). Families: size · values (overflow!) · order · structure · text (ordinal, Unicode) · time · concurrency · contract.

**Testing.** Trace like a debugger → run early and often → parameterised tests. Order: prompt example · riskiest case · boundary · invalid · large.

**Debugging.** Symptom → mechanism hypothesis → cheap evidence → bisect → root cause → rerun all. Narrate.

**Communication.** Decisions not keystrokes · intent/status around blocks · check in · treat interviewer questions as signal · never silent.

**Unstuck.** Restate · tiny instance · relax a constraint · change representation · work backwards · correct baseline. Take hints explicitly.

**Follow-ups.** What breaks · concrete change · correctness caveat · does it matter at our size.

**Quality order.** Correct → clear → robust at boundaries → efficient enough → seam where change is likely → polish. No abstraction without a nameable second implementation.

**API.** General in, specific read-only out · records over tuples · honest nullability · return-vs-throw deliberate · minimal surface · `sealed` · BCL naming · `CancellationToken` last · state invariants.

**Errors.** Contract violation → throw helpers (`ThrowIfNull`, `ThrowIfNegativeOrZero`…) · expected outcome → Try pattern/result · validate once at boundaries · never swallow · `throw;`.

**Concurrency.** Invariant → unit of atomicity → simplest primitive (`lock`) · RW lock useless for LRU · `GetOrAdd` factory may run twice (→ `Lazy<T>`) · no check-then-act · no `await` in `lock` · no callbacks under lock · test with `Parallel.For` + invariants.

**C# environment.** Check version, entry point, test framework (NUnit in CoderPad historically), nullable. Harness and collections from memory.

**Collections.** `Dictionary` unordered, indexer throws, `TryGetValue` · `HashSet.Add` returns bool · `PriorityQueue` unstable min-heap, no decrease-key, `TryPeek(out e, out p)`, `EnqueueDequeue` · `SortedSet.GetViewBetween` · `LinkedList` nodes for LRU · `FrozenDictionary` for read-mostly.

**Modern C#.** Records · switch expressions/patterns · collection expressions · `required`/`init` · nullable · raw strings · careful: primary constructors (mutable captures), LINQ (multiple enumeration, O(n·m)) · later: `Span`, previews.

**Junior tells.** Swallowed exceptions · `throw ex` · `.Result` · `async void` · `new HttpClient()` · `DateTime.Now` · culture-default comparisons · string `+=` in loops · double lookups · `Count() > 0` · mutable statics · magic numbers · overflow · `lock(this)`.

**Testing in .NET.** xUnit v3 `[Theory]`/`[InlineData]` · `TimeProvider` + `FakeTimeProvider` · fakes over mocks · NSubstitute/Moq · FluentAssertions v8 commercial → AwesomeAssertions/Shouldly · all on Microsoft.Testing.Platform.

**Playbooks.**
- *Classic:* contract anyway · pattern + complexity in one breath · steady clean pace · test · spend surplus on evidence.
- *Multi-part:* domain types from part 1 · keep green · restate new rules · refactor at boundaries.
- *LLD:* scope · entities/responsibilities · public API first · one seam · one path end to end · working code.
- *Code review:* intent → skim → correctness, security, concurrency, failure, performance, API, tests, style · severity labels · why + fix · praise · summary + decision.
- *Debugging:* tests and data models first · reproduce · root cause · regression test · mention adjacent issues.
- *Integration:* docs · records + STJ · shared client · timeouts · 4xx vs 429/5xx · idempotency keys · pagination · partial failure.
- *Take-home:* scope + AI policy in writing · timebox · focused core · README as decision record.
- *AI-enabled:* orient and plan yourself · small specified prompts · review aloud · run every time · write the crux · reject what you can't explain · narrate both conversations.

**AI policy.** Ask per round · default no · never covert · disclose in take-homes · AI for prep is fine.

**Prep.** Diagnose against the rubric · four weeks: C# fluency, edge cases, LLD, review, debugging, integration, AI rounds · maintenance set of 40–60 problems re-solved under protocol · mistake log.

**Day of.** Confirm format, AI policy, environment, version · open with policy + restate + contract · close with questions about review culture · after-action review within an hour.

---
# Appendix A — The coding-round protocol card

```text
BEFORE TYPING (≤10 min)
[ ] AI allowed in this round?  Must the code run?
[ ] Restate the problem in one sentence
[ ] Contract: input shape · size · validity · output order/ties/short results · parameter ranges · context
[ ] Ask only design-changing questions; state the rest as defaults
[ ] Write the contract as a comment
[ ] Edge cases named (partitions + boundaries)
[ ] Plan: baseline → bottleneck → options → choice (+ when I'd switch) → code outline
[ ] Interviewer nod

WHILE CODING (to ~min 25)
[ ] Signature / public surface first, explained
[ ] Decompose like the explanation; names from the problem
[ ] Guard clauses at public boundaries; Try pattern for expected failures
[ ] Narrate decisions, not keystrokes
[ ] Clock check at ~15 and ~25

VERIFY (protect this slot)
[ ] Prompt example
[ ] Riskiest edge case for MY implementation
[ ] A boundary
[ ] An invalid input
[ ] (if runnable) a large input
[ ] Fix → rerun everything
[ ] Only now: "I think it's done"

FOLLOW-UPS
[ ] What breaks · concrete change · correctness caveat · does it matter at our size
[ ] Production gaps named (not built)

CLOSE
[ ] Questions about code review, AI use, ownership
[ ] After-action review within the hour
```

---
# Appendix B — Edge-case checklist

```text
SIZE         empty · one · two · exactly k · k±1 · very large · maximum allowed
VALUES       0 · negative · int.MinValue/MaxValue · overflow in sums/products/midpoints · duplicates · all equal
             floating point: NaN, ±0, precision (never double for money)
ORDER        sorted · reverse · nearly sorted · already processed · stable order required?
STRUCTURE    null · null elements · cycles · self-loops · disconnected · single node · deep recursion (stack overflow)
TEXT         "" · whitespace · case · leading/trailing spaces · delimiters inside fields · very long
             Unicode: surrogate pairs (emoji), combining marks, normalisation · culture (ordinal for identifiers)
NUMBERS      parsing culture (InvariantCulture) · leading zeros · exponent formats · numeric enum strings
TIME         time zones · DST gaps/overlaps · leap day · equal timestamps · clock going backwards · UTC vs local
GRIDS/GRAPHS out of bounds · walls at start/goal · start == goal · unreachable goal · multiple shortest paths
CONCURRENCY  simultaneous calls · re-entrancy · cancellation mid-operation · callbacks under lock
I/O & NET    timeout · partial response · 429 · 5xx · duplicate delivery · pagination end · empty page
CONTRACT     invalid parameters · fewer results than requested · ties at the cutoff · malformed records
```

---
# Appendix C — Code review checklist for .NET

```text
0. CONTEXT     What is it for? Hot path? Public API? Who calls it?
1. CORRECTNESS logic · edge cases · off-by-one · null · comparisons (ordinal?) · invariants · business rules in the right place
2. SECURITY    injection (SQL, query string, path, command) · secrets · authz checks · logging PII · unsafe deserialisation
3. CONCURRENCY shared mutable state · static fields · Dictionary across threads · check-then-act · GetOrAdd side effects
   & RESOURCES .Result/.Wait · async void · HttpClient per call · undisposed IDisposable · unbounded growth
4. FAILURE     swallowed exceptions · throw ex · timeouts · retries (only idempotent) · partial failure · null from deserialise
5. PERFORMANCE O(n²) · N+1 queries · allocations in hot loops · multiple enumeration · string concatenation · sync I/O
6. API/DESIGN  naming · mutability of outputs · leaky abstractions · wrong layer · nullability annotations · sealed
7. TESTS       present? behaviour not implementation? edge cases? deterministic (TimeProvider, seeded Random)?
8. STYLE       last, lightly; prefer analyzers/.editorconfig to comments

LABELS  issue (blocking) · issue · suggestion · question · nitpick (non-blocking) · praise
CLOSE   N blocking, M important, nits; decision; offer to pair
```

---
# Appendix D — C# interview snippet kit (type these from memory)

```csharp
// Harness
public static class Solution
{
    public static void Main() { /* Check(...) calls */ }
    static void Check<T>(string name, T expected, T actual) =>
        Console.WriteLine($"{(EqualityComparer<T>.Default.Equals(expected, actual) ? "PASS" : "FAIL")} {name}: {expected} vs {actual}");
    static void CheckSeq<T>(string name, IEnumerable<T> expected, IEnumerable<T> actual) =>
        Console.WriteLine($"{(expected.SequenceEqual(actual) ? "PASS" : "FAIL")} {name}: [{string.Join(",", expected)}] vs [{string.Join(",", actual)}]");
}

// Guards (.NET 6/7/8+)
ArgumentNullException.ThrowIfNull(x);
ArgumentException.ThrowIfNullOrWhiteSpace(s);
ArgumentOutOfRangeException.ThrowIfNegative(n);
ArgumentOutOfRangeException.ThrowIfNegativeOrZero(k);
ArgumentOutOfRangeException.ThrowIfGreaterThan(k, max);

// Try pattern
bool TryParseX(string s, [NotNullWhen(true)] out X? result) { result = null; /* … */ return false; }

// Dictionary counting / grouping
counts[key] = counts.GetValueOrDefault(key) + 1;
if (!map.TryGetValue(key, out var list)) map[key] = list = new List<int>();
list.Add(value);

// Heaps
var minHeap = new PriorityQueue<string, int>();
var maxHeap = new PriorityQueue<string, int>(Comparer<int>.Create((a, b) => b.CompareTo(a)));
minHeap.Enqueue("x", 3);
if (minHeap.TryPeek(out var e, out var p)) { }
minHeap.EnqueueDequeue("y", 5);          // push then pop min
while (minHeap.TryDequeue(out var el, out var pr)) { }

// Ordering with tie-break
var top = items.OrderByDescending(i => i.Score).ThenBy(i => i.Name, StringComparer.Ordinal).Take(k).ToList();

// BFS skeleton on a grid
var dirs = new (int dr, int dc)[] { (-1, 0), (1, 0), (0, -1), (0, 1) };
var queue = new Queue<(int r, int c)>();
var seen = new bool[rows, cols];
queue.Enqueue((sr, sc)); seen[sr, sc] = true;
while (queue.TryDequeue(out var cur))
{
    foreach (var (dr, dc) in dirs)
    {
        int nr = cur.r + dr, nc = cur.c + dc;
        if ((uint)nr >= (uint)rows || (uint)nc >= (uint)cols || seen[nr, nc]) continue;
        seen[nr, nc] = true;
        queue.Enqueue((nr, nc));
    }
}

// Binary search midpoint without overflow
int mid = lo + (hi - lo) / 2;

// Strings
s.Equals(t, StringComparison.Ordinal);
new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
var sb = new StringBuilder(); sb.Append(x); string result = sb.ToString();

// Records
public sealed record Point(int Row, int Col);
var moved = p with { Col = p.Col + 1 };

// Switch expression over types
string Describe(Shape s) => s switch
{
    Circle { Radius: > 10 } => "big circle",
    Circle => "circle",
    Square sq => $"square {sq.Side}",
    _ => throw new ArgumentOutOfRangeException(nameof(s)),
};

// xUnit theory
[Theory]
[InlineData(new[] { 1, 2, 3 }, 6)]
[InlineData(new int[0], 0)]
public void Sums(int[] input, int expected) => Assert.Equal(expected, input.Sum());

// NUnit equivalent (common in online editors)
[TestCase(new[] { 1, 2, 3 }, 6)]
public void Sums(int[] input, int expected) => Assert.That(input.Sum(), Is.EqualTo(expected));
```

---
# Appendix E — Self-scoring rubric for practice rounds

Score each 1–4 (1 strong no hire, 2 lean no hire, 3 lean hire, 4 strong hire) **from observable evidence only**. A senior target is 3+ on every row and 4 on at least three.

| Dimension | 4 — strong hire (senior) | 3 — lean hire | 2 — lean no hire | 1 — strong no hire |
|---|---|---|---|---|
| **Clarification** | Design-changing questions + stated defaults + written contract | Asked some questions | Asked only when stuck | None |
| **Problem solving** | Optimal or right-sized approach unprompted; alternatives with trade-offs; correct complexity | Reached it with minor hints | Major hints; complexity shaky | Couldn't reach a working approach |
| **Code quality** | Clean decomposition, idiomatic current C#, deliberate API, proportionate error handling | Working, readable, some clumsiness | Monolithic, junior tells, syntax errors | Didn't produce working code |
| **Testing / verification** | Deliberate cases traced/run unprompted; found own bugs; nothing left for the interviewer | Tested when asked or partially | Happy path only | Declared done untested; obvious bugs |
| **Communication** | Narrated decisions throughout; interviewer never lost; checked in | Mostly clear; some gaps | Long silences; hard to follow | Silent or incoherent |
| **Follow-ups** | Concrete changes with correctness caveats; production gaps named | Reasonable but generic | Vocabulary without mechanism | Unable to engage |
| **AI use** *(AI rounds only)* | Planned first; small specified prompts; reviewed aloud; ran every time; could explain all | Used well but review thin | Pasted with little review | Prompted through without understanding |

```text
Round: ______  Format: ______  Date: ______
Clar __  PS __  Code __  Test __  Comm __  Follow __  AI __
Bugs I wrote: ___________________   Found by me: ___   Found by interviewer: ___
Edge cases missed: ___________________
Junior C# tells present: ___________________
One fix for next time: ___________________
```

---
# Appendix F — AI-enabled round card

```text
SETUP       Confirm: which tools/models · can it edit files? · tests runnable? · any "no AI" phases?
ORIENT      Read README, tests, data models, entry point MYSELF (5 min)
PLAN        Say the approach and the split: "I'll write X; AI does Y"
PROMPT      Types + constraints + exact behaviour + edge cases; one unit at a time
REVIEW      Read every line aloud-ish: matches spec? Concept 24 tells? edge cases? complexity?
RUN         After EVERY integration. Never stack unrun pieces.
OWN         Write the crux myself. Reject what I can't explain.
NARRATE     Intent before each prompt; judgment after each response
WEAK AI?    Switch model once, then carry it myself — don't burn time arguing with it
CLOSE       Explain the final code end to end; name what I'd harden for production
```

---

*Next: **Module 37 — Worked system design problems with .NET-specific implementation notes**: URL shortener, rate limiter, notification system, distributed cache and order/payment system — end-to-end designs using the method from Modules 3–5 and the platform depth from Phases 3–6, with the .NET implementation choices an interviewer will probe. Several of this module's coding problems — the rate limiter, the LRU cache, the ledger — reappear there at system scale.*
