# Module 3 — The 7-Step Delivery Framework, Applied End to End
*Phase 2: The System Design Method · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Modules 1 and 2 told you *what* gets scored and *how the shape of the evaluation changes* between a senior IC loop and an architect loop. This module is the method that turns that knowledge into a 45- or 60-minute performance. Every design round you will ever sit — product architecture, infrastructure design, an architect's "migrate this estate" case, a Google-style NALSD round, even a low-level design round — collapses into the same seven moves:

1. **Requirements** — functional and non-functional, quantified
2. **Estimation** — only the numbers that change a decision
3. **API design** — the contract, preceded by a one-minute list of core entities
4. **Data model** — access patterns first, then storage, keys and indexes
5. **High-level design** — a working end-to-end system, built requirement by requirement
6. **Deep dives** — two or three components, in real depth, with trade-offs (the step that carries the most weight)
7. **Wrap-up** — bottlenecks, failure modes, operations, evolution

The framework is not a checklist to recite. It is a **dependency chain** (each step's output is the next step's input), a **time budget** (so you deliver a working system *and* get to the deep dives where seniority is visible), and a **protocol** for keeping you and the interviewer looking at the same thing. We build it one concept at a time: first *why* it exists and how it is scored, then each step in depth, then the cross-cutting techniques that separate senior from mid-level delivery, then a full 45-minute worked example on Azure and .NET, and finally an architect-round variant.

Modules 4 and 5 go deeper into steps 1 and 2. Phase 3 (Modules 6–13) supplies the raw material for step 6. This module is the skeleton everything else hangs on.

| # | Concept | The one-line takeaway |
|---|---|---|
| **Part A — Why a framework** | | |
| 1 | The problem the framework solves | Open prompt + hard clock + rubric = you need a track to run on. |
| 2 | What the interviewer records | Every minute should produce scorable evidence on a rubric dimension. |
| 3 | Why these steps, in this order | Each step consumes the previous step's output. The order is a default; the dependencies are the law. |
| 4 | The time budget | High-level design on the board by ~minute 25 of 45, or you cut scope out loud. |
| 5 | Who drives | Mid-level is led; senior leads; staff and architects frame the problem before solving it. |
| **Part B — The seven steps** | | |
| 6 | Step 1 — Requirements | Top three functional, explicit out-of-scope, three to five quantified non-functional, and the one that will drive the design. |
| 7 | Step 2 — Estimation | Estimate to decide, not to decorate. Every number ends with "so…". |
| 8 | The bridge — core entities | Name the nouns in a minute; names are a design signal. |
| 9 | Step 3 — API design | The contract that the rest of the design must satisfy; get the safety details right (identity, idempotency, pagination, async). |
| 10 | Step 4 — Data model | Access patterns choose the store; the partition key is the most consequential line you'll write. |
| 11 | Step 5 — High-level design | Simplest thing that satisfies every functional requirement, built endpoint by endpoint; park optimizations visibly. |
| 12 | Step 6 — Deep dives | Chosen from the NFRs and where the design breaks; each one is problem → naive → options → decision → consequences. |
| 13 | Step 7 — Wrap-up | Not a recap: bottlenecks with numbers, failure modes, operability, evolution. |
| 14 | Iteration | The framework is a spiral. Revisit earlier steps visibly when constraints change. |
| **Part C — Cross-cutting technique** | | |
| 15 | The board as shared state | Layout is communication. Requirements and numbers stay visible all session. |
| 16 | Narrating trade-offs | Option, cost, requirement, decision, revisit trigger — in two sentences. |
| 17 | Interruptions, hints, pushback | An interviewer question is a signal about what they want to score. |
| 18 | When you're stuck | Name the property you need, simplify to one machine, then scale out. |
| 19 | Naming technologies | Generic first, concrete with a reason; only name what you can defend under probing. |
| 20 | Level calibration per step | The same seven steps look different at mid, senior, staff and architect. |
| 21 | Variants of the round | Product architecture, infrastructure, NALSD, architect case, retrospective, low-level, ML/GenAI. |
| 22 | The 2026 context | GenAI prompts, explicit cost/operations grading, AI-assisted rounds — the framework still holds. |
| **Part D — Worked example** | | |
| 23 | Ticketing platform, 45 minutes, end to end | Contention and spikes, not storage, are the design. |
| **Part E — Architect variant** | | |
| 24 | Re-platforming an enterprise .NET system | The same seven steps, translated into drivers, transition architecture, risks and ADRs. |

---

# Part A — Why a framework exists

## Concept 1 — The problem the framework solves

A system design round hands you a deliberately underspecified prompt ("Design a ticketing platform"), a hard clock (45 minutes, of which roughly 35–40 are usable after introductions and closing questions), and an evaluator who is filling in a rubric. Three properties of that situation create the need for structure.

**The prompt is a search space, not a problem.** "Design X" has hundreds of features and dozens of plausible architectures. With no scope you can't finish, and you can't be evaluated against your own goals because you never set any. The first job is to convert the search space into a problem with a boundary.

**The clock is shorter than it feels.** Thirty-five minutes is enough to deliver one coherent design and harden two or three parts of it. It is not enough to explore freely and then converge. Candidates who "think out loud about everything" burn twenty minutes and arrive at the deep-dive section — the part that most distinguishes senior candidates — with five minutes left. Hello Interview, written by former FAANG interviewers, names failing to deliver a working system as the most common reason mid-level candidates fail, usually reported as vague "time management" feedback; the fix is usually focus, not speed.

**Nerves degrade working memory.** Under stress, people lose track of where they are. A fixed sequence gives you a fallback: when your mind goes blank, you ask yourself "which step am I in, and what does it need to produce?" That's enough to restart.

The framework answers all three. It is:

- **A scoping device** — step 1 converts the prompt into a bounded, prioritized problem with measurable goals.
- **A pacing device** — each step has a time budget and a concrete output, so you always know whether you're behind.
- **A protocol** — the step boundaries are natural checkpoints where you and the interviewer agree on what has been decided, so you never spend ten minutes building on an assumption they don't share.

What the framework is *not*: a script. Interviewers can tell the difference between a candidate who uses structure to think and one who recites a template ("Now I will do capacity estimation") and produces numbers that change nothing. Every step must earn its minutes by producing something later steps use.

---

## Concept 2 — What the interviewer is actually recording

Module 1 covered rubrics in detail. Here's the connection to delivery: the interviewer has a small number of dimensions to score, and they can only score what you make visible. Meta, for example, publicly describes its design round as assessing problem navigation, solution design, technical excellence and communication; Google, Amazon and Microsoft use different words for broadly the same things. Mapping each step to the dimension it feeds tells you where to spend your effort.

| Step | Primary dimension fed | What a strong signal looks like |
|---|---|---|
| 1. Requirements | Problem navigation / scoping | Prioritized, bounded scope; quantified NFRs; identifies the hard part early |
| 2. Estimation | Technical excellence / judgment | Numbers that *change a decision*, with the decision stated |
| 3. API | Solution design / communication | A contract that matches the requirements; safety details (idempotency, auth, pagination) |
| 4. Data model | Solution design / technical excellence | Store chosen by access pattern; correct keys, indexes, partition key |
| 5. High-level design | Solution design | A complete, working system traced end to end; complexity deferred deliberately |
| 6. Deep dives | Technical excellence / depth | Options and trade-offs tied to numbers; knows internals of what was named; knows when *not* to |
| 7. Wrap-up | Judgment / operational maturity | Honest bottlenecks, failure modes, monitoring, evolution |
| Throughout | Communication / collaboration | Checkpoints, responsiveness to hints, clean board, no monologues |

**Signal density.** A useful mental model: each minute of the interview should produce at least one piece of evidence the interviewer can write down. "Candidate chose cursor pagination because offsets degrade on deep pages and the feed is append-heavy" is evidence. Five minutes drawing a load balancer in front of a web server is not — the interviewer already assumes you know that. When you're about to spend time on something, ask: *would the interviewer write this down?*

**What is not scored.** Arriving at "the" correct architecture. There isn't one, and interviewers know the canonical answers to their own questions better than you do. interviewing.io's senior guide makes the point directly: in system design, *why* matters more than *what*. A memorized design with no reasoning scores below a derived design with a small flaw, because the first collapses at the first follow-up question.

---

## Concept 3 — Why these seven steps, in this order

The order isn't arbitrary convention. It falls out of a dependency chain: each step consumes the output of the steps before it.

```
 Requirements ──► Estimation ──► API ──► Data model ──► High-level ──► Deep dives ──► Wrap-up
   (goals,         (numbers       (contract   (state +        design         (hardening      (reflection
    scope,          that pick      that the    access          (flow that     against NFRs    on residual
    NFR targets)    approaches)    design      patterns)       satisfies      and probes)     risk)
                                   must meet)                  the contract)
        ▲                                                            │
        └──────────── constraints discovered later flow back ────────┘
```

Read the arrows as "is needed to decide":

- You can't estimate without knowing *what* is being counted (requirements give you the user actions and the scale).
- You can't design a contract without knowing the operations (functional requirements) and their constraints (an NFR like "purchase must be exactly-once" puts an idempotency key in the API).
- You can't model data without knowing how it is accessed (the API's reads and writes *are* the access patterns).
- You can't draw a high-level design without knowing what each request must change and read (API plus data model).
- You can't choose deep dives without a baseline design to find weaknesses in, and without NFRs to judge it against.
- You can't wrap up honestly without knowing what you hardened and what you didn't.

**Variants you'll see in the wild.** Different authors arrange the same moves differently, and recognizing that they're the same moves stops you from being thrown by an interviewer who expects another sequence:

| Source | Shape | Notable difference |
|---|---|---|
| Alex Xu, *System Design Interview* (ByteByteGo) | 4 steps: understand problem & scope → high-level design & buy-in → deep dive → wrap-up | API and data model are folded into the high-level step; explicit "get buy-in" checkpoint |
| Hello Interview delivery framework | Requirements → core entities → API → (optional data flow) → high-level design → deep dives | Estimation deferred and done inline "only if it influences the design"; data model fields written next to the database during HLD |
| ByteByteGo "7 steps" guide | Requirements → estimation → HLD → database design → interface design → … | Data model before interface |
| Google SRE NALSD | Requirements → one machine → distributed → multi-datacenter, with numbers at every stage | Iterative refinement with explicit resource arithmetic; reliability as a first-class requirement |

They agree on the essentials: scope first, a complete simple design before any depth, depth where the requirements demand it, and honest reflection at the end. **This curriculum's seven steps are a default ordering; the dependencies are the law.** If the interviewer says "skip the API, I care about the storage engine," the dependency still holds — you just state the operations in one sentence and move on.

**Iterate, don't waterfall.** The arrow back from deep dives to requirements matters. Real design is a spiral: a deep dive reveals that "holds" need a TTL, so the data model gains a column and the API gains an expiry field. The framework doesn't forbid going back — it requires you to go back *visibly* (Concept 14).

---

## Concept 4 — The time budget

A budget only helps if you can check it against a clock. Here are concrete allocations, with clock times so you can glance at the timer and know whether you're on track.

### 4a. The 45-minute round (the most common format)

| Step | Budget | Clock | Output on the board |
|---|---|---|---|
| Intros (interviewer-led) | 2–4 min | 0:00–0:04 | — |
| 1. Requirements | 4–6 min | 0:04–0:09 | FR list (≤3 core), out-of-scope, NFRs with numbers, "the hard part" |
| 2. Estimation | 2–4 min (or deferred) | 0:09–0:12 | 3–5 numbers, each with its "so…" |
| 3. Core entities + API | 4–6 min | 0:12–0:17 | Entity list; 4–6 endpoints or interface methods |
| 4. Data model | 2–4 min (often merged into HLD) | 0:17–0:20 | Key tables/collections with keys and critical fields |
| 5. High-level design | 8–10 min | 0:20–0:29 | Working end-to-end diagram, flows numbered |
| 6. Deep dives | 10–14 min | 0:29–0:41 | 2–3 hardened components, diagram updated |
| 7. Wrap-up | 2–3 min | 0:41–0:43 | Bottlenecks, failure modes, monitoring, next steps |
| Candidate questions | 2–3 min | 0:43–0:45 | — |

### 4b. The 60-minute round

| Step | Budget | Clock |
|---|---|---|
| Intros | 3–5 | 0:00–0:05 |
| 1. Requirements | 6–8 | 0:05–0:12 |
| 2. Estimation | 3–5 | 0:12–0:16 |
| 3. Entities + API | 5–6 | 0:16–0:22 |
| 4. Data model | 4–5 | 0:22–0:27 |
| 5. High-level design | 10–12 | 0:27–0:38 |
| 6. Deep dives | 15–18 | 0:38–0:55 |
| 7. Wrap-up | 3–4 | 0:55–0:58 |

