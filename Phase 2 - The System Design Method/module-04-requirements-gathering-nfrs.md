# Module 4 — Requirements Gathering & the Non-Functional Questions That Signal Seniority
*Phase 2: The System Design Method · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Module 3 gave you the seven-step skeleton and told you what step 1 must *produce*: three core functional requirements, an explicit out-of-scope list, three to five quantified non-functional requirements, and one sentence naming the hard part. This module is about *how you get there* — and why the questions you ask in those first five minutes are some of the densest evidence an interviewer collects about your level.

The central claim of this module, which we will derive rather than assert:

> **Functional requirements tell you what to build. Non-functional requirements tell you what shape it must have. Two systems with identical features and different non-functional requirements are different architectures.**

That's why seniority shows up here. Anyone can list features. Knowing which *qualities* will dominate a design — and asking the one question that forks it — is knowledge you only acquire by having seen systems fail. When you ask "what happens if this message is delivered twice?", the interviewer hears that you've debugged a duplicate charge. When you ask "how stale can the balance be on this screen?", they hear that you understand caches are replicas. Questions are compressed experience.

We build the material one concept at a time:

- **Part A** — what requirements are for, and the four different things people lump together as "requirements".
- **Part B** — functional requirements and the technique of eliciting them (including when to ask versus assume).
- **Part C** — the non-functional catalogue, one dimension per concept: scale, latency, availability, durability, consistency, security, compliance, operability, evolvability, cost, constraints, and the long tail.
- **Part D** — turning answers into design drivers: per-flow requirements, conflicts, prioritization, traceability, verification.
- **Part E** — the power questions, level calibration, anti-patterns, and the five-minute delivery.
- **Part F** — the architect variant: stakeholders, business drivers, brownfield discovery, documentation that survives.
- **Part G** — three worked examples, including one prompt delivered to three different customers and producing three different architectures.

| # | Concept | The one-line takeaway |
|---|---|---|
| **Part A — What requirements are for** | | |
| 1 | The job of step 1 | You're writing the rubric your own design will be judged against. |
| 2 | Four different things | Functional requirements, quality attributes, constraints and assumptions behave differently — label them. |
| 3 | Why NFRs shape architecture | Same features + different qualities = different system. |
| 4 | Why questions are graded | A question is compressed experience; its reason is the signal. |
| 5 | What makes a requirement usable | Measurable, scoped to an operation, conditioned on a load — the quality-attribute scenario. |
| **Part B — Functional requirements and elicitation** | | |
| 6 | Actors, journeys, the top three | Find the actors, trace their journeys, keep the three that matter. |
| 7 | The hidden functional requirements | Admin, abuse, deletion, backfill, export — name them, then scope them out deliberately. |
| 8 | Ask, or assume and state? | Ask only questions whose answer forks the design; assume the rest out loud. |
| **Part C — The non-functional catalogue** | | |
| 9 | Scale and workload shape | Peak not average, read/write ratio, burstiness, skew, growth. |
| 10 | Latency and performance | Which operation, which percentile, measured where — and tails amplify with fan-out. |
| 11 | Availability | SLI → SLO → SLA; nines are minutes; dependencies multiply; flows degrade separately. |
| 12 | Durability, recovery, data lifecycle | Durability ≠ availability; RPO and RTO are business numbers; replication isn't backup. |
| 13 | Consistency and correctness | Per-operation consistency, invariants, staleness bounds, ordering, idempotency. |
| 14 | Security, identity, tenancy | Who are the principals, what can each do, and how isolated are tenants? |
| 15 | Privacy, compliance, residency | Erasure, residency, PCI scope, sector regulation — the 2026 EU landscape. |
| 16 | Operability and observability | How will we deploy it, know it works, and who gets paged? |
| 17 | Maintainability and evolvability | How long will it live, how often will it change, how many teams touch it? |
| 18 | Cost | A budget is a requirement; unit cost is the language. |
| 19 | Constraints and environment | Existing estate, platform, team, clients and networks, deadlines. |
| 20 | The long tail | Accessibility, internationalization, interoperability, portability, safety. |
| **Part D — From answers to design drivers** | | |
| 21 | NFRs are per flow | Critical flows get strict targets; everything else gets cheaper ones. |
| 22 | NFRs conflict | Every strong requirement costs another quality; name the trade. |
| 23 | Prioritizing: utility tree and the hard part | Rank by importance × difficulty; the (High, High) cell is your deep dive. |
| 24 | Traceability | Every NFR must produce a mechanism, or it's decoration. |
| 25 | Verifying NFRs | Fitness functions, SLO telemetry, load and chaos tests — requirements you can't check aren't requirements. |
| **Part E — Signalling seniority** | | |
| 26 | The power questions | Twenty questions, each with the design fork it reveals. |
| 27 | Calibration by level | Mid lists; senior quantifies; staff reframes; architect elicits. |
| 28 | Anti-patterns | Interrogation, adjectives, gold-plating, asking and ignoring. |
| 29 | The five-minute delivery | A script and a board layout. |
| **Part F — The architect variant** | | |
| 30 | Eliciting drivers from stakeholders | Business goals → quality-attribute scenarios; "never down" → money. |
| 31 | Brownfield discovery | What exists, what hurts, what can't change, what's already promised. |
| 32 | Requirements that survive | Scenarios, SLO documents, arc42 §10, links to ADRs. |
| **Part G — Worked examples** | | |
| 33 | Chat system: the five minutes, verbatim | Ordering, no loss, fan-out — and presence must not hurt messaging. |
| 34 | One prompt, three customers | A URL shortener as an internal tool, a consumer product, and a regulated SaaS. |
| 35 | Architect elicitation: claims modernization | Drivers, constraints, scenarios, utility tree for an EU insurer. |

---

# Part A — What requirements are for

## Concept 1 — The job of step 1: writing your own rubric

Start from what a design is. A design is a set of decisions. A decision is good or bad only *relative to goals*: "use a relational database" is neither right nor wrong until you know whether the system needs multi-row invariants, 2 million writes per second, or schema flexibility across 400 tenants.

In a design interview, nobody hands you those goals. The prompt ("Design a chat system") is deliberately underspecified, which means **the first thing you produce is the evaluation criteria for everything that follows**. Every later decision will be justified against the requirements box on the board — "I'm using a cache here *because* the feed tolerates five seconds of staleness and is read 100 times per write." If the box is empty or vague, every later decision is ungrounded, and the interviewer has to guess whether your choices were reasoned or recited.

Three consequences follow, and they explain most of this module:

1. **Vague requirements make every later step weaker.** "Highly available and scalable" can't justify anything, because every system wants both. You can't trade off against an adjective.
2. **Wrong requirements make a good design score badly.** If you design for global strong consistency and the interviewer had a social feed in mind, your design is correct for the wrong problem — and expensive in latency for no reason.
3. **Too many requirements make a design impossible to finish.** Thirty-five minutes satisfies three functional requirements and a handful of qualities. Writing ten features is volunteering to fail at seven of them.

So step 1 has a precise job: **convert an open search space into a small, prioritized, measurable problem that the interviewer agrees with, and identify which requirement will dominate the design.** It's the scoping half of the "problem navigation" dimension that Module 1 showed appears, under different names, in most company rubrics.

**The interviewer's side of this.** Interviewers usually have a mental "full" version of the problem and a set of follow-ups prepared. Your requirements step tells them which version you're solving and how much ground you'll cover. A candidate who scopes well makes the interviewer's job easy: they can see your plan, nudge it in one sentence ("Let's include group chats"), and then spend the session probing depth. A candidate who scopes badly forces the interviewer to spend their time steering — and steering is evidence of a lower level (Module 3, Concept 5).

---

## Concept 2 — Four different things we call "requirements"

People use "requirements" for four kinds of statement that behave differently. Separating them — even just by labeling them on the board — is a small, cheap seniority signal, because it shows you know which ones you're allowed to trade away.

| Kind | What it is | Example | Can you trade it off? |
|---|---|---|---|
| **Functional requirement** | A behavior the system performs for an actor | "Users can send a message to a group." | Yes, by scoping — but within scope it's binary: it works or it doesn't. |
| **Quality attribute** (non-functional requirement) | A measurable property of *how well* the system performs its functions | "Message delivery p99 < 500 ms when both users are online." | Yes, against other qualities and cost — this is where design happens. |
| **Constraint** | A decision already made, outside your control | "Must run on Azure." "Must integrate with the existing SAP system." "Team of six .NET engineers." | No. You design around it. You may *note* its cost. |
| **Assumption** | Something you believe is true but haven't confirmed | "I'll assume 50 messages per user per day." | It's not traded — it's *validated* or *revised*. Unvalidated assumptions are risks. |

**Why the distinction matters.**

- **Constraints remove options; quality attributes rank them.** "Must use Azure" eliminates DynamoDB; it doesn't tell you anything about whether Cosmos DB or Azure SQL is better. Mixing them up leads to arguments about things that aren't up for discussion.
- **Assumptions should be visible because they're the first thing to break.** In real projects, the assumption "peak is three times average" is the one that turns out to be twelve times average on Black Friday. In interviews, a labeled assumption invites correction: the interviewer can say "actually, assume 10× spikes," and you've learned something cheaply.
- **Functional requirements are binary within scope; qualities are continuous.** You can't half-deliver "users can send messages," but you can deliver 99.9% instead of 99.99% availability, and that difference might save half the infrastructure budget. Design time belongs to the continuous ones.

**Terminology you'll hear.** "Non-functional requirement" (NFR) is the common interview term. Architecture literature prefers **quality attribute** (the SEI and Bass, Clements and Kazman's *Software Architecture in Practice*) or **quality requirement** (arc42, ISO/IEC 25010). Microsoft's Well-Architected Framework speaks of reliability, security, cost, operational excellence and performance "targets." Some people dislike "non-functional" because it sounds like "doesn't function" — Tom Gilb has argued for decades that qualities should be quantified with a scale and a meter, like any engineering requirement. Use whichever term the interviewer uses; know they're the same thing.

**A fifth thing, for architects: architecturally significant requirements (ASRs).** An ASR is any requirement — functional, quality or constraint — that has a measurable effect on the architecture. Most requirements are *not* architecturally significant: "users can change their avatar" won't change your design. The architect's skill is filtering the whole requirement set down to the handful that are. In an interview, the requirements you write on the board should *all* be ASRs; that's why three to five is the right number.

---

## Concept 3 — Why non-functional requirements, not features, shape the architecture

Here's the argument from first principles.

A functional requirement says *what transformation* happens: a message goes from Alice to Bob. Almost any architecture can perform that transformation — a single process with an in-memory dictionary can deliver messages. Functional requirements constrain the **interfaces and the data**, but they barely constrain the **structure**.

Structure — how many processes, where they run, how they communicate, where state lives, how many copies of it exist — is what you change to achieve *qualities*:

- You add replicas because of **availability** and **durability**.
- You add caches because of **latency** and **read scale**.
- You partition because of **write scale** and **data volume**.
- You add queues because of **burst absorption** and **temporal decoupling** between components with different availability.
- You add regions because of **latency to distant users**, **disaster recovery**, or **data residency**.
- You split services because of **team autonomy**, **independent deployability** or **different scaling profiles**.
- You add consensus because of **consistency** requirements under replication.

Every major structural element in a design exists to buy a quality. Remove the quality requirement and the element becomes unjustified complexity.

**A thought experiment.** Take one functional requirement — *"users can transfer money between their own accounts"* — and vary only the qualities:

| Quality profile | Architecture it produces |
|---|---|
| A personal budgeting app for one family, offline-first | A local SQLite file on the phone; sync when online; conflicts resolved by last-writer-wins because the stakes are low |
| A fintech with 2M users, strict invariant (no negative balances), EU only | A relational store with transactions, single region with zones, idempotency keys on every transfer, an outbox for notifications |
| A global bank, 99.999% target, regulator-mandated recovery tests, 30M users across continents | Partitioned ledger by account, consensus-replicated storage across regions, double-entry with immutable journal, reconciliation pipelines, formal DR exercises |

Same feature. Three entirely different systems. The thing that changed was the quality attributes — scale, consistency, availability, regulation. This is why interviewers care so much about step 1: if you get the qualities wrong, *nothing* downstream can be right.

**The corollary for your time.** Don't spend your requirement minutes enumerating features. Spend them finding the two or three qualities that will drive the structure. That's where the interviewer's rubric lives.

---

## Concept 4 — Why the questions you ask are graded

Interviewers don't just score your requirements; they score *how you got them*. Three reasons the questions themselves are evidence:

**1. Questions reveal mental models.** Asking "is it okay if two users briefly see different like counts?" demonstrates that you know like counters are usually served from caches or eventually consistent replicas, that exact counts are expensive at scale, and that the business usually doesn't care. Nobody asks that question without that model in their head. The question is a proof of knowledge that takes five seconds.

**2. Questions reveal priorities.** The order matters. A candidate who first asks about the programming language and the cloud provider is signaling that they think implementation details come first. A candidate who first asks "what's the one thing this system must never get wrong?" is signaling that they think about failure and correctness first. Senior engineers ask about risks before conveniences.

**3. Questions reveal whether you'll use the answer.** This is the subtle one. If you ask "what's the read/write ratio?", get "100 to 1," and then never mention caching or replicas, the question was theater. Interviewers notice unused answers. Conversely, a question with its consequence attached — *"What's the read/write ratio? If it's heavily read-skewed I'll serve the feed from precomputed timelines rather than querying on read"* — is the strongest form, because it shows the fork in the design you're resolving.

That last pattern is worth making a habit. The template is:

> **"[Question]? — because if [answer A], I'd [design choice A]; if [answer B], I'd [design choice B]."**

You don't need it on every question; that would be slow. Use it on the two or three questions that fork the architecture. It converts a question from information-gathering into demonstrated judgment.

**What interviewers are listening for, specifically.** Across public rubrics and interviewer guides, the recurring positive signals in this phase are: identifying ambiguity and resolving it proactively; prioritizing; quantifying; recognizing the hard part early; and *not* over-asking. The recurring negative signals are: diving into design immediately; asking many questions without prioritizing; and failing to establish scale or the key quality requirements, then making choices that the interviewer can't evaluate.

---

## Concept 5 — What makes a requirement usable

A requirement is usable when you can design against it and, later, test whether the design meets it. Most requirements candidates write fail that test. Let's derive what's missing.

Take "the system should be fast." To design against it, you'd need to know:

- **Which operation?** Opening the app? Sending a message? Generating a monthly report? These have targets that differ by three orders of magnitude.
- **Measured how?** Average latency hides the tail. A 100 ms average can coexist with one user in fifty waiting four seconds.
- **Measured where?** Server-side handler time, or what the user perceives on a mobile network?
- **Under what conditions?** At normal load, or during the peak? During a zone failure?
- **How much?** A number.
- **What happens when it's violated?** Is this a contractual SLA with penalties or an internal goal?

Put those together and you've reinvented the **quality-attribute scenario**, the format the Carnegie Mellon Software Engineering Institute (SEI) uses in its architecture methods (the Quality Attribute Workshop and ATAM). It has six parts:

| Part | Question it answers | Example (chat delivery latency) |
|---|---|---|
| **Source** | Who or what triggers it? | An online user |
| **Stimulus** | What happens? | Sends a 1:1 text message |
| **Environment** | Under what conditions? | At peak load, with both users connected in the same region |
| **Artifact** | Which part of the system? | The messaging path |
| **Response** | What should the system do? | Deliver the message to the recipient's connected devices |
| **Response measure** | How do we know it succeeded? | p99 ≤ 500 ms from server receipt to recipient socket write |

In an interview you won't write six columns. But you should be able to **compress a scenario into one line** that keeps the essential parts:

> *"1:1 delivery, both online, same region, at peak: p99 under 500 ms server-side."*

That's operation (stimulus + artifact), condition (environment), percentile and number (response measure). Everything a designer needs.

**A checklist for any NFR you write on the board** — call it the *OPEN* test:

- **O**peration — attached to a specific flow, not the whole system.
- **P**ercentile or probability — p99, 99.95% of minutes, "zero lost acknowledged writes."
- **E**nvironment — normal load, peak, during a failure.
- **N**umber — a target someone could measure.

If a requirement fails OPEN, it's an adjective. Fix it or drop it.

**Another formulation worth knowing: Planguage.** Tom Gilb's Planguage specifies each quality with a *Scale* (what you measure), a *Meter* (how you measure it), a *Must* (the minimum acceptable level — fail below this) and a *Plan* or *Goal* (the target). The Must/Plan split is useful in real life and occasionally in interviews: *"Must: p99 under 1 s or users abandon checkout. Plan: p99 under 300 ms."* It tells the interviewer which number is a hard floor and which is an aspiration.

---

# Part B — Functional requirements and the technique of elicitation

## Concept 6 — Actors, journeys and the top three

Functional requirements are the shortest part of step 1, but there's still a method to doing them quickly and well.

**Step 1: Find the actors.** Who interacts with the system? For a ride-sharing prompt: riders, drivers, support agents, the payments provider, possibly city regulators. For a metrics platform: instrumented services (writers), dashboards and alerting (readers), operators. Naming actors takes ten seconds and reveals flows you'd otherwise miss — infrastructure prompts especially have *system actors* (other services, schedulers, upstream producers) that candidates forget.

**Step 2: Trace the core journey for the primary actor.** What does a rider actually do? Request a ride → get matched → track the driver → ride → pay → rate. That journey is the backbone; the functional requirements are its steps, grouped.