### 4c. The architect case (60–90 minutes)

Architect rounds shift weight to the front and back: understanding drivers and constraints takes longer, and the "wrap-up" becomes a roadmap with risks and decisions. Part E shows this in full.

| Step | Share of time |
|---|---|
| Context, drivers, stakeholders, constraints | ~20% |
| Quality-attribute scenarios and numbers | ~10% |
| Integration contracts and data ownership | ~15% |
| Target and transition architecture | ~20% |
| Deep dives on the top risks | ~25% |
| Roadmap, ADRs, risks, organization | ~10% |

### 4d. Three rules that make the budget work

**The checkpoint rule.** The single most important clock check: *in a 45-minute round, a complete high-level design should be on the board by about minute 25–28; in a 60-minute round, by about minute 35–38.* If it isn't, cut scope out loud: "I'm going to drop search from the high-level design and come back to it if we have time, so we can get to the booking path, which is the interesting part." That sentence costs nothing and demonstrates exactly the prioritization interviewers score.

**Front-load speed, back-load depth.** Steps 1–4 should feel brisk. Their job is to produce shared ground, not to impress. The impressive part is step 6, and you can't get there if you treat step 1 as an essay.

**The budget is yours to defend, not the interviewer's.** If the interviewer pulls you into a deep dive during step 5, go with it — they're telling you what they want to score — but say where you are: "Happy to go into the locking now. I'll finish the purchase flow on the diagram afterwards so we have the full picture." That keeps the system complete while following their lead.

### 4e. Recovering when you're behind

| Situation | Recovery move |
|---|---|
| Minute 15, still on requirements | Freeze scope: "I'll lock these three requirements and these NFRs — tell me if anything's critical that's missing." Move on. |
| Minute 30, no complete HLD | Draw the remaining requirement as a single box with one arrow, say what it would contain, move to the deep dive that matters most. |
| Deep dive running long | "I could go further into this — but I'd rather spend the last eight minutes on the spike problem, unless you'd like more here." |
| Five minutes left, no wrap-up | Skip polish; give the top bottleneck, the top failure mode, and one thing you'd monitor. Ninety seconds. |

---

## Concept 5 — Who drives the interview

Leveling shows up most clearly in *who is steering*. interviewing.io's guide states it plainly: in junior interviews the interviewer expects to drive; at senior levels the expectation shifts to the candidate. The steering expectation by level:

| Level | Requirements | High-level design | Deep dives | Wrap-up |
|---|---|---|---|---|
| Mid-level | Asks reasonable questions; interviewer helps prioritize | Produces a working design with some guidance | Interviewer points at weaknesses; candidate fixes them | Lists some improvements when asked |
| Senior | Prioritizes independently; quantifies NFRs; names the hard part | Produces a complete design unaided, defers complexity deliberately | **Proposes** the deep dives and justifies the choice; handles probes | Volunteers bottlenecks and failure modes with numbers |
| Staff | Reframes the problem if needed ("the real problem is fairness under contention, not scale") | Design reflects the hard part from the start | Goes deep on the highest-risk area; compares approaches with real-world failure stories | Talks about evolution, operability, cost, and what would change the design |
| Architect | Elicits business drivers, constraints, stakeholders; distinguishes must from want | Shows target *and* transition states | Risk-driven: data migration, cutover, consistency across boundaries | Roadmap, decision records, organizational impact |

**Leading without bulldozing.** Hello Interview's guidance on deep dives includes a warning worth taking seriously: senior candidates who are proactive sometimes talk over the interviewer, who has specific signals to collect. Driving means proposing direction and making decisions, not filling every silence. A good rhythm:

- **At each step boundary, check in once.** "Here's the API. Before I draw the design — anything you'd like changed?" One sentence, then move unless they speak.
- **When proposing deep dives, offer a choice.** "I'd like to go deep on double-booking prevention and the on-sale spike. Is there something else you'd rather see?" You show you know what matters; they keep control of their rubric.
- **Don't ask permission for every move.** "Is it OK if I add a cache?" is passive. "I'm adding a cache here because the event page is read 200 times per write — I'll come back to invalidation" is driving.
- **Pause after decisions.** A two-second pause after "so I'm choosing a relational store for inventory" gives the interviewer room to probe. That's where your best evidence comes from.

---

# Part B — The seven steps, one at a time

## Concept 6 — Step 1: Requirements (4–6 minutes)

Requirements convert the prompt into a problem with a boundary and measurable goals. Module 4 covers the full catalogue of non-functional questions; this concept covers what the step must *produce* and how to deliver it at senior pace.

### 6a. The four outputs

**1. Core functional requirements — three, rarely more.** Phrase them as "users should be able to…" or, for infrastructure, "clients should be able to…". The discipline is prioritization: real systems have hundreds of features, and a long list hurts you, because the remaining 35 minutes are spent satisfying what you wrote down. Hello Interview's advice is to identify and prioritize the top three; several large companies explicitly evaluate whether you can focus on what matters.

**2. Explicit out-of-scope.** Write it down. "Out of scope: resale, dynamic pricing, event creation by organizers, payment processing internals." This does three jobs: it shows you know the domain is bigger than what you're designing, it protects your time, and it gives the interviewer an easy place to say "actually, I want resale in."

**3. Non-functional requirements — three to five, quantified, and attached to a specific part of the system.** "Low latency" is meaningless; every system should be fast. "Seat-availability reads p99 < 300 ms; booking confirmation p99 < 1 s excluding the payment provider" is a target you can design against. Use a checklist to find candidates, then keep only the ones that will shape the design:

| Dimension | The question that makes it concrete |
|---|---|
| Consistency vs availability | Where must we never be wrong, and where can we be stale — and for how long? |
| Scale and shape | Peak vs average, read/write ratio, burstiness, skew (hot keys, celebrity events) |
| Latency | Which operation, which percentile, what target? |
| Durability | Can we lose any acknowledged write? Ever? (RPO) |
| Availability | Which flows, what target, what's the cost of downtime? (SLO, RTO) |
| Security and compliance | Authn/authz model, PII, PCI scope, data residency (GDPR) |
| Environment | Mobile clients on poor networks? Existing platform and team? |
| Cost | Is there a budget ceiling? Is this a cost-sensitive internal system? |

**4. The hard part.** This is the senior move, and the one most candidates skip. After listing NFRs, say which one will drive the design: *"The interesting constraint here is that 10 million people will try to buy 50,000 seats in the same ten minutes, and we can never sell a seat twice. That's the design. Everything else is fairly standard."* You've just told the interviewer where the deep dives will go, shown you understand the domain, and given yourself a north star for the high-level design.

### 6b. Ask, or assume and state?

Both are valid; the senior pattern combines them. Ask about things that genuinely fork the design ("Do users pick specific seats, or is it general admission?" — that changes the entire contention model). For everything else, state an assumption and invite correction: "I'll assume 10 million daily users and on-sale spikes of about 10 million concurrent users for top events — stop me if you have different numbers in mind." This keeps momentum and still lets the interviewer redirect.

Questions that are *not* worth asking: anything whose answer won't change the design ("Should we use the cloud?"), and long lists fired at the interviewer without explaining why you're asking. A question with its reason attached ("…because if it's general admission, contention is a counter, not a set of rows") is itself a signal.

### 6c. What it sounds like

> *"Let me make sure I'm designing the right thing. Core flows: users browse and search events; view an event's seat map with availability; and select seats, hold them while they pay, and complete the purchase. I'll leave resale, dynamic pricing, organizer tools and payment internals out — we'll integrate with a payment provider. Does that match what you had in mind?"*
>
> *"For non-functionals: booking has to be strongly consistent — no seat sold twice, ever. Browsing and search can be eventually consistent; a few seconds stale is fine. Search p99 under 500 ms. The system has to survive on-sale spikes — I'll assume up to 10 million users arriving for a top event. Holds last ten minutes. And the hard part is clearly the on-sale: extreme contention on a small inventory under a massive spike, with a hard correctness requirement."*

Ninety seconds. The interviewer now knows your scope, your targets and your plan.

---

## Concept 7 — Step 2: Estimation (2–4 minutes, or deferred)

Module 5 teaches the numbers and the arithmetic. This concept is about *when and why* to estimate inside the framework.

### 7a. Estimate to decide, not to decorate

The failure mode is well known: a candidate computes DAU, QPS, storage and bandwidth, writes four large numbers on the board, says "so it's a lot," and moves on. Nothing in the design changes. The interviewer learns only that you can multiply. Hello Interview goes as far as recommending that you skip upfront estimation unless it will directly influence the design, and do the math inline when a decision depends on it.

The better rule: **every number must end with "so…".**

| Number | "So…" (the decision it drives) |
|---|---|
| Peak write QPS = 5k | …a single relational primary with headroom handles it; no sharding yet |
| Working set = 40 GB | …fits in one large Redis instance; no cache cluster sharding needed |
| Fan-out per post = 1M followers for celebrities | …fan-out-on-write fails for celebrities; hybrid push/pull |
| Storage growth = 500 GB/year | …storage is not the problem; don't spend deep-dive time on it |
| Spike = 10M users polling every 5 s = 2M rps | …the origin can't take this; we need a waiting room and edge caching |

Notice the fourth row: estimation that proves something is *not* a problem is just as valuable, because it justifies where you *won't* spend time.

### 7b. The quick arithmetic you need

| Fact | Use |
|---|---|
| 1 day ≈ 86,400 s ≈ 10⁵ s | Daily volume ÷ 10⁵ ≈ average per second |
| 1M requests/day ≈ 12/s average | Quick conversions |
| Peak ≈ 2–10× average (spiky consumer products can be far higher) | Always design for peak, say which multiplier you're assuming |
| Little's Law: L = λ × W | Concurrency from rate and duration (Module 6) |
| Bytes per object × objects × retention × replication factor | Storage |

### 7c. Upfront or inline?

| Do it upfront when… | Defer and do inline when… |
|---|---|
| The scale *is* the problem (spikes, fan-out, hot keys) | Scale is generic ("it's large, assume distributed") |
| A number determines the architecture's shape (single node vs sharded) | The number only matters for one component's sizing |
| The interviewer asks for it | The interviewer signals they want to get to the design |

If you defer, say so: *"I'll skip the full capacity math for now — the only number that really matters is the on-sale spike, so let me do just that one."* That is a decision, and it reads as judgment, not avoidance.

---

## Concept 8 — The bridge: core entities (about 1–2 minutes)

Before the API, list the nouns. This is quick but not trivial.

**Why it helps.** The entities become the API's resources, the data model's tables or containers, and the boxes' vocabulary. Listing them first gives you and the interviewer shared terms, and it surfaces modeling decisions early. For a ticketing system: `Event`, `Venue`, `Seat`, `EventSeat` (a seat *for a specific event* — the inventory unit), `Hold`, `Order`, `User`. The moment you write `EventSeat` separately from `Seat`, you've shown you understand that a venue's physical seat and an event's sellable ticket have different lifecycles — exactly the kind of distinction DDD calls the ubiquitous language (Module 22).

**Why not the full data model now.** You don't yet know which fields matter. You'll discover them as you trace requests through the high-level design. Write the entity list now; add fields later, next to the database that stores them.

**Names are signal.** Interviewers notice whether `Hold` is called `Reservation`, `Lock` or `TempBooking`, and whether you stay consistent. A good name describes the domain concept, not the mechanism: `Hold` (what the user experiences) rather than `SeatLock` (how you might implement it).

---

## Concept 9 — Step 3: API design (4–6 minutes)

The API is the contract between the system and its users. Its job in the interview is to pin down exactly which operations the high-level design must support, with enough detail that correctness properties (identity, idempotency, ordering, async completion) are visible.

### 9a. Derive it from the functional requirements

Usually it maps one-to-one: each functional requirement becomes one or two endpoints. Use the entities as resources, plural nouns, and default to REST unless there's a reason not to.

### 9b. Choosing the protocol

| Protocol | Choose it when | Interview note |
|---|---|---|
| REST over HTTP | Default for public and product APIs | Say "REST" and move on unless there's a reason |
| gRPC | Internal service-to-service, high call rates, streaming, strong typing | Mention HTTP/2 connection balancing (Module 6) if you choose it |
| GraphQL | Many client types with different data needs; aggregation over many backends | Mention query cost limits and caching complexity |
| WebSockets / SignalR | Bidirectional real-time (chat, collaborative editing, live seat updates) | Stateful connections: backplane or Azure SignalR Service (Module 6) |
| Server-Sent Events | One-way server push over HTTP | Simpler than WebSockets for notifications and live counters |
| Async messages/events | Work that completes later; integration between services | The "API" is the message schema; mention the outbox (Module 11) |
| Library interface | Infrastructure prompts (cache, rate limiter, ID generator) | A C# interface signature is the clearest contract |

For an infrastructure prompt, the contract is often best expressed as code:

```csharp
public interface IRateLimiter
{
    // Returns whether the call is permitted and, if not, when to retry.
    ValueTask<RateLimitDecision> TryAcquireAsync(string clientKey, int permits = 1, CancellationToken ct = default);
}

public readonly record struct RateLimitDecision(bool Allowed, TimeSpan? RetryAfter, long Remaining);
```

### 9c. The details that signal seniority

Most candidates write paths and verbs. Senior candidates put the correctness properties *into the contract*:

- **Identity comes from the token, never the body.** `POST /holds` — the user is derived from the bearer token, not `{"userId": …}` in the request. Writing `userId` in a body is a security smell interviewers notice.
- **Unsafe operations carry an idempotency key.** Anything that charges money or claims inventory gets an `Idempotency-Key` header, so a client retry after a timeout doesn't double-book or double-charge. This is the Stripe pattern, and the IETF HTTP API working group has a draft standardizing the header. The server stores the key with the result and replays the result on retry.
- **Long-running work returns 202 Accepted** with a resource to poll (or a push notification), instead of holding a request open.
- **Lists use cursor pagination**, not offsets: offsets get slower and less consistent as data shifts under them; a cursor (an opaque encoding of the last seen sort key) is stable and index-friendly.
- **Conflicts are explicit.** `409 Conflict` when the seats are gone, with a machine-readable body. In ASP.NET Core, RFC 9457 Problem Details (`Results.Problem`, `AddProblemDetails()`) is the standard shape.
- **Versioning is decided, not ignored.** A `/v1/` prefix or a header; say which and move on.
- **Optimistic concurrency where updates race.** `If-Match` with an ETag (or an expected version in the body) for updates to shared resources.

### 9d. What it looks like on the board

```
GET  /v1/events?q=&city=&from=&cursor=         → Page<EventSummary>
GET  /v1/events/{eventId}                       → Event
GET  /v1/events/{eventId}/availability          → SectionAvailability[]   (cached, seconds stale)
POST /v1/events/{eventId}/holds                 Idempotency-Key
     { seatIds: [...] }                         → 201 { holdId, expiresAt } | 409 SeatsUnavailable
POST /v1/orders                                 Idempotency-Key
     { holdId }                                 → 202 { orderId, status: "PendingPayment", paymentSession }
GET  /v1/orders/{orderId}                       → Order
POST /v1/payments/webhook                       (from the payment provider; signature-verified)
```

Six lines, and the interviewer can already see your consistency story: availability is cached and approximate, holds are the authoritative claim, orders complete asynchronously with the payment provider.

**Don't overspend.** API design is a five-minute step. Full schemas, every query parameter, every error code — skip them. Interviewers at Meta's product architecture round weight the API more heavily (Concept 21), so stretch slightly there; elsewhere, keep it tight.

---

## Concept 10 — Step 4: Data model (2–4 minutes, often merged into step 5)

The data model answers three questions: *what state exists, where does it live, and how is it found?* The order in which you answer them matters.

### 10a. Access patterns first

Before choosing a store, list how the data is read and written. The API you just wrote *is* the access-pattern list. Turn it into a small table:

| Access pattern | Frequency | Consistency need | Shape |
|---|---|---|---|
| Search events by text, city, date | High | Seconds stale OK | Full-text + filters |
| Get event details | Very high | Seconds stale OK | Key lookup |
| Get availability for an event | Extreme during on-sale | Seconds stale OK (display only) | Aggregate by section |
| Claim N specific seats atomically | High during on-sale | **Strict — linearizable per seat** | Conditional multi-row update |
| Confirm order, mark seats sold | Moderate | **Strict** | Transaction across seats + order |
| Get my orders | Low | Read-your-writes | Lookup by user |

This table chooses the stores for you. The strict, multi-row, conditional pattern says *relational with transactions* (or a store with equivalent conditional multi-item writes). The full-text pattern says *search index*. The extreme, stale-tolerant read says *cache*. Module 12 covers the SQL-vs-NoSQL axes in depth; the interview move is to derive the choice from this table instead of asserting a favorite.

### 10b. Keys, indexes and the partition key

Write only the fields that matter to the design, next to the store, with the keys marked:

```
event_seats  PK (event_id, seat_id)
             status            available | held | sold
             hold_id           nullable
             hold_expires_at   nullable
             order_id          nullable
             -- index: (event_id, section_id, status) for availability aggregation

holds        PK hold_id · user_id · event_id · expires_at · status
orders       PK order_id · user_id · hold_id · status · amount · psp_payment_id
             UNIQUE (user_id, idempotency_key)
```

**The partition key is the most consequential line in the data model.** In a partitioned store (Cosmos DB, a sharded SQL database, Kafka topics), it decides which operations are cheap single-partition operations and which become expensive cross-partition ones, and it decides where hot spots form. For inventory, `event_id` is natural: every booking operation is scoped to one event, so transactions stay single-partition. The risk is the mega-event — all traffic on one partition — which is precisely what the on-sale deep dive must address. Saying this out loud during the data model step is a strong signal (Module 8 covers partitioning strategy).

### 10c. Common data-model mistakes

- Listing every column (`name`, `email`, `created_at`) — the interviewer assumes them.
- Choosing the store before the access patterns ("I'll use Cassandra because it scales").
- Forgetting the uniqueness constraint that enforces an invariant (the `UNIQUE` on the idempotency key *is* the idempotency mechanism).
- Ignoring the lifecycle of state (when does a hold become invalid? who cleans it up? does correctness depend on the cleanup?).

---

## Concept 11 — Step 5: High-level design (8–10 minutes)

The high-level design is a **working system**: every functional requirement can be traced end to end through it. Not optimized, not hardened — working.

### 11a. Build it endpoint by endpoint

Take the API line by line and draw what each request touches. This is the most reliable way to produce a complete design under pressure, and it naturally narrates the data flow:

1. `GET /events` — client → edge/CDN → API gateway → event service → search index.
2. `GET /availability` — event service → cache → inventory database (aggregate query) on miss.
3. `POST /holds` — gateway → booking service → inventory database (conditional update).
4. `POST /orders` — booking service → creates order → payment provider session; webhook → booking service → marks seats sold.

As each request reaches a store, say what state changes: *"This write flips those seats from available to held, with this hold ID and an expiry ten minutes out."* That sentence is worth more than the box.

### 11b. Simplest thing that works, with visible parking

Complexity added in step 5 is complexity you have to explain before you've shown a working system. Candidates routinely add Kafka, three caches and a service mesh, run out of time, and never finish the purchase flow. The discipline:

- **Start with one service per responsibility, not microservices by reflex.** "Booking service" can be a module in a monolith at this point; say so if it matters (Module 21).
- **Park optimizations visibly.** When you notice the availability endpoint will melt during an on-sale, write "⚠ spike — deep dive" next to it and keep going. You get credit for noticing, and you finish the design.
- **Name the async boundary.** Where work completes later (payment, emails, search indexing), draw it dashed and say why it's async.

### 11c. Diagram conventions that help the interviewer

- Clients on the left, data stores on the right, flow left to right.
- Solid arrows for synchronous calls, dashed for asynchronous messages.
- Number the flows (①, ②, ③) and refer to the numbers when talking.
- Label stores with what they hold, not just the technology: "Inventory (Azure SQL)" not "DB".

### 11d. The walkthrough test

Before leaving step 5, walk one complete user journey aloud using the numbers on the diagram: *"A user searches, ① hits the search index; opens the event, ② gets cached availability; selects two seats, ③ we claim them conditionally; pays, ④ the provider's webhook ⑤ marks them sold and ⑥ the outbox publishes OrderConfirmed for the email."* If you can't narrate a requirement through the diagram, the design isn't complete. It takes thirty seconds, and it's the cleanest possible handoff to the deep dives.

---

## Concept 12 — Step 6: Deep dives (10–14 minutes in 45; 15–18 in 60)

This is where senior and staff candidates separate from mid-level ones. The high-level design proves you can assemble components; deep dives prove you understand how they behave under the constraints you wrote in step 1.

### 12a. Choosing what to dive into

Pick two or three, and derive the choice — don't just pick your favorite topic:

1. **From the non-functional requirements.** Each NFR the baseline design doesn't yet meet is a candidate. "No double-booking" → concurrency control. "Survive 10M-user spike" → admission control and edge caching.
2. **From where the design breaks.** Look at the numbers from step 2 against the diagram from step 5. Which box gets a load it can't take? Which flow has a correctness hole if a step fails halfway?
3. **From the interviewer's probes.** If they've asked twice about the payment flow, that's a deep dive whether you planned it or not.

Then offer the choice (Concept 5): *"I'd like to go deep on two things — preventing double-booking with holds, and surviving the on-sale spike. Then, if there's time, the consistency between search and inventory. Sound good?"*

### 12b. The anatomy of a good deep dive

Every deep dive has the same internal structure, and following it makes even unfamiliar territory sound rigorous:

1. **State the problem precisely, with a number.** "Two users clicking the same seat within milliseconds; during an on-sale, thousands of concurrent attempts on the best sections."
2. **Show the naive approach and why it fails.** "Read the seat, check it's available, then write — a classic check-then-act race; both readers see 'available' and both write."
3. **Lay out two or three real options.** Pessimistic locking, conditional (optimistic) updates, an external lock service.
4. **Compare them on the criteria that matter *here*.** Correctness under failure, contention behavior, latency, operational complexity, number of sources of truth.
5. **Decide, and say why in terms of the requirements.** "Conditional update in the database, because it keeps a single source of truth and the expiry falls out of the predicate."
6. **State the consequences.** What new failure modes appear, what you'd monitor, what would make you revisit the decision.

The "options ladder" variation — *good, better, best* — works well when the options are incremental (a single cache → cache with stampede protection → cache plus push updates). It shows you know the cheap solution and know exactly when it stops being enough.

### 12c. Depth means mechanisms, not more boxes

A deep dive that adds three new boxes and no mechanism is still a high-level design. Depth sounds like: *which* isolation level and why it suffices; what happens to an in-flight transaction when the process dies; which key the lock is on and what happens when the lease expires; how the cache is invalidated and what a reader sees in the gap. If you name a technology, expect the interviewer to probe its internals — that is the point of naming it (Concept 19).

### 12d. Handling probes mid-dive

When the interviewer interjects ("What if the payment webhook never arrives?"), answer it fully — that question is them telling you what they want to score — then return explicitly: "So that's the reconciliation path. Back to the spike: …". Never treat an interviewer question as an interruption to your plan; it *is* the plan.

---

## Concept 13 — Step 7: Wrap-up (2–3 minutes)

The wrap-up is not a recap of what's already on the board. It's a short demonstration that you know where your design is weak and how you'd operate it.

**What to cover, in priority order:**

1. **The top bottleneck, with a number.** "For a mega-event, all booking traffic hits one inventory partition. At the admission rate we chose, that's around 3k conditional updates a second, which a large primary handles — but it's the first thing that breaks if we raise admission."
2. **The top failure mode and its behavior.** "If the payment provider is down, holds expire and seats return to the pool — we fail closed on sales, which is the right direction for a ticketing business."
3. **How you'd know it's working.** One or two SLOs and the alerts that matter. For ticketing, a periodic invariant check — *no seat with two confirmed orders* — that should always return zero.
4. **Evolution.** What changes at 10×, or with the next obvious feature (resale, multi-region).
5. **What you'd do with more time.** One or two items. Honest, specific.

Two things to never say: "I think the design is pretty complete" (no design is), and a generic list ("we'd add monitoring, logging, security") with no specifics. ByteByteGo's framework phrases the first rule bluntly: never say your design is perfect.

---

## Concept 14 — Iteration: the framework is a spiral

Real interviews don't proceed linearly. Constraints arrive mid-way ("Actually, assume users can't pick seats — best available only"), deep dives expose missing state, and interviewers change the scale to see what breaks. The framework handles this if you revisit earlier steps *visibly*:

1. **Acknowledge the change and update the source.** Cross out or edit the requirement on the board. The requirements box should always be true.
2. **Trace the impact downstream, out loud.** "Best-available changes the hold API — the client sends a quantity and section, not seat IDs — and the claim becomes 'find N available seats in this section and take them,' which is a different query. The rest of the design stands."
3. **Decide whether it changes a deep dive.** Sometimes the new constraint *is* the next deep dive.

Visible iteration is positive signal. It shows the interviewer exactly the behavior they want from a senior engineer when requirements change in real life: absorb the change, assess the blast radius, and adjust only what needs adjusting.

---

# Part C — Cross-cutting technique

## Concept 15 — The board as shared state

Whether you're at a physical whiteboard or in Excalidraw, Miro or a company's own drawing tool, the board is not decoration. It is the shared memory between you and the interviewer, and the interviewer will often photograph or export it for the debrief. A messy board makes a good design look worse in the write-up.