**Step 3: Keep the top three, and say why they're the top three.** Prioritize by: (a) what the product *is* — the requirement without which the system has no reason to exist; (b) what's architecturally interesting — the requirement that exercises the hard quality; (c) what the interviewer signals. For ride-sharing: *riders can request a ride and be matched to a nearby driver; riders and drivers see each other's location in near-real time during the trip; riders are charged when the trip ends.* Rating, scheduling, ride pooling and surge pricing are real features — and out of scope for 45 minutes.

**Step 4: Phrase them as capabilities, not implementations.** "Users can see nearby drivers" — not "the system queries a geospatial index." The implementation is your design; the requirement is the problem.

**Prioritization frameworks you can name.** MoSCoW (Must, Should, Could, Won't) is the one most likely to be recognized by non-engineering stakeholders and is useful in architect rounds. In a 45-minute interview, "Must" is your top three and "Won't (this session)" is your out-of-scope list — that's all the framework you need.

**For infrastructure prompts, the "functions" are operations on an interface.** "Design a distributed cache" → *clients can get and set values by key with a TTL; the cache can be scaled by adding nodes without a full flush; clients can invalidate keys.* "Design a rate limiter" → *a caller can ask whether a request from a client key is allowed under a configured policy; policies can be changed without redeploying.* Write them as operations, because Module 3's API step will turn them into an interface directly.

---

## Concept 7 — The hidden functional requirements

Every real system has functions that nobody puts in the product brief, because they're not what users *see*. They matter in interviews for two reasons: naming one or two shows production experience, and some of them turn out to be architecturally significant.

| Hidden requirement | Why it can matter architecturally |
|---|---|
| **Deletion and erasure** | Users (and GDPR Article 17) can demand deletion. If data has been copied into caches, search indexes, analytics stores, event logs and backups, deletion becomes a distributed problem (Concept 15). |
| **Abuse and fraud prevention** | Spam, scraping, credential stuffing, fake accounts, phishing links. Often needs rate limiting, reputation, async scanning — new components and new write paths. |
| **Administration and support** | Support agents need to look up a user's state, replay a failed action, issue refunds. Implies indexed lookups by attributes the product doesn't query, audit trails, privileged access. |
| **Moderation** | Reporting content, takedowns, legal holds. A takedown must propagate to caches and CDNs — a consistency requirement in disguise. |
| **Backfill and reprocessing** | "Recompute everyone's recommendations after the model changes." "Rebuild the search index." Requires the source of truth to be replayable and pipelines to be idempotent. |
| **Migration** | The system replaces something. Dual-running, data migration, cutover. In architect rounds this is often *the* hard part. |
| **Export and portability** | "Download my data" (GDPR Article 20), tenant offboarding, B2B data exports. Bulk reads that mustn't hurt the OLTP path. |
| **Notifications** | Email, push, SMS on events. Introduces third-party dependencies with their own rate limits and failure modes. |
| **Reporting and analytics** | Business wants dashboards. Analytical queries on the OLTP store are a classic way to hurt production; usually means a separate read model or pipeline. |
| **Configuration and feature flags** | Gradual rollouts, kill switches. Affects deployability and incident response. |
| **Billing and metering** | Multi-tenant SaaS has to measure usage per tenant — which forces tenant identity through every hot path. |

**How to use this in an interview.** Don't add these to your functional requirements — that blows the scope. Instead, name one or two in your out-of-scope line *with a hint of why they'd matter*:

> *"Out of scope: moderation, analytics and account deletion — though I'll make sure message storage doesn't make deletion impossible later."*

That one clause shows you know deletion is a cross-cutting concern, and it costs four seconds. If the interviewer wants to go there, they will.

---

## Concept 8 — Ask, or assume and state?

Module 3 introduced the senior pattern: ask questions that fork the design; state assumptions for everything else. Here's the reasoning behind that rule, and a quick way to apply it under pressure.

**The value of a question** is roughly:

> **(how much the answer changes the design) × (how uncertain you are about the answer) − (time it costs)**

- A question whose answer won't change your design has zero value no matter how uncertain you are. ("What language should the services be written in?")
- A question whose answer you can reasonably predict has low value; state the assumption instead and let the interviewer correct it. ("I'll assume this is a mobile-first consumer app.")
- A question that forks the architecture *and* could plausibly go either way is worth asking — and worth attaching a reason to. ("Do users pick specific seats, or is it general admission?")

**A decision procedure you can run in your head:**

```
For each unknown:
  Would different answers lead to structurally different designs?
     no  → skip it, or fold it into a stated assumption
     yes → Can I predict the answer with reasonable confidence from the prompt?
              yes → state it as an assumption, invite correction ("stop me if…")
              no  → ask it, with the design fork attached
```

**Fork questions by prompt type.** A few examples of questions that genuinely fork designs, so you recognize the category:

| Prompt | Fork question | Why it forks |
|---|---|---|
| Ticketing | Assigned seats or general admission? | Contention on rows vs on a counter |
| Chat | Maximum group size? | Fan-out-on-write vs fan-out-on-read; cost per message |
| Chat | End-to-end encryption? | Server can't read content → no server-side search or moderation |
| News feed | Ranked or chronological? | Precomputed timelines vs ranking at read time with a model |
| File sync | Do we need real-time collaborative editing, or just sync? | OT/CRDTs vs file-level versioning |
| Payments | Are we the processor or integrating with one? | PCI scope, ledger design, reconciliation |
| Ride sharing | Do we need to guarantee one driver per ride under contention? | Distributed claim/lock vs best-effort matching |
| Metrics | What query latency over what retention? | Raw storage vs downsampled rollups; hot/cold tiers |
| Rate limiter | Single region or global limits? | Local counters vs coordinated/global counters with a staleness budget |
| Any B2B SaaS | Is it multi-tenant, and do tenants need isolation guarantees? | Pooled vs siloed tenancy, noisy-neighbor controls |

**When the interviewer says "you decide."** Many interviewers deliberately return questions: "What do you think?" That's not evasion; it's a test of whether you can make a reasonable call. Make it quickly, with the reasoning: *"Then I'll cap groups at 500, which keeps fan-out-on-write viable — WhatsApp-scale products have used caps in that range. If we needed broadcast channels with millions of members, that'd be a different read path."* Don't bounce it back.

**When the interviewer gives no numbers.** Derive them. "I'll assume 100 million daily users, each sending about 40 messages a day" is better than asking "how many users?", because you've shown you can estimate, and you've put a number on the board the interviewer can correct. Module 5 covers the arithmetic.

**Batching.** Don't fire questions one at a time and wait for each answer — that turns step 1 into a twenty-minute dialogue. Group them: *"Three things would change the design: group size, whether we need end-to-end encryption, and whether history must sync across devices. What would you like for those?"* Then state the rest as assumptions in one breath.

---

# Part C — The non-functional catalogue, one dimension at a time

Each concept in this part covers one quality dimension: what it actually means, how to make it concrete, the questions that elicit it, and — most importantly — what each answer *changes* in the design. You won't ask about all twelve in any single interview. The point of knowing all of them is to quickly spot the two or three that matter for the prompt in front of you.

## Concept 9 — Scale and workload shape

"How many users?" is the question everyone asks. It's the least informative of the scale questions, because a user count alone doesn't tell you the *shape* of the load, and shape decides architecture more than size does.

| Dimension | The question | What the answer decides |
|---|---|---|
| **Population** | Registered users, daily actives, peak concurrent? | Concurrent users drive connection counts and session state; DAU drives daily volume; registered drives storage. |
| **Rate per action** | How often does a user do each core action? | Converts users into requests per second, per operation. |
| **Peak factor** | What's peak versus average — daily cycle, weekly cycle, known events? | You provision for peak. A 3× diurnal peak and a 100× on-sale spike need different mechanisms (autoscale vs admission control). |
| **Read/write ratio** | Per entity: how many reads per write? | Heavily read-skewed → caches, replicas, precomputation. Write-heavy → partitioning, log-structured storage, batching. |
| **Burstiness** | Do requests arrive smoothly or in bursts — a push notification sent to 10M users, a cron at midnight? | Bursts need queues for load leveling, or admission control; smooth load needs only capacity. |
| **Skew** | Are some keys, users or tenants vastly hotter than others? | Hot partitions, celebrity fan-out, noisy-neighbor tenants — design for the hottest key, not the average. |
| **Object size** | Typical and maximum payload size; distribution? | Large objects go to blob storage, not the database; size distribution affects caching and network budgets. |
| **Data volume and growth** | How much data now, how fast does it grow, how long is it kept? | Single-node versus partitioned storage; tiering; how soon a one-way door (the partition key) gets tested. |
| **Fan-out** | When one thing happens, how many others are affected? | A post to 1M followers, a group message to 500 members, an invalidation to 300 edge nodes — fan-out is often the real write volume. |
| **Geography** | Where are users and producers? One region or many? | Latency floors, regional deployment, data residency, follow-the-sun peaks. |
| **Growth horizon** | What scale must we support in 1–3 years? | How much headroom to build in now versus designing an evolution path. |

**Skew deserves its own paragraph** because it's where averages lie most dangerously. Real access patterns are usually heavy-tailed: a small fraction of keys gets most of the traffic (Zipf-like distributions are common in social, e-commerce and content systems). A system sized for the average key will melt on the hottest one, because the hottest key usually lives on *one* partition and therefore one machine. In Cosmos DB, for instance, throughput per physical partition is capped (Module 8), so a "celebrity" partition key throttles regardless of how much total throughput you've provisioned. Asking *"Are there celebrity users, huge tenants or viral items?"* is a seniority signal because it shows you design for the distribution, not the mean.

**Growth: design for 10×, plan for 100×.** Designing today for 1000× tomorrow is over-engineering; designing for exactly today's load is negligence. The usual senior stance is: *the current design should handle 10× with configuration and capacity changes; beyond that, I'll name the evolution path.* The things to get right early are the **one-way doors** — decisions that are expensive to reverse at scale: partition keys, ID formats, public API shapes, the data model of the source of truth, tenancy model. Scale-dependent but reversible choices (cache size, instance counts, even many technology choices) can wait.

**Questions that sound senior:**

- *"What's the peak-to-average ratio, and is there a known event that drives spikes?"*
- *"Is the load evenly spread across keys, or are there celebrities, big tenants or viral items?"*
- *"When one user acts, how many others does it affect?"*
- *"What scale do we need in three years? I want to get the one-way doors right now and leave the rest to evolve."*

---

## Concept 10 — Latency and performance

### 10a. Make it concrete: operation, percentile, where measured

"Low latency" becomes a requirement when you specify **which operation**, **which percentile**, **measured where**, and **under what load**.

**Why percentiles, not averages.** Latency distributions are skewed: most requests are fast, a few are very slow (GC pauses, cache misses, lock waits, retries, noisy neighbors). The average hides the slow ones. A service with a 50 ms mean might have a p99 of 900 ms — meaning one request in a hundred takes nearly a second. If a page makes twenty requests, a large share of page loads include at least one slow call. That's why serious latency targets use high percentiles: p95, p99, sometimes p99.9 for high-volume paths.

Two measurement traps worth knowing, because interviewers who've run production systems will appreciate them:

- **Percentiles don't average.** You can't compute a fleet-wide p99 by averaging each host's p99. You aggregate histograms (bucket counts) and compute the percentile from the merged distribution. OpenTelemetry histograms exist for exactly this reason.
- **Coordinated omission.** Many load generators wait for a response before sending the next request, so when the system stalls, the tool *stops measuring* during the stall and under-reports the tail. Gil Tene's talk on measuring latency is the classic explanation. If you mention load testing a latency SLO, mention constant-rate (open-model) load generation.

**Where measured.** Server-side handler time is easy to measure but isn't what users feel. User-perceived latency adds DNS, TLS, network round trips, client rendering and retries. For mobile users on cellular networks, the network can dominate. A good requirement says which: *"p99 ≤ 200 ms server-side; we'll track client-side time-to-interactive separately."*

### 10b. Human thresholds worth knowing

| Threshold | What it feels like | Implication |
|---|---|---|
| ~100 ms | Instantaneous — the system feels like it's reacting directly | Typing feedback, UI interactions; usually requires local or edge-served responses |
| ~1 s | The user's flow of thought is preserved, but they notice the delay | Interactive page loads and actions |
| ~10 s | The limit of attention; users switch tasks | Beyond this, use progress indication or make the work asynchronous |

These thresholds come from long-standing human-factors research (popularized by Jakob Nielsen). For the web specifically, Google's Core Web Vitals give current targets: Largest Contentful Paint at or under 2.5 s and Interaction to Next Paint at or under 200 ms are considered "good."

### 10c. Tails amplify with fan-out

This is the most important performance idea to bring into a design interview, because it changes architecture.

If one request fans out to *n* backends in parallel and must wait for all of them, the request is slow if *any* backend is slow. If each backend independently exceeds its own p99 on 1% of calls, the probability that a fan-out request hits at least one slow backend is:

> **P(slow) = 1 − 0.99ⁿ**

| Fan-out *n* | Probability at least one call is beyond its p99 |
|---|---|
| 1 | 1.0% |
| 10 | 9.6% |
| 100 | 63.4% |

At a fan-out of 100, the backend's *p99* becomes the request's *median* experience. Dean and Barroso's *The Tail at Scale* (CACM, 2013) made this famous and proposed tail-tolerant techniques: hedged requests (send a second copy after a short delay, take the first answer), tied requests, micro-partitioning, and selective replication of hot items.

**The requirement-gathering consequence:** when a prompt involves scatter-gather (search across shards, a feed assembled from many sources, a dashboard calling many services), a latency requirement on the user-facing operation implies a much stricter requirement on each backend — or a design that doesn't wait for all of them (partial results, timeouts with degraded answers). Ask: *"Is a partial result acceptable if one shard is slow?"* That single question can remove an entire class of tail problems.

### 10d. Latency budgets

A budget decomposes an end-to-end target into allowances per hop. It makes trade-offs concrete and tells you early whether a design is even feasible.

Example: a mobile API call with a target of **p99 ≤ 400 ms user-perceived**, users in Europe, servers in one EU region.

| Segment | Allowance | Notes |
|---|---|---|
| Mobile network RTT + TLS (connection reused) | 120 ms | Cellular variance dominates; reused connections avoid repeated handshakes |
| Edge / gateway (WAF, auth token validation) | 20 ms | Validate JWTs locally with cached keys; no per-request call to the identity provider |
| Service logic | 30 ms | |
| Primary data read (cache hit / miss) | 5 ms / 25 ms | Cache-aside; most reads should hit |
| Downstream service call | 60 ms | One call, not a chain — chains add their tails |
| Serialization, response | 15 ms | |
| **Headroom** | ~130 ms | For retries, GC, noisy neighbors; tails don't add linearly, so keep slack |

Two lessons the budget teaches immediately: a **synchronous chain of three downstream services doesn't fit**, and **a cross-region call doesn't fit** (Frankfurt to US East is ~6,200 km of great-circle distance, so even at the ~200,000 km/s speed of light in fibre the theoretical round trip is ~62 ms, and real round trips are higher). Both are architectural conclusions reached in requirements time.

### 10e. Throughput, utilization and the knee

Latency and throughput are coupled through queueing. As a resource's utilization approaches 100%, waiting time grows non-linearly — in the simplest queueing model (M/M/1), the mean time in system is the service time divided by (1 − utilization): at 50% utilization requests take twice the service time on average; at 90%, ten times. Real systems aren't M/M/1, but the shape — a "knee" past which latency explodes — is universal (Module 6 covered Little's Law and the Universal Scalability Law).

**Requirement consequence:** a latency target is only meaningful *together with* a throughput at which it must hold. "p99 under 200 ms at 20,000 requests per second" is a requirement; "p99 under 200 ms" alone is satisfied by an idle system.

### 10f. Batch and asynchronous work have different targets

Not everything is interactive. For batch and pipeline work, the requirement is usually a **deadline** or **freshness** rather than per-request latency: "the nightly settlement file must be produced by 06:00," "dashboards reflect events within 2 minutes p99," "a video is playable within 5 minutes of upload." These targets drive throughput, parallelism and scheduling decisions rather than caching and edge placement.

### 10g. Measuring it in .NET

ASP.NET Core already emits the `http.server.request.duration` histogram through `System.Diagnostics.Metrics`, which OpenTelemetry exports; that's your default HTTP latency SLI. For business flows that span several HTTP calls or messages, define your own instrument and align the histogram buckets with your SLO threshold so "good events" can be counted exactly rather than interpolated:

```csharp
using System.Diagnostics.Metrics;

public static class CheckoutTelemetry
{
    public static readonly Meter Meter = new("Shop.Checkout", "1.0.0");

    // Bucket boundaries include the SLO threshold (0.3 s), so the fraction of
    // requests under 300 ms is read directly from a bucket, not estimated.
    public static readonly Histogram<double> PlaceOrderDuration =
        Meter.CreateHistogram<double>(
            name: "shop.checkout.place_order.duration",
            unit: "s",
            description: "Server-side duration of PlaceOrder (SLI for the checkout latency SLO)",
            tags: null,
            advice: new InstrumentAdvice<double>
            {
                HistogramBucketBoundaries = [0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5]
            });
}

// Usage
var started = TimeProvider.System.GetTimestamp();
// ... place the order ...
CheckoutTelemetry.PlaceOrderDuration.Record(
    TimeProvider.System.GetElapsedTime(started).TotalSeconds,
    new KeyValuePair<string, object?>("outcome", "success"));
```

Register the meter with `AddOpenTelemetry().WithMetrics(m => m.AddMeter("Shop.Checkout"))`. Mentioning that you'd measure the SLI this way — and that you'd keep tag cardinality low (outcome, region, tier — never user ID) — is the kind of one-level-deeper detail that lands well in .NET-focused interviews.

**Questions that sound senior:**

- *"Which operations are latency-critical, and what percentile do we care about?"*
- *"Is that measured server-side, or what the user perceives on mobile?"*
- *"At what throughput must that latency hold?"*
- *"Is a partial result acceptable if one shard is slow?"*
- *"Is this interactive, or is it a freshness or deadline requirement?"*

---

## Concept 11 — Availability

### 11a. From first principles: what does "available" mean?

Availability is the fraction of time — or the fraction of requests — for which the system does its job correctly. Two ways to measure:

- **Time-based:** the fraction of minutes during which the service was "up" (by some health definition). Simple, but a minute where 30% of requests fail is hard to classify.
- **Request-based:** the fraction of valid requests that succeeded (within a latency bound). Google's SRE practice favors this form: *SLI = good events ÷ valid events*. It weights outages by how many users they actually affected.

Either way, availability must be defined **per flow** (Concept 21), and "success" must include a latency bound — a response after 30 seconds is not a success for an interactive user.

### 11b. SLI, SLO, SLA — three different things

| Term | What it is | Who owns it | Example |
|---|---|---|---|
| **SLI** (indicator) | A measurement | Engineering | Proportion of `POST /orders` returning 2xx/4xx (not 5xx) within 1 s |
| **SLO** (objective) | A target for the SLI over a window | Engineering + product | 99.9% over a rolling 28 days |
| **SLA** (agreement) | A contract with consequences, usually service credits | Business + legal | 99.5% monthly, or 10% credit |

The SLA is deliberately *looser* than the SLO — you want to notice and fix problems internally before they cost money. Cloud provider SLAs are financial commitments, not engineering predictions; Microsoft's Well-Architected guidance treats them as a *proxy* for platform reliability when calculating your own composite targets, not as a guarantee of what your workload will achieve.

**Error budgets.** If your SLO is 99.9%, you're *allowed* 0.1% failure. That budget is spent on deployments, experiments and incidents. When it's exhausted, the error-budget policy says what changes (typically: freeze risky releases, prioritize reliability work). Mentioning error budgets turns availability from an aspiration into a management mechanism, which interviewers at SRE-minded companies value.

### 11c. The nines, in minutes

You should be able to convert nines to downtime without a calculator. Year = 8,760 h; 30-day month = 720 h; week = 168 h.

| Availability | Per year | Per 30-day month | Per week |
|---|---|---|---|
| 99% | 3.65 days | 7.2 h | 1.68 h |
| 99.5% | 1.83 days | 3.6 h | 50.4 min |
| 99.9% | 8.76 h | 43.2 min | 10.1 min |
| 99.95% | 4.38 h | 21.6 min | 5.0 min |
| 99.99% | 52.6 min | 4.3 min | 1.0 min |
| 99.995% | 26.3 min | 2.2 min | 30 s |
| 99.999% | 5.3 min | 26 s | 6 s |

The operational meaning of the table is more important than the numbers. At **99.9%**, you have 43 minutes a month — a human can be paged, diagnose and fix an issue. At **99.99%**, you have about four minutes a month — no human-in-the-loop recovery fits; failover must be automatic. At **99.999%**, you have 26 seconds a month — even automated failover that takes a minute blows the budget, so you need active-active redundancy where failures are absorbed rather than recovered from. **Each additional nine changes the *kind* of mechanism you need, not just the amount of hardware.**

### 11d. Dependencies multiply

If a request must pass through components in series, and each is up independently, the end-to-end availability is the **product** of their availabilities. If redundant alternatives exist in parallel (and failover works), the unavailability is the product of their unavailabilities.

> **Serial:** A = A₁ × A₂ × … × Aₙ  
> **Parallel (n redundant copies):** A = 1 − (1 − A₁)(1 − A₂)…(1 − Aₙ)

```csharp
static double Serial(params double[] a)   => a.Aggregate(1.0, (acc, x) => acc * x);
static double Parallel(params double[] a) => 1 - a.Aggregate(1.0, (acc, x) => acc * (1 - x));
```

**Worked example (illustrative SLA figures — check the current Azure SLA page before relying on them).** A checkout flow uses App Service (99.95% SLA), Azure SQL Database without zone redundancy (99.99%), and an external payment provider you'll treat as 99.9%:

> 0.9995 × 0.9999 × 0.999 ≈ **0.9984 → 99.84%**, about **14 hours** of expected downtime per year.

Two observations an interviewer will love:

1. **You can't be more available than a hard dependency in series.** No amount of engineering inside your service gets the checkout above 99.9% while it synchronously depends on a 99.9% provider. To exceed that, you must *decouple*: accept the order and authorize payment asynchronously, cache what can be cached, or degrade (accept orders and charge later, with a risk policy). This is the reasoning behind static stability and graceful degradation (the Amazon Builders' Library covers both).
2. **Redundancy helps only if failures are independent.** Two regions at 99.84% each give a theoretical 1 − (0.0016)² ≈ 99.9997%. In reality, failures are correlated — shared identity providers, shared global DNS or edge, the same bad deployment rolled to both, the same configuration error — and the failover mechanism itself can fail. Microsoft explicitly cautions against treating multiplied SLA percentages as a workload guarantee. Use the math to compare designs, not to promise numbers.

Two rules of thumb from Google's *The Calculus of Service Availability* (ACM Queue, 2017) are worth quoting in an interview. First, the **"rule of the extra 9"**: a service's critical dependencies should each be roughly one nine more available than the service itself, because a service typically has several of them and its own bugs too. Second, **availability is bounded by incident frequency times recovery time**: three full outages a year that take 20 minutes each to detect and fix is 60 minutes — already over a 99.99% budget of ~53 minutes, however good the rest of the year is. That second rule turns availability requirements into *operational* requirements: how fast you detect, and how fast you can roll back.

Useful Azure reference points (verified for 2026; SLAs change, so treat as orders of magnitude): Azure SQL Database offers 99.99% baseline and 99.995% for zone-redundant configurations; Cosmos DB offers 99.99% for single-region accounts and 99.999% for multi-region accounts (with multi-region writes for write availability); App Service offers 99.95%.

### 11e. Availability is per flow, and failure can be partial

"The system is 99.99% available" is almost never the right requirement. Different flows deserve different targets (Concept 21), and many flows can **degrade** instead of failing:

| Degradation mode | Example |
|---|---|
| Serve stale | Product pages from cache when the catalog database is down |
| Read-only mode | Banking app shows balances and history but disables transfers |
| Queue writes | Accept orders into a durable queue; process when the downstream recovers |
| Drop the optional | Hide recommendations, typing indicators, presence |
| Reduce fidelity | Lower-resolution images, approximate counts |
| Shed load | Reject low-priority requests (prefetch, analytics) first |

A requirement like *"if the recommendation service is down, the home page still loads within its latency SLO with a generic list"* is a design decision disguised as a requirement — and exactly the kind of statement that signals operational experience.

### 11f. Questions that elicit availability requirements

- *"Which flows are business-critical, and what does an hour of downtime on each cost?"* (Concept 30 turns this into money.)
- *"Does planned maintenance count against availability, or do we need zero-downtime deployments and migrations?"*
- *"Which flows can degrade — serve stale, go read-only, queue — rather than fail?"*
- *"Which external dependencies are in the critical path, and what are their SLAs?"*
- *"Is the target contractual (an SLA with credits) or internal?"*
- *"Do we need to survive a zone failure, a region failure, or both?"*

---

## Concept 12 — Durability, recovery and data lifecycle

### 12a. Durability is not availability

These are confused constantly, and separating them cleanly is a quick signal.

- **Availability:** can I read or write the data *now*?
- **Durability:** once the system said "saved," will the data still exist later?

A storage account in a region that's down is unavailable but durable — the data comes back when the region does. A database that's up but silently lost the last hour of writes after a failover is available but not durable. You need separate requirements for each.

Reference figures (Azure Storage documentation, current): locally redundant storage (LRS) is designed for at least **11 nines** of object durability over a year, zone-redundant storage (ZRS) for at least **12 nines**, and geo-redundant options (GRS/GZRS) for at least **16 nines**. Those numbers protect against *hardware* failure. They do nothing against an application bug that overwrites data, an operator who deletes a container, or ransomware — all of which replicate faithfully to every copy.

### 12b. RPO and RTO

```
            last good          disaster          service restored
            recovery point        │                    │
  ──────────────●─────────────────✕────────────────────●──────────►  time
                ◄────── RPO ──────►◄────── RTO ────────►
              (data you may lose)   (time you're down)
```

- **RPO (recovery point objective):** the maximum acceptable data loss, measured in time. "RPO 5 minutes" means we may lose up to the last five minutes of acknowledged writes in a disaster.
- **RTO (recovery time objective):** the maximum acceptable time to restore service.

Both are **business numbers** — they come from the cost of lost data and the cost of downtime — but they map directly onto mechanisms:

| Target | Mechanism it implies | Azure examples |
|---|---|---|
| RPO = 0 within a region | Synchronous replication across zones before acknowledging | Zone-redundant Azure SQL Database; ZRS storage |
| RPO = 0 across regions | Synchronous cross-region replication or quorum (costs write latency) | Cosmos DB with strong consistency across regions |
| RPO of seconds across regions | Asynchronous geo-replication | Azure SQL active geo-replication / failover groups (Microsoft offers a business-continuity SLA on Business Critical geo-replication with a 5 s RPO and 30 s RTO — check current terms) |
| RPO of hours | Periodic backups shipped off-site | Geo-redundant backup storage, point-in-time restore |
| RTO of seconds | Automatic failover, warm standby or active-active | Failover groups with automatic policy; multi-region writes |
| RTO of hours | Restore from backup, redeploy with infrastructure as code | Bicep/Terraform plus restore runbooks |

**The trade at the core of this:** an RPO of zero across regions means a write isn't acknowledged until a distant region has it — adding a cross-region round trip (tens of milliseconds) to every write. That's PACELC (Module 7) appearing as a business requirement. Asking *"After a regional disaster, how many seconds of acknowledged writes can we lose?"* makes the trade explicit, and the honest answer often turns out to be "a few seconds is fine for most data, zero for payments" — which leads to different replication for different data.

### 12c. When do we say "saved"?

A sharper version of the durability question: **what does the acknowledgment mean?** When the API returns 201, is the write on one disk, on a quorum of replicas in different zones, or in two regions? Users and downstream systems act on acknowledgments — the client deletes its local copy, the seller ships the goods. A mismatch between what the acknowledgment promises and what it guarantees is how data gets lost "with no errors in the logs."

### 12d. Replication is not backup

Replication protects against loss of a *copy*. Backups protect against loss of *correctness* — the logical corruption that replication spreads instantly. You need:

- **Point-in-time restore** for "the bad migration ran at 14:02; restore to 14:01."
- **Soft delete and versioning** for accidental deletes.
- **Immutable or logically separated backups** for ransomware and compromised credentials.
- **Restore tests**, because an untested backup is a hypothesis. GitLab's 2017 database incident postmortem is the standard cautionary tale: several backup mechanisms existed, and when they were needed, most turned out not to be working.

### 12e. Data lifecycle

Durability requirements have an end. Lifecycle questions are frequently forgotten and frequently architecturally significant:

- **Retention minimums** — "financial records kept for N years" (legal requirement varies by jurisdiction).
- **Retention maximums** — "delete personal data when no longer needed" (GDPR's storage-limitation principle). Yes, both can apply to the same system.
- **Access temperature over time** — data read constantly for a week and rarely thereafter suggests hot/cool/archive tiers (Azure Blob Storage has hot, cool, cold and archive tiers; archive has hours of rehydration latency, so "how fast must old data be retrievable?" is a real question).
- **TTL on derived data** — caches, projections, search indexes, analytics copies. Short TTLs on derived copies make erasure tractable (Concept 15).

**Questions that sound senior:**

- *"Can we ever lose an acknowledged write? What about after a regional disaster — how many seconds?"*
- *"How quickly must we be back after a regional outage, and is that automatic or can a human decide?"*
- *"What protects us from our own bugs — what's the point-in-time restore requirement?"*
- *"How long is data kept, and is there a maximum as well as a minimum?"*

---

## Concept 13 — Consistency and correctness

Module 7 covered the consistency spectrum in depth. Here the job is different: **eliciting which guarantee each operation needs, in business language, and translating the answer into a model.** Users and product managers never say "linearizable." They say "a seat is never sold twice" or "I posted it, why can't I see it?"

### 13a. Ask per operation, not per system

Global strong consistency is expensive (latency, availability under partition, throughput). Global eventual consistency is often wrong for the operations that matter. The senior pattern is to decide **per operation, per piece of data**:

| What the business says | What it means technically | Typical mechanism |
|---|---|---|
| "A seat/username/coupon can never be claimed twice." | Linearizable claim on that key (uniqueness invariant) | Conditional write, unique constraint, single-leader per key |
| "A balance can never go negative." | Invariant across concurrent debits on one account | Serialize per account: row lock, conditional update, single writer per partition |
| "After I post, I must see my post." | Read-your-writes (a session guarantee) | Session consistency (Cosmos DB default), sticky reads, read from leader after write |
| "My feed shouldn't jump backwards when I refresh." | Monotonic reads | Session tokens, sticky replicas |
| "Replies must never appear before the comment they reply to." | Causal consistency | Causal metadata, per-thread ordering |
| "Everyone in the chat sees messages in the same order." | Total order *per conversation* (not globally) | Sequence numbers assigned per conversation partition |
| "Like counts can be a bit off." | Eventual consistency with bounded staleness acceptable | Async aggregation, approximate counters, caches |
| "Search should reflect edits within a few seconds." | Bounded staleness, stated as a number | Change feed / outbox → indexer, monitored lag |
| "The price shown must be the price charged." | Validation at commit time against current version | Optimistic concurrency (ETag/rowversion) on checkout |

Doug Terry's short paper *Replicated Data Consistency Explained Through Baseball* is the best illustration of this per-actor, per-operation thinking: the scorekeeper, the umpire, the radio reporter and the sportswriter each need a different consistency guarantee from the *same* data.

### 13b. Invariants: local versus global

An **invariant** is a statement that must always be true: "seats sold ≤ capacity," "every order has exactly one payment record," "a user has one active subscription." Two kinds behave very differently:

- **Local invariants** involve data that can live together (one account, one order, one event's inventory). They can be enforced cheaply inside a single partition or transaction — which is a strong reason to choose partition keys and aggregate boundaries around them (Modules 8 and 22).
- **Global invariants** span partitions (globally unique usernames, a company-wide spending limit across many accounts). They require coordination — a single authority, consensus, or a reservation scheme — and coordination is what limits scalability and availability.

Research on *invariant confluence* (Bailis and colleagues) formalized which invariants can be maintained without coordination: some, like "this ID references an existing record," can often be preserved by merge-friendly designs; others, like uniqueness and "never below zero," fundamentally need coordination. You don't need the formalism in an interview, but asking *"Which invariants must hold across the whole system, versus within one account or one order?"* is a staff-level question, because the answer determines where coordination — and therefore cost — lives.

### 13c. Ordering

Ask what ordering is required, and **over what scope**. Global total order is almost never needed and is expensive. Typical real requirements:

- Per entity (all events for one order are processed in order) → partition by entity key in Kafka/Event Hubs, or Service Bus sessions keyed by entity.
- Per conversation or per user → sequence numbers assigned within that scope.
- Causal ("effects after causes") → causal metadata or same-partition routing for related events.
- None (independent events) → maximize parallelism.

### 13d. Duplicates and replays

In any distributed system with retries, messages are delivered **at least once** (Module 11). The requirement question isn't "do we want exactly-once?" — everyone does — it's:

> *"What happens if this is processed twice? What happens if it arrives late or out of order?"*

The answers sort operations into: naturally idempotent (setting a value), idempotent with a key (charging a card with an idempotency key), and order-sensitive (state transitions that must reject stale events, often via version checks). This question frequently reveals the most dangerous operation in the system.

### 13e. The cost of being wrong

When the interviewer is vague, classify the consequence of an inconsistency — it tells you how much to pay to prevent it:

| Consequence | Example | Typical stance |
|---|---|---|
| Cosmetic | Like count off by a few | Eventual; don't spend money |
| Annoying | User doesn't see their own edit | Session guarantees; cheap |
| Financial | Double charge, oversold inventory | Strong per key; idempotency; reconciliation |
| Legal / regulatory | Data shown to the wrong tenant; revoked access still works | Strong, plus audit and bounded revocation time |
| Safety | Wrong medication dose displayed | Strong, plus verification, fail-safe defaults |

**Questions that sound senior:**

- *"Which operations must never be wrong, and what does it cost if they are?"*
- *"After a user writes, must they immediately see their own write? Must others?"*
- *"How stale can this view be — in seconds?"*
- *"What ordering do we need, and over what scope — per user, per conversation, global?"*
- *"What happens if this event is processed twice, or out of order?"*
- *"Which invariants span the whole system rather than one account?"*

---

## Concept 14 — Security, identity and tenancy

Security in requirements gathering isn't about listing controls ("we'll use TLS"). It's about identifying the **principals**, the **assets**, the **trust boundaries** and the **isolation guarantees** — because those determine structure. Module 29 covers threat modeling (STRIDE) and identity protocols in depth.

### 14a. Who are the principals?

| Principal | Typical Azure-side answer | Architectural consequence |
|---|---|---|
| Employees / workforce | Microsoft Entra ID (with Conditional Access) | Single sign-on, group-based roles, possibly on-premises sync |
| Consumers / customers | Microsoft Entra External ID (Azure AD B2C has been closed to new customers since May 1, 2025; existing tenants remain supported) | Self-service sign-up, social identity providers, high-volume token issuance |
| Business partners / B2B tenants | External ID B2B collaboration, or federation with each customer's own identity provider | Per-tenant identity configuration; "bring your own IdP" is a frequent enterprise requirement |
| Services and workloads | Managed identities, workload identity federation | No secrets in code; per-service least privilege |
| Devices / IoT | Device certificates, IoT Hub identities | Provisioning, rotation, revocation at fleet scale |

The fork question: *"Who signs in, and with whose identity provider?"* For an enterprise SaaS, "each customer federates their own Entra ID or Okta" is a requirement that shapes the whole authentication layer.

### 14b. What can each principal do?

The authorization model is often architecturally significant:

- **Role-based (RBAC)** — users have roles; roles have permissions. Fine for most internal and admin systems.
- **Attribute-based (ABAC)** — decisions on attributes of user, resource and context ("managers can approve expenses under €5,000 in their own cost center"). Policy engines.
- **Relationship-based (ReBAC)** — permissions follow relationships ("anyone a folder is shared with can read documents in it"). Google's Zanzibar paper describes the model behind Google Drive's sharing; open-source implementations exist. If the prompt involves **sharing** (documents, files, boards), this is the fork: per-object sharing at scale needs a dedicated authorization service and a strategy for checking permissions in list and search operations, not just on single-object reads.

Ask: *"Can users share individual objects with arbitrary other users, or are permissions role-based?"*

### 14c. Multi-tenancy and isolation

For B2B SaaS, tenancy is one of the most consequential requirements in the entire design. Microsoft's Azure Architecture Center describes a spectrum:

| Model | Description | Trade-off |
|---|---|---|
| **Pooled** (shared) | All tenants share compute and data stores, separated by a tenant ID | Cheapest and simplest to operate; noisy neighbors; isolation enforced in code (and optionally row-level security) |
| **Siloed** (dedicated) | Each tenant gets its own stack or its own database | Strong isolation, per-tenant customization and residency; expensive; operational sprawl |
| **Hybrid** (tiered) | Pooled for most; siloed for large or regulated tenants | Common in practice; complexity in routing and tooling |

The elicitation questions: *"How many tenants, and how unequal in size? Do any require dedicated infrastructure, their own encryption keys, or data in a specific region? What's the blast radius if one tenant's workload misbehaves?"* The answers decide your partitioning (tenant ID as partition key prefix, Cosmos DB hierarchical partition keys), your throttling (per-tenant rate limits), and whether "tenant" appears in every API path and every log line.

### 14d. Assets, exposure and abuse

- **What's valuable?** PII, payment data, health data, credentials, intellectual property, money movement. Each raises the bar for encryption, access control, logging and retention.
- **Who can reach it?** Public internet? Partner networks? Only internal? Exposure determines edge protection (WAF, DDoS protection, bot management) and network isolation (private endpoints).
- **How will it be abused?** Spam, scraping, credential stuffing, enumeration, fake accounts, payment fraud. Abuse requirements produce real components: rate limiters (ASP.NET Core has built-in rate-limiting middleware since .NET 7), reputation scoring, CAPTCHAs at the edge, asynchronous scanning pipelines.

### 14e. Audit and evidence

Enterprise and regulated customers ask: *who did what, when, and can you prove the log wasn't altered?* That's an append-only, tamper-evident audit requirement. Azure SQL's ledger tables provide cryptographic tamper evidence for database history; immutable Blob Storage policies provide write-once retention for logs. Mention these only if the domain warrants them — but ask about audit whenever money, access rights or regulated data are involved.

**Questions that sound senior:**

- *"Who are the principals — consumers, employees, partner organizations, other services — and whose identity provider do they use?"*
- *"Is authorization role-based, or can users share individual objects?"*
- *"Is this multi-tenant? Do any tenants need dedicated resources, their own keys, or regional isolation?"*
- *"What are the abuse cases?"*
- *"Is there an audit requirement — who did what, provably?"*

---

## Concept 15 — Privacy, compliance and data residency

Regulation becomes architecture when it constrains **where data may live**, **how data must flow**, **what must be deletable**, or **what must be provable**. You're not expected to be a lawyer in a design interview — and you should say so when relevant — but you're expected to recognize when a regime applies and what it does to the design.

### 15a. Personal data (GDPR and its relatives)

The EU's General Data Protection Regulation shapes systems that process personal data of people in the EU, and many other jurisdictions have similar laws. The architecturally significant parts:

| Obligation | Architectural consequence |
|---|---|
| **Right to erasure** (Article 17) | Personal data must be deletable across *every* copy: primary store, caches, search indexes, analytics, event logs, data lake, backups. |
| **Right to data portability** (Article 20) | A user (or tenant) export path that doesn't hurt production. |
| **Data minimization and storage limitation** | Don't collect or keep what you don't need; TTLs on derived data. |
| **Special categories** (Article 9 — health, biometrics, etc.) | Stricter controls; often encryption, segregated stores, access logging. |
| **Breach notification** (Article 33 — to the supervisory authority within 72 hours where feasible) | You need detection and forensic logging good enough to know what was accessed. |
| **International transfers** | Where data may be processed and by whom — a legal question that becomes a region and vendor choice. |

**The erasure problem in event-driven and event-sourced systems** is a classic senior topic (Module 24 touched on it). Immutable logs and personal data are in tension. Common designs:

- **Keep PII out of events and logs** — reference users by an opaque ID; store PII in one deletable place.
- **Crypto-shredding** — encrypt each person's data with a per-person key; deleting the key renders every copy, including backups, unreadable.
- **Short retention on derived copies** — caches, indexes and analytics copies expire, so deletions propagate within a bounded time.
- **Backups** — a common approach is a bounded backup retention plus a process that re-applies erasures after any restore; confirm the specific approach with the organization's data protection officer.

In .NET, `Microsoft.Extensions.Compliance` provides data classification and redaction support so that classified fields are redacted in logs — a concrete way to meet "no PII in logs."

### 15b. Residency and sovereignty

- **Data residency** — *where* data is stored and processed ("customer data stays in the EU").
- **Data sovereignty** — *whose law and whose control* applies, including access by foreign authorities and by the provider's personnel. Sovereignty requirements can rule out vendors or demand customer-managed keys and specific operational controls.

Practical Azure facts: Microsoft completed the **EU Data Boundary** in February 2025 (in three phases since 2023), committing to store and process customer data, pseudonymized personal data and, in the final phase, professional-services (support) data for its core cloud services within the EU and EFTA; some Azure services require additional customer configuration. For a design, residency mainly means: choose regions deliberately, make sure *every* component (including logs, backups, CDN caches, AI services and support tooling) respects the boundary, and design the tenancy model so tenants can be placed per region if they require it.

The fork question: *"Are there residency requirements — must any data stay within a jurisdiction?"* If yes, global active-active replication of that data is off the table, and "global" features (cross-region search, global leaderboards) need data that's allowed to move.

### 15c. Payments (PCI DSS)

The Payment Card Industry Data Security Standard governs systems that store, process or transmit cardholder data. As of 2026, **PCI DSS v4.0.1 is the only active version**; the requirements that v4.0 introduced as "future-dated" became mandatory on March 31, 2025. The architectural move is almost always **scope reduction**: keep card data out of your systems entirely by using the payment provider's hosted payment page or hosted fields and storing only tokens. That single decision can turn a heavy assessment into a much lighter one. In an interview: *"I'll keep us out of PCI scope by tokenizing at the provider — we never see a card number."*

### 15d. Sector regulation in the EU

If you're interviewing with European banks, insurers or infrastructure operators, two regimes come up often:

- **DORA (Digital Operational Resilience Act)** — applies since **January 17, 2025** to EU financial entities, including banks, insurers and investment firms. It imposes ICT risk management, major-incident classification and reporting, resilience testing (including threat-led penetration testing for significant entities), and management of ICT third-party risk — including exit strategies for critical providers such as cloud platforms. Architectural consequences: documented recovery capabilities, tested failover, incident detection and evidence, and a credible plan for leaving a provider.
- **NIS2** — the EU directive on cybersecurity for "essential" and "important" entities across many sectors (energy, transport, health, digital infrastructure, some digital providers). It requires risk management measures and staged incident reporting with tight deadlines.

Note the name clash: in engineering metrics, "DORA" also refers to the DevOps Research and Assessment metrics (deployment frequency, lead time, change failure rate, time to restore). Clarify which one you mean if both could be relevant.

### 15e. AI systems

If the system includes AI components, the **EU AI Act** may apply. Its timeline changed in 2026: obligations for general-purpose AI model providers have applied since August 2, 2025; most Article 50 transparency obligations (for example, telling people they're interacting with an AI system) were not deferred; and the **Digital Omnibus on AI** — Regulation (EU) 2026/1744, in force since July 27, 2026 — moved obligations for stand-alone high-risk systems in Annex III (employment, credit scoring, education, essential services and others) from August 2, 2026 to **December 2, 2027**, and for high-risk AI embedded in regulated products (Annex I) to **August 2, 2028**. Architecturally, high-risk classification implies logging and traceability of decisions, human oversight paths, data governance and post-market monitoring — requirements worth surfacing early if the prompt involves automated decisions about people.

### 15f. How to handle compliance in an interview

- **Ask once, early:** *"Any regulatory, residency or audit requirements I should know about?"* — and move on if the answer is no.
- **For regulated prompts, name the one or two regimes that apply and their architectural consequence,** not a list of acronyms. "Health claims data is special-category under GDPR, so I'll keep it in a segregated, encrypted store with access logging" beats reciting HIPAA, SOC 2 and ISO 27001.
- **Be explicit about your role:** "Legal decides what the regulation requires; my job is a design that makes compliance achievable and provable." That's the right professional boundary and interviewers recognize it.

---

## Concept 16 — Operability and observability

Operability is the set of qualities that determine whether the people running the system can keep it healthy. It's frequently omitted by candidates and frequently weighted by interviewers, especially for senior and staff roles where you'd own production.

### 16a. Deployability

- **How often will we deploy?** Many times a day favors small services or a well-modularized monolith with fast pipelines; quarterly releases with change boards is a constraint to design around.
- **Is zero-downtime deployment required?** If yes, every schema change must be backward compatible: the *expand/contract* (parallel change) pattern — add the new column, write both, backfill, switch reads, remove the old one. This affects how you design the data model *today*.
- **How fast must we be able to roll back?** Rollback of code is easy; rollback of data migrations is not. Feature flags (Azure App Configuration's feature management) decouple deployment from release.
- **Progressive delivery?** Canary or ring-based rollouts need routing support and per-version telemetry.

### 16b. Observability

- **What are the SLIs per critical flow** (Concept 11), and where are they measured?
- **Can we trace a request across asynchronous boundaries?** OpenTelemetry with W3C Trace Context propagated through HTTP headers *and* message properties (Module 11).
- **Can support find a specific user's failing request?** Correlation IDs surfaced to the client ("Reference: 7f3a…") and indexed in logs.
- **What does telemetry cost?** At scale, log ingestion is a major line item (Concept 18). Sampling, log levels and retention are requirements, not afterthoughts.

### 16c. Incident response

- **Who is on call, and how many people?** A two-person team can't operate a twenty-service microservice estate at 99.99%. Team size is an operability constraint.
- **What are the detection and recovery targets?** Time to detect, time to mitigate. These drive alerting design (SLO burn-rate alerts rather than threshold alerts on CPU).
- **Are there runbooks and automated mitigations** (failover, scale-out, feature kill switches)?

### 16d. Questions that sound senior

- *"How often will we deploy, and do deployments and migrations need to be zero-downtime?"*
- *"Who operates this, and how big is the on-call rotation?"*
- *"What would we alert on? I'd want burn-rate alerts on the SLOs rather than resource thresholds."*
- *"Does support need to look up and replay individual requests?"*

---

## Concept 17 — Maintainability, evolvability and the team

These qualities are about the system's future: how expensive it is to change. They're rarely the "hard part" in a 45-minute product design, but they're often *the* hard part in architect rounds, and asking about them at the right moment signals that you think beyond the whiteboard.

**First principles: modularize around what will change.** David Parnas's 1972 paper *On the Criteria To Be Used in Decomposing Systems into Modules* argued that modules should hide *design decisions likely to change*, not mirror processing steps. That's still the best requirements-to-structure bridge for maintainability: ask what's likely to change, then put boundaries around it.

| Question | Why it matters |
|---|---|
| *How long will this system live?* | A 6-month experiment and a 10-year core platform justify different investments in structure and testing. |
| *What will change most often?* | Pricing rules, partner integrations, UI flows, ML models — isolate them behind stable interfaces. |
| *How many teams will work on it, and how are they organized?* | Conway's law: the architecture will mirror the communication structure. Three teams with separate release cadences push toward three deployable units with explicit contracts (Module 21). |
| *What are the team's skills?* | A .NET team asked to operate a Scala/Kafka Streams pipeline has a real maintainability problem. |
| *Are there extension points customers need?* | Plugins, webhooks, custom fields, per-tenant rules — designing these later is expensive. |
| *How will we test it?* | Requirements like "a new partner integration can be certified in a day" imply contract tests and sandbox environments. |

**Team cognitive load.** *Team Topologies* (Skelton and Pais) frames the limit as the cognitive load a team can carry. A requirement like "one team of six will build and run this" is an architectural constraint: it argues for a modular monolith and managed services over a distributed estate (Module 21).

---

## Concept 18 — Cost

A budget is a requirement. Many candidates treat cost as something to mention in the wrap-up; strong candidates ask about it in step 1, because cost constraints eliminate options as decisively as latency targets do.

### 18a. Express cost as a unit

Absolute monthly cost is useful; **unit cost** is what businesses manage: cost per monthly active user, per 1,000 transactions, per tenant, per GB stored, per hour of video processed. Unit cost tells you which dimension cost scales with — and therefore which design choices matter. A product with thin margins per transaction needs a design whose cost per transaction is tiny; an enterprise SaaS with high per-seat revenue can afford dedicated resources for big tenants.

### 18b. Common cost drivers in Azure designs

| Driver | Why it surprises people |
|---|---|
| Always-on compute for spiky load | Paying for peak capacity 24/7; consumption or autoscale models help |
| Provisioned throughput (e.g., Cosmos DB RU/s) | Hot partitions force over-provisioning; autoscale has a cost floor |
| Zone and geo redundancy | Premium tiers, duplicate capacity, cross-region replication traffic |
| Data egress and cross-region traffic | Moving data out of a region or to the internet is billed |
| Telemetry ingestion and retention | Often one of the largest bills in a mature system |
| Licensing | SQL Server per-core licensing; Azure Hybrid Benefit can offset it |
| Engineer time | Self-hosting Kafka or Kubernetes costs people; managed services trade money for operational load |

**A quick telemetry estimate, the kind worth doing out loud:** 10,000 requests per second × 1 KB of logs per request = 10 MB/s ≈ 864 GB/day ≈ 26 TB/month. At typical per-GB ingestion pricing, that's plausibly a five-figure monthly bill on its own — so log sampling, log levels and retention tiers become explicit requirements. (Check the current pricing page for exact numbers; the order of magnitude is the point.)

### 18c. The cost of each nine

Moving from 99.9% to 99.99% typically means zone redundancy everywhere, automated failover, more rigorous testing and on-call; moving to 99.999% typically means active-active multi-region, which roughly doubles infrastructure and multiplies operational complexity. That's why availability targets should be justified against the cost of downtime (Concept 30), not chosen by instinct.

**Questions that sound senior:**

- *"Is there a budget ceiling, or a target cost per user or per transaction?"*
- *"Is this cost-sensitive — an internal tool or a thin-margin product — or is reliability worth paying for?"*
- *"Would we rather pay a managed-service premium or carry the operational load ourselves?"*

---

## Concept 19 — Constraints and environment

Constraints are decisions already made (Concept 2). You don't trade them off; you discover them early so you don't design something that can't be built.

| Category | Examples | Why ask early |
|---|---|---|
| **Existing estate** | An ERP that only accepts nightly batch files; a mainframe; a partner API limited to 50 requests/s | The integration's limits become your design's limits; queues and caches appear around them |
| **Mandated platform** | Azure only; must run on the company's AKS platform; approved-vendor lists | Eliminates options; may give you platform services for free |
| **Team** | Six .NET engineers; no SRE function; hiring freeze | Operational complexity budget |
| **Timeline** | MVP in three months; must ship before a regulatory deadline | Favors managed services, fewer components, buy over build |
| **Clients** | Mobile on poor networks; offline field workers; kiosks; constrained IoT devices; old browsers | Offline sync, conflict resolution, payload size, retry/idempotency design |
| **Geography and physics** | Users across continents | The speed of light sets latency floors (~62 ms theoretical minimum round trip Frankfurt–US East); distance can't be optimized away, only avoided with regional presence |
| **Third parties** | Payment providers, SMS gateways, identity providers | Their SLAs, rate limits and failure modes are now yours |
| **Organizational** | Security review required for new data stores; architecture board approval | Affects what's realistic in the timeline |

**The client environment question is underrated.** "Is this mobile, and do users have reliable connectivity?" forks a design: unreliable networks mean client-side retries (so every write needs idempotency), offline queues, sync protocols with conflict resolution, small payloads, and push rather than poll. Field-service and logistics prompts almost always hinge on this.

---

## Concept 20 — The long tail of quality attributes

The ISO/IEC 25010 product quality model is a useful checklist for "did I forget anything?". Its 2023 revision lists nine characteristics: **functional suitability, performance efficiency, compatibility, interaction capability** (formerly *usability*), **reliability, security, maintainability, flexibility** (formerly *portability*, now including scalability) and **safety** (new). Most were covered above; these are the ones left.

**Accessibility.** Beyond being the right thing to do, it's increasingly a legal requirement: the **European Accessibility Act** applies from June 28, 2025 to many consumer-facing products and services in the EU — including e-commerce, consumer banking, e-books and parts of passenger transport. The usual technical reference is the harmonized standard EN 301 549, which builds on WCAG; WCAG 2.2 is the current W3C recommendation. Accessibility is mostly a front-end concern, but it can create back-end requirements: captioning and transcript pipelines for video, accessible document generation, alternative text storage.

**Internationalization and localization.** Requirements here produce subtle data-model decisions:

- **Time:** store instants in UTC; store the user's IANA time zone for anything recurring or calendar-based ("every Monday at 9:00 local" can't be stored as a UTC instant because daylight saving moves it). In .NET, `DateTimeOffset` plus a time-zone ID, and `TimeProvider` for testability.
- **Money:** `decimal`, an ISO 4217 currency code, and the currency's minor units; never floating point.
- **Text:** Unicode normalization for comparisons and uniqueness ("é" has two encodings), culture-aware collation for sorting, language-specific analyzers in search indexes, right-to-left layouts.
- **.NET-specific trap:** containers often run with `InvariantGlobalization` enabled to avoid shipping ICU; that's fine for services that never do culture-sensitive operations and a bug factory for ones that do. If localization is a requirement, verify the runtime's globalization mode.

**Interoperability.** Domain standards can be requirements: HL7 FHIR in healthcare, ISO 20022 in payments, open banking APIs, OpenID Connect for identity. Using them shapes your API and data model.

**Portability and exit.** "Must be cloud-agnostic" is expensive — you forgo managed services or build abstractions over them. Probe what's really needed: often it's an *exit strategy* (required under DORA for critical ICT providers in finance) rather than active multi-cloud operation. Containers, standard protocols, open data formats and infrastructure as code make exit credible without running two clouds.

**Safety.** New as a top-level ISO 25010 characteristic in 2023: the capability not to endanger people, property or the environment. Relevant for systems that control physical devices, medical workflows, vehicles or industrial processes. Safety requirements push toward fail-safe defaults, interlocks, independent verification and very conservative change processes.

---

# Part D — From answers to design drivers

Part C gave you the questions. This part is about what you *do* with the answers: split them by flow, notice where they conflict, rank them, connect each to a mechanism, and make them checkable.

## Concept 21 — NFRs are per flow, not per system

A system isn't one thing with one availability and one latency. It's a set of **flows** — user journeys and system processes — with very different importance. Microsoft's Well-Architected Framework makes this the starting point for reliability work: identify user and system flows, rate them by criticality, and set targets per flow. The same idea applies to every quality.

Example: an e-commerce platform.

| Flow | Criticality | Availability | Latency (p99) | Consistency | Data-loss tolerance |
|---|---|---|---|---|---|
| Browse and search catalog | High | 99.95%, degrade to cached | 300 ms | Seconds stale OK | N/A (derived data) |
| Add to cart | High | 99.95% | 200 ms | Read-your-writes | Losing a cart is bad but recoverable |
| Checkout and payment | **Mission-critical** | 99.99% | 1 s excluding provider | **Strong per order; idempotent** | **Zero acknowledged orders lost** |
| Order history | Medium | 99.9% | 500 ms | Read-your-writes after checkout | N/A (reads the order store) |
| Recommendations | Low | Best effort; hide on failure | 150 ms or omit | Hours stale OK | N/A |
| Merchandising admin | Low | 99.5%, business hours | 2 s | Strong for edits | Minutes OK |
| Nightly finance export | Medium | Must complete by 06:00 | Deadline, not latency | Snapshot-consistent | Must be reproducible |

**Why this is a seniority signal.** It shows you know where money is spent. Applying the checkout row's requirements to every flow would make the whole platform expensive and slow; applying the recommendation row's requirements to checkout would lose orders. Splitting by flow lets you spend the reliability budget where it's worth it — and it directly produces architecture: the checkout path gets zone-redundant transactional storage and an outbox; the catalog gets caching and a CDN; recommendations get a timeout and a fallback.

**In the interview**, you don't need a seven-row table. Two or three tiers are enough: *"Checkout must be strongly consistent with zero lost orders and 99.99%; browsing can be a few seconds stale and should degrade to cached results; recommendations are best effort."* One sentence, three tiers, and the interviewer can see your whole cost structure.

---

## Concept 22 — NFRs conflict

Every strong requirement is paid for with some other quality. If your requirements don't conflict anywhere, you probably haven't made them strong enough to matter — or you haven't noticed the conflicts yet.

| Tension | Why they conflict | Typical resolution |
|---|---|---|
| **Consistency ↔ latency / availability** | Strong consistency across replicas needs coordination; coordination costs round trips and fails during partitions (CAP, PACELC — Module 7) | Strong only where invariants live; session or eventual elsewhere |
| **Durability (cross-region RPO 0) ↔ write latency** | Synchronous remote acknowledgment adds a cross-region round trip to each write | RPO 0 for critical data only; async geo-replication for the rest |
| **Availability ↔ cost** | Each nine needs more redundancy and automation (Concept 18c) | Tie targets to the cost of downtime per flow |
| **Latency ↔ cost** | Caches, edge presence, over-provisioning and premium tiers cost money | Budget latency per flow; cache only hot paths |
| **Security ↔ latency / usability** | Encryption, extra authorization checks, step-up authentication, inspection proxies add time and friction | Risk-based controls: strong where assets are valuable |
| **Isolation ↔ cost** | Dedicated per-tenant resources are expensive | Tiered tenancy: pool by default, silo on demand |
| **Evolvability ↔ performance** | Service boundaries and abstractions add hops and serialization | Modular monolith first; split where scaling or team needs justify it |
| **Observability ↔ cost / privacy** | More telemetry costs money and risks leaking PII | Sampling, redaction, tiered retention |
| **Time to market ↔ almost everything** | Every quality costs engineering time | Explicit "not yet" list with revisit triggers |

**How to handle a conflict in the interview.** Don't hide it, and don't resolve it by preference. Name it, split it by flow if possible, and tie the resolution to a number or a business consequence:

> *"Global strong consistency would make every write wait for a cross-region round trip — around 80–100 ms between Europe and the US East coast. Only the payment ledger needs it. So I'll make the ledger strongly consistent within one home region per account and let everything else replicate asynchronously."*

When a conflict can't be split and is genuinely a business choice, say so: *"That's a product decision — fail closed and lose sales during an outage, or fail open and risk overselling. I'd recommend failing closed for tickets because overselling is a customer-trust problem, but I'd want the business to own that call."* Architects are evaluated on exactly this: knowing which decisions are theirs and which are the business's.

---

## Concept 23 — Prioritizing: the utility tree and the hard part

You now have a set of quantified, per-flow requirements, some of which conflict. Which ones drive the design?

**The utility tree** (from the SEI's ATAM method) is a simple structure: quality attributes at the top, refined into concrete scenarios at the leaves, and each leaf rated on two axes:

- **Importance** to the business (H/M/L)
- **Difficulty** to achieve architecturally (H/M/L)

```
Utility
├── Correctness
│   └── (H,H) Two users can never hold the same seat, under 3k claims/s on one event
├── Performance
│   ├── (H,M) Seat-map availability p99 < 300 ms during on-sale
│   └── (M,L) Search p99 < 500 ms
├── Availability
│   ├── (H,H) Booking path survives a 10M-user spike without collapsing
│   └── (M,L) Browse degrades to cached pages if the catalog store is down
├── Security
│   └── (H,M) Bots can't monopolize inventory
└── Cost
    └── (M,M) Normal-day infrastructure cost stays flat; spike capacity is elastic
```

Reading the tree:

- **(High, High)** — the architecturally significant core. These become your **hard part** and your **deep dives**.
- **(High, Low/Medium)** — important but solved by well-known patterns. Satisfy them in the high-level design with a sentence each.
- **(Low, High)** — expensive and not important. Push back, defer, or simplify ("I'd drop exact seat-level live updates and show section-level counts instead").
- **(Low, Low)** — mention only if asked.

You won't draw a utility tree in a 45-minute product design round. But **running it in your head is how you choose the hard part** — and in architect rounds, drawing a small one is a strong, recognizable move.

**Stating the hard part.** The output of prioritization is one or two sentences:

> *"The hard part is the (H,H) pair: never selling a seat twice while 10 million users arrive in minutes for 50,000 seats. Everything else is standard."*

This is the bridge to deep dives (Module 3, Concept 12): you've told the interviewer where the depth will be *and why*.

---

## Concept 24 — Traceability: every NFR must produce a mechanism

Here's a test that separates decorative requirements from real ones: **for each NFR on the board, can you name the design mechanism that satisfies it?** If you can't, either the requirement doesn't matter (drop it) or it's genuinely hard (it's a deep dive).

| NFR (one line) | Mechanism(s) it produces | How you'd verify it |
|---|---|---|
| Feed read p99 < 200 ms at 50k rps | Precomputed timelines in a cache; fan-out-on-write for normal users | Load test at 50k rps; latency SLI histogram |
| Celebrity posts don't melt fan-out | Hybrid push/pull: pull for accounts above a follower threshold | Synthetic celebrity in load test |
| Zero lost acknowledged orders | Zone-redundant transactional store; ack after commit; outbox for side effects | Zone-failure chaos experiment; invariant check |
| Payment never charged twice | Idempotency key on the API; provider-side idempotency; unique constraint | Retry storm test; duplicate-webhook test |
| Search reflects edits within 5 s p99 | Change feed/outbox → indexer; lag metric | Indexer-lag SLI and alert |
| 99.99% for checkout | Zone redundancy throughout the path; no synchronous non-critical dependencies; degradation for optional calls | Composite availability estimate; SLO with burn-rate alerts |
| Survive regional outage with RTO 1 h, RPO 5 min | Warm standby region; async geo-replication; infrastructure as code; runbook | Scheduled DR drill with measured RTO/RPO |
| EU data stays in EU | EU-only regions for all stores, logs, backups, CDN origin; residency policy enforcement | Policy compliance scans; data-flow inventory review |
| User erasure within 30 days | PII isolated by user ID; TTL on derived copies; crypto-shredding for logs/backups | Automated test: create user, erase, assert absence everywhere |
| One team of six can operate it | Modular monolith; managed services; few deployables | Toil and on-call load review |
| Cost < €0.002 per transaction | Serverless/consumption for spiky paths; reserved capacity for baseline | Cost per transaction metric from billing exports |

Use the middle column as a **completeness check on your high-level design**: every row should be visible somewhere on the diagram. If "zero lost orders" is a requirement and the diagram shows a single-zone database with fire-and-forget event publishing, the interviewer will notice before you do.

---

## Concept 25 — Verifying NFRs: requirements you can check

A requirement you can't check is a hope. The senior and architect move is to say, briefly, **how each critical NFR would be verified continuously** — not just tested once before launch. *Building Evolutionary Architectures* (Ford, Parsons, Kua) calls these checks **fitness functions**: automated, objective measures of how well the architecture meets a quality goal, run in pipelines or continuously in production.

| Quality | Fitness function | .NET / Azure tooling |
|---|---|---|
| Latency under load | Pipeline load test that fails the build if p99 exceeds target at a fixed arrival rate | Azure Load Testing (JMeter or Locust scripts), k6; use open-model (constant-rate) load to avoid coordinated omission |
| Allocation/CPU on hot paths | Microbenchmarks with regression thresholds | BenchmarkDotNet with `[MemoryDiagnoser]` (Module 17) |
| Availability | SLO burn-rate alerts in production | OpenTelemetry → Azure Monitor; alert rules on error-budget burn |
| Resilience | Scheduled fault injection: kill a zone, add latency to a dependency | Azure Chaos Studio |
| Recoverability | Scheduled restore drill measuring achieved RPO/RTO | Point-in-time restore automation, failover group test failovers |
| Structural boundaries | Architecture tests that fail when a forbidden dependency appears | NetArchTest, ArchUnitNET |
| Security | Dependency and container scanning; DAST in staging | GitHub Advanced Security / Defender for Cloud, OWASP ZAP |
| Privacy | Erasure end-to-end test; log redaction test | `Microsoft.Extensions.Compliance` redaction + integration tests |

An architecture test is the cheapest example to show, because it turns a maintainability requirement ("the domain doesn't depend on infrastructure") into a failing build:

```csharp
using NetArchTest.Rules;
using Xunit;

public class ArchitectureFitness
{
    [Fact]
    public void Domain_must_not_depend_on_infrastructure_or_web()
    {
        var result = Types.InAssembly(typeof(Shop.Domain.Order).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny("Shop.Infrastructure", "Microsoft.AspNetCore", "Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful,
            "Domain types with forbidden dependencies: " +
            string.Join(", ", result.FailingTypeNames ?? []));
    }
}
```

**SLO burn-rate alerting, in one paragraph.** Rather than alerting on "error rate > 1%," alert on how fast you're consuming the error budget. With a 30-day window (720 hours), consuming 2% of the monthly budget in one hour means you're burning at 0.02 × 720 = **14.4×** the sustainable rate — page someone. Slower burns (say, 5% of budget over six hours, a 6× burn) open a ticket. The Google SRE Workbook's chapter on alerting on SLOs gives the full multi-window scheme. Saying "I'd alert on burn rate, not thresholds" in a wrap-up is a compact operational-maturity signal.

---

# Part E — Signalling seniority

## Concept 26 — The power questions

This is the centerpiece of the module. Each question below is short, sounds natural, and forks a design. The third column is what you're really demonstrating when you ask it. You'll use three to six of these in any given interview — choose by prompt.

| # | Question | What the answer decides | What asking it signals |
|---|---|---|---|
| 1 | *"What's the one thing this system must never get wrong, and what does it cost if it does?"* | Where strong consistency and invariants live; where to spend correctness effort | You think about failure and its business impact first |
| 2 | *"What's peak versus average, and is there a known spike event?"* | Autoscaling vs admission control vs queue-based load leveling | You've seen systems fall over at peak, not at average |
| 3 | *"Is load evenly spread, or are there celebrities, huge tenants or viral items?"* | Partition design, hot-key mitigation, hybrid fan-out | You design for the distribution, not the mean |
| 4 | *"When one user acts, how many others are affected?"* | Fan-out strategy and the real write volume | You know fan-out, not user count, is often the scaling problem |
| 5 | *"What happens if this is processed twice — or out of order?"* | Idempotency keys, deduplication, version checks, ordering scope | You know delivery is at-least-once in practice |
| 6 | *"After a user writes, must they see it immediately? Must everyone else?"* | Read-your-writes vs global consistency; caching strategy | You know caches and replicas are stale and that the fix is usually session-scoped |
| 7 | *"How stale can this view be — in seconds?"* | Cache TTLs, replica reads, projection lag budgets | You quantify consistency instead of choosing a side |
| 8 | *"Can we ever lose an acknowledged write? How much after a regional disaster?"* | Synchronous vs asynchronous replication; RPO per data class | You separate durability from availability |
| 9 | *"How long can each flow be down, and which flows can degrade instead of failing?"* | Per-flow availability targets; degradation modes; dependency isolation | You know availability is per flow and failure can be partial |
| 10 | *"Which external dependencies sit in the critical path, and what are their SLAs?"* | Composite availability; async decoupling; bulkheads and fallbacks | You know you can't beat a hard dependency in series |
| 11 | *"Where are users, and what are their networks like — mobile, offline, global?"* | Regions, edge, offline sync, retries, payload design | You know physics and networks set floors |
| 12 | *"Is a partial or approximate result acceptable?"* | Scatter-gather timeouts, approximate counts, degraded answers | You know tail latency amplifies with fan-out |
| 13 | *"Who are the principals, and can users share individual objects?"* | Identity provider choice; RBAC vs relationship-based authorization | You know sharing is an authorization-architecture problem |
| 14 | *"Is it multi-tenant, and do any tenants need isolation, their own keys, or a region?"* | Pooled/siloed/hybrid tenancy; partition keys; noisy-neighbor controls | You've built or operated B2B SaaS |
| 15 | *"Any residency, regulatory or audit requirements?"* | Region choices; erasure design; PCI scope; tamper-evident logs | You know regulation constrains data placement and flow |
| 16 | *"What are the abuse cases?"* | Rate limiting, bot protection, scanning pipelines, fraud controls | You've run something on the public internet |
| 17 | *"What scale do we need in three years?"* | Which one-way doors (partition keys, IDs, API shape) to get right now | You distinguish reversible from irreversible decisions |
| 18 | *"What exists already — platform, systems we must integrate with, team size and skills?"* | Constraints; build vs buy; operational complexity budget | You design for the organization, not a blank slate |
| 19 | *"Is there a budget, or a target cost per user or transaction?"* | Consumption vs provisioned; managed vs self-hosted; redundancy level | You treat cost as a first-class requirement |
| 20 | *"How will we know it's working — what's the SLO, and who gets paged?"* | SLIs, alerting, observability, team operating model | You've been on call |

**Phrasing tips.**

- **Attach the fork to the most important two or three**: *"What happens if a payment webhook arrives twice? — if it can, the confirmation path needs an inbox table keyed on the provider's event ID."*
- **Batch them**: three questions in one breath, then state the rest as assumptions.
- **Don't ask all twenty.** Asking every question regardless of prompt is a recited checklist, and interviewers recognize it. The skill is selecting the four that matter for *this* prompt.

---

## Concept 27 — Calibration by level

The same five minutes look different at each level. Use this to check recordings of your mocks.

| Aspect | Mid-level | Senior | Staff | Architect |
|---|---|---|---|---|
| **What they ask about** | Features and user counts | Scale shape, latency, consistency, availability — quantified | Non-obvious constraints: skew, abuse, fairness, deletion, migration, cost | Business drivers, stakeholders, constraints, success metrics, existing commitments |
| **How NFRs are stated** | Adjectives ("scalable, fast") | Numbers attached to operations | Numbers per flow, with the trade-off each implies | Quality-attribute scenarios tied to business outcomes and money |
| **Assumptions** | Implicit, or asked as questions | Stated and invited for correction | Stated, with which ones are risky if wrong | Logged as risks with owners and validation plans |
| **Prioritization** | Lists everything | Top three FRs; names the hard part | Reframes the problem around the real constraint | Utility tree; distinguishes architect decisions from business decisions |
| **Use of answers** | Some answers unused | Each answer changes a choice | Answers rule options in or out on the spot | Answers become documented drivers traced to decisions |
| **Typical blind spots** | Peak vs average; tails; data loss | Organizational constraints; cost; operability | Rarely — sometimes over-scopes | Must avoid losing technical depth in the deep dives |

**The staff move: reframing.** Staff candidates sometimes change the question. "Design a URL shortener" → *"At bit.ly scale the shortening itself is trivial; the real problems are the read path at the edge and abuse — phishing links. I'd like to spend the depth there."* Reframing is powerful when it's right and grounded in the requirements; it's damaging when it's a way to avoid the interviewer's intended problem. Offer it as a proposal, and accept the interviewer's answer.

---

## Concept 28 — Anti-patterns

| Anti-pattern | What it looks like | Why it hurts | The fix |
|---|---|---|---|
| **The interrogation** | Fifteen questions, one at a time, before any structure | Burns ten minutes; looks like you can't make decisions | Batch the three fork questions; assume the rest out loud |
| **Adjective requirements** | "Scalable, highly available, low latency, secure" | Nothing can be designed or traded against an adjective | OPEN test: operation, percentile, environment, number |
| **Gold-plating** | "Five nines, strongly consistent, globally" for a photo-sharing app | Shows no sense of cost; makes the design needlessly hard | Per-flow targets justified by consequence |
| **System-wide NFRs** | One availability and one latency for everything | Hides where the money and risk are | Two or three tiers of flows |
| **Asking for numbers you should derive** | "What's the QPS?" | Hands the interviewer your estimation work | Derive from DAU and actions; state the result |
| **Ask and ignore** | Asks the read/write ratio, never uses it | Questions look like ritual | Attach a fork; refer back to answers later |
| **The recited checklist** | The same twenty questions for every prompt | Looks memorized; wastes time | Pick the four that fork *this* design |
| **Missing the hard part** | Requirements are fine but nothing is flagged as dominant | Deep dives later look arbitrary | One sentence: "The hard part is…" |
| **Conflating terms** | Durability vs availability; SLA vs SLO; consistency vs durability | Small but noticeable precision gaps | Use the definitions in Concepts 11–13 |
| **Compliance lecture** | Five minutes on GDPR for a URL shortener | Wrong priorities | One question; deep only when the domain warrants |
| **Never revisiting** | New constraint arrives, requirements box unchanged | The board stops being true | Edit the box visibly (Module 3, Concept 14) |
| **Designing during requirements** | "I'll use Kafka" in minute two | Commits before understanding; invites probing too early | Park mechanisms; requirements describe the problem |

---

## Concept 29 — The five-minute delivery

### 29a. A script with timings

| Clock (45-min round) | What you do | Example phrase |
|---|---|---|
| 0:00–0:30 | Restate the problem and actors | *"So we're building a chat product — consumers on mobile and web, messaging one-to-one and in groups."* |
| 0:30–1:30 | Top three functional requirements + out of scope | *"Core: send and receive messages in real time; delivery and read receipts; history synced across devices. Out: calls, stories, payments, search."* |
| 1:30–2:30 | Batched fork questions, each with its reason | *"Three things change the design: max group size, whether we need end-to-end encryption, and how long history is kept. What would you like?"* |
| 2:30–4:00 | NFRs per flow, quantified; assumptions stated | *"Messages are never lost once acknowledged; delivery p99 under 500 ms when both are online; per-conversation ordering; presence and typing are best effort. I'll assume 500M DAU at about 50 messages a day — stop me if you have other numbers."* |
| 4:00–4:30 | The hard part | *"The hard part is fan-out with ordering and no loss across ~150M persistent connections — and keeping presence, which is a huge but low-value write stream, from hurting messaging."* |
| 4:30–5:00 | Check-in | *"Does that match what you had in mind before I estimate?"* |

### 29b. The board

Keep the requirements in the permanent left column (Module 3, Concept 15). Label the four kinds of statement so the interviewer can see which are negotiable:

```
FR    1 send/receive 1:1 + group (≤500)   2 receipts   3 multi-device history sync
OUT   calls, stories, payments, search, E2E crypto internals
NFR   [messaging]  no loss after ack · per-conversation order · p99 < 500 ms online/same region
      [presence]   best effort, may lag/drop under load
      [history]    1 yr retention · sync new device < 10 s for last 30 days
CON   Azure · .NET team
ASM   500M DAU · 50 msgs/user/day · 30% online at peak
★     fan-out + ordering + no loss at ~150M connections; isolate presence
```

Twelve lines. Every later decision can point to one of them.

---

# Part F — The architect variant

Architect rounds (and real architecture work) stretch step 1 from five minutes into a significant share of the session — Module 3's architect budget gives it about a fifth of the time. The prompt is usually a business situation, not a product: "We need to modernize X," "Two acquired companies must share Y," "We're expanding to the US." The requirements aren't hiding in the prompt; they're in the heads of stakeholders who disagree.

## Concept 30 — Eliciting drivers from stakeholders

### 30a. Who are the stakeholders?

List them early, because each holds different requirements and different veto power:

| Stakeholder | What they care about | Requirements they hold |
|---|---|---|
| Business sponsor | Outcomes, deadlines, budget | Success metrics, timeline, investment ceiling |
| Product / business owners | Capabilities, customer experience | Functional scope, latency targets, availability expectations |
| End users / operations staff | Getting work done | Usability, workflow fit, offline needs |
| IT operations / SRE | Running it | Operability, monitoring, on-call load, deployment constraints |
| Security / CISO | Risk | Identity, data protection, audit, threat model |
| Data protection officer / legal | Compliance | Residency, retention, erasure, lawful processing |
| Finance | Cost | Opex vs capex, unit cost, licensing |
| Owners of adjacent systems | Their stability | Integration contracts, batch windows, rate limits |
| Regulators (indirectly) | Systemic risk | Resilience testing, incident reporting, third-party risk (DORA) |

### 30b. Business drivers: "why now?"

The most useful architect question is *"Why are we doing this now?"* The answer reveals the dominant driver — a data-center contract ending, a regulatory deadline, a competitor's feature, a cost overrun, an outage that embarrassed the CEO. The driver tells you which qualities dominate and how much risk the organization will accept in the transition.

Follow with *"How will we know this succeeded?"* — success metrics turn vague goals into requirements: "claim cycle time for simple claims from 10 days to 2," "infrastructure cost down 25%," "no P1 incidents during peak season."

### 30c. Quality Attribute Workshops, briefly

The SEI's **Quality Attribute Workshop (QAW)** is a facilitated method for eliciting quality requirements *before* the architecture exists: business and mission presentations, identification of drivers, scenario brainstorming by stakeholders, consolidation, prioritization by voting, and refinement of the top scenarios into the six-part format (Concept 5). You'd rarely run a full QAW in an interview, but describing a light version — *"I'd run a two-hour workshop with ops, security and the business to brainstorm and vote on quality scenarios"* — is a credible architect answer to "how would you gather requirements?"

### 30d. Turning "it must never go down" into money

Stakeholders often state absolutes. The architect's job is to convert them into a decision with costs on both sides.

> **Example.** An online channel generates about €2M of revenue per day, so roughly **€83,000 per hour** of outage (ignoring reputational damage and recovered sales).
>
> - At **99.9%**, expected downtime is ~8.76 h/year → ~**€730,000** exposure.
> - At **99.99%**, ~53 min/year → ~**€73,000** exposure.
>
> Moving from 99.9% to 99.99% reduces expected exposure by roughly €650,000 a year. If the zone redundancy, automation and extra on-call needed for that step cost about €300,000 a year, the investment pays for itself. Going on to 99.999% saves at most another ~€66,000 a year of exposure — almost certainly less than the cost of active-active multi-region. **So the recommendation is 99.99% for the revenue path, not five nines.**

This calculation is crude — downtime isn't uniformly distributed, some sales are only delayed, brand damage is real — but it changes the conversation from adjectives to a decision the business can own. Present options, not a single answer: *"Here's what 99.9, 99.95 and 99.99 cost and protect; I recommend 99.99 for checkout and 99.9 for everything else."*

### 30e. When stakeholders conflict

Security wants every request re-authorized against a central service; product wants 100 ms page loads. Finance wants single-region; the business wants regional disaster recovery. The architect doesn't pick a winner by authority. The pattern:

1. **Make the conflict explicit and quantified** — "central authorization adds ~20 ms and a new critical dependency."
2. **Find the split** — risk-based: re-authorize sensitive operations, use short-lived tokens with local validation for the rest.
3. **Escalate what's genuinely a business decision** with options and consequences.
4. **Record the decision** in an ADR with the requirement it serves and the trigger to revisit (Module 31).

---

## Concept 31 — Brownfield discovery

Most architect work changes existing systems. The requirements of a brownfield system include everything it does *today* that someone depends on — including things nobody wrote down. Hyrum's law states the problem: with enough users, every observable behavior of a system will be depended on by somebody.

**Discovery questions that consistently find the real requirements:**

| Question | What it uncovers |
|---|---|
| *"What hurts today? Show me the last year of major incidents."* | The real quality problems — often different from the stated ones |
| *"Who reads from this database, directly?"* | Hidden consumers: reports, other teams' jobs, spreadsheets with ODBC connections |
| *"What does the system do on a schedule?"* | Batch jobs, file drops at 02:00, month-end processes that are implicit requirements |
| *"What have we promised customers or regulators?"* | Contractual SLAs, data-retention commitments, audit obligations already in force |
| *"What can't change?"* | Constraints: the ERP, the mainframe, the partner file format, freeze windows |
| *"Which behaviors are bugs that someone relies on?"* | Hyrum's-law dependencies that a rewrite would break |
| *"What's the data quality like?"* | Duplicates, orphans, encoding problems that block migration |
| *"Who knows how this works?"* | Knowledge concentration; key-person risk |
| *"When is peak, and when can we never deploy?"* | Seasonal freezes that constrain the migration plan |
| *"What happens if we're late?"* | Real versus soft deadlines; the cost of the old system per month |

**The principle:** *in brownfield work, the current system's behavior is the specification until proven otherwise.* Requirements gathering includes characterizing that behavior — through logs, traffic analysis, database query stores and conversations — before redesigning it.

---

## Concept 32 — Requirements that survive

Interview requirements live on a whiteboard for an hour. Real requirements must survive years, reorganizations and personnel changes. Architect candidates are often asked how they'd document them.

**A minimal quality-requirements register** — one row per architecturally significant scenario:

| ID | Scenario (six-part, compressed) | Priority (I, D) | Owner | Verification | Linked decisions |
|---|---|---|---|---|---|
| QS-01 | During peak (20× normal), a customer submitting a claim from mobile gets confirmation within 5 s p95 | (H, M) | Claims product owner | Quarterly load test at 20× | ADR-004 (queue-based intake) |
| QS-02 | If the primary region fails, accepted claims are not lost (RPO 0 for intake) and intake resumes within 15 min | (H, H) | Head of IT operations | Semi-annual DR drill | ADR-007 (multi-region intake) |
| QS-03 | A new product line's claim rules can be deployed by the claims team within 2 sprints, without a core release | (H, M) | Engineering lead | Measured on next product launch | ADR-010 (rules engine boundary) |

**Where it lives.** The arc42 template's section 10 ("Quality Requirements") holds exactly this: a quality tree plus scenarios; arc42's companion site (Q42) offers a large catalogue of example quality requirements. SLOs get their own living document per service (SLI definitions, targets, error-budget policy). ADRs (Module 31) reference requirement IDs, so that when someone asks in two years "why is intake multi-region?", the answer points to QS-02 and the business reason behind it.

**Keep it alive.** Requirements drift. Review the register when drivers change (new market, new regulation, an incident that revealed a missing scenario), and retire scenarios that no longer matter. A stale register is worse than none, because people trust it.

---

# Part G — Worked examples

## Concept 33 — "Design a chat application": the five minutes, verbatim

This walkthrough shows the requirements step only — what you'd say, what the interviewer answers, and what each answer changes. Estimation (Module 5) and the design itself follow it.

### 0:00 — Restate and find the actors

> *"Let me make sure I've got the shape. A consumer messaging app — people on mobile and web, one-to-one and group conversations. Actors are users, possibly on several devices each, and our own back-end services. Is this consumer-scale, like WhatsApp or Messenger, rather than an enterprise tool like Teams?"*
>
> **Interviewer:** "Consumer scale."

*Why:* consumer versus enterprise forks identity (self-sign-up versus Entra ID), compliance (eDiscovery and retention policies are enterprise requirements) and scale.

### 0:30 — Functional requirements and scope

> *"Core functional requirements: one, users can send and receive text messages, one-to-one and in groups, in real time. Two, senders see delivery and read status. Three, a user's history is available on all their devices, including a newly added one. Out of scope: voice and video calls, stories, payments, message search, and media processing — I'll treat attachments as blobs referenced by messages. Also out of scope, but I'll keep it possible: account deletion and moderation."*

### 1:30 — The fork questions, batched

> *"Three things would change the design substantially. First, maximum group size — small groups let us fan out on write; huge broadcast channels need a different read path. Second, end-to-end encryption — if the server can't read messages, server-side search and moderation are off the table and multi-device needs key management. Third, how long the server keeps history — that decides storage volume and tiering. What would you like for those?"*
>
> **Interviewer:** "Groups up to 500. Don't worry about end-to-end encryption — assume TLS in transit and encryption at rest. Keep history for a year on the server."

| Answer | What it changes |
|---|---|
| Groups ≤ 500 | Fan-out on write per message is viable (bounded cost per message); no separate broadcast-channel design |
| No E2E encryption | Server stores readable messages; multi-device sync is a server-side read; moderation possible later |
| One-year server history | Storage grows to petabyte scale; hot/cold tiering by age; new-device sync pulls recent history first |

### 2:30 — Non-functional requirements, per flow, with assumptions

> *"Non-functionals. For messaging: a message is never lost once the server acknowledges it to the sender — that acknowledgment is what turns the single check mark on, so it must mean 'durably stored,' replicated across zones. Ordering: everyone in a conversation sees messages in the same order; I don't need global order across conversations. Delivery is at-least-once, and clients deduplicate by message ID, so users see each message exactly once. Latency: when both users are online in the same region, p99 under 500 ms from send to delivery on the recipient's device. Availability: 99.99% for sending and receiving."*
>
> *"For presence and typing indicators: best effort. They can lag or drop under load — and they must never be allowed to degrade messaging."*
>
> *"For history sync: a new device shows the last 30 days within about 10 seconds; older history loads on demand."*
>
> *"Scale assumptions — stop me if you have different numbers: 500 million daily users, about 50 messages each per day, and around 30% online at peak. That's roughly 25 billion messages a day, about 290 thousand a second on average and maybe 900 thousand at peak — before multiplying by recipients and devices for deliveries. Around 150 million concurrent connections at peak. At ~200 bytes a message that's ~5 TB a day, so a year of history is on the order of 1.8 PB before replication."*

### 4:00 — The hard part

> *"The hard part is delivering fan-out with per-conversation ordering and no loss across roughly 150 million persistent connections and multiple devices per user — while presence, which generates far more events than messages and is worth far less, stays isolated so it can be shed under load. Storage is big but straightforward; connection management and fan-out are the design."*

### 4:30 — Check-in

> *"Does that match what you had in mind? If so, I'll do a quick estimate of connection servers and storage throughput, then move to the API."*

### What made this work

- **Three fork questions, batched, each with its reason** — and every answer visibly changed something.
- **The acknowledgment was defined** ("the check mark means durably stored across zones") — durability stated in user-visible terms.
- **Ordering was scoped** (per conversation, not global), and **delivery semantics were stated** (at-least-once plus client deduplication).
- **Two tiers of flows**: messaging strict; presence best effort and isolated — a requirement that becomes a bulkhead in the design.
- **Numbers were derived, not requested**, and only the decision-relevant ones were mentioned.
- **The hard part named a mechanism problem** (connections, fan-out, ordering), not "scale."

---

## Concept 34 — One prompt, three customers: "Design a URL shortener"

This is the most compact demonstration of Concept 3. The functional requirements are identical in all three cases: *create a short link for a long URL; redirect a short link to its target; see basic click counts.* Only the non-functional requirements differ — and the architectures diverge completely.

| | **A. Internal tool** | **B. Public consumer service** | **C. Regulated B2B SaaS** |
|---|---|---|---|
| Who | 2,000 employees sharing links internally | Anyone on the internet, at bit.ly-like scale | Marketing teams at EU banks and insurers, on their own branded domains |
| Scale | ~10k redirects/day | Billions of redirects/month; heavily read-skewed (≈100:1) with viral spikes | Thousands of tenants; spiky per campaign; tenant sizes vary by orders of magnitude |
| Latency | "Feels instant" — a few hundred ms is fine | Redirect p99 tens of ms, globally (the redirect *is* the product) | p99 < 100 ms in the EU |
| Availability | 99.9%, business hours matter most | 99.99%+ for redirects; link creation can be lower | 99.95% contractual SLA with service credits |
| Consistency | Anything reasonable | New links resolvable within seconds; analytics approximate | **Revoked or expired links must stop redirecting within 60 s everywhere** — a bounded-staleness requirement on every cache and edge |
| Security | Entra ID single sign-on; employees only | Abuse is the main threat: phishing and malware links, spam, enumeration of codes | Each tenant federates its own identity provider; per-tenant roles; audit of every link change |
| Compliance | Internal policy | Takedown process; click-data privacy | GDPR (click data is personal data); **EU residency**; customers subject to **DORA** will assess you as an ICT third party and ask for incident reporting and exit support |
| Cost | Near zero | Cost per redirect must be tiny | Per-tenant cost visibility; premium tiers justified by contract |
| **Resulting architecture** | One small App Service plus a small Azure SQL database, or simply buy an existing product. No caching, no sharding. | Edge-served redirects (CDN or edge compute with caching); a key-value store partitioned by short code (e.g., Cosmos DB); coordination-free ID generation; async click analytics through an event stream; link-reputation scanning pipeline; aggressive rate limiting on creation | EU-only regions for *every* component including logs and backups; pooled multitenancy with tenant ID in every key, siloed option for large tenants; custom-domain TLS automation; cache TTLs bounded and active invalidation on revocation to meet the 60 s requirement; tamper-evident audit trail (e.g., Azure SQL ledger); documented DR and exit plan |
| **The hard part** | Nothing — choose the simplest thing | Read path latency at global scale, and abuse | Revocation propagation within a bound, tenant isolation, and evidencing compliance |

**What to take from this:** if an interviewer gives you "design a URL shortener," the *questions* determine which of these three you're designing. A candidate who jumps to "base62-encode an auto-increment ID and put it in Redis" has answered column B without knowing whether the interviewer meant column C — where the hardest requirement is a *consistency bound on revocation*, something no canonical URL-shortener answer mentions.

---

## Concept 35 — Architect elicitation: modernizing claims at an EU insurer

> *Prompt: "We're a mid-sized European insurer. Claims run on a 15-year-old .NET Framework application with a SQL Server database in our own data center, plus nightly batch integration with the policy administration system. We want customers to report claims from their phones, process simple claims automatically, and reduce IT costs. How would you approach the architecture?"*

An architect spends the first fifteen to twenty minutes of a ninety-minute session here. A condensed version:

### Step 1 — Stakeholders and drivers

> *"Before designing, I'd like to understand who needs what and why now. Who are the key stakeholders — the head of claims, IT operations, security, the data protection officer, finance, and the team that owns policy administration? And what's driving the timing?"*
>
> **Interviewer:** "The data-center contract ends in 18 months. The board wants simple claims settled in two days instead of ten. And our regulator has been asking about operational resilience."

The candidate notes three drivers with very different natures: a **hard deadline** (data-center exit), a **business outcome** (cycle time 10 → 2 days for simple claims), and a **regulatory pressure** (DORA applies to insurers since January 2025, covering ICT risk, incident reporting, resilience testing and third-party risk — which includes the cloud provider they're moving to).

### Step 2 — Constraints and current state

> *"What can't change? How does the policy administration system integrate today, and when can it change? What does the team look like? And what does a peak look like?"*
>
> **Interviewer:** "Policy admin is a vendor product; it exposes a nightly batch export and a read API that's unavailable from 01:00 to 05:00. The vendor releases twice a year. We have twelve .NET developers and a small ops team. Peaks happen during storms — claim volume can jump 20× for a couple of days."

| Constraint | Consequence the candidate states |
|---|---|
| Policy admin read API down 01:00–05:00, changes twice a year | Claim intake must not depend synchronously on it; cache or replicate policy data needed for validation |
| 18-month data-center exit | Transition architecture must move the existing system first (rehost/replatform), with modernization incremental — not a big-bang rewrite |
| Twelve .NET developers, small ops team | Managed services and a modular monolith or few services; operational load is a real budget |
| 20× storm peaks | Intake must absorb bursts — queue-based load leveling; processing can lag, intake can't fail |

### Step 3 — Quality-attribute scenarios

The candidate converts what they've heard into a handful of scenarios, compressed to one line each:

| ID | Scenario | (Importance, Difficulty) |
|---|---|---|
| QS-1 | During a storm surge at 20× normal volume, a customer submitting a claim from mobile receives confirmation within 5 s p95, and no accepted claim is lost | (H, H) |
| QS-2 | If the primary Azure region is unavailable, claim intake continues within 15 minutes; processing may resume later | (H, H) |
| QS-3 | A simple claim (clear coverage, under a threshold amount) is decided automatically within 1 hour of submission, with the decision and its inputs logged for audit | (H, M) |
| QS-4 | Injury claims contain health data — special-category data under GDPR — and are accessible only to authorized adjusters, with every access logged | (H, M) |
| QS-5 | The claims team can change straight-through-processing rules and thresholds without a code release, with changes reviewed and audited | (M, M) |
| QS-6 | A major ICT incident is detected, classified and evidenced fast enough to meet regulatory reporting deadlines | (H, M) |
| QS-7 | Running cost of the claims platform is at least 20% below the current data-center cost after migration | (M, M) |

### Step 4 — Prioritize and name the hard part

> *"The two (High, High) scenarios are about intake: absorbing a 20× storm surge without losing claims, and continuing intake through a regional outage. That's where the architecture risk is — and conveniently, intake is also the new capability, so we can build it cloud-native without touching the legacy core first. The second area of risk is the transition itself: we have 18 months to leave the data center with a team of twelve, so I'd separate 'move the existing system' from 'modernize it.'"*

### Step 5 — Separate architect decisions from business decisions

> *"Two decisions I'd bring back to the business with options rather than make myself. First, the threshold for automatic settlement — it trades cost of fraud against cycle time; that's a claims and risk decision, and I'd design for it to be configurable. Second, the recovery target for claims processing, as opposed to intake — I've assumed hours are acceptable, but if the business wants minutes, that roughly doubles the infrastructure for the processing tier."*

### What made this an architect-level answer

- **"Why now?"** surfaced three drivers of different types, including the regulatory one.
- **Constraints were turned into design consequences immediately** — the policy-admin outage window became an asynchronous-intake requirement.
- **Requirements became scenarios** with measures, ratings and a clear (H,H) core.
- **The transition was treated as a requirement in its own right**, not an afterthought.
- **Business decisions were identified and handed back with options**, which is precisely the judgment architect rounds score.

---

## Putting it together: how this shows up in interviews

### Common questions and model answers

**"What non-functional requirements would you consider for this system?"**
Don't recite a list. Pick the three or four that drive the design, quantify them per flow, and say why the others matter less: *"The ones that drive this design are no lost messages after acknowledgment, per-conversation ordering, and delivery latency under 500 ms. Availability matters but follows from the redundancy those need. Cost matters at this scale mainly through storage tiering."*

**"How available should it be?"**
Answer per flow and tie the number to a consequence: *"Messaging 99.99% — that's about four minutes a month, which means failover must be automatic. Presence can be much lower because it degrades gracefully. If you'd like 99.999%, that's 26 seconds a month and implies active-active across regions, which roughly doubles cost — I'd want a business case."*

**"What's the difference between an SLA, an SLO and an SLI?"**
The SLI is the measurement (good events over valid events), the SLO is the internal target for it over a window, and the SLA is the contract with consequences — usually looser than the SLO so you can react before you owe credits. Add error budgets: the SLO defines how much failure you can spend on change.

**"What's the difference between RPO and RTO?"**
RPO is how much data you can afford to lose, measured in time; RTO is how long you can afford to be down. They're business numbers that map to mechanisms: RPO 0 needs synchronous replication; seconds means async geo-replication; hours means backups. RTO seconds needs automatic failover; hours allows restore-and-redeploy.

**"Isn't durability the same as availability?"**
No. Availability is whether I can use the data now; durability is whether acknowledged data still exists later. A region outage makes data unavailable but not lost; a bad failover that drops recent writes leaves the system available but not durable. Also: replication protects durability against hardware failure, not against bugs or deletes — that's what point-in-time restore and backups are for.

**"What does strongly consistent mean here, exactly?"**
Translate to the operation: *"For seat claims, linearizable per seat — once one user's claim succeeds, every later claim on that seat fails. I don't need strong consistency for the seat map display; that can be a second stale."* Scope the guarantee to the data and the operation.

**"The interviewer says: 'You decide.'"**
Decide quickly, with reasoning and a revisit trigger: *"Then groups cap at 500 — fan-out on write stays viable. If we later need broadcast channels with millions of followers, that becomes a pull-based read path."*

**"The interviewer gives you no numbers at all."**
Derive and state them: *"I'll assume 100M DAU and 20 actions a day — about 23k requests a second on average and perhaps 70k at peak. Correct me if that's off."* Then use them.

**"How would you handle a user's request to delete their data?"**
Identify every copy: primary store, caches, search index, analytics, event log, backups. Keep PII in one place referenced by opaque IDs; give derived copies short TTLs; use crypto-shredding for immutable logs and backups; run an end-to-end erasure test. Confirm the backup approach with the data protection officer.

**"Two stakeholders want contradictory things. What do you do?"**
Quantify the conflict, look for a split (by flow, by risk level), present options with consequences to whoever owns the business decision, and record the outcome in an ADR with a revisit trigger.

**"The business says the system must never go down."**
Convert to money: cost of downtime per hour × downtime per year at each availability level, compared with the cost of achieving each level. Recommend per-flow targets; let the business own the final trade.

**"How would you make sure the system actually meets these requirements over time?"**
Fitness functions: load tests with latency thresholds in the pipeline, SLO burn-rate alerts in production, scheduled chaos experiments and restore drills, architecture tests for structural rules, and an erasure test for privacy.

**"What's an architecturally significant requirement?"**
A requirement with a measurable effect on the architecture — usually high business value or high technical risk. Most requirements aren't; finding the few that are is the core of step 1.

**"Why did you ask about group size?"**
*"Because it decides fan-out strategy. With groups capped at a few hundred, writing one copy per recipient's inbox is affordable; with millions of members, each message would be millions of writes, so I'd switch to pull for large groups."* Every question you ask should survive this follow-up.

### Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Lists ten features | Three core FRs, explicit out-of-scope, hidden requirements named in a clause |
| "Scalable, available, fast, secure" | OPEN test: operation, percentile, environment, number |
| One availability number for the whole system | Two or three flow tiers with different targets and degradation modes |
| Averages for latency | Percentiles, with where they're measured and at what throughput |
| Ignores fan-out and tails | Asks about fan-out; knows 1 − 0.99ⁿ; asks if partial results are acceptable |
| "Strong consistency" or "eventual consistency" for everything | Per-operation guarantees translated from business language |
| Confuses durability and availability | Separates them; defines what an acknowledgment means |
| Treats replication as backup | Point-in-time restore, soft delete, restore drills |
| Assumes exactly-once delivery | Asks what happens on duplicates and reordering; plans idempotency |
| Forgets external dependencies | Multiplies availabilities; decouples from dependencies it can't beat |
| Ignores tenancy in B2B prompts | Asks about tenant size skew, isolation, keys and residency |
| Compliance lecture, or none at all | One question; deep only when the domain warrants; states the architectural consequence |
| Asks "what's the QPS?" | Derives it from users and actions, states it, invites correction |
| Asks questions and never uses the answers | Attaches design forks; refers back to answers during the design |
| Asks the same checklist every time | Picks the four questions that fork this prompt |
| No hard part | Names the (High, High) driver in one sentence |
| Cost only in the wrap-up | Asks for budget or unit-cost targets in step 1 |
| Picks five nines by instinct | Converts downtime to money and recommends per-flow targets |
| Never revisits requirements | Edits the requirements box visibly when constraints change |
| (Architect rounds) Jumps to target architecture | Stakeholders, "why now?", constraints, scenarios, then design — and hands business decisions back |

---

## Practice exercises

**Exercise 1 — The OPEN rewrite drill.** Take these ten adjective requirements and rewrite each one to pass the OPEN test (operation, percentile, environment, number) for a system of your choice: "fast search," "highly available checkout," "scalable feed," "secure login," "reliable notifications," "consistent inventory," "low-latency trading," "durable uploads," "responsive dashboard," "real-time location." Then, for each, write the mechanism it produces (Concept 24).

**Exercise 2 — Fork-question hunting.** For ten prompts (chat, news feed, ride sharing, food delivery, file sync, payments, metrics, rate limiter, job scheduler, collaborative document editor), write the *two* questions whose answers would most change the design, and for each, the two designs it forks between. Compare with published breakdowns. If your questions don't fork anything, replace them.

**Exercise 3 — Three customers.** Pick a prompt and write Concept 34's table for three very different customers (internal, consumer, regulated B2B). Identify the hard part for each. The goal is to internalize that NFRs, not features, choose the architecture.

**Exercise 4 — Per-flow targets.** For an e-commerce or banking app, list seven flows and fill in Concept 21's table: criticality, availability, latency, consistency, data-loss tolerance. Then compute a composite availability for the most critical flow using real Azure SLA figures from the current SLA page, and identify which dependency caps you.

**Exercise 5 — Nines and money.** Pick a business (or your own project) and estimate revenue or cost per hour of downtime. Compute expected annual exposure at 99.9%, 99.95%, 99.99% and 99.999%. Write a two-paragraph recommendation to a non-technical sponsor.

**Exercise 6 — Build an SLI and a fitness function in .NET.** In a small ASP.NET Core service, add a custom histogram for one business operation with bucket boundaries aligned to an SLO, export it with OpenTelemetry to a local collector or Aspire's dashboard, and write a k6 or Azure Load Testing script with a p99 threshold that fails the run. Add a NetArchTest rule enforcing one structural requirement.

**Exercise 7 — The erasure audit.** Take a system you know (or one of your designs from Module 3's exercises) and list every place a user's personal data ends up: primary store, caches, search, logs, traces, analytics, event streams, backups, third parties. For each, write how erasure would propagate and within what time bound.

**Exercise 8 — Architect elicitation role-play.** With a peer playing a business sponsor who answers only in business language ("it has to be rock solid," "we can't lose claims," "make it cheaper"), run a 20-minute elicitation for a brownfield modernization. Produce: stakeholder list, three drivers, five constraints with consequences, five quality-attribute scenarios with (Importance, Difficulty) ratings, and two decisions you'd hand back to the business with options.

### Self-scoring checklist for the requirements step

| # | Check | ✓ |
|---|---|---|
| 1 | Restated the problem and named the actors | |
| 2 | ≤ 3 core functional requirements; explicit out-of-scope | |
| 3 | Asked 2–4 fork questions, batched, with reasons | |
| 4 | Stated assumptions out loud and invited correction | |
| 5 | Every NFR passes the OPEN test | |
| 6 | NFRs split into at least two flow tiers | |
| 7 | Consistency stated per operation, in business terms | |
| 8 | Durability/data-loss tolerance stated separately from availability | |
| 9 | Derived (didn't request) the key numbers | |
| 10 | Named the hard part in one sentence | |
| 11 | Every answer received changed something in the plan | |
| 12 | Finished within ~5 minutes (45-minute round) | |
| 13 | Checked in with the interviewer before moving on | |
| 14 | (Architect) Asked "why now?" and listed stakeholders and constraints | |
| 15 | (Architect) Identified which decisions belong to the business | |

---

## Free resources

### Interview method and requirements in design rounds

| Resource | What it covers |
|---|---|
| [Hello Interview — Delivery Framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery) | Requirements as the first step, with timings and examples of good NFRs |
| [Hello Interview — Core Concepts](https://www.hellointerview.com/learn/system-design/core-concepts/numbers-to-know) | Numbers and concepts that make NFRs concrete |
| [ByteByteGo — A Framework for System Design Interviews](https://bytebytego.com/courses/system-design-interview/a-framework-for-system-design-interviews) | Step 1 "understand the problem and establish design scope," with sample questions |
| [interviewing.io — A Senior Engineer's Guide to the System Design Interview](https://interviewing.io/guides/system-design-interview) | Expectations by level, including requirement gathering |
| [The System Design Primer](https://github.com/donnemartin/system-design-primer) | "Outline use cases, constraints, and assumptions" as step 1 |
| [Tech Interview Handbook — System Design](https://www.techinterviewhandbook.org/system-design/) | Interview approach and curated resources |

### Quality attributes: theory and catalogues

| Resource | What it covers |
|---|---|
| [ISO/IEC 25010 overview (iso25000.com)](https://iso25000.com/index.php/en/iso-25000-standards/iso-25010) | The product quality model as a checklist |
| [Q42 — Update on ISO 25010, version 2023](https://quality.arc42.org/articles/iso-25010-update-2023) | What changed in 2023: safety added, usability → interaction capability, portability → flexibility |
| [Q42 — The arc42 quality model](https://quality.arc42.org/) | Large catalogue of quality attributes and example requirements |
| [arc42 — Section 10: Quality Requirements](https://docs.arc42.org/section-10/) | Quality trees and scenarios in an architecture document |
| [SEI — Quality Attribute Workshops (report)](https://www.sei.cmu.edu/library/quality-attribute-workshops/) | The original QAW method |
| [SEI — Quality Attribute Workshops, Third Edition (PDF)](https://www.sei.cmu.edu/documents/716/2003_005_001_14249.pdf) | The refined QAW: scenario generation, prioritization, refinement |
| [InfoQ — Toward Agile Architecture: Insights from 15 Years of ATAM Data](https://www.infoq.com/articles/atam-quality-attributes/) | Quality-attribute scenarios and utility trees in practice |
| [Wikipedia — Architecturally significant requirements](https://en.wikipedia.org/wiki/Architecturally_significant_requirements) | Definition and criteria for ASRs |
| [Wikipedia — Non-functional requirement](https://en.wikipedia.org/wiki/Non-functional_requirement) | Terminology and examples |
| [Wikipedia — List of system quality attributes](https://en.wikipedia.org/wiki/List_of_system_quality_attributes) | A long checklist of "-ilities" |
| [Wikipedia — MoSCoW method](https://en.wikipedia.org/wiki/MoSCoW_method) | Prioritization vocabulary for stakeholder conversations |
| [Volere Requirements Specification Template](https://www.volere.org/templates/volere-requirements-specification-template/) | A classic, thorough requirements template including fit criteria |
| [Joel Spolsky — Painless Functional Specifications, Part 1](https://www.joelonsoftware.com/2000/10/02/painless-functional-specifications-part-1-why-bother/) | Why writing requirements down saves time |

### SLIs, SLOs, availability and error budgets

| Resource | What it covers |
|---|---|
| [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) | SLI/SLO/SLA definitions and choosing indicators |
| [Google SRE Book — Embracing Risk](https://sre.google/sre-book/embracing-risk/) | Why 100% is the wrong target; the cost of each nine |
| [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/) | Per-journey SLIs, windows, documentation |
| [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) | Burn-rate and multi-window alerting |
| [Google SRE Workbook — Example Error Budget Policy](https://sre.google/workbook/error-budget-policy/) | What happens when the budget runs out |
| [Google Cloud — SRE fundamentals: SLIs, SLAs and SLOs](https://cloud.google.com/blog/products/devops-sre/sre-fundamentals-slis-slas-and-slos) | A concise introduction |
| [ACM Queue — The Calculus of Service Availability](https://queue.acm.org/detail.cfm?id=3096459) | How dependencies constrain achievable availability |
| [uptime.is](https://uptime.is/) | Instant nines-to-downtime conversion |
| [Alex Ewerlof — Calculating composite SLA](https://alexewerlof.medium.com/calculating-composite-sla-d855eaf2c655) | Serial and parallel dependency math with examples |
| [AWS Well-Architected — Availability (Reliability pillar)](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/availability.html) | Hard vs redundant dependencies, availability math |

### Azure Well-Architected and Azure reliability facts

| Resource | What it covers |
|---|---|
| [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/) | The five pillars as requirement prompts |
| [WAF — Defining reliability targets (composite SLOs)](https://learn.microsoft.com/en-us/azure/well-architected/reliability/metrics) | Composite SLO calculation and its caveats |
| [WAF — Identify and rate user and system flows](https://learn.microsoft.com/en-us/azure/well-architected/reliability/identify-flows) | Per-flow criticality |
| [WAF — Failure mode analysis](https://learn.microsoft.com/en-us/azure/well-architected/reliability/failure-mode-analysis) | Mapping flows to failure points and mitigations |
| [WAF — Reliability maturity model](https://learn.microsoft.com/en-us/azure/well-architected/reliability/maturity-model) | How reliability practice grows over time |
| [WAF — Mission-critical workloads](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-overview) | Design when availability targets are very high |
| [WAF — Cost Optimization pillar](https://learn.microsoft.com/en-us/azure/well-architected/cost-optimization/) | Cost as a requirement |
| [Service Level Agreements for Microsoft Online Services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services) | Current Azure SLAs |
| [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy) | LRS/ZRS/GRS/GZRS durability and availability |
| [Azure blog — Understanding and leveraging Azure SQL Database's SLA](https://azure.microsoft.com/en-us/blog/understanding-and-leveraging-azure-sql-database-sla/) | 99.995% zone-redundant SLA and business-continuity SLA |
| [Azure Cosmos DB — Global distribution](https://learn.microsoft.com/en-us/azure/cosmos-db/distribute-data-globally) | Multi-region writes and 99.999% availability |
| [Azure Cosmos DB — Consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels) | Five consistency levels as requirement vocabulary |
| [Azure Architecture Center — Multitenant architecture guide](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) | Tenancy requirements and models |
| [Azure Architecture Center — Tenancy models](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models) | Pooled, siloed and hybrid trade-offs |
| [Noisy Neighbor antipattern](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor) | Isolation requirements in shared systems |
| [Azure Architecture Center — Performance antipatterns](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/) | What happens when performance requirements are ignored |

### Latency and performance

| Resource | What it covers |
|---|---|
| [Dean & Barroso — The Tail at Scale (CACM)](https://cacm.acm.org/research/the-tail-at-scale/) | Tail amplification with fan-out; hedged and tied requests |
| [Google Research — The Tail at Scale](https://research.google/pubs/the-tail-at-scale/) | Paper page |
| [Nielsen Norman Group — Response Times: The 3 Important Limits](https://www.nngroup.com/articles/response-times-3-important-limits/) | 0.1 s, 1 s, 10 s |
| [web.dev — Web Vitals](https://web.dev/articles/vitals) | Current user-centric web performance targets |
| [Gil Tene — How NOT to Measure Latency (InfoQ)](https://www.infoq.com/presentations/latency-response-time/) | Percentiles and coordinated omission |
| [Marc Brooker — Tail latency might matter more than you think](https://brooker.co.za/blog/2021/04/19/latency.html) | Why tails dominate user experience |
| [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) | Utilization, saturation, errors for performance analysis |

### Consistency and correctness

| Resource | What it covers |
|---|---|
| [Doug Terry — Replicated Data Consistency Explained Through Baseball](https://www.microsoft.com/en-us/research/publication/replicated-data-consistency-explained-through-baseball/) | Per-actor consistency needs on the same data |
| [Werner Vogels — Eventually Consistent](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) | Client-side consistency guarantees in business terms |
| [Jepsen — Consistency Models](https://jepsen.io/consistency) | Precise definitions with a map of the hierarchy |
| [Amazon Builders' Library — Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) | Duplicate handling as a requirement |
| [Stripe — Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) | Idempotency keys in practice |

### Durability, recovery and resilience

| Resource | What it covers |
|---|---|
| [Amazon Builders' Library — Static stability using Availability Zones](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/) | Designing so dependencies' failures don't propagate |
| [Amazon Builders' Library — Avoiding fallback in distributed systems](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/) | When degradation modes help and when they hurt |
| [Amazon Builders' Library — Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) | Shedding low-priority work to protect critical flows |
| [GitLab — Postmortem of database outage of January 31, 2017](https://about.gitlab.com/blog/2017/02/10/postmortem-of-database-outage-of-january-31/) | Why replication isn't backup, and untested backups aren't backups |
| [AWS — Summary of the December 7, 2021 us-east-1 event](https://aws.amazon.com/message/12721/) | Correlated failure and dependency chains in practice |
| [Azure status history](https://azure.status.microsoft/en-us/status/history/) | Post-incident reviews of real Azure outages |
| [Azure Chaos Studio overview](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-overview) | Fault injection to verify resilience requirements |

### Security, identity and privacy

| Resource | What it covers |
|---|---|
| [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/) | Security requirements you can verify, by level |
| [OWASP API Security Top 10](https://owasp.org/API-Security/) | API-specific threats to turn into requirements |
| [Microsoft Threat Modeling Tool](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool) | STRIDE-based threat modeling (Module 29) |
| [Microsoft Entra External ID overview](https://learn.microsoft.com/en-us/entra/external-id/external-identities-overview) | Customer and partner identity; Azure AD B2C's end of sale |
| [Google — Zanzibar: Google's Consistent, Global Authorization System](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/) | Relationship-based authorization at scale |
| [Azure SQL — Ledger overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-overview) | Tamper-evident audit requirements |
| [.NET — Data redaction](https://learn.microsoft.com/en-us/dotnet/core/extensions/data-redaction) | Keeping classified data out of logs |
| [ASP.NET Core — Rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) | Abuse-related requirements in .NET |
| [Wikipedia — Crypto-shredding](https://en.wikipedia.org/wiki/Crypto-shredding) | Erasure in immutable stores and backups |

### Compliance and regulation (EU focus)

| Resource | What it covers |
|---|---|
| [GDPR text (gdpr-info.eu)](https://gdpr-info.eu/) | The regulation, article by article |
| [GDPR Article 17 — Right to erasure](https://gdpr-info.eu/art-17-gdpr/) | The source of the deletion requirement |
| [European Data Protection Board](https://www.edpb.europa.eu/) | Official guidelines |
| [UK ICO — UK GDPR guidance](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/) | Very readable practical guidance |
| [Microsoft completes the EU Data Boundary (DCD)](https://www.datacenterdynamics.com/en/news/microsoft-completes-eu-data-boundary-for-microsoft-cloud/) | What the EU Data Boundary covers |
| [PCI SSC Document Library](https://www.pcisecuritystandards.org/document_library/) | PCI DSS v4.0.1 and SAQs |
| [EIOPA — Digital Operational Resilience Act](https://www.eiopa.europa.eu/digital-operational-resilience-act-dora_en) | DORA for insurers and pension funds |
| [European Commission — NIS2 Directive](https://digital-strategy.ec.europa.eu/en/policies/nis2-directive) | Cybersecurity obligations by sector |
| [Orrick — EU AI Act: Digital Omnibus finalizes compliance changes](https://www.orrick.com/en/Insights/2026/07/EU-AI-Act-Update-Digital-Omnibus-Finalizes-8-Compliance-Changes) | The 2026 timeline changes |
| [EU AI Act explorer](https://artificialintelligenceact.eu/) | Navigable text and summaries |
| [European Commission — European Accessibility Act](https://commission.europa.eu/strategy-and-policy/policies/justice-and-fundamental-rights/disability/european-accessibility-act-eaa_en) | Accessibility obligations from June 2025 |
| [W3C — WCAG 2.2](https://www.w3.org/TR/WCAG22/) | The current accessibility guidelines |

### Operability, evolvability, cost and verification

| Resource | What it covers |
|---|---|
| [.NET observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel) | Measuring SLIs in .NET |
| [ASP.NET Core health checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) | Liveness and readiness requirements |
| [Azure App Configuration — .NET feature management](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference) | Decoupling deploy from release |
| [Thoughtworks — Fitness function-driven development](https://www.thoughtworks.com/insights/articles/fitness-function-driven-development) | Turning quality goals into automated checks |
| [NetArchTest](https://github.com/BenMorris/NetArchTest) | Architecture rules as unit tests in .NET |
| [ArchUnitNET](https://github.com/TNG/ArchUnitNET) | An alternative architecture-testing library |
| [Azure Load Testing overview](https://learn.microsoft.com/en-us/azure/load-testing/overview-what-is-azure-load-testing) | Load tests with pass/fail criteria in pipelines |
| [Grafana k6 documentation](https://grafana.com/docs/k6/latest/) | Scriptable load tests with thresholds |
| [BenchmarkDotNet](https://benchmarkdotnet.org/) | Micro-level performance requirements |
| [.NET globalization and localization](https://learn.microsoft.com/en-us/dotnet/core/extensions/globalization-and-localization) | Internationalization requirements in .NET |
| [FinOps Foundation — What is FinOps?](https://www.finops.org/introduction/what-is-finops/) | Cost as an engineering discipline |
| [Azure Cost Management overview](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management) | Measuring unit cost on Azure |
| [Azure pricing calculator](https://azure.microsoft.com/en-us/pricing/calculator/) | Quick cost estimates for options |

### Architect-round material

| Resource | What it covers |
|---|---|
| [Martin Fowler — Who Needs an Architect? (PDF)](https://martinfowler.com/ieeeSoftware/whoNeedsArchitect.pdf) | Architecture as the decisions that are hard to change |
| [Martin Fowler — Conway's Law](https://martinfowler.com/bliki/ConwaysLaw.html) | Team structure as a requirement on architecture |
| [Team Topologies — Key concepts](https://teamtopologies.com/key-concepts) | Cognitive load and team boundaries |
| [Hyrum's Law](https://www.hyrumslaw.com/) | Every observable behavior gets depended on |
| [Parnas — On the Criteria To Be Used in Decomposing Systems into Modules (PDF)](https://www.win.tue.nl/~wstomv/edu/2ip30/references/criteria_for_modularization.pdf) | Modularize around what will change |
| [Gregor Hohpe — The Architect Elevator](https://architectelevator.com/) | Connecting business drivers to technical decisions |
| [Architecture Decision Records](https://adr.github.io/) | Linking decisions to requirements (Module 31) |
| [The C4 model](https://c4model.com/) | Communicating structure to stakeholders |

### Engineering case studies where requirements drove the design

| Resource | What it covers |
|---|---|
| [Discord — How Discord stores trillions of messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Chat storage at scale; hot partitions |
| [Slack Engineering — Real-time Messaging](https://slack.engineering/real-time-messaging/) | Connection management and fan-out for chat |
| [Stripe — Online migrations at scale](https://stripe.com/blog/online-migrations) | Zero-downtime as a requirement during data migration |
| [Hello Interview — WhatsApp breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/whatsapp) | A full chat design to compare with Concept 33 |
| [Hello Interview — Bit.ly breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly) | The consumer URL shortener (Concept 34, column B) |

---

## Quick-recall sheet

| If you need to… | Remember… |
|---|---|
| State the purpose of step 1 | You're writing the rubric your design will be judged against |
| Label requirement types | Functional · quality attribute · constraint · assumption (+ ASRs are the ones that matter) |
| Explain why NFRs matter most | Same features + different qualities = different architecture |
| Make an NFR usable | OPEN: Operation, Percentile, Environment, Number |
| Know the formal version | Quality-attribute scenario: source, stimulus, environment, artifact, response, response measure |
| Decide ask vs assume | Ask only if the answer forks the design *and* you can't predict it; attach the fork |
| Describe scale | Peak not average, read/write ratio, burstiness, skew, fan-out, growth, geography |
| Reason about latency | Percentiles; where measured; at what throughput; 1 − 0.99ⁿ for fan-out |
| Convert nines | 99.9 ≈ 43 min/month · 99.99 ≈ 4.3 min/month · 99.999 ≈ 26 s/month |
| Combine availabilities | Serial: multiply availabilities · Parallel: multiply unavailabilities · correlation breaks both |
| Separate durability from availability | Can I use it now vs does acknowledged data still exist later |
| Define RPO/RTO | Data you may lose (time) / time you're down; both are business numbers |
| Elicit consistency | Per operation, in business words → model; "how stale, in seconds?" |
| Handle duplicates | "What if it's processed twice or out of order?" |
| Cover security | Principals, authorization model, tenancy and isolation, abuse, audit |
| Cover compliance | One question; erasure, residency, PCI scope, DORA/NIS2/AI Act when relevant |
| Cover operations | Deploy frequency, zero-downtime, SLIs, who gets paged |
| Cover cost | Budget and unit cost; cost of each nine |
| Prioritize | Per-flow tiers; utility tree (Importance, Difficulty); (H,H) = hard part |
| Check completeness | Every NFR → a mechanism on the diagram → a fitness function |
| Deliver in 5 minutes | Restate → 3 FRs + out → batched forks → per-flow NFRs + assumptions → hard part → check-in |
| Architect round | Stakeholders, "why now?", constraints, scenarios, utility tree, hand business decisions back |
| Turn "never down" into a decision | €/hour × downtime per year at each level vs cost of achieving it |

---

## Progress

Module 4 complete. It deepens step 1 of Module 3's framework and connects forward:

- **Module 5** takes the numbers you elicited or assumed here and turns them into estimates that decide something — QPS, storage, bandwidth, connection counts — with the latency figures to know cold.
- **Phase 3 (Modules 6–13)** supplies the mechanisms that each NFR produces: scalability and admission control (6), consistency models (7), replication and partitioning for skew and durability (8), coordination for global invariants (9), caching with stated staleness (10), messaging with at-least-once semantics (11), transactions and invariants (12), and reliability patterns for availability and recovery targets (13).
- **Module 28** (observability) builds out SLIs, SLOs and burn-rate alerting; **Module 29** (security architecture) develops identity, tenancy and threat modeling; **Modules 30–33** turn the architect variant here into design documents, ADRs, brownfield strategy and cost conversations.

Use the self-scoring checklist above on the first five minutes of every mock from here on — it's the part of the interview where small, rehearsable habits produce the largest change in how senior you sound.