**A layout that works.** Reserve regions before you start, and keep the left column permanent:

```
┌─────────────────────────┬───────────────────────────────────────────────────────┐
│ REQUIREMENTS            │                                                       │
│ FR: 1… 2… 3…            │                                                       │
│ Out: …                  │              HIGH-LEVEL DESIGN                        │
│ NFR: …(numbers)         │       clients → edge → services → stores              │
│ ★ Hard part: …          │       (flows numbered ① ② ③, async dashed)            │
├─────────────────────────┤                                                       │
│ NUMBERS                 │                                                       │
│ peak QPS … so …         │                                                       │
│ spike … so …            ├───────────────────────────────────────────────────────┤
├─────────────────────────┤ DEEP DIVES                                            │
│ ENTITIES / API          │ DD1: problem → options → decision                     │
│ Event, EventSeat, Hold… │ DD2: …                                                │
│ POST /holds …           │                                                       │
├─────────────────────────┤                                                       │
│ PARKING LOT             │                                                       │
│ ⚠ availability spike    │                                                       │
│ ⚠ search staleness      │                                                       │
└─────────────────────────┴───────────────────────────────────────────────────────┘
```

**Why the left column stays.** Every decision you make later should be justified against something in that column. When you say "this is fine because peak writes are 5k/s," the interviewer can see the number. When they ask "why not eventual consistency here?", you point at the NFR.

**The parking lot.** A small list of things you noticed but deliberately deferred. It proves awareness without derailing the build, and it's a ready-made menu for choosing deep dives.

**Practice the tool.** Ask the recruiter which tool you'll use, and practice it until drawing a box with a label takes two seconds. Fumbling with a drawing tool for five minutes is a real, avoidable cost. In Excalidraw, learn the shortcuts for rectangle, arrow and text, and how to duplicate a group.

---

## Concept 16 — Narrating trade-offs

Every decision in a design round is a trade-off, and the interviewer wants to hear that you know what you're giving up. A compact template that works under pressure:

> **"Option A gives us *X* at the cost of *Y*. Given *requirement R* (with its number), I'll take A. I'd revisit if *Z*."**

Applied:

> *"A conditional update in the inventory database gives us one source of truth and atomic multi-seat claims, at the cost of putting all contention on the database primary. Given that admission control caps us at about 3,000 claims per second per event, a single primary handles that comfortably, so I'll take it. I'd revisit if we needed to admit far more concurrent buyers — then I'd look at pre-partitioning an event's inventory by section."*

Four things make this land:

- **The cost is named.** Candidates often describe only the benefit. Naming the cost is the signal.
- **The decision is tied to a requirement and a number on the board.** That's the difference between preference and judgment.
- **The revisit trigger is specific.** It shows you think of decisions as conditional, which is how real architecture works.
- **Reversibility is considered.** Some decisions are cheap to change (a cache TTL), some are expensive (a partition key, a public API shape, a database engine). Spending more care on one-way doors is itself a senior signal: *"I'm more careful about the partition key than the cache choice, because re-partitioning live inventory is a migration, while swapping cache settings is a deploy."*

**Avoid false trade-offs.** "SQL is consistent but doesn't scale; NoSQL scales but isn't consistent" is a cliché that interviewers hear daily and that is wrong in both directions (Module 12). Trade-offs should be specific to the operation and the store, not to whole categories.

---

## Concept 17 — Interruptions, hints and pushback

An interviewer's question is information. It tells you what they want to score, what they think you've missed, or that they're managing time. Classify it, then respond accordingly:

| Type | How it sounds | What to do |
|---|---|---|
| Clarifying | "What do you mean by a hold?" | Answer briefly; possibly tighten your vocabulary on the board |
| Hint | "What happens if two users click the same seat at once?" | They see a gap. Treat it as the next deep dive, now. |
| Challenge | "Why not just use Redis locks?" | Take it seriously. Compare honestly; change your mind if they're right. |
| Scale or constraint change | "Now assume 10× the users." | Iterate visibly (Concept 14). |
| Time management | "Let's move on to…" | Move immediately; don't finish your sentence's subplot. |
| Probing depth | "How does that actually work?" | Go one level deeper into mechanism. If you hit the limit of your knowledge, say so (Concept 18). |

**Pushback is often a test, not a correction.** Interviewers frequently challenge a *good* decision to see whether you can defend it with reasons rather than abandon it at the first frown. The right response is neither stubbornness nor instant capitulation: restate the requirement driving the choice, acknowledge the alternative's real advantage, and decide again with the reasoning visible. If they're right, say so plainly — *"That's a better approach; the external lock adds a second source of truth I hadn't accounted for"* — and update the board. Conceding gracefully when wrong is a strong collaboration signal; defending a weak choice for five minutes is a strong negative one.

**It's fine to ask what's behind a question**, once: *"Are you asking because you see a failure case with the conditional update, or about its performance?"* That's collaborative and it saves you from answering the wrong question.

**Correcting your own mistakes is positive signal.** If you realize mid-dive that something earlier was wrong, say it: *"Actually, I need to revise the hold expiry — if a sweeper job releases expired holds, correctness depends on the sweeper. Let me put the expiry in the claim predicate instead."* Interviewers watch for exactly this self-correction in senior engineers.

---

## Concept 18 — When you're stuck, or don't know a technology

Everyone hits the edge of their knowledge in a good interview — that's how interviewers find your level. What matters is how you behave at the edge.

**Name the property, not the product.** If you don't know the right technology, describe what you need: *"I need a store that gives me atomic conditional writes across several items in the same partition, with strong consistency on that partition."* That's a correct answer that any interviewer can map to concrete technologies, and it shows you understand the requirement more deeply than a candidate who names a product.

**Be honest about experience, and reason anyway.** *"I haven't run Cassandra in production, but from its design — leaderless replication, tunable quorums — I'd expect lightweight transactions to be expensive here, so I'd avoid relying on them for every seat claim."* Honesty plus first-principles reasoning beats bluffing every time; bluffing collapses at the first follow-up.

**Simplify, then scale.** Google's NALSD method, from the SRE Workbook, starts by asking whether the problem can be solved on one machine and only then distributes it, computing resources at each stage. When stuck in a deep dive, do the same: *"Let me solve this for one server first — a single process with an in-memory map of seats and a lock. Now: what breaks when there are twenty servers?"* The single-machine version usually makes the distributed problem obvious.

**Use the requirements as a compass.** When you don't know what to do next, read the left column of the board. Which requirement isn't met yet? Which number hasn't been used? That's where to go.

---

## Concept 19 — Naming technologies

**Generic first, concrete when it matters.** In the high-level design, "relational database," "message queue" and "search index" are often enough. Name a specific technology when its specific properties drive the design, and say which property: *"Azure SQL here, because I want multi-row transactions and the conditional update semantics I'm relying on."*

**For .NET and Azure roles, map boxes to services — with reasons.** Interviewers hiring for an Azure-heavy team often want to hear that you can translate an abstract design into their platform. Doing it with reasons is valuable; doing it as a list of logos ("Front Door, APIM, AKS, Cosmos, Service Bus, Event Grid, Redis, AI Search…") is vendor-dumping, and it invites probing on every one.

| Generic component | Azure option (2026) | AWS rough equivalent | When the specific choice matters |
|---|---|---|---|
| Global edge, CDN, WAF | Azure Front Door | CloudFront + WAF | Anycast failover speed, caching rules, WAF policies |
| API gateway | Azure API Management, or YARP in your own ingress | API Gateway | Auth, quotas, transformation; APIM cost vs self-hosted |
| Compute | App Service, Container Apps, AKS, Functions (Module 26) | Elastic Beanstalk, ECS/Fargate, EKS, Lambda | Scaling model, cold start, operational load |
| Relational OLTP | Azure SQL Database, Azure Database for PostgreSQL flexible server | RDS / Aurora | Transactions, isolation, scale-up ceiling, read replicas |
| Multi-model NoSQL | Azure Cosmos DB | DynamoDB | Partition key, RU cost, consistency levels (Module 7) |
| Cache | Azure Managed Redis (Azure Cache for Redis is retiring) | ElastiCache / MemoryDB | Clustering, persistence, eviction (Module 10) |
| Queue / pub-sub | Azure Service Bus | SQS / SNS | Sessions, ordering, DLQ, transactions (Module 11) |
| Event streaming | Azure Event Hubs (Kafka-compatible) | Kinesis / MSK | Partitions, retention, replay |
| Search | Azure AI Search, or Elasticsearch | OpenSearch | Indexing latency, relevance tuning |
| Real-time push | Azure SignalR Service, Azure Web PubSub | API Gateway WebSockets / AppSync | Connection counts, backplane |
| Object storage | Azure Blob Storage | S3 | Tiers, lifecycle, CDN integration |

A fast-moving fact worth having current: Microsoft has announced the retirement of Azure Cache for Redis in favor of Azure Managed Redis — Enterprise tiers on March 31, 2027 and Basic/Standard/Premium on September 30, 2028, with new-customer creation of the older tiers blocked since April 2026. In a 2026 interview, proposing "Azure Cache for Redis" for a new design is a small but noticeable staleness signal; "Azure Managed Redis" is the current name.

**Only name what you can defend.** Naming a technology is an invitation to be probed on it. If you say Cosmos DB, expect "what's your partition key, and what happens to a hot partition?" If you say Kafka, expect "how many partitions, and what's your ordering guarantee?" If you can't answer at least one level below the name, describe the property instead (Concept 18).

**Know when to say no to a technology.** Some of the strongest signals in a design round are the things you *don't* add: *"I'm not adding a message broker between the API and the booking service — the user needs a synchronous answer about whether they got the seats, and a queue would add latency and a pending state for no benefit"* (Module 11, Concept 7).

---

## Concept 20 — Level calibration per step

The same seven steps produce visibly different output at different levels. Use this table to check your mock interviews against the level you're targeting.

| Step | Mid-level | Senior | Staff | Architect |
|---|---|---|---|---|
| Requirements | Lists features; some NFRs, mostly unquantified | Top 3 FRs, explicit out-of-scope, quantified NFRs, names the hard part | Reframes the problem around the real constraint; surfaces non-obvious requirements (fairness, abuse, compliance) | Business drivers, stakeholders, constraints (budget, team, existing estate, regulation), quality-attribute scenarios |
| Estimation | Computes generic numbers | Computes only decision-relevant numbers, each with "so…" | Uses numbers to rule options in and out on the fly; sizes headroom for failure | Adds cost, licensing and capacity over time; growth scenarios |
| API | Endpoints match features | Identity, idempotency, pagination, async completion in the contract | Contract designed for evolution and backward compatibility; client retry semantics | Integration contracts between systems and teams; versioning policy; consumer-driven contracts |
| Data model | Tables with many columns | Store chosen by access pattern; keys, indexes, partition key justified | Anticipates hot partitions, lifecycle and retention, migrations | Data ownership per bounded context, master data, migration strategy |
| High-level design | Working design with some guidance; may over-engineer | Complete, simple design unaided; complexity parked deliberately | Design already reflects the hard part; clear sync/async boundaries | Target *and* transition architecture; build-vs-buy |
| Deep dives | Fixes issues the interviewer points out | Proposes and leads 2–3 dives with options and trade-offs | Picks the highest-risk area; knows real-world failure modes and incidents; knows when not to apply a pattern | Risk-driven dives: consistency across boundaries, cutover, rollback, operating model |
| Wrap-up | Lists generic improvements | Bottlenecks with numbers, failure modes, SLOs | Operability, cost, evolution, what would invalidate the design | Roadmap, ADRs, risks with mitigations, team and organizational impact |

Module 2's distinction between the IC ladder and the enterprise-architect track maps onto the last two columns: staff candidates win on depth and judgment within a system; enterprise architects win on framing, transition and risk across systems.

---

## Concept 21 — Variants of the design round

The framework adapts; the dependency chain doesn't change. Here's how the seven steps stretch or compress in each common variant.

**Product architecture (Meta's term; others say "product design").** Meta offers this as an alternative design round, typically for product-focused engineers, built around *user-facing* products ("Design Ticketmaster," "Design a news feed") rather than infrastructure ("Design a distributed cache"). Hello Interview's guide, written by a former Meta staff engineer, notes the questions can overlap heavily with system design and that preparation is nearly the same, with more emphasis on API design. Stretch steps 3 and 4; keep step 6 focused on product-relevant behaviors (pagination, real-time updates, consistency the user can see).

**Infrastructure system design.** "Design a distributed cache / rate limiter / job scheduler / message queue." The API is a client interface (Concept 9b), the data model is often an in-memory structure plus a persistence format, and the deep dives go into internals: replication, partitioning, consensus, eviction. Expect more probing on Phase 3 material.

**Non-Abstract Large System Design (Google SRE).** The design must be concrete enough to provision: how many machines, disks and network links, with arithmetic. The method is iterative — one machine, then distributed, then multiple datacenters — with an SLO driving each refinement. Step 2 is not optional here; it runs through every stage.

**The architect case.** "Our company runs X; design how it should evolve to support Y." Requirements become business drivers and constraints; the API becomes integration contracts; the data model becomes data ownership and migration; the high-level design becomes target plus transition states; deep dives target the riskiest parts of the transition; the wrap-up becomes a roadmap with decision records. Part E walks through one.

**The retrospective ("design something you built").** Some companies ask you — and Google's "Googleyness and Leadership" round is reportedly being revised to include a technical design conversation about past work — to present and defend a system you built. Run the same seven steps in the past tense: what the requirements and constraints were, what numbers mattered, the contract, the data model, the architecture, the two hardest problems and how you solved them, and what you'd change now. Prepare two of these in advance; they are rehearsable in a way that live prompts aren't.

**Low-level / object-oriented design.** "Design a parking lot," "design an elevator controller." The framework compresses: requirements (operations and error cases) → entities → class design and interfaces → key flows → extensibility and edge cases. In C#, this is where records, interfaces, enums and state machines appear on the board, and where you're judged on cohesion, coupling and naming.

**ML and generative-AI system design.** Requirements start from the business objective and the metric that measures it; estimation includes inference cost and latency per request; the data model includes training data, features or the retrieval corpus; deep dives include evaluation, drift, guardrails and fallbacks. The framework holds, with "model" as a component that is probabilistic, slow and expensive.

**Take-home and AI-assisted formats.** When you have hours rather than minutes, the framework becomes a document: requirements and assumptions up front, decisions recorded with alternatives, explicit out-of-scope. When AI tools are allowed, the evaluation shifts toward how well you verify and direct them; the design reasoning is still yours to show.

---

## Concept 22 — The 2026 context

A few shifts are worth knowing about as you prepare, without overreacting to them:

- **Generative-AI prompts have entered general loops.** Interview-prep providers report that "design an LLM-backed X" (retrieval-augmented search, an LLM gateway with routing and cost controls, an agent orchestration service) now appears in general engineering loops, not just ML roles. For these, the seven steps still apply; the deep dives change — latency and cost per request, caching of model outputs, evaluation, guardrails, fallbacks when the model is slow or wrong.
- **Cost and operations are graded more explicitly.** Several 2026 prep guides report that interviewers now expect cost reasoning and operational thinking as part of the answer, not as optional extras. Put a cost line in your estimation when it matters, and make the wrap-up operational.
- **AI-assisted rounds exist, mostly in coding.** Meta has added an AI-assisted coding round; Google has piloted AI-assisted code-comprehension rounds; Canva announced in 2025 that it expects candidates to use AI tools in technical interviews. Policies vary by company and even team, and Microsoft's published default is no outside assistance unless explicitly permitted — so if the rules aren't stated, ask before the interview. In design rounds, the human judgment the framework surfaces is precisely what's being assessed.
- **System design reaches further down the ladder.** Reports suggest design rounds now appear in some mid-level loops. For senior and architect candidates, that raises the bar: a "working system" is table stakes, and the differentiation is in steps 1, 6 and 7.

---

# Part D — Worked example: a ticketing platform, 45 minutes, end to end

## Concept 23 — "Design a ticketing platform like Ticketmaster"

This walkthrough follows the 45-minute budget from Concept 4a, with the clock in the margin, the words you'd actually say, the board state at each step, and the decisions with their trade-offs. The platform is Azure and the services are .NET, because that's the context of the roles you're targeting — but every decision is stated in generic terms first.

### 0:04 — Step 1: Requirements

> *"Before designing, I want to pin down scope. Do users pick specific seats, or is it general admission?"*
>
> **Interviewer:** "Assigned seating — users pick seats on a map."
>
> *"Great — that makes contention per seat, not just a counter, which matters a lot. Core functional requirements, then: one, users can search and browse events; two, users can view an event and its seat availability; three, users can select seats, hold them while paying, and complete the purchase. Out of scope: resale, dynamic pricing, organizer tooling, and payment processing itself — we'll integrate with a payment provider."*
>
> *"Non-functional: booking is strongly consistent — a seat is never sold twice. Browse and search can be a few seconds stale. Search p99 under 500 ms. Holds last ten minutes. The platform must survive on-sale spikes — I'll assume a top event draws 10 million users for 50,000 seats. Payments must never charge twice. And I'll assume we're on Azure with a .NET team."*
>
> *"The hard part is the on-sale: extreme contention on a tiny inventory, under a spike several orders of magnitude above normal, with a hard correctness requirement. That's where I'll spend the deep dives."*

Board, left column:

```
FR  1 search/browse events  2 view event + seat availability  3 hold seats → purchase
OUT resale, dynamic pricing, organizer tools, payment internals
NFR booking: strongly consistent (never double-sell) · browse/search: ≤ few s stale
    search p99 < 500 ms · hold TTL 10 min · on-sale: 10M users / 50k seats
    payments: no double charge · Azure + .NET
★   hard part: contention + spike + correctness at on-sale
```

### 0:09 — Step 2: Estimation (only what decides something)

> *"I'll only compute the numbers that change the design."*

| Quantity | Arithmetic | So… |
|---|---|---|
| Normal browse load | 10M DAU × ~20 page views ≈ 2×10⁸/day ≈ 2.3k rps avg, ~10k rps peak | Ordinary; cache plus a few replicas. Not a deep dive. |
| On-sale availability reads | 10M users refreshing every ~5 s ≈ **2M rps** on one event | The origin can't take this. We need to control *who* reaches the booking path, and serve availability from cache or push. |
| Successful bookings for that event | 50k seats ÷ ~2 per order ≈ 25k orders | The writes that *succeed* are few; the problem is contention and failed attempts, not write volume. |
| Storage | ~500M tickets/year × ~1 KB ≈ 500 GB/year | Storage is not the problem. A relational store handles it; partition by event later if needed. |

> *"So: storage and normal traffic are routine. The design is shaped by the 2M-rps spike and per-seat contention."*

### 0:12 — Core entities and API

```
Entities: Event · Venue · Seat (physical) · EventSeat (sellable unit, per event) · Hold · Order · User

GET  /v1/events?q=&city=&from=&cursor=      → Page<EventSummary>
GET  /v1/events/{id}                         → Event
GET  /v1/events/{id}/availability            → SectionAvailability[]      (cached; display only)
POST /v1/events/{id}/holds   Idempotency-Key { seatIds[] }
                                             → 201 { holdId, expiresAt } | 409 SeatsUnavailable
POST /v1/orders              Idempotency-Key { holdId }
                                             → 202 { orderId, status: PendingPayment, paymentSession }
GET  /v1/orders/{id}                         → Order
POST /v1/payments/webhook    (provider → us; signature verified)
```

> *"User identity comes from the token on every call. Holds and orders take idempotency keys, so a client retrying after a timeout can't double-hold or double-pay. Ordering is asynchronous because the payment provider completes it — the client polls the order or gets a push."*

### 0:16 — Step 4: Data model

> *"Access patterns first."* (Points at the API.) *"Claiming seats is a conditional multi-row write that must be linearizable per seat — that's the pattern that picks the store. I want transactions and conditional updates, so relational: Azure SQL Database, zone-redundant. Search is full-text with filters, so a search index fed asynchronously. Availability is a stale-tolerant aggregate, so cache."*

```
Inventory DB (Azure SQL, zone-redundant)
  event_seats  PK (event_id, seat_id) · section_id · status {available|held|sold}
               hold_id · hold_expires_at · order_id
               IX (event_id, section_id, status)
  holds        PK hold_id · user_id · event_id · expires_at · status {active|checkout|released|converted}
  orders       PK order_id · user_id · hold_id · status · amount · psp_payment_id
               UNIQUE (user_id, idempotency_key)
  outbox       PK id · type · payload · created_at · dispatched_at

Search index (Azure AI Search)   events: name, performers, venue, city, date, category
Cache (Azure Managed Redis)      availability:{eventId} → section counts, TTL ~1–2 s
```

> *"Partition key thinking: every booking operation is scoped to one event, so `event_id` keeps transactions local if we ever shard. The risk is the mega-event — all traffic on one key — which is exactly the on-sale deep dive."*

### 0:20 — Step 5: High-level design

```
                    ┌──────────────┐
 Browser/App ──①───►│ Front Door   │── static + cached GETs (CDN)
                    │ (WAF, edge)  │
                    └──────┬───────┘
                           │②
                    ┌──────▼───────┐          ┌──────────────────┐
                    │ API gateway  │──③──────►│ Event service    │──► Search index (AI Search)
                    │ (auth, rate  │          │ (browse, detail, │──► Redis (availability cache)
                    │  limits)     │          │  availability)   │──► Inventory DB (on miss)
                    └──────┬───────┘          └──────────────────┘
                           │④
                    ┌──────▼───────────┐  ⑤ conditional claim     ┌──────────────────────┐
                    │ Booking service  │─────────────────────────►│ Inventory DB          │
                    │ (holds, orders)  │◄─────────────────────────│ (Azure SQL, ZR)       │
                    └──────┬─────▲─────┘                          │ event_seats, holds,   │
                         ⑥ │     │ ⑦ webhook                      │ orders, outbox        │
                    ┌──────▼─────┴─────┐                          └──────────┬───────────┘
                    │ Payment provider │                                     ┊ ⑧ outbox relay
                    └──────────────────┘                          ┌──────────▼───────────┐
                                                                  │ Service Bus topic     │
                                                                  └──┬────────────┬──────┘
                                                                     ┊            ┊
                                                         Search indexer      Notifications
                                                                             (email/push)
 Parking lot: ⚠ availability at 2M rps   ⚠ hold expiry vs payment duration   ⚠ search staleness
```

Narrated, endpoint by endpoint:

> *"① Clients hit Front Door, which serves static assets and short-TTL cached GETs and gives us WAF and bot protection. ② The gateway authenticates and applies coarse rate limits. ③ Browse and detail go to the event service, which reads the search index and caches event details. Availability is a cached aggregate of section counts with a TTL of a second or two."*
>
> *"④ Holds go to the booking service, which ⑤ claims seats in the inventory database — I'll go deep on exactly how. On purchase, ⑥ we create an order in PendingPayment and a payment session with the provider; the client pays directly with the provider. ⑦ The provider's webhook tells us the payment succeeded; in one transaction we mark the seats sold, convert the hold, confirm the order and write an outbox row. ⑧ An outbox relay publishes OrderConfirmed to Service Bus for emails and to keep search in sync."*
>
> *"Walkthrough: a user searches ③, opens the event and sees cached availability ③, picks two seats and gets a hold ⑤, pays ⑥, and the webhook ⑦ confirms; the email goes out via ⑧. All three requirements trace through. I've parked three things — the spike, holds expiring mid-payment, and search staleness."*

**Why no queue between the gateway and the booking service?** Because the user needs a synchronous answer — "you got these seats" or "they're gone" — and a queue would add latency and a pending state for no benefit (Concept 19). The async boundary is drawn where work genuinely completes later: payment confirmation, notifications, indexing.

### 0:29 — Step 6: Deep dives

> *"I'd like to go deep on two things: preventing double-booking with holds, and surviving the on-sale spike. If there's time, the staleness contract between search, availability and inventory. Sound good?"*

#### Deep dive 1 — Never sell a seat twice

**The problem.** Thousands of concurrent claim attempts on the best sections. The naive version is check-then-act: read the seat, see `available`, write `held`. Two requests interleave, both read `available`, both write — a lost-update race (Module 12's anomaly catalogue).

**Options.**

| Option | How it works | Strengths | Weaknesses |
|---|---|---|---|
| A. Pessimistic lock | `SELECT … WITH (UPDLOCK, ROWLOCK)` (SQL Server) or `SELECT … FOR UPDATE` (PostgreSQL), check, update, commit | Simple mental model | Lock held across a round trip; long waits and deadlocks under contention; throughput drops exactly when you need it |
| B. Conditional update | One `UPDATE … WHERE status is claimable`; check affected rows | Atomic per row; no read-then-write window; short transactions; single source of truth | Must handle partial success on multi-seat holds; contention still lands on the primary |
| C. External lock (Redis key per seat with TTL) | `SET seat:{id} holdId NX PX 600000`, then write the database | Very fast; TTL gives expiry for free | Two sources of truth that can disagree; lock expiry during a GC pause or failover can let two holders proceed; needs fencing (Module 9) |

**Decision: B, with the expiry in the predicate.** The claim is a single set-based statement that only succeeds for seats that are available *or* held with an expired hold. All seats in the request are claimed or none are:

```csharp
public sealed class HoldService(BookingDbContext db, TimeProvider clock)
{
    private static readonly TimeSpan HoldDuration = TimeSpan.FromMinutes(10);

    public async Task<HoldResult> PlaceHoldAsync(
        Guid eventId, Guid userId, IReadOnlyCollection<int> seatIds, CancellationToken ct)
    {
        var now = clock.GetUtcNow();
        var holdId = Guid.CreateVersion7();
        var expiresAt = now + HoldDuration;

        await using var tx = await db.Database.BeginTransactionAsync(ct);

        // One atomic statement. A seat is claimable if it is available,
        // or held under a hold that has already expired — so expiry needs no sweeper.
        int claimed = await db.EventSeats
            .Where(s => s.EventId == eventId
                     && seatIds.Contains(s.SeatId)
                     && (s.Status == SeatStatus.Available
                         || (s.Status == SeatStatus.Held && s.HoldExpiresAt < now)))
            .ExecuteUpdateAsync(set => set
                .SetProperty(s => s.Status, SeatStatus.Held)
                .SetProperty(s => s.HoldId, holdId)
                .SetProperty(s => s.HoldExpiresAt, expiresAt), ct);

        if (claimed != seatIds.Count)
        {
            await tx.RollbackAsync(ct);           // all-or-nothing: release partial claims
            return HoldResult.SeatsUnavailable();
        }

        db.Holds.Add(new Hold(holdId, userId, eventId, expiresAt, HoldStatus.Active));
        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
        return HoldResult.Held(holdId, expiresAt);
    }
}
```

**Why this is correct, stated as reasoning rather than vocabulary:**

- **Atomicity per row comes from the engine.** Each row update takes a row lock and re-evaluates the predicate against the current committed value, so two concurrent claims on the same seat can't both match. Read Committed is sufficient; there's no read-then-write gap to protect with a higher isolation level.
- **All-or-nothing comes from the transaction.** If the user asked for four seats and only three matched, the rollback releases the three.
- **Expiry needs no background job for correctness.** An expired hold is claimable by the predicate itself. A cleanup job can still run to tidy statuses for display, but if it stops, nothing double-sells — correctness doesn't depend on it.
- **Deadlocks are possible but rare and safe.** Two multi-seat claims that overlap can lock rows in different orders. The engine kills one as a deadlock victim (SQL Server error 1205, PostgreSQL 40P01); the booking service retries the whole transaction once with jitter, outside the transaction, or returns 409. Retrying inside the failed transaction would be a bug (Module 25).
- **Idempotency.** The hold endpoint stores the `Idempotency-Key` with the result; a retried request replays the original `holdId` rather than trying to claim again.

**The payment race (the parked item).** A user might take nine minutes to type card details, and the hold expires at ten. The fix is a state transition, not a longer TTL for everyone. When the user starts checkout, the hold moves to `checkout` and its expiry is extended once to a bounded maximum (say 15 minutes total), conditional on the hold still being active. When the webhook arrives, confirmation is conditional on ownership:

```sql
UPDATE event_seats SET status = 'sold', order_id = @orderId
WHERE event_id = @eventId AND hold_id = @holdId AND status = 'held';
-- affected rows must equal the hold's seat count, in the same transaction
-- that confirms the order and inserts the outbox row
```

If the affected row count is short — the hold genuinely lapsed and someone else claimed a seat — the order fails and the payment is voided or refunded. That compensation path is rare by construction (the extension makes it so), but it must exist. Using authorize-then-capture with the provider makes it a void rather than a refund, which is better for users.

**Payments never charge twice.** The order creation call to the provider carries our order ID as the provider-side idempotency key; the webhook handler is idempotent on the provider's event ID (an inbox table with a unique constraint), because webhooks are delivered at least once (Module 11).

**What I'd monitor.** Claim latency p99, claim conflict rate, deadlock retries per second, hold-to-order conversion, and a scheduled invariant query that must always return zero: seats with more than one confirmed order, or sold seats without a confirmed order.

> **Interviewer:** "Why not just use Redis locks? They'd be faster."
>
> *"They would — but then the lock and the inventory row are two sources of truth. If a Redis lock expires during a long GC pause or a failover while the holder still thinks it owns the seat, two users can proceed, and I'd need fencing tokens checked by the database to make it safe. At that point the database is doing the real work anyway. Since admission control — the next deep dive — caps the claim rate at a level the database handles comfortably, I'd keep one source of truth. If we ever needed far higher claim rates, I'd partition an event's inventory by section across databases before introducing a second source of truth."*

#### Deep dive 2 — Surviving the on-sale spike

**The problem, with the number from step 2.** Ten million users arriving within minutes, each refreshing the seat map every few seconds: about 2M requests per second on one event. Fifty thousand seats means only a small fraction of these users can ever buy. Scaling the booking path to 2M rps would be enormously expensive, and even then every request would contend on the same inventory rows.

**Options ladder.**

1. **Good — cache availability aggressively.** Serve section-level counts (not per-seat maps) from Redis with a 1–2 s TTL, behind Front Door caching. This turns 2M rps of database reads into a trickle — but 2M rps still reaches the edge and the cache tier, and nothing limits how many users hammer the booking endpoint.
2. **Better — push instead of poll for admitted users.** Users on the seat map subscribe to availability deltas over Azure SignalR Service; the booking service publishes "section X now has N left" when holds change. This removes polling but doesn't solve the fundamental mismatch between 10M people and 50k seats.
3. **Best — admission control with a virtual waiting room.** Only let in as many buyers as the booking path can serve; queue everyone else *before* they touch the origin.

**Decision: a waiting room, sized with Little's Law.**

- **Booking capacity.** Suppose load testing shows the inventory database comfortably sustains about 3,000 claim attempts per second for one event, and an admitted user makes roughly one claim attempt every 20 seconds (browsing, failed attempts, retries) — 0.05 attempts per second per user.
- **Concurrent admitted users** L = 3,000 ÷ 0.05 = 60,000.
- **Admitted session time** W ≈ 5 minutes = 300 s.
- **Admission rate** λ = L ÷ W = 60,000 ÷ 300 = **200 users per second**.

That sets the waiting room's output rate. Implementation choices:

| Approach | How | Trade-off |
|---|---|---|
| Edge waiting room (e.g., Cloudflare Waiting Room) | Edge workers queue users and admit at a configured rate with a cookie | Absorbs the spike before it reaches your infrastructure; less control, vendor coupling |
| Custom waiting room | Issue queue positions from an atomic counter in Redis; a status endpoint (edge-cacheable, cheap) tells users their position; an admitter grants signed, short-lived admission tokens (a JWT) at λ; the gateway rejects booking calls without a valid token | Full control over fairness and messaging; you build and operate it |

**Fairness.** First-come-first-served rewards whoever has the lowest latency and the most bots; many platforms randomize positions for everyone who arrives before the on-sale opens, then queue later arrivals FIFO. Cloudflare's product supports FIFO, random and lottery admission for this reason. Combine with bot protection at the edge, one admission token per verified account, and per-account hold limits.

**The insight that makes this a staff-level answer.** With 25,000 orders available and, say, a 50% conversion rate among admitted users, the event sells out after roughly 50,000 admissions — about four minutes at 200 per second. Everyone beyond position ~50,000 in a 10M-person queue will not get a ticket. The waiting room should *say so*: show sellout likelihood by position, stop admitting when inventory reaches zero, and switch the page to "sold out" and a waitlist. That isn't just better UX; it collapses the load from millions of users who would otherwise keep refreshing.

**What the waiting room itself must survive.** 10M users polling status every 20 s is about 500k rps — so the status response must be cacheable at the edge or computed without touching a central store (position is in the user's signed cookie; the "now serving" number is a single value refreshed every second). If the waiting room fails, it must **fail closed** — admit no one rather than everyone — because failing open sends 10M users straight to the booking path.

#### Deep dive 3 (brief) — The staleness contract

> *"Three views of availability exist, deliberately: search results (seconds stale, via the outbox and an indexer), the availability cache (one to two seconds stale), and the inventory database (authoritative). The rule is that only the claim decides; everything else is a hint. The UI says 'few left' rather than an exact count during on-sale, and a 409 on a hold is a normal, well-designed outcome — the client refreshes the section and suggests alternatives."*

This is Module 10's framing in action: a cache is an asynchronous replica with weak consistency, and the design states the bound instead of pretending it's exact.

### 0:41 — Step 7: Wrap-up

> *"Main bottleneck: for a mega-event, all claims hit one inventory primary. With admission at 200 users per second that's about 3,000 claims per second, which is comfortable — but it's the ceiling on how fast we can let people in. If that's not enough, I'd partition the event's inventory by section across databases before adding a second source of truth."*
>
> *"Failure modes: if the payment provider is down, checkouts stall, holds expire and seats return — we fail closed on sales. If Service Bus is unavailable, the outbox buffers in the database and emails are delayed, not lost. If the waiting room fails, it fails closed. Regionally, booking runs active-passive with a zone-redundant primary; I'd avoid async geo-replication for automatic failover of inventory, because losing the last few seconds of writes could mean selling a seat twice after failover — that's a deliberate RPO decision (Module 13)."*
>
> *"I'd monitor claim p99 and conflict rate, deadlock retries, admission rate versus target, queue length, hold conversion, and the double-sale invariant, which must always be zero."*
>
> *"With more time: general-admission sections, where contention becomes a counter we'd decrement atomically, possibly sharded counters per section; resale with the same hold machinery; and a load-test plan that rehearses the on-sale against a production-sized copy of the inventory."*

### The board at minute 43

```
REQUIREMENTS  FR 1 search  2 view+availability  3 hold→purchase   OUT resale, pricing…
              NFR never double-sell · ≤ few s stale reads · p99 search < 500 ms · 10M→50k spike
NUMBERS       browse ~10k rps (routine) · on-sale ~2M rps (!) · 25k orders · 500 GB/yr (not a problem)
API           holds + orders with Idempotency-Key · 202 for orders · cursor pagination
DATA          event_seats PK(event_id, seat_id) · conditional claim · outbox
HLD           Front Door → gateway → event svc / booking svc → Azure SQL · Redis · AI Search · Service Bus
DD1           conditional UPDATE, expiry in predicate · checkout extension · ownership-conditional confirm · idempotent webhook
DD2           waiting room: λ = L/W = 200/s · random-then-FIFO · fail closed · tell users when sold out
DD3           only the claim decides; other views are hints
WRAP          bottleneck: inventory primary per event · fail closed · no async geo-failover for inventory
```

### What made this a senior answer

- The hard part was named in minute five, and every later choice pointed at it.
- Estimation produced four numbers, each with a "so…", including two that ruled things *out*.
- The API carried the correctness story (idempotency, async completion, approximate availability).
- The store was derived from access patterns, and the partition key's risk was flagged before the deep dive.
- The high-level design was complete and simple, with three items parked visibly.
- Deep dives followed the anatomy: problem with a number, naive failure, options, decision tied to requirements, consequences and monitoring.
- The interviewer's challenge was answered with reasons, and the answer named the condition under which the decision would change.
- The wrap-up was operational and honest, including a deliberate trade-off on regional failover.

---

# Part E — The architect variant

## Concept 24 — "Re-platform our order management system"

Architect rounds look different on the surface but run the same seven steps. Here is a condensed walkthrough of a typical enterprise prompt, showing how each step translates.

> *Prompt: "We run an order management system — a .NET Framework 4.8 monolith on IIS with a single SQL Server, on-premises. It integrates with an ERP and a warehouse system. The business wants to move to Azure, support 5× order growth over three years, and let three teams ship independently. Design the target architecture and how we get there."*

**Step 1 → Drivers, constraints and quality-attribute scenarios.** The functional requirements are largely given (the system exists); the work is eliciting *why* change and *what can't change*. Ask about: the business drivers (growth, team autonomy, data-center exit date), constraints (budget, compliance, the ERP's integration capabilities, team skills, a freeze period around peak season), stakeholders (operations, finance, the ERP owners), and the non-negotiables (no lost orders, cutover downtime limits). Turn NFRs into scenarios: *"During the holiday peak at 5× today's volume, order submission p99 stays under 800 ms and no order is lost if one availability zone fails."*

**Step 2 → Capacity and cost.** Current and projected volumes, the database's size and growth, and the hot tables. Add cost: what does the target cost per month, and how does that compare with the current estate? Licensing matters here (SQL Server per-core licensing, Azure Hybrid Benefit).

**Step 3 → Integration contracts and boundaries.** Instead of public endpoints, define the contracts between bounded contexts and with external systems: which events the order context publishes, which commands the warehouse accepts, which data the ERP owns. A context map is the natural artifact (Module 22).

**Step 4 → Data ownership and migration.** Who owns each table today, and who will own it tomorrow? Which shared tables have to be split, and in what order? This is usually the hardest part of the whole exercise (Modules 12 and 21).

**Step 5 → Target *and* transition architecture.** Draw where you're going *and* the intermediate states. A typical shape: rehost or replatform the monolith first (upgrade to .NET 10 incrementally, move it to App Service or Container Apps, move the database to Azure SQL Managed Instance for compatibility), then modularize inside the monolith (Module 21's modular monolith), then extract the parts with genuinely different scaling or team needs using the strangler-fig pattern behind a YARP or API Management façade.

**Step 6 → Risk-driven deep dives.** Choose the riskiest transitions: the data split (dual writes vs change data capture vs outbox-based synchronization during migration), the cutover plan with rollback, and consistency across the new service boundaries (sagas replacing distributed transactions, Module 12).

**Step 7 → Roadmap, decisions, risks, organization.** A phased roadmap with exit criteria for each phase, architecture decision records for the one-way doors (database platform, integration style, hosting model), a risk register with mitigations, and the team topology that matches the target boundaries (Conway's law, Module 21). Name what you would *not* do: "I would not split into microservices before the modular boundaries are proven inside the monolith."

The difference from Part D isn't the steps — it's the weight. The architect round spends far more time on constraints, transition and risk, and far less on the internals of any one component.

---

## Putting it together: how this shows up in interviews

The framework itself is rarely asked about directly. It shows up as the *shape* of every design round, and as a handful of meta-questions and situations that test whether you can manage the session. Here are the common ones, with what a strong answer contains.

### Common questions and situations

**"Can we skip the estimation?"**
Yes — and say what you'll do instead: "Sure. The one number I'd want is the peak on-sale load, because it determines whether we need admission control. I'll assume about 2M requests a second and move on." You've complied and kept the one estimate that matters.

**"Assume infinite scale" / "Don't worry about scale."**
Take the interviewer at their word and redirect your depth to whatever *is* the hard part — usually correctness, consistency or the data model. Don't sneak scale back in; they've told you what they want to score.

**You're at minute 30 and the high-level design isn't finished.**
Say it, cut scope, and protect the deep dive: "I'm going to represent search as a single box fed from the outbox and come back to it if there's time — I'd rather get to the booking path." Prioritizing out loud is itself evidence of judgment.

**The interviewer asks about something you parked.**
"Good — that's the first item on my parking list." Then treat it as a deep dive with the anatomy from Concept 12. You get credit for having seen it and for handling it now.

**"Why didn't you use microservices / Kafka / Cosmos DB here?"**
Answer with the requirement that made the simpler choice sufficient and the condition under which you'd change: "One booking service is enough at three thousand claims a second; I'd split it if a separate team needed to own payments, or if payments needed to scale independently." Knowing when *not* to use a pattern is one of the clearest senior signals.

**"What would you do differently at 10× the traffic?"**
Don't redraw everything. Go to the numbers, find which box breaks first, and change that: "At 10×, admission would have to rise, which pushes claims to 30k per second per event — beyond one primary. I'd partition the event's inventory by section across databases; nothing else changes."

**"Walk me through exactly what happens when a user clicks Buy."**
This is a request for the walkthrough test (Concept 11d) with mechanisms. Use the flow numbers and name the state change at every hop, including what happens if each step fails.

**"Which of these would you deep-dive first, and why?"**
Rank by risk to the requirements, not by interest: correctness failures first (they are unrecoverable), then availability under load, then performance, then cost.

**"Tell me about a system you designed."**
Run the seven steps in the past tense (Concept 21): context and requirements, the numbers that mattered, the contract, the data model, the architecture, the two hardest problems and their trade-offs, and what you'd change now. End with what you learned. Prepare two of these.

**"What are the weaknesses of your design?"**
This is the wrap-up arriving early. Give the top bottleneck with a number, the top failure mode with its behavior, and what you'd monitor. Never "I think it's pretty solid."

**The interviewer is silent and gives no feedback.**
Silence is not disapproval. Keep checking in at step boundaries, keep narrating decisions, and make your reasoning visible on the board. Some interviewers are trained to stay neutral.

**The interviewer pushes a design you believe is worse.**
Engage with it seriously and compare: "That would work, and it's faster. My concern is that it gives us two sources of truth for seat ownership — here's the failure case. If you'd like, I can design it that way with fencing to close that gap." Then follow their lead; it's their session.

### Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Starts drawing in the first minute | Spends five minutes on scope and quantified NFRs, then names the hard part |
| Lists ten functional requirements | Prioritizes three, writes explicit out-of-scope |
| "Low latency, highly available, scalable" | Attaches numbers to specific operations ("hold p99 < 300 ms") |
| Computes DAU, QPS, storage, bandwidth, then "so it's a lot" | Computes only numbers that decide something; each ends with "so…" |
| Puts `userId` in request bodies | Derives identity from the token |
| No idempotency on money or inventory | Idempotency keys on unsafe operations, unique constraints to enforce them |
| Offset pagination by default | Cursor pagination, with the reason |
| Picks the database first, then the access patterns | Derives the store from an access-pattern table |
| Lists every column | Writes only the fields that matter, with keys and the partition key |
| Adds Kafka, caches and microservices in the high-level design | Simplest complete design first; parks optimizations visibly |
| High-level design never finished | Checks the clock at minute 25 and cuts scope out loud |
| Waits for the interviewer to choose deep dives | Proposes two or three, derived from NFRs and breaking points, and offers the choice |
| Deep dive adds boxes but no mechanism | Explains isolation, locking, failure behavior, invalidation, expiry |
| One option presented as the answer | Two or three options, compared on criteria that matter here, decision tied to a number |
| Defends a weak choice under pushback | Re-reasons openly; concedes gracefully when the interviewer is right |
| Treats interviewer questions as interruptions | Treats them as the agenda; answers, then returns explicitly |
| Names technologies as logos | Names a technology when a property drives the choice, and can defend it one level down |
| "I'd add monitoring and logging" | Names the SLOs, alerts and invariant checks that matter for this system |
| "I think the design is complete" | Names the top bottleneck, top failure mode and what would change the design |
| Talks continuously | Pauses after decisions; checks in at step boundaries |
| Same performance for every round type | Adapts weight: API-heavy for product architecture, internals for infrastructure, transition and risk for architect cases |

---

## Practice exercises

**Exercise 1 — The five-minute requirements drill (repeat daily for a week).** Take ten prompts (ticketing, chat, news feed, URL shortener, rate limiter, payment system, ride sharing, file sync, metrics monitoring, collaborative editing). For each, set a five-minute timer and produce only: three functional requirements, out-of-scope, three to five quantified NFRs, and one sentence naming the hard part. Compare your "hard part" with a published breakdown (Hello Interview's are free). The goal is to make step 1 automatic and fast.

**Exercise 2 — "So…" estimation.** For five of those prompts, write at most four numbers, each followed by the decision it drives. Delete any number whose "so…" is "it's large." Then write one number that proves something is *not* a problem.

**Exercise 3 — Contract-first API.** For three prompts, write the API in five minutes, then audit it against the Concept 9c list: identity from token, idempotency keys, cursor pagination, 202 for async work, explicit conflicts, versioning. Implement one in ASP.NET Core minimal APIs with `AddProblemDetails()` and an idempotency filter backed by a unique constraint.

**Exercise 4 — Deep-dive anatomy on paper.** Pick three deep dives from Phase 3 (fan-out on write vs read, cache invalidation for a product catalog, exactly-once payment processing). For each, write one page: problem with a number, naive approach and its failure, three options in a table, decision tied to a requirement, consequences and monitoring. These pages become reusable material.

**Exercise 5 — Implement and break Deep Dive 1.** Build the hold service from Part D with EF Core against SQL Server or PostgreSQL (Aspire makes the local setup quick). Write a concurrency test that fires 1,000 parallel claims at the same 10 seats and asserts exactly 10 succeed. Then replace the conditional update with read-check-write and watch the test fail. Add a deadlock-retry policy and measure retries under overlapping multi-seat claims.

**Exercise 6 — Change one requirement, trace the impact.** Re-run the ticketing design with *general admission* instead of assigned seats. Write down what changes in each step (API, data model, the contention mechanism, the waiting room math) and what doesn't. This trains Concept 14.

**Exercise 7 — Full timed mock, recorded.** Do a 45-minute mock with a peer or a mock-interview platform, using the same drawing tool you'll use on the day. Record it. Afterwards, mark the clock time at each step boundary and compare with Concept 4a. Score yourself with the checklist below.

**Exercise 8 — The architect case.** Take a real system you've worked on (anonymized) and run Part E's translation: drivers and constraints, scenarios, contracts, data ownership, target and transition states, top three risks, roadmap with ADRs. Present it in 60 minutes to a peer. This doubles as preparation for "tell me about a system you designed."

### Self-scoring checklist for mocks

| # | Check | ✓ |
|---|---|---|
| 1 | Requirements finished by ~minute 9 (of 45) | |
| 2 | ≤ 3 core FRs, explicit out-of-scope | |
| 3 | NFRs quantified and attached to operations | |
| 4 | Named the hard part before designing | |
| 5 | Every estimate had a "so…" | |
| 6 | API had identity-from-token and idempotency where needed | |
| 7 | Store derived from access patterns; partition key discussed | |
| 8 | Complete HLD by ~minute 28, with a walkthrough | |
| 9 | Optimizations parked visibly, not added prematurely | |
| 10 | Proposed deep dives and offered the interviewer a choice | |
| 11 | Each deep dive: problem → naive → options → decision → consequences | |
| 12 | Tied at least two decisions to numbers on the board | |
| 13 | Answered probes fully, then returned explicitly | |
| 14 | Wrap-up: bottleneck with number, failure mode, monitoring, evolution | |
| 15 | Checked in at step boundaries; didn't monologue | |

---

## Free resources

### Delivery frameworks and interview method

| Resource | What it covers | Why read it |
|---|---|---|
| [Hello Interview — Delivery Framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery) | Step-by-step structure with timings | Closest published match to this module; written by former FAANG interviewers |
| [Hello Interview — System Design in a Hurry: Introduction](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction) | What the round assesses | Short orientation to the rubric dimensions |
| [Hello Interview — How to Prepare](https://www.hellointerview.com/learn/system-design/in-a-hurry/how-to-prepare) | A study plan around the framework | Good for sequencing practice |
| [ByteByteGo — A Framework for System Design Interviews](https://bytebytego.com/courses/system-design-interview/a-framework-for-system-design-interviews) | Alex Xu's 4-step process with time allocations | The most widely known framework; free chapter |
| [ByteByteGo — How to Ace System Design Interviews](https://bytebytego.com/guides/how-to-ace-system-design-interviews-like-a-boss/) | A 7-step variant | Compare orderings with this module |
| [interviewing.io — A Senior Engineer's Guide to the System Design Interview](https://interviewing.io/guides/system-design-interview) | Approach, expectations by level, who drives | Explicit on senior expectations |
| [interviewing.io — Part 2: Fundamental concepts](https://interviewing.io/guides/system-design-interview/part-two) | Why "why" beats "what" | Reinforces trade-off narration |
| [interviewing.io — Part 4: Worked examples](https://interviewing.io/guides/system-design-interview/part-four) | Includes a ticketing walkthrough with SQL schema | A second take on Part D |
| [The System Design Primer](https://github.com/donnemartin/system-design-primer) | How to approach a design question, plus fundamentals | Classic free overview |
| [Tech Interview Handbook — System Design](https://www.techinterviewhandbook.org/system-design/) | Interview approach and resources | Concise, curated |

### Variants of the round

| Resource | What it covers |
|---|---|
| [Hello Interview — Meta System Design vs Product Architecture](https://www.hellointerview.com/blog/meta-system-vs-product-design) | How the two Meta rounds differ |
| [Hello Interview — How to prepare for Meta's Product Architecture interview](https://www.hellointerview.com/blog/how-to-prepare-meta-pa) | User-facing prompts, API emphasis |
| [IGotAnOffer — Meta Product Architecture interview](https://igotanoffer.com/en/advice/meta-product-architecture-interview) | Format and example questions |
| [Google SRE Workbook — Introducing Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/) | One machine → distributed → multi-datacenter, with arithmetic |
| [Google Cloud — SRE Classroom: exercises for NALSD](https://cloud.google.com/blog/products/devops-sre/join-sre-classroom-nalsd-workshops) | Workshop format for practicing NALSD |
| [NALSD workshop workbook (PDF)](https://static.googleusercontent.com/media/sre.google/en//static/pdf/nalsd-workbook-a4.pdf) | A full practice problem with solutions |
| [Hello Interview — Low-Level Design Delivery Framework](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/delivery) | The framework compressed for object-oriented design |
| [Hello Interview — ML System Design Delivery Framework](https://www.hellointerview.com/learn/ml-system-design/in-a-hurry/delivery) | The framework adapted for ML systems |
| [Exponent — System Design Interview Guide (2026)](https://www.tryexponent.com/blog/system-design-interview-guide) | What's changed in 2026: AI prompts, cost and operations grading |
| [Formation — 3 ways AI has changed FAANG interviews in 2026](https://formation.dev/blog/3-ways-ai-has-changed-faang-interviews-in-2026) | GenAI design prompts, AI-assisted rounds |

### Step 1 — Requirements and quality attributes

| Resource | What it covers |
|---|---|
| [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) | How to make "available" and "fast" measurable |
| [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/) | Choosing SLIs per user journey |
| [ISO/IEC 25010 quality model](https://iso25000.com/index.php/en/iso-25000-standards/iso-25010) | A complete checklist of quality attributes |
| [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/) | Reliability, security, cost, operations, performance pillars as requirement prompts |
| [Azure Well-Architected — Identify and rate user and system flows](https://learn.microsoft.com/en-us/azure/well-architected/reliability/identify-flows) | Attaching NFRs to specific flows |

### Step 2 — Estimation

| Resource | What it covers |
|---|---|
| [Hello Interview — Numbers to Know](https://www.hellointerview.com/learn/system-design/core-concepts/numbers-to-know) | Modern hardware and service limits |
| [Hello Interview — Mastering Estimation](https://www.hellointerview.com/blog/mastering-estimation) | Estimating to decide |
| [Latency numbers every programmer should know](https://gist.github.com/jboner/2841832) | The classic table |
| [Interactive latency numbers by year](https://colin-scott.github.io/personal_website/research/interactive_latency.html) | How the numbers changed over time |
| [Jeff Dean — Designs, Lessons and Advice from Building Large Distributed Systems (PDF)](https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf) | Back-of-the-envelope design in practice |

### Step 3 — API design

| Resource | What it covers |
|---|---|
| [Hello Interview — API Design](https://www.hellointerview.com/learn/system-design/core-concepts/api-design) | REST, GraphQL, RPC for interviews |
| [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) | Microsoft's and Azure's API conventions |
| [Google API Improvement Proposals (AIP)](https://google.aip.dev/) | Resource-oriented design, pagination, long-running operations |
| [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/) | Thorough, opinionated, practical |
| [Stripe — Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) | The canonical idempotency-key explanation |
| [Stripe API — Idempotent requests](https://docs.stripe.com/api/idempotent_requests) | How a production API implements it |
| [IETF draft — The Idempotency-Key HTTP header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) | The standardization effort |
| [Amazon Builders' Library — Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) | Idempotency design at scale |
| [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457) | The standard error format |
| [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) | Method safety, idempotency, status codes |
| [ASP.NET Core minimal APIs overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview) | Building the contract in .NET |
| [ASP.NET Core error handling and Problem Details](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling) | `AddProblemDetails`, `Results.Problem` |
| [gRPC on .NET](https://learn.microsoft.com/en-us/aspnet/core/grpc/) | When the contract is internal RPC |
| [ASP.NET API Versioning](https://github.com/dotnet/aspnet-api-versioning) | Versioning in practice |

### Step 4 — Data modeling

| Resource | What it covers |
|---|---|
| [Hello Interview — Data Modeling](https://www.hellointerview.com/learn/system-design/core-concepts/data-modeling) | Interview-level data modeling |
| [Azure Architecture Center — Understand data store models](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/data-store-overview) | Choosing a store by workload |
| [Azure Cosmos DB — Data modeling](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/modeling-data) | Embedding vs referencing, access-pattern-first |
| [Azure Cosmos DB — Partitioning overview](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview) | Choosing a partition key |
| [Alex DeBrie — The what, why, and when of single-table design](https://www.alexdebrie.com/posts/dynamodb-single-table/) | Access-pattern-driven modeling in NoSQL |
| [Use The Index, Luke](https://use-the-index-luke.com/) | Indexing as a design discipline |
| [EF Core — ExecuteUpdate and ExecuteDelete](https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete) | Set-based conditional updates (Deep Dive 1) |
| [EF Core — Handling concurrency conflicts](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) | Optimistic concurrency tokens |

### Step 5 — High-level design and diagramming

| Resource | What it covers |
|---|---|
| [Excalidraw](https://excalidraw.com/) | The most common virtual whiteboard in interviews |
| [draw.io / diagrams.net](https://www.drawio.com/) | Free diagramming, including Azure shapes |
| [The C4 model](https://c4model.com/) | Levels of architecture diagrams; great vocabulary for architect rounds |
| [Mermaid](https://mermaid.js.org/) | Diagrams as code, useful for take-homes |
| [Azure architecture icons](https://learn.microsoft.com/en-us/azure/architecture/icons/) | Official icons for Azure diagrams |
| [Azure Architecture Center — Architecture styles](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/) | N-tier, web-queue-worker, microservices, event-driven |
| [Azure Architecture Center — Browse reference architectures](https://learn.microsoft.com/en-us/azure/architecture/browse/) | Real Azure designs to study |

### Step 6 — Deep-dive patterns

| Resource | What it covers |
|---|---|
| [Hello Interview — Dealing with Contention](https://www.hellointerview.com/learn/system-design/patterns/dealing-with-contention) | Pessimistic, optimistic, and distributed approaches |
| [Hello Interview — Multi-step Processes](https://www.hellointerview.com/learn/system-design/patterns/multi-step-processes) | Sagas and workflows |
| [Hello Interview — Scaling Reads](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads) | Caching, replicas, denormalization |
| [Hello Interview — Scaling Writes](https://www.hellointerview.com/learn/system-design/patterns/scaling-writes) | Partitioning, batching, queues |
| [Hello Interview — Ticketmaster breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ticketmaster) | A full breakdown of Part D's prompt |
| [Hello Interview — Flash Sale breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/flash-sale) | Spike and contention under a related prompt |
| [Hello Interview — Payment System breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/payment-system) | Idempotency and reconciliation |
| [Hello Interview — Shopify Inventory Reservations (in the wild)](https://www.hellointerview.com/learn/system-design/in-the-wild/shopify-inventory-reservations) | A real reservation system |
| [Azure cloud design patterns catalog](https://learn.microsoft.com/en-us/azure/architecture/patterns/) | Named patterns with Azure context |
| [Queue-Based Load Leveling](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) | Absorbing bursts |
| [Throttling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/throttling) | Admission and shedding |
| [Rate Limiting pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/rate-limiting-pattern) | Protecting downstream capacity |
| [Cache-Aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) | The default caching pattern |
| [Compensating Transaction pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction) | Undoing completed steps (payment refunds) |
| [Saga distributed transactions](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga) | Coordinating multi-service workflows |
| [Transactional Outbox with Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-outbox-cosmos) | The outbox on Azure |
| [Strangler Fig pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig) | Incremental migration (Part E) |
| [Cloudflare Waiting Room — About](https://developers.cloudflare.com/waiting-room/about/) | How an edge waiting room admits users |
| [Cloudflare — How Waiting Room makes queueing decisions](https://blog.cloudflare.com/how-waiting-room-queues/) | Distributed admission control internals |

### Step 7 and architect-round artifacts

| Resource | What it covers |
|---|---|
| [Architecture Decision Records (adr.github.io)](https://adr.github.io/) | ADR formats and tooling |
| [Michael Nygard — Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) | The original ADR post |
| [Azure Well-Architected — Architecture decision record](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) | Microsoft's guidance on ADRs |
| [arc42](https://arc42.org/) | A template for architecture documentation |
| [Azure Well-Architected — Mission-critical workloads](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-overview) | Reliability-first design end to end |
| [Martin Fowler — Software Architecture Guide](https://martinfowler.com/architecture/) | Essays on architecture and evolution |

### .NET and Azure services used in the worked example

| Resource | What it covers |
|---|---|
| [Azure Front Door overview](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview) | Global edge, CDN, WAF |
| [Azure SQL Database overview](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview) | The inventory store |
| [Azure Database for PostgreSQL flexible server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) | The alternative relational choice |
| [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) | The current managed Redis offering |
| [What's new in Azure Cache for Redis (retirement timeline)](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-whats-new) | Why to name Azure Managed Redis in 2026 |
| [Azure Service Bus overview](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview) | Queues and topics |
| [Azure AI Search overview](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search) | The search index |
| [Azure SignalR Service overview](https://learn.microsoft.com/en-us/azure/azure-signalr/signalr-overview) | Push updates to admitted users |
| [Aspire](https://aspire.dev/) | Local orchestration for Exercise 5 |
| [eShop reference application](https://github.com/dotnet/eShop) | A modern .NET reference architecture |
| [.NET Microservices: Architecture for Containerized .NET Applications (free e-book)](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/) | .NET architecture patterns, boundaries, communication |

### Engineering writing worth reading for deep-dive material

| Resource | What it covers |
|---|---|
| [Amazon Builders' Library](https://aws.amazon.com/builders-library/) | Production lessons on retries, load shedding, health checks, idempotency |
| [Google SRE books (free online)](https://sre.google/books/) | SLOs, overload, cascading failures, NALSD |
| [Discord — How Discord stores trillions of messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | A real data-model and hot-partition story |
| [Netflix Tech Blog](https://netflixtechblog.com/) | Large-scale architecture case studies |
| [InfoQ — Architecture & Design](https://www.infoq.com/architecture-design/) | Talks and articles from practitioners |
| [High Scalability](http://highscalability.com/) | Architecture write-ups of real systems |

### Video and mock practice

| Resource | What it covers |
|---|---|
| [Hello Interview (YouTube)](https://www.youtube.com/@hello_interview) | Full mock walkthroughs using a delivery framework |
| [ByteByteGo (YouTube)](https://www.youtube.com/@ByteByteGo) | Short visual explanations of components and patterns |
| [Jordan has no life (YouTube)](https://www.youtube.com/@jordanhasnolife5163) | Deep, theory-heavy design walkthroughs |
| [interviewing.io — Learning Center](https://interviewing.io/learn) | Recorded expert mock interviews and guides |
| [Hussein Nasser (YouTube)](https://www.youtube.com/@hnasr) | Backend fundamentals for deep dives |

---

## Quick-recall sheet

| If you need to… | Remember… |
|---|---|
| Recall the seven steps | Requirements → Estimation → API (entities first) → Data model → HLD → Deep dives → Wrap-up |
| Explain why that order | Each step consumes the previous step's output; order is default, dependencies are law |
| Budget a 45-minute round | Req ~5 · Est ~3 · API ~5 · Data ~3 · HLD ~9 · Deep dives ~12 · Wrap ~3 |
| Know if you're behind | Complete HLD by ~minute 25–28 of 45 (35–38 of 60), or cut scope aloud |
| Open strongly | Three FRs, out-of-scope, quantified NFRs, **name the hard part** |
| Estimate | Only numbers that decide something; every number ends with "so…" |
| Make the API senior | Identity from token, idempotency keys, cursor pagination, 202 for async, explicit 409 |
| Choose the store | Access-pattern table first; partition key is the most consequential line |
| Build the HLD | Endpoint by endpoint; simplest working design; park optimizations; walkthrough test |
| Choose deep dives | From unmet NFRs, breaking points, and interviewer probes; propose 2–3, offer a choice |
| Run a deep dive | Problem with a number → naive failure → options → decision tied to requirement → consequences |
| Narrate a trade-off | "A gives X at cost Y; given R, I take A; I'd revisit if Z" |
| Handle pushback | Re-reason openly; concede gracefully when they're right |
| Handle being stuck | Name the property; solve on one machine; then distribute |
| Name technology | Only with a reason, and only what you can defend one level down |
| Wrap up | Bottleneck with number, failure mode, monitoring, evolution — never "it's complete" |
| Adapt to the round | Product → API-heavy; infrastructure → internals; NALSD → arithmetic; architect → transition, risk, ADRs |

---

## Progress

Module 3 complete. It is the method that the rest of the curriculum fills in:

- **Module 4** deepens step 1 — the full catalogue of non-functional questions and how asking them signals seniority.
- **Module 5** deepens step 2 — the latency and throughput numbers to know cold and how to use them under pressure.
- **Phase 3 (Modules 6–13)** supplies the raw material for step 6: scalability and admission control (Module 6, used in Deep Dive 2), consistency models (Module 7, the staleness contract), partitioning (Module 8, the partition-key risk), coordination and fencing (Module 9, the case against Redis locks), caching (Module 10, availability as a weakly consistent replica), messaging and the outbox (Module 11, webhooks and notifications), transactions and anomalies (Module 12, the conditional claim) and regional failure (Module 13, the RPO decision).
- **Phases 4 and 5** supply the .NET implementation fluency that lets you go one level deeper than the box — EF Core's set-based updates (Module 19), modular boundaries (Module 21), aggregates and ubiquitous language (Module 22) and resilience composition (Module 25).

Use this module's self-scoring checklist on every mock from here on; it turns "that felt OK" into specific, fixable gaps.
